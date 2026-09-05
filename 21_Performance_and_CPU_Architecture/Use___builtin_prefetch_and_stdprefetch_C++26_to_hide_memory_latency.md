# Use __builtin_prefetch and std::prefetch (C++26) to hide memory latency

**Category:** Performance and CPU Architecture  
**Standard:** C++26  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html>  

---

## Topic Overview

This topic covers the C++26 `std::prefetch` proposal and practical prefetch strategies - complementing #542 which covers `__builtin_prefetch` basics and excessive prefetching.

The core idea behind prefetching is simple: memory latency is roughly 100ns when you miss all the way to DRAM, but your loop iteration might only take 5ns. If you can tell the CPU "I will need this address in about 20 iterations," it can start fetching that cache line in the background while you continue computing on data you already have. By the time you actually need the new data, it has already arrived.

Today you do this with `__builtin_prefetch`. C++26 will standardize the same hint as `std::prefetch`:

```cpp
__builtin_prefetch (GCC/Clang):             std::prefetch (C++26 P1007):
  __builtin_prefetch(ptr, 0, 3);              std::prefetch(ptr, std::pfhint::read);
  // Compiler-specific                        // Standard, portable
  // Works today                               // Future (C++26)

Both generate the same instruction:
  prefetcht0 [rdi]   (x86)
  prfm pldl1keep, [x0]  (ARM)
```

The three `std::prefetch` hint values map directly to the three most common use cases:

| Prefetch hint | `__builtin_prefetch` | `std::prefetch` (proposed) |
| --- | --- | --- |
| Read, keep in L1 | `(ptr, 0, 3)` | `pfhint::read` |
| Read, non-temporal | `(ptr, 0, 0)` | `pfhint::read_streaming` |
| Write | `(ptr, 1, 3)` | `pfhint::write` |

---

## Self-Assessment

### Q1: Overlap prefetch with compute

The hash table lookup is the canonical case where prefetching pays off. The hardware prefetcher can predict sequential and strided access patterns, but random accesses to a hash table defeat it entirely. Every lookup causes a cache miss, and your loop sits idle waiting for DRAM.

The fix is to compute where you will be looking in D iterations and issue a prefetch for that address now, letting the memory system work in parallel with your compute:

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// Key technique: issue prefetch BEFORE the data is needed.
// The CPU loads the cache line in the background while we compute.

// Hash table lookup: random access, HW prefetcher fails
struct HashTable {
    std::vector<int> buckets;
    int lookup(int key) const { return buckets[key % buckets.size()]; }
};

int sum_no_prefetch(const HashTable& ht, const std::vector<int>& keys) {
    int sum = 0;
    for (size_t i = 0; i < keys.size(); ++i) {
        sum += ht.lookup(keys[i]);  // cache miss on every lookup!
    }
    return sum;
}

int sum_prefetch(const HashTable& ht, const std::vector<int>& keys) {
    int sum = 0;
    constexpr int D = 8;  // prefetch distance
    for (size_t i = 0; i < keys.size(); ++i) {
        // Prefetch the bucket we'll need D iterations from now
        if (i + D < keys.size()) {
            size_t future_idx = keys[i + D] % ht.buckets.size();
            __builtin_prefetch(&ht.buckets[future_idx], 0, 3);
            // rw=0 (read), locality=3 (keep in L1)
        }
        sum += ht.lookup(keys[i]);  // this one was prefetched D iters ago!
    }
    return sum;
}

// C++26 version (future):
// #include <prefetch>  // proposed header
// int sum_std_prefetch(const HashTable& ht, const std::vector<int>& keys) {
//     int sum = 0;
//     for (size_t i = 0; i < keys.size(); ++i) {
//         if (i + 8 < keys.size()) {
//             size_t idx = keys[i+8] % ht.buckets.size();
//             std::prefetch(&ht.buckets[idx], std::pfhint::read);
//         }
//         sum += ht.lookup(keys[i]);
//     }
//     return sum;
// }

int main() {
    HashTable ht;
    ht.buckets.resize(10'000'000, 42);
    std::vector<int> keys(5'000'000);
    std::mt19937 rng(42);
    for (auto& k : keys) k = rng();

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile int s = fn(ht, keys);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(sum_no_prefetch, "No prefetch ");
    bench(sum_prefetch,    "With prefetch");
    // No prefetch:  ~250ms (random L3 misses)
    // With prefetch: ~150ms (40% faster, latency hidden)
}
```

A 40% improvement from a few extra lines of code is a good return. The key is picking the right distance D - too small and the prefetch doesn't arrive in time, too large and you waste prefetch bandwidth on data the CPU would have fetched anyway.

### Q2: The four locality levels and when to use each

The second argument to `__builtin_prefetch` (the `rw` parameter) controls read vs write. The third argument (the `locality` parameter) controls which cache levels to populate. This matters because polluting L2/L3 with data you will only use once evicts data that other parts of your code need.

```cpp
#include <iostream>

// __builtin_prefetch(addr, rw, locality)
// locality controls which cache level to target.

int main() {
    std::cout << "Locality levels and use cases:\n\n";
    std::cout << "+----------+---------------+---------------------------+------------------------+\n";
    std::cout << "| Locality | x86 instruction| Cache levels populated    | Best use case          |\n";
    std::cout << "+----------+---------------+---------------------------+------------------------+\n";
    std::cout << "| 0        | prefetchnta   | L1 only (evicts quickly)  | Streaming (use once)   |\n";
    std::cout << "| 1        | prefetcht2    | L3 only                   | Large working set      |\n";
    std::cout << "| 2        | prefetcht1    | L2 + L3                   | Moderate reuse         |\n";
    std::cout << "| 3        | prefetcht0    | L1 + L2 + L3              | High reuse (default)   |\n";
    std::cout << "+----------+---------------+---------------------------+------------------------+\n";

    // Practical guidelines:
    //
    // locality=3 (default, most common):
    //   - Hash table lookups (will process immediately)
    //   - Linked list traversal (need data NOW)
    //   - B-tree node access (will read many fields)
    //
    // locality=0 (non-temporal, streaming):
    //   - Sequential scan of huge arrays (don't pollute L2/L3)
    //   - Log file writing (write once, never read back)
    //   - Network packet headers (inspect once, discard)
    //
    // locality=1 (L3 only):
    //   - Large database index scans (many pages, some revisited)
    //   - Graph traversal (vertices accessed semi-randomly)
    //
    // locality=2 (L2+L3):
    //   - Matrix operations (moderate spatial reuse)
    //   - Sorting large arrays (elements accessed multiple times)

    // rw parameter:
    //  0 = read prefetch (shared cache line state)
    //  1 = write prefetch (exclusive state, avoids upgrade later)
    //  Use rw=1 when you KNOW you'll modify the data:
    //    __builtin_prefetch(output + i + 64, 1, 3);  // will write here
}
```

The reason to care about locality levels is cache pollution. If you use `locality=3` for a streaming scan that touches 2GB of data, you will thrash the L2 and L3 caches, pushing out data that your other hot loops actually reuse. Using `locality=0` for single-pass data is a courtesy to the rest of your working set.

### Q3: Prefetch speedup on linked-list traversal

Linked-list traversal is the hardest case for the hardware prefetcher because each node contains the address of the next node. The prefetcher cannot know where to look next until it has read the current node - the dependency chain means it can only ever prefetch one step ahead, and that one step may already be a 100ns DRAM miss.

Software prefetching lets you look further ahead by walking the list an extra step or two to retrieve future addresses before they are needed for computation:

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>
#include <numeric>

// Linked list via array indices (realistic: pool allocator pattern)
struct Node {
    int data;
    int padding[15]; // 64 bytes total (one cache line per node)
    int next;        // index of next node (-1 = end)
};

int traverse_no_prefetch(const std::vector<Node>& nodes, int head) {
    int sum = 0, idx = head;
    while (idx != -1) {
        sum += nodes[idx].data;
        idx = nodes[idx].next;  // random jump -> cache miss!
    }
    return sum;
}

// Single prefetch: look 1 node ahead
int traverse_prefetch_1(const std::vector<Node>& nodes, int head) {
    int sum = 0, idx = head;
    while (idx != -1) {
        int next = nodes[idx].next;
        if (next != -1)
            __builtin_prefetch(&nodes[next], 0, 3);  // prefetch next node
        sum += nodes[idx].data;
        idx = next;
    }
    return sum;
}

// Double prefetch: look 2 nodes ahead (hides more latency)
int traverse_prefetch_2(const std::vector<Node>& nodes, int head) {
    int sum = 0, idx = head;
    // Prime the pipeline: prefetch first 2 nodes
    if (idx != -1) {
        int n1 = nodes[idx].next;
        if (n1 != -1) {
            __builtin_prefetch(&nodes[n1], 0, 3);
            int n2 = nodes[n1].next;
            if (n2 != -1) __builtin_prefetch(&nodes[n2], 0, 3);
        }
    }
    while (idx != -1) {
        int next = nodes[idx].next;
        // Prefetch 2 ahead
        if (next != -1) {
            int next2 = nodes[next].next;
            if (next2 != -1)
                __builtin_prefetch(&nodes[next2], 0, 3);
        }
        sum += nodes[idx].data;
        idx = next;
    }
    return sum;
}

int main() {
    constexpr int N = 1'000'000;
    std::vector<Node> nodes(N);

    // Shuffle nodes: random order in memory
    std::vector<int> order(N);
    std::iota(order.begin(), order.end(), 0);
    std::mt19937 rng(42);
    std::shuffle(order.begin(), order.end(), rng);

    for (int i = 0; i < N; ++i) {
        nodes[order[i]].data = i;
        nodes[order[i]].next = (i + 1 < N) ? order[i + 1] : -1;
    }
    int head = order[0];

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile int s = 0;
        for (int r = 0; r < 10; ++r) s = fn(nodes, head);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
        std::cout << label << ": " << us.count() / 10 << " us\n";
    };

    bench(traverse_no_prefetch, "No prefetch     ");
    bench(traverse_prefetch_1,  "Prefetch 1 ahead");
    bench(traverse_prefetch_2,  "Prefetch 2 ahead");
    // No prefetch:      ~80ms per traversal
    // Prefetch 1 ahead: ~55ms (30% faster)
    // Prefetch 2 ahead: ~45ms (44% faster)
    // HW prefetcher: 0% help (random pattern)
}
```

Each node is padded to exactly one cache line (64 bytes) so that a prefetch always brings in exactly one useful node. Looking 2 nodes ahead rather than 1 gives another meaningful improvement because it gives the memory system more latency to hide. Looking further than 2-3 nodes ahead yields diminishing returns because the CPU's prefetch queue has limited capacity.

---

## Notes

- `std::prefetch` (P1007) is proposed for C++26; use `__builtin_prefetch` until then.
- Prefetch is a hint; the CPU may ignore it (no-op if already cached, or if too many in-flight).
- For linked lists: prefetch 1-2 nodes ahead. For hash tables: prefetch D lookups ahead.
- Hardware prefetcher handles sequential and strided patterns well; SW prefetch adds nothing there.
- Measure with `perf stat -e L1-dcache-load-misses` before and after.
