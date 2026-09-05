# Understand unordered_map load factor, rehashing, and bucket management

**Category:** Standard Library Containers  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/container/unordered_map>  

---

## Topic Overview

`std::unordered_map` uses a hash table internally. Three concepts govern its performance: **load factor**, **rehashing**, and **bucket management**. Understanding these lets you avoid hidden O(n) operations and tune for throughput or memory.

The reason this matters in practice: if you insert a million elements into an `unordered_map` without calling `reserve()` first, the map will rehash roughly 20 times, each time copying every existing element into a larger bucket array. You pay for all of that silently. One call to `reserve()` eliminates every one of those rehashes.

### Internal Structure

A picture of how separate-chaining works helps make the load factor formula concrete:

```cpp
Hash Table (separate chaining):
                           bucket_count() = 8
  Index:  [0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]
           |    |         |              |
           v    v         v              v
          K:A  K:B       K:D            K:E   <- linked lists (chains)
           |              |
           v              v
          K:C            K:F   <- collision chain

  size()        = 6  (total elements)
  bucket_count  = 8  (number of buckets)
  load_factor() = 6/8 = 0.75
```

Each bucket is just the head of a linked list. When two keys hash to the same bucket index, they form a chain. Short chains mean fast lookups; long chains mean slow ones.

### Key Definitions

| Term | Formula / Meaning | Default |
| --- | --- | --- |
| `load_factor()` | `size() / bucket_count()` | - |
| `max_load_factor()` | Threshold that triggers rehash | **1.0** |
| `bucket_count()` | Number of buckets (slots) | Implementation-defined |
| `bucket_size(n)` | Elements in bucket `n` (chain length) | - |
| `bucket(key)` | Which bucket `key` hashes to | - |
| `reserve(n)` | Pre-allocates for `n` elements without rehash | - |
| `rehash(n)` | Sets `bucket_count >= n` and redistributes | - |

### When Rehashing Happens

Rehashing is the expensive event you want to avoid. Here is what the container does when it triggers:

```cpp
After insert, if load_factor() > max_load_factor():

  1. New bucket_count is chosen (typically ~2x current)
  2. Every existing element is re-hashed into new buckets
  3. All iterators are invalidated
  4. This single insert becomes O(n) instead of O(1)
```

That O(n) cost on step 4 is amortized across many inserts (the table doubles each time, so it happens logarithmically often), but it still causes a latency spike - which matters in real-time or low-latency code.

### The Cost of Rehashing

| Elements | Bucket count before | After rehash | Elements moved |
| --- | --- | --- | --- |
| 1000 | 1000 | ~2000 | all 1000 |
| 10,000 | 10,000 | ~20,000 | all 10,000 |
| 1,000,000 | 1,000,000 | ~2,000,000 | all 1,000,000 |

Each rehash moves every single element. For a million-element map, that is a million hash computations and pointer rewirings in a single insert call.

### Core API

Here is the basic API for querying and controlling the bucket structure. Notice that `reserve()` is expressed in terms of *elements* while `rehash()` is expressed in terms of *buckets* - `reserve()` is almost always the right choice because you think in elements, not buckets:

```cpp
#include <iostream>
#include <unordered_map>
#include <algorithm>
#include <string>

int main() {
    std::unordered_map<int, std::string> m;

    // Query current state
    std::cout << "bucket_count: " << m.bucket_count() << "\n";
    std::cout << "max_load_factor: " << m.max_load_factor() << "\n";
    std::cout << "load_factor: " << m.load_factor() << "\n";

    // Pre-allocate for 1000 elements
    m.reserve(1000);
    std::cout << "\nAfter reserve(1000):\n";
    std::cout << "  bucket_count: " << m.bucket_count() << "\n";
    // Output: bucket_count >= 1000 (enough for 1000 elements at max_load_factor=1.0)

    // Insert elements
    for (int i = 0; i < 1000; ++i)
        m[i] = "val_" + std::to_string(i);

    std::cout << "\nAfter 1000 inserts:\n";
    std::cout << "  size: " << m.size() << "\n";
    std::cout << "  bucket_count: " << m.bucket_count() << "\n";
    std::cout << "  load_factor: " << m.load_factor() << "\n";
    // No rehash occurred because we reserved!

    // Inspect individual buckets
    for (size_t i = 0; i < std::min(m.bucket_count(), size_t(5)); ++i)
        std::cout << "  bucket[" << i << "] size: " << m.bucket_size(i) << "\n";

    // Force a rehash
    m.rehash(5000);
    std::cout << "\nAfter rehash(5000):\n";
    std::cout << "  bucket_count: " << m.bucket_count() << "\n";
    std::cout << "  load_factor: " << m.load_factor() << "\n";

    return 0;
}
```

After the forced `rehash(5000)`, the load factor drops because the same 1000 elements are now spread across at least 5000 buckets. Fewer elements per bucket means shorter chains and faster lookups.

### reserve() vs rehash()

| Method | Argument meaning | Sets bucket_count to |
| --- | --- | --- |
| `reserve(n)` | Number of **elements** | `>= ceil(n / max_load_factor())` |
| `rehash(n)` | Number of **buckets** | `>= n` |

**Prefer `reserve()`** - it thinks in terms of elements, which is what you know.

---

## Self-Assessment

### Q1: Use reserve() on an unordered_map before bulk insertion to avoid rehashing

This is the most important practical lesson in this topic. The benchmark counts how many times the bucket array is reallocated, which is your direct measure of how many O(n) rehash events occurred:

```cpp
#include <iostream>
#include <unordered_map>
#include <chrono>
#include <string>

int main() {
    constexpr int N = 1'000'000;

    // === Without reserve: multiple rehashes during insertion ===
    {
        std::unordered_map<int, int> m;
        std::cout << "=== Without reserve() ===\n";
        std::cout << "Initial bucket_count: " << m.bucket_count() << "\n";

        auto start = std::chrono::steady_clock::now();
        size_t rehash_count = 0;
        size_t prev_buckets = m.bucket_count();

        for (int i = 0; i < N; ++i) {
            m[i] = i;
            if (m.bucket_count() != prev_buckets) {
                ++rehash_count;
                prev_buckets = m.bucket_count();
            }
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();

        std::cout << "Final bucket_count: " << m.bucket_count() << "\n";
        std::cout << "Rehashes detected: " << rehash_count << "\n";
        std::cout << "Time: " << ms << " ms\n";
        std::cout << "Load factor: " << m.load_factor() << "\n\n";
    }

    // === With reserve: zero rehashes ===
    {
        std::unordered_map<int, int> m;
        m.reserve(N);  // Pre-allocate for N elements
        std::cout << "=== With reserve(" << N << ") ===\n";
        std::cout << "Initial bucket_count: " << m.bucket_count() << "\n";

        auto start = std::chrono::steady_clock::now();
        size_t rehash_count = 0;
        size_t prev_buckets = m.bucket_count();

        for (int i = 0; i < N; ++i) {
            m[i] = i;
            if (m.bucket_count() != prev_buckets) {
                ++rehash_count;
                prev_buckets = m.bucket_count();
            }
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();

        std::cout << "Final bucket_count: " << m.bucket_count() << "\n";
        std::cout << "Rehashes detected: " << rehash_count << "\n";
        std::cout << "Time: " << ms << " ms\n";
        std::cout << "Load factor: " << m.load_factor() << "\n";
    }

    return 0;
}
// Typical output:
// === Without reserve() ===
// Initial bucket_count: 1
// Final bucket_count: 1056323
// Rehashes detected: ~20
// Time: ~350 ms
//
// === With reserve(1000000) ===
// Initial bucket_count: 1056323
// Rehashes detected: 0
// Time: ~250 ms (faster! no rehash overhead)
```

About 20 rehashes without `reserve()`, zero with it - and a measurable time saving. The amortized cost is still O(1) per insert either way, but the constant factor and peak latency are both better when you reserve upfront.

**How it works:**

- Without `reserve()`, the map starts with a small bucket count (often 1 or 8). As elements are inserted, `load_factor()` exceeds `max_load_factor()`, and the map rehashes - doubling buckets and moving ALL existing elements.
- With `reserve(N)`, enough buckets are allocated upfront: `bucket_count >= ceil(N / max_load_factor())`. No rehashing occurs during insertion.
- Each rehash is O(n) where n is the current size, so ~20 rehashes through exponential growth still gives amortized O(1) per insert, but the constant factor and memory fragmentation are worse.
- **Rule of thumb:** If you know (or can estimate) the final size, always `reserve()`.

### Q2: Show that max_load_factor(0.25f) reduces collisions at the cost of memory

Lowering the max load factor is a memory-for-speed trade. The analysis function here measures the actual chain lengths so you can see the collision reduction directly:

```cpp
#include <iostream>
#include <unordered_map>
#include <numeric>
#include <algorithm>

void analyze(const std::unordered_map<int, int>& m, const std::string& label) {
    std::cout << "=== " << label << " ===\n";
    std::cout << "  size:             " << m.size() << "\n";
    std::cout << "  bucket_count:     " << m.bucket_count() << "\n";
    std::cout << "  load_factor:      " << m.load_factor() << "\n";
    std::cout << "  max_load_factor:  " << m.max_load_factor() << "\n";

    // Count non-empty buckets and find max chain length
    size_t non_empty = 0;
    size_t max_chain = 0;
    size_t total_chain_length = 0;
    size_t collisions = 0;  // buckets with >1 element

    for (size_t i = 0; i < m.bucket_count(); ++i) {
        size_t bs = m.bucket_size(i);
        if (bs > 0) {
            ++non_empty;
            total_chain_length += bs;
            max_chain = std::max(max_chain, bs);
            if (bs > 1) ++collisions;
        }
    }

    double avg_chain = non_empty > 0 ? double(total_chain_length) / non_empty : 0;
    double utilization = double(non_empty) / m.bucket_count() * 100;

    std::cout << "  Non-empty buckets: " << non_empty
              << " (" << utilization << "% utilization)\n";
    std::cout << "  Collision buckets: " << collisions << "\n";
    std::cout << "  Max chain length:  " << max_chain << "\n";
    std::cout << "  Avg chain length:  " << avg_chain << "\n";
    std::cout << "  Memory overhead:   ~"
              << m.bucket_count() * sizeof(void*) / 1024 << " KB (bucket array)\n\n";
}

int main() {
    constexpr int N = 100'000;

    // Default: max_load_factor = 1.0
    {
        std::unordered_map<int, int> m;
        // max_load_factor defaults to 1.0
        m.reserve(N);
        for (int i = 0; i < N; ++i) m[i] = i;
        analyze(m, "max_load_factor = 1.0 (default)");
    }

    // Low: max_load_factor = 0.25
    {
        std::unordered_map<int, int> m;
        m.max_load_factor(0.25f);  // Set BEFORE inserting or reserving!
        m.reserve(N);
        for (int i = 0; i < N; ++i) m[i] = i;
        analyze(m, "max_load_factor = 0.25");
    }

    // High: max_load_factor = 2.0 (save memory, more collisions)
    {
        std::unordered_map<int, int> m;
        m.max_load_factor(2.0f);
        m.reserve(N);
        for (int i = 0; i < N; ++i) m[i] = i;
        analyze(m, "max_load_factor = 2.0");
    }

    return 0;
}
// Typical output (values depend on implementation):
// max_load_factor=1.0:  ~100K buckets, ~100K*8 bytes, avg chain ~1.0
// max_load_factor=0.25: ~400K buckets, ~400K*8 bytes, avg chain ~0.25 (fewer collisions!)
// max_load_factor=2.0:  ~50K buckets,  ~50K*8 bytes,  avg chain ~2.0 (more collisions)
```

At `max_load_factor(0.25f)` you use 4x the memory for the bucket array, but the average chain length drops to ~0.25, meaning most lookups hit an element immediately with no chain traversal at all. That is the trade-off in one number.

**How it works:**

- `max_load_factor(f)` sets the threshold at which rehashing is triggered: when `size() / bucket_count() > f`.
- **Lower max_load_factor** -> more buckets for the same number of elements -> shorter chains -> fewer collisions -> faster lookups.
- **Trade-off:** More buckets means more memory consumed by the bucket array (each bucket is typically a pointer, 8 bytes on 64-bit).
- At `max_load_factor(0.25f)`, you need 4x the buckets compared to `1.0`, but average chain length drops to ~0.25, making lookups nearly always O(1) with no chain traversal.
- **When to use low load factor:** Latency-sensitive code (trading, real-time systems) where worst-case lookup time matters more than memory.
- **Set `max_load_factor` BEFORE `reserve()`** - `reserve()` uses the current max_load_factor to compute how many buckets to allocate.

### Q3: Explain the amortized O(1) insert and show the worst-case O(n) scenario

The amortized analysis is elegant but abstract. This example makes both the good case (standard use) and the catastrophic case (bad hash) concrete and measurable:

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>
#include <functional>
#include <chrono>

// === Amortized O(1) Analysis ===
//
// Hash table insert:
//   1. Compute hash:      O(1) for simple keys
//   2. Find bucket:       O(1) (modulo operation)
//   3. Traverse chain:    O(chain_length) - typically O(1) with good hash
//   4. Insert node:       O(1)
//   5. Check load factor: O(1)
//   6. If rehash needed:  O(n) - but happens rarely
//
// Amortized analysis (geometric series):
//   With doubling strategy, total rehash cost for n inserts:
//   n + n/2 + n/4 + n/8 + ... <= 2n
//   So cost per insert = 2n/n = O(1) amortized

// === Worst-case O(n): all keys hash to same bucket ===
struct BadHash {
    size_t operator()(int) const {
        return 42;  // Every key goes to bucket 42 % bucket_count!
    }
};

int main() {
    constexpr int N = 50'000;

    // === Good hash: amortized O(1) ===
    {
        std::unordered_map<int, int> m;
        m.reserve(N);

        auto start = std::chrono::steady_clock::now();
        for (int i = 0; i < N; ++i) m[i] = i;
        auto end = std::chrono::steady_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();

        std::cout << "=== Good hash (std::hash<int>) ===\n";
        std::cout << "  Inserted " << N << " elements in " << us << " us\n";
        std::cout << "  Per-insert: ~" << us * 1000 / N << " ns\n";

        // Find is O(1)
        start = std::chrono::steady_clock::now();
        volatile int sum = 0;
        for (int i = 0; i < N; ++i) sum += m[i];
        end = std::chrono::steady_clock::now();
        us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
        std::cout << "  Lookup " << N << " elements in " << us << " us\n\n";
    }

    // === Bad hash: O(n) per operation (hash collision attack) ===
    {
        std::unordered_map<int, int, BadHash> m;
        m.reserve(N);

        auto start = std::chrono::steady_clock::now();
        for (int i = 0; i < N; ++i) m[i] = i;
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();

        std::cout << "=== Bad hash (all collide) ===\n";
        std::cout << "  Inserted " << N << " elements in " << ms << " ms\n";
        std::cout << "  This is O(n^2) total - each insert walks the full chain!\n";

        // Find is O(n) - must traverse entire chain
        start = std::chrono::steady_clock::now();
        volatile bool found = m.count(N - 1);
        end = std::chrono::steady_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
        std::cout << "  Single lookup of last element: " << us << " us\n";
        std::cout << "  (Traverses chain of " << N << " elements)\n\n";
    }

    // === Worst-case from rehashing ===
    {
        std::cout << "=== Rehash cost demonstration ===\n";
        std::unordered_map<int, int> m;
        // Do NOT reserve - let rehashing happen

        size_t prev_bc = m.bucket_count();
        for (int i = 0; i < N; ++i) {
            m[i] = i;
            if (m.bucket_count() != prev_bc) {
                std::cout << "  Rehash at size=" << m.size()
                          << ": buckets " << prev_bc
                          << " -> " << m.bucket_count()
                          << " (moved " << m.size() << " elements)\n";
                prev_bc = m.bucket_count();
            }
        }
    }

    return 0;
}
// Typical output:
// === Good hash (std::hash<int>) ===
//   Inserted 50000 elements in ~2000 us
//   Per-insert: ~40 ns
//   Lookup 50000 elements in ~1500 us
//
// === Bad hash (all collide) ===
//   Inserted 50000 elements in ~5000 ms (2500x slower!)
//   Single lookup of last element: ~100 us (traverses 50000-node chain)
//
// === Rehash cost demonstration ===
//   Rehash at size=2: buckets 1 -> 3
//   Rehash at size=4: buckets 3 -> 7
//   ... (exponential growth)
//   Rehash at size=49153: buckets 49157 -> 98317
```

The `BadHash` result is shocking on purpose - 2500x slower than a good hash for the same number of elements. This is what a hash collision attack does to a server that uses user-supplied input as map keys without a randomized hash.

**How it works:**

**Amortized O(1):**

- With a good hash function, elements distribute uniformly across buckets. Average chain length approximately equals `load_factor()` <= 1.0.
- Rehashes happen when `size() > bucket_count() * max_load_factor()`. Bucket count roughly doubles each time.
- Total rehash work across n inserts: `n + n/2 + n/4 + ... <= 2n` -> amortized O(1) per insert.

**Worst-case O(n):**

1. **Hash collision attack:** If all keys hash to the same bucket (deliberately or via a poor hash function), the bucket becomes a linked list of all n elements. Every insert/find/erase traverses O(n) chain nodes -> total insert time is O(n²).
2. **Single rehash event:** When a rehash triggers, that one insert pays O(n) to relocate all elements. This doesn't happen often enough to break amortized O(1), but it causes a latency spike.

**Mitigations:**

- Use `reserve()` to eliminate rehash spikes.
- Use a good hash function (the default `std::hash` is fine for simple types).
- For security-critical code, use a randomized hash to prevent collision attacks (e.g., `absl::Hash`).

---

## Notes

- **`reserve()` vs `rehash()`:** `reserve(n)` says "I'll insert n elements", and it computes the needed bucket count. `rehash(n)` directly requests n buckets. Use `reserve()` unless you're specifically tuning bucket count.
- **Iterator invalidation:** `rehash()`, `reserve()`, and any insert that triggers rehashing invalidate **all** iterators. References and pointers to elements remain valid.
- **`max_load_factor` cannot be 0:** Setting it to 0 or negative is undefined behavior.
- **Hash flooding defense:** Some implementations (e.g., `absl::flat_hash_map`) use open addressing + randomized seeds to mitigate collision attacks. Standard `std::unordered_map` with separate chaining is more vulnerable.
- **Memory layout:** Standard `unordered_map` uses separate chaining (linked lists per bucket). This means poor cache locality - each node is a separate heap allocation. For better performance, consider `absl::flat_hash_map` or `robin_map` which use open addressing with inline storage.
- **`bucket_size(i)` for profiling:** You can iterate all buckets and check chain lengths to detect poor hash distribution at runtime.
