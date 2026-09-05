# Use std::shared_mutex for read-write locking

**Category:** Concurrency and Parallelism  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/thread/shared_mutex>  

---

## Topic Overview

`std::shared_mutex` allows multiple concurrent readers OR one exclusive writer. This is the classic reader-writer lock pattern, and it exists because a plain `std::mutex` is unnecessarily restrictive when most of your threads are only reading.

### Lock Modes

The two lock modes map directly onto two RAII wrappers. Use `std::shared_lock` for reads and `std::unique_lock` for writes - the naming makes the intent clear at every call site.

```cpp
Shared (read) lock:        Exclusive (write) lock:
──────────────────          ─────────────────────
std::shared_lock lk(mtx);  std::unique_lock lk(mtx);
// Multiple threads can     // Only ONE thread at a time
// hold shared locks        // Blocks all readers AND writers
// simultaneously

Compatibility matrix:
┌──────────┬──────────┬───────────┐
│          │ Shared   │ Exclusive │
├──────────┼──────────┼───────────┤
│ Shared   │ Yes      │ blocks    │
│ Exclusive│ blocks   │ blocks    │
└──────────┴──────────┴───────────┘
```

---

## Self-Assessment

### Q1: Implement a thread-safe cache using shared_mutex where multiple readers can proceed concurrently

The key pattern here is `mutable std::shared_mutex` on the class - `mutable` is needed because `const` member functions like `get()` still need to acquire the lock. Notice how clearly the RAII wrappers communicate intent: `shared_lock` means "read-only, others may join," `unique_lock` means "exclusive, everyone else waits."

```cpp
#include <shared_mutex>
#include <mutex>
#include <unordered_map>
#include <string>
#include <thread>
#include <vector>
#include <iostream>
#include <optional>

class ThreadSafeCache {
    mutable std::shared_mutex mtx_; // mutable: lock in const methods
    std::unordered_map<std::string, std::string> cache_;

public:
    // READ: shared lock - multiple readers simultaneously
    std::optional<std::string> get(const std::string& key) const {
        std::shared_lock lock(mtx_); // shared (read) lock
        auto it = cache_.find(key);
        if (it != cache_.end())
            return it->second;
        return std::nullopt;
    }

    // WRITE: exclusive lock - blocks all readers and writers
    void put(const std::string& key, const std::string& value) {
        std::unique_lock lock(mtx_); // exclusive (write) lock
        cache_[key] = value;
    }

    // WRITE: exclusive lock for deletion
    bool remove(const std::string& key) {
        std::unique_lock lock(mtx_);
        return cache_.erase(key) > 0;
    }

    // READ: shared lock for size check
    size_t size() const {
        std::shared_lock lock(mtx_);
        return cache_.size();
    }
};

int main() {
    ThreadSafeCache cache;

    // Pre-populate
    for (int i = 0; i < 100; ++i)
        cache.put("key" + std::to_string(i), "value" + std::to_string(i));

    std::atomic<int> read_count{0};
    std::atomic<int> write_count{0};

    // 8 reader threads (concurrent reads)
    std::vector<std::jthread> readers;
    for (int t = 0; t < 8; ++t) {
        readers.emplace_back([&cache, &read_count, t] {
            for (int i = 0; i < 10'000; ++i) {
                auto val = cache.get("key" + std::to_string(i % 100));
                if (val) read_count.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }

    // 2 writer threads (exclusive writes)
    std::vector<std::jthread> writers;
    for (int t = 0; t < 2; ++t) {
        writers.emplace_back([&cache, &write_count, t] {
            for (int i = 0; i < 1'000; ++i) {
                cache.put("key" + std::to_string(i % 100),
                           "updated" + std::to_string(i));
                write_count.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }

    // jthreads auto-join
    readers.clear();
    writers.clear();

    std::cout << "Reads: " << read_count.load() << "\n";
    std::cout << "Writes: " << write_count.load() << "\n";
    std::cout << "Cache size: " << cache.size() << "\n";

    // Output:
    // Reads: 80000
    // Writes: 2000
    // Cache size: 100
}
```

The 8 reader threads can all hold their `shared_lock` simultaneously. Only when a writer arrives does anyone have to wait. This is the whole benefit: reads are free to parallelize as long as no write is in progress.

### Q2: Show the upgrade pattern: acquire shared lock, detect stale data, re-acquire as exclusive

This pattern comes up in lazy caching: you want to check first (cheap, shared lock) and only write if the value is missing (expensive, exclusive lock). The critical thing to understand is that `shared_mutex` does NOT support upgrading a lock in place - you must release the shared lock before acquiring the exclusive one, which creates a small window where another thread could beat you to the write.

```cpp
#include <shared_mutex>
#include <unordered_map>
#include <string>
#include <iostream>
#include <thread>
#include <chrono>

class LazyCache {
    mutable std::shared_mutex mtx_;
    std::unordered_map<std::string, std::string> cache_;

    std::string compute_value(const std::string& key) const {
        // Simulate expensive computation
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        return "computed_" + key;
    }

public:
    // "Upgrade" pattern: read first, write only if needed
    std::string get_or_compute(const std::string& key) {
        // Step 1: Try shared (read) lock first
        {
            std::shared_lock read_lock(mtx_);
            auto it = cache_.find(key);
            if (it != cache_.end()) {
                return it->second; // cache hit - fast path
            }
        }
        // read_lock released here!

        // Step 2: Cache miss - need exclusive (write) lock
        // NOTE: We CANNOT "upgrade" a shared_lock to unique_lock!
        // We must release the shared lock and acquire exclusive.
        {
            std::unique_lock write_lock(mtx_);

            // CRITICAL: Re-check! Another thread may have populated
            // the cache between our shared unlock and exclusive lock
            auto it = cache_.find(key);
            if (it != cache_.end()) {
                return it->second; // another thread beat us to it
            }

            // Actually compute and insert
            std::string value = compute_value(key);
            cache_[key] = value;
            return value;
        }
    }

    // === WHY CAN'T WE UPGRADE IN-PLACE? ===
    // std::shared_mutex does NOT support atomic upgrade from shared -> exclusive.
    // If two threads both hold shared locks and both try to upgrade,
    // they would deadlock (each waiting for the other to release shared).
    //
    // The double-check pattern is the standard workaround:
    //   1. shared_lock: check cache
    //   2. Release shared_lock
    //   3. unique_lock: re-check + update
    //
    // The re-check in step 3 is essential (TOCTOU guard).
};

int main() {
    LazyCache cache;
    std::vector<std::jthread> threads;

    for (int t = 0; t < 8; ++t) {
        threads.emplace_back([&cache, t] {
            // All threads ask for the same keys - only first computes
            for (int i = 0; i < 5; ++i) {
                std::string key = "item" + std::to_string(i);
                auto val = cache.get_or_compute(key);
                std::cout << "Thread " << t << ": " << key
                          << " = " << val << "\n";
            }
        });
    }

    // Output (order varies):
    // Thread 0: item0 = computed_item0
    // Thread 3: item0 = computed_item0  (cache hit)
    // Thread 1: item1 = computed_item1
    // ...
}
```

The double-check after acquiring the exclusive lock is not optional - it's the TOCTOU guard. Between releasing the shared lock and acquiring the exclusive one, another thread may have computed and inserted the value. The re-check ensures you don't redo the work unnecessarily.

### Q3: Benchmark shared_mutex vs plain mutex for a read-heavy workload

The benchmark below is illuminating because it reveals a trap: `shared_mutex` is not unconditionally faster than plain `mutex`. It only wins when reads substantially outnumber writes and there is actual contention to relieve.

```cpp
#include <shared_mutex>
#include <mutex>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>
#include <unordered_map>

template<typename MutexType>
long long benchmark(int n_readers, int n_writers, int ops_per_thread) {
    MutexType mtx;
    std::unordered_map<int, int> data;
    for (int i = 0; i < 100; ++i) data[i] = i;

    auto start = std::chrono::steady_clock::now();
    std::vector<std::jthread> threads;

    // Readers
    for (int t = 0; t < n_readers; ++t) {
        threads.emplace_back([&mtx, &data, ops_per_thread, t] {
            volatile int sink = 0;
            for (int i = 0; i < ops_per_thread; ++i) {
                if constexpr (std::is_same_v<MutexType, std::shared_mutex>) {
                    std::shared_lock lock(mtx); // shared
                    sink = data.at(i % 100);
                } else {
                    std::lock_guard lock(mtx); // exclusive (even for reads!)
                    sink = data.at(i % 100);
                }
            }
        });
    }

    // Writers
    for (int t = 0; t < n_writers; ++t) {
        threads.emplace_back([&mtx, &data, ops_per_thread, t] {
            for (int i = 0; i < ops_per_thread / 10; ++i) {
                if constexpr (std::is_same_v<MutexType, std::shared_mutex>) {
                    std::unique_lock lock(mtx); // exclusive
                    data[i % 100] = i + t;
                } else {
                    std::lock_guard lock(mtx); // exclusive
                    data[i % 100] = i + t;
                }
            }
        });
    }

    threads.clear(); // join all
    auto ns = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();
    return ns;
}

int main() {
    constexpr int OPS = 500'000;

    std::cout << "Read-heavy workload (8 readers, 1 writer):\n";
    auto ms_plain = benchmark<std::mutex>(8, 1, OPS);
    auto ms_shared = benchmark<std::shared_mutex>(8, 1, OPS);
    std::cout << "  std::mutex:        " << ms_plain << " ms\n";
    std::cout << "  std::shared_mutex: " << ms_shared << " ms\n";
    std::cout << "  Speedup: " << (double)ms_plain / ms_shared << "x\n";

    std::cout << "\nWrite-heavy workload (2 readers, 6 writers):\n";
    ms_plain = benchmark<std::mutex>(2, 6, OPS);
    ms_shared = benchmark<std::shared_mutex>(2, 6, OPS);
    std::cout << "  std::mutex:        " << ms_plain << " ms\n";
    std::cout << "  std::shared_mutex: " << ms_shared << " ms\n";
    std::cout << "  Speedup: " << (double)ms_plain / ms_shared << "x\n";

    // Typical output:
    // Read-heavy workload (8 readers, 1 writer):
    //   std::mutex:        320 ms
    //   std::shared_mutex: 85 ms
    //   Speedup: 3.76x
    //
    // Write-heavy workload (2 readers, 6 writers):
    //   std::mutex:        180 ms
    //   std::shared_mutex: 210 ms
    //   Speedup: 0.86x    <- shared_mutex is SLOWER when write-heavy!
    //
    // KEY INSIGHT:
    //   shared_mutex has HIGHER per-operation overhead than plain mutex
    //   (it must maintain reader count atomically).
    //   It only wins when reads outnumber writes AND there's contention.
}
```

The write-heavy result is the important lesson. `shared_mutex` has to maintain an atomic reader count, which means it is intrinsically more expensive per operation than a plain mutex. That overhead only pays off when concurrent readers would otherwise be serialized unnecessarily. When writes are frequent, the extra overhead makes `shared_mutex` a net loss.

**When shared_mutex wins:** many readers, few writers, readers hold the lock for significant time. **When it loses:** write-heavy workloads, very short critical sections, or low contention (plain mutex is simpler and faster).

---

## Notes

- `std::shared_lock` (C++14) is the RAII wrapper for shared (read) locks. Use it with `shared_mutex`.
- `std::unique_lock` is the RAII wrapper for exclusive (write) locks.
- Writer starvation: some implementations may starve writers if readers keep arriving. The standard doesn't specify scheduling policy.
- `shared_timed_mutex` (C++14) adds `try_lock_for`/`try_lock_shared_for` for timed operations.
- No upgrade: `shared_mutex` does not support upgrading from shared to exclusive. Always release shared, then acquire exclusive (with double-check).
- Rule of thumb: if the read/write ratio is less than 10:1, benchmark before using `shared_mutex`. Plain `mutex` might be faster due to lower overhead.
- Compile with `-std=c++17 -O2 -pthread`.
