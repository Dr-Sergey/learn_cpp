# Implement a concurrent hash map using striped locking

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/mutex>  

---

## Topic Overview

A **concurrent hash map** allows multiple threads to read and write simultaneously without corrupting the data structure. The simplest approach - one global mutex - serializes every operation and wastes all the parallelism you paid for. **Striped locking** is a practical middle ground: partition the hash table into N groups called "stripes," each protected by its own mutex. Threads operating on different stripes proceed in parallel. Only threads that happen to hash to the same stripe need to wait for each other.

### Striped Locking Architecture

The key idea is that the stripe index is determined by the hash of the key, not by the bucket index. So 64 buckets might be covered by only 16 mutexes, and any two operations whose keys hash to different stripes run concurrently:

```cpp
Hash table with 16 buckets, 4 stripes:

Stripe 0 (mutex_0):  | Bucket 0 | Bucket 4 | Bucket 8  | Bucket 12 |
Stripe 1 (mutex_1):  | Bucket 1 | Bucket 5 | Bucket 9  | Bucket 13 |
Stripe 2 (mutex_2):  | Bucket 2 | Bucket 6 | Bucket 10 | Bucket 14 |
Stripe 3 (mutex_3):  | Bucket 3 | Bucket 7 | Bucket 11 | Bucket 15 |

stripe_index = hash(key) % num_stripes
Thread A (bucket 3) locks stripe 3
Thread B (bucket 5) locks stripe 1  <- runs in PARALLEL!
```

### Approach Comparison

| Approach | Contention | Complexity | Memory overhead |
| --- | --- | --- | --- |
| Global mutex | O(1) bottleneck | Simple | 1 mutex |
| Striped locking | O(1/N) per stripe | Moderate | N mutexes |
| Per-bucket mutex | Minimal | Moderate | 1 mutex per bucket |
| Lock-free | None (CAS retries) | Very complex | Tagged pointers |

Striped locking is the "sweet spot" for most real applications: dramatically better concurrency than a global mutex, and dramatically simpler than lock-free code.

---

## Self-Assessment

### Q1: Partition the hash map into N buckets each protected by its own mutex (striped locking)

**Answer:**

Here is a complete implementation. Notice that `stripe_index` and `bucket_index` are computed separately from the same hash - they do not have to be the same value, which is the point:

```cpp
#include <vector>
#include <list>
#include <mutex>
#include <functional>
#include <optional>
#include <iostream>
#include <string>
#include <array>
#include <utility>

template<typename K, typename V, typename Hash = std::hash<K>>
class StripedHashMap {
    struct Bucket {
        std::list<std::pair<K, V>> entries;
    };

    static constexpr size_t NUM_STRIPES = 16;
    static constexpr size_t NUM_BUCKETS = 64;

    std::vector<Bucket> buckets_;
    mutable std::array<std::mutex, NUM_STRIPES> stripes_;
    Hash hasher_;

    size_t bucket_index(const K& key) const {
        return hasher_(key) % NUM_BUCKETS;
    }

    size_t stripe_index(const K& key) const {
        return hasher_(key) % NUM_STRIPES;
    }

public:
    StripedHashMap() : buckets_(NUM_BUCKETS) {}

    void insert(const K& key, const V& value) {
        std::lock_guard lock(stripes_[stripe_index(key)]);
        auto& bucket = buckets_[bucket_index(key)];
        for (auto& [k, v] : bucket.entries) {
            if (k == key) { v = value; return; } // update existing
        }
        bucket.entries.emplace_back(key, value);
    }

    std::optional<V> find(const K& key) const {
        std::lock_guard lock(stripes_[stripe_index(key)]);
        const auto& bucket = buckets_[bucket_index(key)];
        for (const auto& [k, v] : bucket.entries) {
            if (k == key) return v;
        }
        return std::nullopt;
    }

    bool erase(const K& key) {
        std::lock_guard lock(stripes_[stripe_index(key)]);
        auto& bucket = buckets_[bucket_index(key)];
        for (auto it = bucket.entries.begin(); it != bucket.entries.end(); ++it) {
            if (it->first == key) {
                bucket.entries.erase(it);
                return true;
            }
        }
        return false;
    }
};

int main() {
    StripedHashMap<std::string, int> map;
    map.insert("alice", 95);
    map.insert("bob", 87);

    if (auto val = map.find("alice"))
        std::cout << "alice: " << *val << "\n"; // alice: 95

    map.erase("bob");
    std::cout << "bob found: " << map.find("bob").has_value() << "\n"; // 0
}
```

The map has 64 buckets but only 16 mutexes. `stripe_index = hash(key) % 16` determines which mutex protects a given key. Multiple threads accessing keys in different stripes run fully in parallel. Keys in the same stripe serialize, but with 16 stripes the probability of contention is low.

### Q2: Show that striped locking reduces contention from O(1) bottleneck to O(1/N) per thread

**Answer:**

Numbers are worth more than theory here. The benchmark below runs 8 threads doing half a million inserts each, comparing a global-lock map against a 16-stripe map:

```cpp
#include <unordered_map>
#include <mutex>
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>
#include <array>
#include <string>

// === Global mutex version ===
class GlobalLockMap {
    std::unordered_map<int, int> data_;
    std::mutex mtx_;
public:
    void insert(int k, int v) {
        std::lock_guard lock(mtx_);
        data_[k] = v;
    }
};

// === Striped locking version ===
class StripedMap {
    static constexpr size_t STRIPES = 16;
    std::array<std::unordered_map<int, int>, STRIPES> shards_;
    std::array<std::mutex, STRIPES> mutexes_;

    size_t shard(int key) const { return std::hash<int>{}(key) % STRIPES; }
public:
    void insert(int k, int v) {
        auto s = shard(k);
        std::lock_guard lock(mutexes_[s]);
        shards_[s][k] = v;
    }
};

template<typename Map>
long long benchmark(Map& map, int num_threads, int ops_per_thread) {
    auto start = std::chrono::steady_clock::now();

    std::vector<std::thread> threads;
    for (int t = 0; t < num_threads; ++t) {
        threads.emplace_back([&, t] {
            for (int i = 0; i < ops_per_thread; ++i)
                map.insert(t * ops_per_thread + i, i);
        });
    }
    for (auto& t : threads) t.join();

    return std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();
}

int main() {
    constexpr int THREADS = 8;
    constexpr int OPS = 500'000;

    GlobalLockMap gm;
    auto global_ms = benchmark(gm, THREADS, OPS);

    StripedMap sm;
    auto striped_ms = benchmark(sm, THREADS, OPS);

    std::cout << "Global mutex:    " << global_ms << " ms\n";
    std::cout << "Striped (16):    " << striped_ms << " ms\n";
    std::cout << "Speedup:         " << static_cast<double>(global_ms) / striped_ms << "x\n";
    // Typical output (8 threads):
    // Global mutex:    320 ms
    // Striped (16):    65 ms
    // Speedup:         4.9x
    //
    // With N stripes and T threads, probability of contention ~= T/N
    // More stripes -> less contention -> better scaling
}
```

With a global mutex, all threads compete for one lock and throughput plateaus regardless of core count. With 16 stripes and 8 threads, the contention probability per operation is roughly 0.5 (8/16), compared to 1.0 for the global mutex. In practice this yields 3 to 6x speedup on 8 cores.

### Q3: Compare striped locking with a single global mutex and with a fully lock-free map

**Answer:**

The table below summarizes the trade-offs across all three approaches. After that, a short decision guide tells you when to pick each one:

```cpp
+---------------------+------------------+------------------+--------------------+
| Criterion           | Global Mutex     | Striped Locking  | Lock-Free          |
+---------------------+------------------+------------------+--------------------+
| Implementation      | Trivial          | Moderate         | Very complex       |
| Concurrency         | None (serial)    | O(N) parallelism | Maximum            |
| Contention          | O(T) (all clash) | O(T/N)           | CAS retry loops    |
| Latency             | Predictable      | Predictable      | Variable (retries) |
| Dead/live-lock risk | Deadlock if >1   | Deadlock if >1   | Livelock possible  |
| Priority inversion  | Yes              | Yes              | No                 |
| Memory overhead     | 1 mutex          | N mutexes        | Tagged pointers    |
| Resize support      | Easy             | Complex (re-stripe)| Very complex      |
| Debugging           | Easy             | Moderate         | Very hard          |
| Best for            | Low contention   | Moderate-high    | Extreme throughput |
+---------------------+------------------+------------------+--------------------+
```

```cpp
#include <iostream>

int main() {
    // DECISION GUIDE:
    //
    // 1. Global mutex:
    //    Use when: contention is rare, simplicity matters most
    //    Example: config map read at startup, rarely updated
    //
    // 2. Striped locking:
    //    Use when: moderate-to-high write contention, reasonable complexity
    //    Example: web server session store, in-memory cache
    //    Tuning: set num_stripes >= 2 * num_threads
    //
    // 3. Lock-free:
    //    Use when: maximum throughput required, expert-level team
    //    Example: HFT order book, real-time systems
    //    Warning: extremely hard to get right (ABA, memory reclamation)

    // If unsure, START WITH STRIPED LOCKING - it offers the best
    // complexity-to-performance ratio for most applications.

    std::cout << "Start simple, measure, optimize\n";
}
```

Striped locking is the right default for high-contention maps. Start with a global mutex during prototyping, profile to confirm contention is actually a bottleneck, then upgrade to striped locking. Only go lock-free when striped locking is proven insufficient and you have the expertise to handle ABA and safe memory reclamation.

---

## Notes

- Stripe count: Use at least `2 * max_threads` stripes to minimize contention. Power-of-2 counts enable `& (N-1)` instead of `% N` for faster index computation.
- Reader-writer locking: Use `std::shared_mutex` per stripe if reads vastly outnumber writes - readers can then proceed concurrently within the same stripe.
- Resize challenge: Increasing bucket count with striped locking requires locking all stripes during resize. Java's `ConcurrentHashMap` solves this with a complex "transfer" protocol, which gives you a sense of how much engineering goes into production-quality concurrent maps.
- `std::unordered_map` is not thread-safe. Even concurrent reads can race with a concurrent write because a rehash invalidates iterators.
- Test with `-fsanitize=thread` to verify correctness before trusting performance measurements.
