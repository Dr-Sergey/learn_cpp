# Use std::shared_mutex for reader-writer locking

**Category:** Concurrency and Parallelism  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/thread/shared_mutex>  

---

## Topic Overview

The reader-writer problem is one of the classic concurrency problems: multiple threads need concurrent read access, but writes must be exclusive. `std::shared_mutex` (C++17) solves this directly by supporting two distinct lock modes - shared for readers and exclusive for writers.

### Why a Plain Mutex Is Not Enough

A plain `std::mutex` is binary: locked or unlocked. Every lock acquisition is exclusive, even if the thread holding it only intends to read. That means ten reader threads all end up serializing against each other for no good reason. `std::shared_mutex` removes that unnecessary restriction.

| Scenario | `std::mutex` | `std::shared_mutex` |
| --- | --- | --- |
| N readers, 0 writers | Only 1 reader at a time (serial) | All N readers concurrently |
| 0 readers, 1 writer | 1 writer exclusively | 1 writer exclusively |
| N readers + 1 writer | Serial access for all | Writer waits for readers to drain, then exclusive |

### Lock Types

Here is a quick reference for which RAII wrapper to use with `shared_mutex`. The naming is intentional: `shared_lock` acquires shared ownership, everything else acquires exclusive.

```cpp
┌─────────────────────────────────────────────────┐
│            std::shared_mutex smtx;              │
│                                                 │
│  shared_lock<shared_mutex> rlock(smtx);         │
│     -> acquires SHARED ownership                │
│     -> multiple threads hold simultaneously     │
│                                                 │
│  unique_lock<shared_mutex> wlock(smtx);         │
│     -> acquires EXCLUSIVE ownership             │
│     -> blocks until all shared locks released   │
│                                                 │
│  lock_guard<shared_mutex> wlock(smtx);          │
│     -> also EXCLUSIVE (same as unique_lock but  │
│       no unlock/relock capability)              │
└─────────────────────────────────────────────────┘
```

### Reader-Writer Fairness Policies

This is one of the subtler aspects of `shared_mutex` that trips people up. The standard says nothing about how the implementation decides who gets the lock next - that's implementation-defined. In practice, most platforms use writer-preference to prevent the writer-starvation problem, but you can't rely on it.

| Policy | Behavior | Risk |
| --- | --- | --- |
| **Reader-preference** | Readers enter immediately when other readers hold the lock | **Writer starvation** - continuous readers starve waiting writers |
| **Writer-preference** | Once a writer waits, new readers queue behind it | **Reader starvation** - frequent writers starve readers |
| **Fair / FIFO** | Lock granted in arrival order | Balanced, slightly higher overhead |

Most implementations (glibc pthreads, Windows SRW locks) use writer-preference or FIFO to prevent writer starvation. The C++ standard does not mandate a policy.

### Core API

You will rarely call these raw methods directly - prefer the RAII wrappers - but it is useful to know they exist:

```cpp
#include <shared_mutex>

std::shared_mutex smtx;

// Exclusive (write) operations:
smtx.lock();          // blocks until exclusive
smtx.try_lock();      // non-blocking attempt
smtx.unlock();

// Shared (read) operations:
smtx.lock_shared();       // blocks until shared available
smtx.try_lock_shared();   // non-blocking attempt
smtx.unlock_shared();

// RAII wrappers (preferred):
std::shared_lock  rlock(smtx);   // shared
std::unique_lock  wlock(smtx);   // exclusive
std::lock_guard   wlock2(smtx);  // exclusive (simpler)
```

### Important Notes

- `shared_lock` is the only RAII wrapper that acquires shared ownership.
- `unique_lock` and `lock_guard` both acquire exclusive ownership on a `shared_mutex`.
- A thread that holds a shared lock must not attempt to upgrade to exclusive - this is undefined behavior.
- `std::shared_timed_mutex` adds `try_lock_for` / `try_lock_until` for timed waits.
- Benchmark before adopting - for low-contention or write-heavy workloads, a plain `std::mutex` can be faster due to lower overhead.

---

## Self-Assessment

### Q1: Protect a read-heavy data structure with `shared_mutex` using `shared_lock` for reads and `unique_lock` for writes

A DNS cache is a natural fit for this pattern: lookups happen constantly, updates happen rarely. The `mutable` on the mutex is the standard trick for allowing `const` member functions to lock.

```cpp
#include <iostream>
#include <shared_mutex>
#include <string>
#include <unordered_map>
#include <thread>
#include <vector>

class DnsCache {
    mutable std::shared_mutex mtx_;
    std::unordered_map<std::string, std::string> entries_;

public:
    // Multiple readers can call this concurrently
    std::string resolve(const std::string& host) const {
        std::shared_lock lock(mtx_);   // shared access
        auto it = entries_.find(host);
        return (it != entries_.end()) ? it->second : "";
    }

    // Only one writer at a time; blocks all readers
    void update(const std::string& host, const std::string& ip) {
        std::unique_lock lock(mtx_);   // exclusive access
        entries_[host] = ip;
    }

    size_t size() const {
        std::shared_lock lock(mtx_);
        return entries_.size();
    }
};

int main() {
    DnsCache cache;

    // Populate some entries
    cache.update("example.com", "93.184.216.34");
    cache.update("google.com",  "142.250.80.46");
    cache.update("github.com",  "140.82.121.4");

    // Launch 8 reader threads - all run concurrently
    std::vector<std::thread> readers;
    for (int i = 0; i < 8; ++i) {
        readers.emplace_back([&cache, i] {
            for (int j = 0; j < 1000; ++j) {
                auto ip = cache.resolve("github.com");
                // Readers proceed in parallel - no serialization
            }
            std::cout << "Reader " << i << " done\n";
        });
    }

    // 1 writer thread - acquires exclusive lock, blocks readers briefly
    std::thread writer([&cache] {
        for (int j = 0; j < 100; ++j) {
            cache.update("new-host.com", "10.0.0." + std::to_string(j));
        }
        std::cout << "Writer done\n";
    });

    for (auto& t : readers) t.join();
    writer.join();

    std::cout << "Cache size: " << cache.size() << "\n";
    // Expected output: 4 (3 initial + 1 from writer's last update)
}
```

The 8 reader threads all hold `shared_lock` simultaneously during their lookups, with zero serialization between them. Only when the writer arrives and acquires `unique_lock` do the remaining readers have to wait. This is the whole point of the pattern.

---

### Q2: Show a performance comparison between `mutex` and `shared_mutex` for a 10:1 read-write ratio

A benchmark is worth a thousand words here, because the performance story is not as simple as "shared_mutex is always better." Run this to see the actual numbers on your hardware.

```cpp
#include <iostream>
#include <shared_mutex>
#include <mutex>
#include <thread>
#include <vector>
#include <chrono>
#include <atomic>

static const int NUM_READERS = 10;
static const int NUM_WRITERS = 1;
static const int OPS_PER_THREAD = 100'000;

int shared_data = 0;

// ---------- Plain mutex ----------
std::mutex plain_mtx;

void plain_reader() {
    for (int i = 0; i < OPS_PER_THREAD; ++i) {
        std::lock_guard lock(plain_mtx);
        volatile int x = shared_data;  // simulate read
        (void)x;
    }
}

void plain_writer() {
    for (int i = 0; i < OPS_PER_THREAD; ++i) {
        std::lock_guard lock(plain_mtx);
        ++shared_data;
    }
}

// ---------- shared_mutex ----------
std::shared_mutex sh_mtx;

void shared_reader() {
    for (int i = 0; i < OPS_PER_THREAD; ++i) {
        std::shared_lock lock(sh_mtx);
        volatile int x = shared_data;
        (void)x;
    }
}

void shared_writer() {
    for (int i = 0; i < OPS_PER_THREAD; ++i) {
        std::unique_lock lock(sh_mtx);
        ++shared_data;
    }
}

template <typename ReaderFn, typename WriterFn>
long long benchmark(ReaderFn rfn, WriterFn wfn, const char* label) {
    shared_data = 0;
    std::vector<std::thread> threads;

    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < NUM_READERS; ++i)
        threads.emplace_back(rfn);
    for (int i = 0; i < NUM_WRITERS; ++i)
        threads.emplace_back(wfn);

    for (auto& t : threads) t.join();

    auto end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();

    std::cout << label << ": " << ms << " ms\n";
    return ms;
}

int main() {
    auto plain_ms  = benchmark(plain_reader, plain_writer, "std::mutex       ");
    auto shared_ms = benchmark(shared_reader, shared_writer, "std::shared_mutex");

    if (shared_ms > 0) {
        double speedup = static_cast<double>(plain_ms) / shared_ms;
        std::cout << "Speedup: " << speedup << "x\n";
    }
    // Typical result (10:1 read-write, 8+ cores):
    //   std::mutex       : 450 ms
    //   std::shared_mutex: 120 ms
    //   Speedup: 3.75x
    //
    // With fewer cores or higher write ratio, the gap shrinks.
}
```

At a 10:1 read-write ratio on a multi-core machine, the speedup is dramatic. On a single core or with a high write ratio, the overhead of tracking the reader count atomically means `shared_mutex` actually loses.

**When `shared_mutex` Wins vs Loses:**

| Condition | Winner |
| --- | --- |
| Read-heavy (10:1 or more), many cores | `shared_mutex` - readers run in parallel |
| Write-heavy (1:1 or worse) | `mutex` - shared_mutex overhead not amortized |
| Single-core machine | `mutex` - no parallelism to exploit |
| Very short critical sections | `mutex` - lock overhead dominates |
| Long reads (e.g., serializing a container) | `shared_mutex` - parallel reads save wall time |

---

### Q3: Explain the reader starvation risk and how `shared_mutex` implementations mitigate it

This is the subtlest part of reader-writer locks. The problem is that if new readers keep arriving before existing readers finish, the shared lock count never drops to zero, and a waiting writer can be blocked indefinitely. The diagram below shows the worst case.

```cpp
Timeline - Reader-Preference Policy:

Writer waits ──────────────────────> still waiting ────> STARVED
    │
    ▼
R1: ████████████
R2:    ████████████
R3:       ████████████            <- continuous readers keep arriving
R4:          ████████████           before R1 finishes, so shared
R5:             ████████████        count never drops to 0
R6:                ████████████
    ────────────────────────────────────> time

The writer can NEVER acquire exclusive access if readers keep arriving.
```

**How Implementations Mitigate This:**

1. **Writer-preference (most common - glibc, MSVC SRW locks):** Once a writer is waiting, new reader requests queue behind the writer. Existing readers finish; then the writer gets exclusive access. New readers resume after the writer releases.

```cpp
Writer-Preference Timeline:

R1: ████████                          R4: ██████████
R2:    ██████████                     R5:    ██████████
R3:       █████████
                    W1: ████████████
                    ↑
                    Writer arrived - new readers R4, R5 wait
```

2. **FIFO / ticket-based:** Every lock request gets a ticket number. Served in order - no starvation possible for either side. Slightly higher overhead due to ticket management.

3. **Timed locks (`std::shared_timed_mutex`):** Writers use `try_lock_for()` - if readers hold too long, the writer can detect the timeout and retry or escalate.

**Practical Recommendations:**

```cpp
// 1. Minimize time under lock to reduce starvation window
std::string read_data() const {
    std::shared_lock lock(mtx_);
    return data_;                // copy under lock, fast
}   // lock released immediately

// 2. For guaranteed fairness, implement ticket-based approach:
//    std::atomic<uint64_t> next_ticket{0};
//    std::atomic<uint64_t> now_serving{0};

// 3. Profile with perf/VTune to detect actual starvation
//    Look for: high lock contention, writer wait time >> read time
```

The C++ standard does not specify fairness policy - it is implementation-defined. Most platforms use writer-preference, making writer starvation rare but reader starvation possible under heavy write load. If fairness is critical, consider `std::shared_timed_mutex` with timeouts or an explicit ticket-based scheme. Always benchmark your actual workload - theoretical starvation may not be a real problem if write frequency is low.

---

## Notes

- No upgrade path: you cannot atomically upgrade a `shared_lock` to `unique_lock`. You must release the shared lock, then acquire exclusive - this creates a TOCTOU gap where data may change.
- `std::shared_timed_mutex` adds: `try_lock_for()`, `try_lock_until()`, `try_lock_shared_for()`, `try_lock_shared_until()`.
- Recursive locking: `shared_mutex` is not recursive. A single thread acquiring the same lock twice (shared or exclusive) is undefined behavior.
- Compile flags: use `-pthread` on GCC/Clang. Use `/std:c++17` on MSVC.
- Thread Sanitizer: always test with `-fsanitize=thread` to catch missed synchronization.
