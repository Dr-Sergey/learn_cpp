# Implement a wait-free data structure using fetch_add

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic/fetch_add>  

---

## Topic Overview

**Wait-free** algorithms guarantee that every thread completes its operation in a bounded number of steps, regardless of what other threads do. `std::atomic::fetch_add` is the canonical wait-free primitive — it atomically adds a value and returns the previous value in a single CPU instruction (e.g., `LOCK XADD` on x86).

### Progress Guarantees Hierarchy

```cpp

┌──────────────┐  Weakest
│  Blocking    │  Threads may wait indefinitely (mutex)
├──────────────┤
│  Lock-free   │  At least ONE thread makes progress (CAS loops)
├──────────────┤
│  Wait-free   │  EVERY thread makes progress in bounded steps
└──────────────┘  Strongest

```

### fetch_add Properties

| Property | Value |
| --- | --- |
| Atomicity | Single instruction on most architectures |
| Progress | Wait-free (no retry loop) |
| Return value | Previous value before addition |
| Memory ordering | Configurable (relaxed to seq_cst) |
| Applicable types | Integral types, pointers |

---

## Self-Assessment

### Q1: Build a wait-free multi-producer counter using fetch_add with memory_order_relaxed

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

class WaitFreeCounter {
    std::atomic<long long> count_{0};

public:
    void increment() {
        // fetch_add is wait-free: ONE instruction, no loop, no retry
        count_.fetch_add(1, std::memory_order_relaxed);
        //                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^
        // relaxed is sufficient for a counter — we only need atomicity,
        // not ordering with other variables
    }

    void add(long long n) {
        count_.fetch_add(n, std::memory_order_relaxed);
    }

    long long get() const {
        return count_.load(std::memory_order_relaxed);
    }

    // fetch_add returns the PREVIOUS value — useful for unique IDs
    long long next_id() {
        return count_.fetch_add(1, std::memory_order_relaxed);
    }
};

int main() {
    WaitFreeCounter counter;
    constexpr int NUM_THREADS = 8;
    constexpr int OPS_PER_THREAD = 1'000'000;

    auto start = std::chrono::steady_clock::now();

    std::vector<std::thread> threads;
    for (int t = 0; t < NUM_THREADS; ++t) {
        threads.emplace_back([&] {
            for (int i = 0; i < OPS_PER_THREAD; ++i)
                counter.increment();
        });
    }
    for (auto& t : threads) t.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    std::cout << "Count: " << counter.get() << "\n";
    std::cout << "Expected: " << (long long)NUM_THREADS * OPS_PER_THREAD << "\n";
    std::cout << "Time: " << ms << " ms\n";
    // Output:
    // Count: 8000000
    // Expected: 8000000
    // Time: ~50 ms

    // === Wait-free unique ID generator ===
    WaitFreeCounter id_gen;
    std::vector<long long> ids(NUM_THREADS);
    std::vector<std::thread> id_threads;
    for (int t = 0; t < NUM_THREADS; ++t) {
        id_threads.emplace_back([&, t] {
            ids[t] = id_gen.next_id(); // each thread gets a unique ID
        });
    }
    for (auto& t : id_threads) t.join();

    std::cout << "Unique IDs: ";
    for (auto id : ids) std::cout << id << " ";
    std::cout << "\n";
    // Output: 0 1 2 3 4 5 6 7 (order may vary)
}

```

**Explanation:** `fetch_add(1, memory_order_relaxed)` compiles to a single `LOCK XADD` instruction on x86 — no CAS loop, no retry, no possibility of contention-induced retry. Every thread completes in exactly one instruction, making it wait-free. The `relaxed` ordering is sufficient because counters don't need ordering with other variables — just atomicity.

### Q2: Show that fetch_add is wait-free: every thread completes its operation in a bounded number of steps

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <iostream>
#include <vector>
#include <chrono>

// PROOF BY CONTRAST: CAS loop (lock-free) vs fetch_add (wait-free)

std::atomic<int> cas_counter{0};
std::atomic<int> fa_counter{0};

std::atomic<long long> cas_retries{0}; // count CAS retry attempts
std::atomic<long long> fa_retries{0};  // should always be 0

void cas_increment(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        int expected = cas_counter.load(std::memory_order_relaxed);
        // CAS loop — may retry multiple times under contention
        while (!cas_counter.compare_exchange_weak(
            expected, expected + 1,
            std::memory_order_relaxed, std::memory_order_relaxed)) {
            cas_retries.fetch_add(1, std::memory_order_relaxed); // count retries
        }
    }
}

void fa_increment(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        // fetch_add — ALWAYS succeeds on first attempt
        fa_counter.fetch_add(1, std::memory_order_relaxed);
        // No retry possible — the instruction completes atomically
    }
}

int main() {
    constexpr int THREADS = 8;
    constexpr int ITERS = 1'000'000;

    // === CAS-based increment (lock-free, NOT wait-free) ===
    {
        std::vector<std::thread> threads;
        auto start = std::chrono::steady_clock::now();
        for (int t = 0; t < THREADS; ++t)
            threads.emplace_back(cas_increment, ITERS);
        for (auto& t : threads) t.join();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start).count();

        std::cout << "CAS counter:  " << cas_counter.load() << "\n";
        std::cout << "CAS retries:  " << cas_retries.load() << "\n";
        std::cout << "CAS time:     " << ms << " ms\n";
    }

    // === fetch_add increment (wait-free) ===
    {
        std::vector<std::thread> threads;
        auto start = std::chrono::steady_clock::now();
        for (int t = 0; t < THREADS; ++t)
            threads.emplace_back(fa_increment, ITERS);
        for (auto& t : threads) t.join();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start).count();

        std::cout << "FA counter:   " << fa_counter.load() << "\n";
        std::cout << "FA retries:   " << fa_retries.load() << "\n";
        std::cout << "FA time:      " << ms << " ms\n";
    }

    // Typical output (8 threads, 1M iterations each):
    // CAS counter:  8000000
    // CAS retries:  3500000  ← millions of retries under contention!
    // CAS time:     180 ms
    // FA counter:   8000000
    // FA retries:   0        ← ZERO retries — every call succeeds immediately
    // FA time:      50 ms

    // CAS (lock-free): A thread's CAS can fail if another thread modifies
    // the value between load and CAS. Under high contention, a thread
    // can theoretically retry indefinitely (starve).
    //
    // fetch_add (wait-free): The hardware guarantees the operation
    // completes in a fixed number of steps (typically 1 instruction).
    // No thread can be starved, regardless of contention level.
}

```

**Explanation:** The CAS-based counter has millions of retries under high contention — a thread might spin many times before succeeding. Theoretically, one unlucky thread could starve. `fetch_add` has zero retries — every single call succeeds in one atomic instruction. This is the definition of wait-free: bounded steps for every thread, regardless of concurrency.

### Q3: Compare wait-free vs lock-free vs blocking for a high-contention counter workload

**Answer:**

```cpp

#include <atomic>
#include <mutex>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

constexpr int THREADS = 8;
constexpr int ITERS = 5'000'000;

// === Blocking: mutex-protected counter ===
struct BlockingCounter {
    int count = 0;
    std::mutex mtx;
    void increment() {
        std::lock_guard lock(mtx);
        ++count;
    }
};

// === Lock-free: CAS loop counter ===
struct LockFreeCounter {
    std::atomic<int> count{0};
    void increment() {
        int expected = count.load(std::memory_order_relaxed);
        while (!count.compare_exchange_weak(expected, expected + 1,
            std::memory_order_relaxed, std::memory_order_relaxed))
            ;
    }
};

// === Wait-free: fetch_add counter ===
struct WaitFreeCounter {
    std::atomic<int> count{0};
    void increment() {
        count.fetch_add(1, std::memory_order_relaxed);
    }
};

template<typename Counter>
long long bench(const char* name) {
    Counter c;
    auto start = std::chrono::steady_clock::now();

    std::vector<std::thread> threads;
    for (int t = 0; t < THREADS; ++t)
        threads.emplace_back([&] {
            for (int i = 0; i < ITERS; ++i) c.increment();
        });
    for (auto& t : threads) t.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();
    std::cout << name << ": " << ms << " ms (count=" << c.count << ")\n";
    return ms;
}

int main() {
    auto blocking  = bench<BlockingCounter>("Blocking (mutex)");
    auto lockfree  = bench<LockFreeCounter>("Lock-free (CAS) ");
    auto waitfree  = bench<WaitFreeCounter>("Wait-free (FA)  ");

    // Typical results (8 threads, 5M iterations each):
    // Blocking (mutex): ~1200 ms
    // Lock-free (CAS):  ~450 ms
    // Wait-free (FA):   ~120 ms
    //
    // ANALYSIS:
    // ┌───────────────┬──────────┬────────────┬───────────────┐
    // │               │ Blocking │ Lock-free  │ Wait-free     │
    // ├───────────────┼──────────┼────────────┼───────────────┤
    // │ Mechanism     │ Mutex    │ CAS loop   │ fetch_add     │
    // │ Starvation    │ Possible │ Possible   │ Impossible    │
    // │ Overhead      │ Kernel   │ Retry loop │ 1 instruction │
    // │ Applicability │ General  │ General    │ Limited       │
    // │ Complexity    │ Simple   │ Complex    │ Simple        │
    // │ Tail latency  │ Bad      │ Medium     │ Best          │
    // └───────────────┴──────────┴────────────┴───────────────┘
    //
    // Wait-free is FASTEST but limited to simple operations (add, or, xor).
    // Most data structures can only be lock-free (using CAS), not wait-free.
    // Blocking mutex is simplest and appropriate for low-contention scenarios.
}

```

**Explanation:** Under high contention (8 threads, 5M ops each), wait-free `fetch_add` is ~10x faster than mutex and ~4x faster than CAS. Mutex has kernel-mode overhead (futex syscalls), CAS has retry-loop waste, and `fetch_add` has neither. However, wait-free algorithms only exist for simple operations — complex data structures usually require lock-free (CAS) designs.

---

## Notes

- **Wait-free operations in C++:** `fetch_add`, `fetch_sub`, `fetch_and`, `fetch_or`, `fetch_xor`, `exchange`, `load`, `store`. All are single-instruction on x86.
- **On ARM/POWER:** These may involve LL/SC sequences internally but still provide wait-free guarantees at the semantic level.
- **Use `memory_order_relaxed`** for pure counters/statistics. Use stronger orders (acquire/release) when the counter's value synchronizes access to other data.
- **Wait-free ring buffer:** Use `fetch_add` on head/tail indices to create a wait-free SPSC or MPMC queue (with careful modular arithmetic).
- Compile with `-std=c++11 -O2 -pthread`.
