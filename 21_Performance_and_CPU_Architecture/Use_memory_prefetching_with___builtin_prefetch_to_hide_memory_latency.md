# Use memory prefetching with __builtin_prefetch to hide memory latency

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html>  

---

## Topic Overview

When the CPU needs data that isn't in cache, it has to go all the way to DRAM to get it - and that round-trip takes about 100 nanoseconds. On a 3 GHz processor, that is roughly 300 cycles of stalling while the CPU sits idle waiting. Software prefetching is the technique of *asking for the data early*, before you actually need it, so the memory request is in flight while you're still computing on previous data.

The diagram below shows the key idea - with prefetch, the two operations overlap instead of stacking:

```cpp
Without prefetch:        With prefetch:
  load &data[i]            prefetch &data[i+D]  (D = prefetch distance)
  STALL 100ns              load &data[i]         (compute while prefetch in flight)
  use data[i]              use data[i]           (data already in L1/L2)
  load &data[i+1]          prefetch &data[i+D+1]
  STALL 100ns              load &data[i+1]       (hit!)
  use data[i+1]            use data[i+1]
```

The three parameters of `__builtin_prefetch` let you tune exactly how the prefetch behaves:

| Parameter | Value | Meaning |
| --- | --- | --- |
| `address` | pointer | Address to prefetch (cache-line-aligned recommended) |
| `rw` | 0 | Prefetch for read |
| `rw` | 1 | Prefetch for write (exclusive ownership) |
| `locality` | 0 | Non-temporal (use once, don't keep in cache) |
| `locality` | 1 | Low temporal locality (L3 only) |
| `locality` | 2 | Moderate temporal locality (L2 + L3) |
| `locality` | 3 | High temporal locality (L1 + L2 + L3) |

---

## Self-Assessment

### Q1: Prefetch ahead in a pointer-chasing loop

Pointer-chasing is the worst case for the hardware prefetcher because each memory address is hidden inside the data you just loaded - there is no pattern to predict. This is exactly where software prefetching shines: you read the next address a step early and issue the prefetch yourself.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>
#include <numeric>

// Pointer-chasing: each access depends on the previous result.
// Hardware prefetcher CANNOT predict the pattern!
struct Node {
    int value;
    int next;  // index into array (simulates pointer chase)
};

int sum_no_prefetch(const std::vector<Node>& nodes) {
    int sum = 0, idx = 0;
    while (idx != -1) {
        sum += nodes[idx].value;
        idx = nodes[idx].next;  // random next -> cache miss!
    }
    return sum;
}

int sum_with_prefetch(const std::vector<Node>& nodes) {
    int sum = 0, idx = 0;
    while (idx != -1) {
        int next = nodes[idx].next;
        if (next != -1) {
            // Prefetch the NEXT node while we process current
            __builtin_prefetch(&nodes[next], 0, 1);
            // Parameters: address, read(0), L3 locality(1)
        }
        sum += nodes[idx].value;
        idx = next;
    }
    return sum;
}

int main() {
    constexpr int N = 1'000'000;
    std::vector<Node> nodes(N);

    // Create random linked list (worst case for hardware prefetcher)
    std::vector<int> order(N);
    std::iota(order.begin(), order.end(), 0);
    std::mt19937 rng(42);
    std::shuffle(order.begin(), order.end(), rng);
    for (int i = 0; i < N; ++i) {
        nodes[order[i]].value = i;
        nodes[order[i]].next = (i + 1 < N) ? order[i + 1] : -1;
    }

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile int s = 0;
        for (int r = 0; r < 10; ++r) s = fn(nodes);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
        std::cout << label << ": " << us.count() / 10 << " us\n";
    };

    bench(sum_no_prefetch,   "No prefetch ");
    bench(sum_with_prefetch, "With prefetch");
    // N=1M random: No prefetch ~80ms, With prefetch ~50ms (1.6x speedup)
}
```

The 1.6x speedup here comes from overlapping the memory latency with the work of processing the current node. The hardware prefetcher simply can't help here because it has no idea what `nodes[idx].next` will be until after the load completes.

### Q2: The three parameters of `__builtin_prefetch`

Understanding the locality hint matters because choosing the wrong one can actually hurt performance. Using `locality=3` for streaming data that you'll never reuse wastes L1 space; using `locality=0` for data you're about to read repeatedly causes repeated cache misses.

```cpp
#include <iostream>
#include <vector>

// __builtin_prefetch(addr, rw, locality)
//
// addr:     any pointer expression (prefetches the cache line containing it)
// rw:       0 = read (prefetchnta/prefetch0/1/2)
//           1 = write (prefetchw - brings line in Modified state)
// locality: 0 = non-temporal (prefetchnta: L1 only, evicts quickly)
//           1 = low locality  (prefetch2: L3 only)
//           2 = moderate       (prefetch1: L2 + L3)
//           3 = high locality  (prefetch0: L1 + L2 + L3)

void example_read_sequential(const float* data, int n) {
    // Sequential read with moderate reuse
    for (int i = 0; i < n; ++i) {
        __builtin_prefetch(data + i + 64, 0, 3);  // read, keep in L1
        // 64 elements * 4 bytes = 256 bytes = 4 cache lines ahead
        volatile float x = data[i] * data[i];
    }
}

void example_write_streaming(float* out, int n) {
    // Write-only streaming (no reuse)
    for (int i = 0; i < n; ++i) {
        __builtin_prefetch(out + i + 128, 1, 0);  // write, non-temporal
        out[i] = static_cast<float>(i);
    }
}

void example_read_random(const int* table, const int* indices, int n) {
    // Random access with no reuse (hash table lookup)
    for (int i = 0; i < n; ++i) {
        __builtin_prefetch(&table[indices[i + 4]], 0, 0);  // read, non-temporal
        volatile int v = table[indices[i]];
    }
}

int main() {
    // The x86 instructions generated:
    std::cout << "Locality mapping to x86 prefetch instructions:\n";
    std::cout << "  locality=0, rw=0 -> prefetchnta  (L1 only, non-temporal)\n";
    std::cout << "  locality=1, rw=0 -> prefetcht2   (L3)\n";
    std::cout << "  locality=2, rw=0 -> prefetcht1   (L2+L3)\n";
    std::cout << "  locality=3, rw=0 -> prefetcht0   (L1+L2+L3)\n";
    std::cout << "  locality=*, rw=1 -> prefetchw    (exclusive ownership)\n";
}
```

The `rw=1` (write prefetch) is worth a special mention. When the CPU writes to a cache line it doesn't own yet, it first has to request exclusive ownership - that's a bus transaction. By prefetching for write, you initiate that ownership request early, so by the time you actually write, the line is already yours.

### Q3: Excessive prefetching hurting performance

The reason this trips people up is that prefetching too aggressively can actually make things *worse*. If you prefetch data that is too far ahead, you pollute the cache with data that won't be used for a while, evicting the data you need *right now*. The sweet spot is prefetching just far enough ahead to hide the latency without causing evictions.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// TOO MUCH PREFETCHING: evicts useful data from L1
void bad_prefetch(const float* a, const float* b, float* c, int n) {
    for (int i = 0; i < n; ++i) {
        // Prefetching TOO FAR ahead (4096 elements = 16KB = half of L1!)
        __builtin_prefetch(a + i + 4096, 0, 3);
        __builtin_prefetch(b + i + 4096, 0, 3);
        __builtin_prefetch(c + i + 4096, 1, 3);
        // Three prefetches per iteration, 16KB ahead each.
        // This fills L1 with prefetched data, evicting 'a[i]' and 'b[i]'
        // that we're about to use RIGHT NOW!
        c[i] = a[i] + b[i];  // MISS! our data was evicted by prefetches
    }
    // Result: SLOWER than no prefetching at all!
}

// CORRECT: moderate prefetch distance, matching memory latency
void good_prefetch(const float* a, const float* b, float* c, int n) {
    // Prefetch distance = memory_latency / loop_time
    // ~100ns latency / ~1ns per iteration = ~100 elements
    constexpr int D = 64; // 64 elements = 256 bytes = 4 cache lines
    for (int i = 0; i < n; ++i) {
        if (i + D < n) {
            __builtin_prefetch(a + i + D, 0, 0);  // non-temporal (streaming)
            __builtin_prefetch(b + i + D, 0, 0);
        }
        c[i] = a[i] + b[i];
    }
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N, 1.0f), b(N, 2.0f), c(N);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 50; ++r) fn(a.data(), b.data(), c.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(bad_prefetch,  "Bad prefetch  ");
    bench(good_prefetch, "Good prefetch ");
    // bad_prefetch:  ~600ms (cache thrashing)
    // good_prefetch: ~400ms (smooth streaming)
    // no prefetch:   ~450ms (HW prefetcher handles sequential OK)

    // Rules of thumb:
    std::cout << "\nPrefetch guidelines:\n";
    std::cout << "  Distance: 100-500 bytes ahead (depends on latency)\n";
    std::cout << "  Don't prefetch sequential access (HW does it)\n";
    std::cout << "  DO prefetch random/pointer-chasing access\n";
    std::cout << "  Use locality=0 for streaming, 3 for reused data\n";
    std::cout << "  Limit to 1-2 prefetches per loop iteration\n";
}
```

Notice that the "bad" version is slower than no prefetching at all. The formula in the comment - `latency / loop_time` - is a useful starting point for picking the prefetch distance. From there, measure and tune.

---

## Notes

- `__builtin_prefetch` is a hint; CPU may ignore it (no-op if target already in cache).
- Best use case: random/pointer-chasing access where HW prefetcher fails.
- Sequential access rarely benefits (HW prefetcher handles it well).
- Optimal distance = memory_latency x loop_throughput (typically 4-8 cache lines).
- C++26 proposes `std::prefetch` as a portable alternative.
