# Use __builtin_prefetch to hide memory latency in hot loops

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html>  

---

## Topic Overview

This topic focuses on practical prefetch patterns in hot loops - complementing #542 (basics + excessive prefetch) and #624 (C++26 std::prefetch + locality levels).

The basic hot-loop prefetch pattern is straightforward: in each iteration, issue a prefetch for the address you will need D iterations from now. By the time the loop reaches that iteration, the data has already been fetched from DRAM into cache.

```cpp
Hot loop prefetch pattern:
  for (int i = 0; i < N; ++i) {
      __builtin_prefetch(&data[i + D]);  // D = prefetch distance
      process(data[i]);                  // uses data prefetched D iters ago
  }

  Optimal D = memory_latency / iteration_time
  DRAM latency: ~100ns, iteration: ~5ns -> D ~= 20
  L3 latency:   ~10ns, iteration:  ~5ns -> D ~= 2
```

The reason this only pays off for irregular access is the hardware prefetcher. Modern CPUs track sequential and strided access patterns automatically and issue prefetches without any help from you. Software prefetching only adds value for patterns the hardware cannot predict - pointer chasing, random lookups, and indirect indexing.

| Access pattern | HW prefetcher | SW prefetch needed? |
| --- | --- | --- |
| Sequential (stride=1) | Excellent | No (HW handles it) |
| Strided (constant stride) | Good | Usually no |
| Pointer-chasing (linked list) | Fails | Yes (big win) |
| Random (hash table) | Fails | Yes (moderate win) |
| Indirect (a[b[i]]) | Fails | Yes (gather prefetch) |

---

## Self-Assessment

### Q1: Prefetch next cache line ahead of loop iteration

The indirect array access pattern `a[indices[i]]` is a common source of cache misses in practice - think database lookups, graph traversal, or any scatter/gather operation. The hardware can predict the sequential access through `indices[]`, but cannot predict where `a[indices[i]]` will land.

The fix: each iteration, look D steps ahead in the indices array and prefetch the corresponding element of `a`:

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// Indirect array access: a[indices[i]]
// Hardware prefetcher can predict indices[] (sequential)
// but CANNOT predict a[indices[i]] (random)

void sum_indirect_no_pf(const int* a, const int* indices, int n, long long& out) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += a[indices[i]];  // random access into a[] -> cache miss
    }
    out = sum;
}

void sum_indirect_pf(const int* a, const int* indices, int n, long long& out) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        // Prefetch the element we'll need in D iterations
        if (i + 16 < n) {
            __builtin_prefetch(&a[indices[i + 16]], 0, 3);
            // Why 16? ~100ns DRAM latency / ~6ns per iteration = ~16
        }
        sum += a[indices[i]];  // prefetched 16 iterations ago!
    }
    out = sum;
}

int main() {
    constexpr int TABLE_SIZE = 10'000'000;  // 40 MB (doesn't fit in L3)
    constexpr int LOOKUPS = 5'000'000;

    std::vector<int> table(TABLE_SIZE, 42);
    std::vector<int> indices(LOOKUPS);
    std::mt19937 rng(42);
    for (auto& idx : indices) idx = rng() % TABLE_SIZE;

    auto bench = [&](auto fn, const char* label) {
        long long result = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 5; ++r) fn(table.data(), indices.data(), LOOKUPS, result);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() / 5 << " ms\n";
    };

    bench(sum_indirect_no_pf, "No prefetch ");
    bench(sum_indirect_pf,    "With prefetch");
    // No prefetch:  ~120ms (constant DRAM misses)
    // With prefetch: ~80ms  (33% faster, latency partially hidden)
}
```

The improvement is "partial" because latency hiding is never perfect. Some cache misses still complete late when the computation catches up with the prefetch pipeline. Experiment with the distance value - on some machines and some data sizes, D=8 beats D=16, and on others the opposite is true.

### Q2: Prefetch 8-16 iterations ahead for pointer-chasing

Pointer-chasing in a linked list is fundamentally hard to prefetch because each node's next pointer is only readable after the current node has been fetched. Single-step prefetch (look 1 node ahead) only hides L3 latency (~15ns). To hide full DRAM latency (~100ns), you need to walk 8 or more nodes ahead - which means paying the cost of reading those intermediate next pointers early.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>
#include <numeric>

struct Node {
    int value;
    int next; // array index
    int pad[14]; // 64 bytes per node (1 cache line)
};

// Single prefetch: 1 node ahead (hides ~15ns = L3 latency)
int traverse_pf1(const std::vector<Node>& nodes, int head) {
    int sum = 0, idx = head;
    while (idx != -1) {
        int next = nodes[idx].next;
        if (next != -1)
            __builtin_prefetch(&nodes[next], 0, 3);
        sum += nodes[idx].value;
        idx = next;
    }
    return sum;
}

// Deep prefetch: 8 nodes ahead (hides ~100ns = DRAM latency)
// Requires "pre-walking" the list to get future pointers
int traverse_pf8(const std::vector<Node>& nodes, int head) {
    int sum = 0;
    // Maintain a sliding window of future indices
    constexpr int D = 8;
    int window[D + 1];
    int idx = head;
    // Fill initial window
    for (int i = 0; i <= D && idx != -1; ++i) {
        window[i] = idx;
        idx = nodes[idx].next;
    }
    int front = 0, back = std::min(D, static_cast<int>(nodes.size()));

    // Process with deep prefetch
    idx = head;
    int lookahead = idx;
    for (int i = 0; i < D && lookahead != -1; ++i)
        lookahead = nodes[lookahead].next;

    while (idx != -1) {
        // Prefetch D nodes ahead
        if (lookahead != -1) {
            __builtin_prefetch(&nodes[lookahead], 0, 3);
            lookahead = nodes[lookahead].next;
        }
        sum += nodes[idx].value;
        idx = nodes[idx].next;
    }
    return sum;
}

int main() {
    constexpr int N = 1'000'000;
    std::vector<Node> nodes(N);
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
        for (int r = 0; r < 10; ++r) s = fn(nodes, order[0]);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
        std::cout << label << ": " << us.count() / 10 << " us\n";
    };

    bench(traverse_pf1, "Prefetch 1 ahead");
    bench(traverse_pf8, "Prefetch 8 ahead");
    // Prefetch 1: ~55ms (L3 latency hidden)
    // Prefetch 8: ~40ms (DRAM latency partially hidden)
    // Diminishing returns beyond 8-16 (prefetch queue limited)
}
```

The deep-prefetch version is more complex because it maintains a `lookahead` pointer that is always D nodes ahead of the working `idx`. The prefetch queue in typical CPUs holds about 10-16 entries, so going beyond D=16 yields diminishing returns.

### Q3: Temporal vs non-temporal prefetch

The locality parameter in `__builtin_prefetch` controls which cache levels receive the data. This distinction matters when your working set is large: pulling data into L1+L2+L3 with every prefetch can evict other useful data from those caches. For single-pass workloads, it is better to use a non-temporal prefetch that brings data only into L1 and lets it evict quickly.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// TEMPORAL (locality=3): put in L1+L2+L3
//   Use when data will be accessed MULTIPLE TIMES soon.
//   Example: hash table bucket (read key, compare, read value)
void sum_temporal(const int* data, const int* indices, int n) {
    volatile long long sum = 0;
    for (int i = 0; i < n; ++i) {
        if (i + 16 < n)
            __builtin_prefetch(&data[indices[i + 16]], 0, 3);  // L1+L2+L3
        sum += data[indices[i]];
    }
}

// NON-TEMPORAL (locality=0): put in L1 only, evict quickly
//   Use when data is accessed ONCE and shouldn't pollute higher caches.
//   Example: scanning log entries (process once, never again)
void sum_nontemporal(const int* data, const int* indices, int n) {
    volatile long long sum = 0;
    for (int i = 0; i < n; ++i) {
        if (i + 16 < n)
            __builtin_prefetch(&data[indices[i + 16]], 0, 0);  // L1 only
        sum += data[indices[i]];
    }
}

int main() {
    constexpr int N = 5'000'000;
    constexpr int TABLE = 10'000'000;
    std::vector<int> data(TABLE, 1);
    std::vector<int> indices(N);

    // Random indices (simulates hash table lookups)
    for (int i = 0; i < N; ++i) indices[i] = (i * 7919) % TABLE;

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 5; ++r) fn(data.data(), indices.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() / 5 << " ms\n";
    };

    bench(sum_temporal,    "Temporal (L1+L2+L3)");
    bench(sum_nontemporal, "Non-temporal (L1)   ");
    // For single-pass scans: non-temporal is better (doesn't pollute L2/L3)
    // For repeated access: temporal is better (stays in cache)
    // Difference: 5-15% depending on working set vs cache size

    std::cout << "\nWhen to use each:\n";
    std::cout << "  Temporal (3): hash tables, B-trees, reused data\n";
    std::cout << "  Non-temporal (0): log scans, streaming, one-pass\n";
    std::cout << "  L3 only (1): large working sets with some reuse\n";
    std::cout << "  L2+L3 (2): moderate working sets\n";
}
```

The 5-15% difference between temporal and non-temporal depends on the ratio of your working set to the available cache size. If the data fits in L3 anyway, it doesn't matter. If you are scanning multiple gigabytes and you have other hot data competing for cache space, the non-temporal hint is worth using.

---

## Notes

- Complementary to #542 (basics/excessive prefetch) and #624 (C++26 std::prefetch).
- Optimal prefetch distance D = memory_latency / iteration_time (typically 8-32).
- Diminishing returns: CPU has limited prefetch queue (~10-16 entries).
- For indirect access `a[b[i]]`: prefetch `a[b[i+D]]` not `b[i+D]` (b[] is sequential, a[] is random).
- Always benchmark: prefetch can hurt if distance is wrong or data is already cached.
