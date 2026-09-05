# Use relaxed atomics correctly for counters and statistics

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/memory_order>  

---

## Topic Overview

`memory_order_relaxed` provides **atomicity** (no torn reads/writes) but **no ordering guarantees** relative to other memory operations. This makes it the cheapest memory order - ideal for counters and statistics where you only need the final aggregate value, not real-time consistency with other data. The mental model is: relaxed says "this operation itself won't be torn," but says nothing about whether other memory operations before or after it are visible to other threads.

### When Relaxed Is Safe

If the table feels dense, the rule boils down to this: relaxed is safe when the atomic variable stands alone - when no other data's correctness depends on seeing the atomic value at a particular moment. Counters are perfect for this. A flag that signals "my data is ready for you to read" is not.

```cpp
SAFE (only need atomicity):                UNSAFE (need ordering):
-----------------------------------------  --------------------------
Hit counters / statistics                  Flag to signal "data ready"
Unique ID generation (fetch_add)           Reference counts (last decrement)
Progress indicators                        Publishing a pointer to new data
Approximate size queries                   Lock/unlock implementations
```

### Relaxed vs Stronger Orders

| Operation | relaxed | acquire/release | seq_cst |
| --- | --- | --- | --- |
| Atomicity | Yes | Yes | Yes |
| Ordering with other ops | No | Paired ops only | Total order |
| Cost on x86 | Free | Free | `MFENCE` |
| Cost on ARM | Free | `ldar`/`stlr` | `DMB ISH` |
| Use case | Counters | Producer-consumer | Global coordination |

---

## Self-Assessment

### Q1: Use memory_order_relaxed for a hit counter that only needs eventual consistency

**Answer:**

Notice that every `fetch_add` here uses relaxed. Each individual increment is atomic, so no update is ever lost. But reading all the counters at once isn't a consistent snapshot - `page_views` might be from time T and `api_calls` from time T+1. For statistics dashboards, that approximation is perfectly fine:

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

class StatsCollector {
    std::atomic<long long> page_views_{0};
    std::atomic<long long> api_calls_{0};
    std::atomic<long long> errors_{0};
    std::atomic<long long> bytes_sent_{0};

public:
    // All increments use relaxed - we don't need ordering with other data
    void record_page_view() {
        page_views_.fetch_add(1, std::memory_order_relaxed);
    }
    void record_api_call() {
        api_calls_.fetch_add(1, std::memory_order_relaxed);
    }
    void record_error() {
        errors_.fetch_add(1, std::memory_order_relaxed);
    }
    void record_bytes(long long n) {
        bytes_sent_.fetch_add(n, std::memory_order_relaxed);
    }

    // Read with relaxed - values are approximate but eventually consistent
    void print_stats() const {
        std::cout << "Page views: " << page_views_.load(std::memory_order_relaxed)
                  << "\nAPI calls:  " << api_calls_.load(std::memory_order_relaxed)
                  << "\nErrors:     " << errors_.load(std::memory_order_relaxed)
                  << "\nBytes sent: " << bytes_sent_.load(std::memory_order_relaxed)
                  << "\n";
    }

    long long total_page_views() const {
        return page_views_.load(std::memory_order_relaxed);
    }
};

int main() {
    StatsCollector stats;
    constexpr int THREADS = 8;
    constexpr int OPS = 1'000'000;

    auto start = std::chrono::steady_clock::now();

    std::vector<std::thread> threads;
    for (int t = 0; t < THREADS; ++t) {
        threads.emplace_back([&] {
            for (int i = 0; i < OPS; ++i) {
                stats.record_page_view();
                if (i % 10 == 0) stats.record_api_call();
                if (i % 100 == 0) stats.record_error();
                stats.record_bytes(256);
            }
        });
    }
    for (auto& t : threads) t.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    stats.print_stats();
    std::cout << "Time: " << ms << " ms\n";
    // Output:
    // Page views: 8000000    (exact - fetch_add is atomic)
    // API calls:  800000
    // Errors:     80000
    // Bytes sent: 2048000000
    // Time: ~50 ms

    // Note: each INDIVIDUAL counter is exact (atomic increment).
    // But reading ALL counters is NOT a consistent snapshot -
    // page_views might be from time T and api_calls from time T+1.
    // For statistics, this "approximate" snapshot is perfectly fine.
}
```

**Explanation:** Relaxed atomics are perfect for counters because: (1) each increment is individually atomic (no lost updates), (2) the final value after joining all threads is exact, (3) mid-run reads give an approximate-but-useful snapshot, (4) zero synchronization overhead on all architectures.

### Q2: Explain why relaxed ordering is safe for a monotonically increasing counter read by one thread

**Answer:**

There are a few guarantees that make relaxed safe for counters, and it's worth being explicit about each one:

```text
WHY RELAXED IS SAFE FOR COUNTERS:
==================================

1. ATOMICITY guarantees:
   - fetch_add is indivisible - no thread sees a partial update
   - Two concurrent fetch_add(1) always produce count+2, never count+1

2. NO ordering needed because:
   - The counter value doesn't "protect" other data
   - Reading counter=5000 doesn't mean anything about what other
     variables look like - and we don't need it to

3. For monotonically increasing counters:
   - Thread A: counter.fetch_add(1, relaxed) -> returns 0
   - Thread A: counter.fetch_add(1, relaxed) -> returns 1 or higher

   Within a SINGLE thread, relaxed operations on the SAME atomic
   are guaranteed to be consistent (modification order coherence).
   A thread never sees an "older" value after seeing a "newer" one
   on the same variable.

4. After thread::join():
   - join() provides a happens-before relationship
   - All relaxed increments are guaranteed visible to the joining thread

   Thread 1: counter += 1M (relaxed)
   Thread 1 completes -> join() in main
   Main: counter.load(relaxed) -> exactly 1M

WHAT RELAXED DOES NOT GUARANTEE (between threads, mid-execution):
   Thread A: counter.store(42, relaxed)
   Thread B: counter.load(relaxed) -> might still see 0!
   (But eventually will see 42 - hardware caches propagate)
```

Here's a short code example that exercises these properties:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <cassert>

int main() {
    std::atomic<int> counter{0};

    // Within ONE thread: relaxed modifications are always consistent
    std::thread t([&] {
        counter.fetch_add(1, std::memory_order_relaxed); // 0->1
        counter.fetch_add(1, std::memory_order_relaxed); // 1->2
        counter.fetch_add(1, std::memory_order_relaxed); // 2->3

        // A single thread always sees its own relaxed writes in order
        int val = counter.load(std::memory_order_relaxed);
        assert(val >= 3); // guaranteed (might be higher if other threads add)
    });
    t.join();

    // After join: happens-before guarantees all relaxed ops visible
    assert(counter.load(std::memory_order_relaxed) >= 3); // guaranteed
    std::cout << "Counter: " << counter.load() << "\n";
}
```

### Q3: Show a case where relaxed ordering is insufficient and acquire/release is required

**Answer:**

Here's the classic mistake: using relaxed on a flag that signals "my data is ready." The flag increment is atomic, but there's no ordering - the consumer might observe the flag as set before seeing the data that was supposed to precede it:

```cpp
#include <atomic>
#include <thread>
#include <cassert>
#include <iostream>

// === BROKEN: relaxed ordering for data publication ===
std::atomic<bool> ready{false};
int data = 0; // non-atomic

void broken_producer() {
    data = 42;
    ready.store(true, std::memory_order_relaxed); // <- BUG!
}

void broken_consumer() {
    while (!ready.load(std::memory_order_relaxed))
        ;
    // BUG: data might still be 0!
    // relaxed load of 'ready' does NOT guarantee we see data=42
    std::cout << "Broken: data = " << data << "\n"; // might be 0!
}

// === FIXED: acquire/release ordering ===
std::atomic<bool> ready2{false};
int data2 = 0;

void fixed_producer() {
    data2 = 42;
    ready2.store(true, std::memory_order_release); // <- RELEASE
    // release: all writes before this store (data2=42) are
    // guaranteed visible after a paired acquire load
}

void fixed_consumer() {
    while (!ready2.load(std::memory_order_acquire)) // <- ACQUIRE
        ;
    // acquire: all writes before the paired release store
    // are guaranteed visible here
    assert(data2 == 42); // GUARANTEED
    std::cout << "Fixed: data = " << data2 << "\n"; // always 42
}

int main() {
    // Broken version (UB on ARM, might work on x86 by luck)
    {
        std::thread t1(broken_producer);
        std::thread t2(broken_consumer);
        t1.join(); t2.join();
    }

    // Fixed version (correct on all architectures)
    {
        std::thread t1(fixed_producer);
        std::thread t2(fixed_consumer);
        t1.join(); t2.join();
    }

    // RULE: Use relaxed ONLY when the atomic variable is independent.
    //       Use acquire/release when the atomic coordinates access
    //       to other (non-atomic) data.
    //
    // If atomic SIGNALS something about other data:
    //   -> acquire/release (or seq_cst)
    // If atomic is STANDALONE (counter, ID, stat):
    //   -> relaxed is fine
}
```

**Explanation:** When an atomic variable acts as a *signal* that non-atomic data is ready, you need ordering: release ensures the data is committed before the signal, and acquire ensures the reader sees the data after seeing the signal. Relaxed only provides atomicity on the flag itself - it says nothing about when other memory operations become visible.

---

## Notes

- **Performance difference:** On x86, relaxed vs acquire/release generates identical code (x86's hardware model provides acquire/release for free). On ARM, relaxed avoids `ldar`/`stlr` instructions.
- **Relaxed is always safe for:** `fetch_add` counters, `exchange` for unique IDs, `store`/`load` for progress reporting where approximate values are acceptable.
- **`memory_order_relaxed` is NOT `volatile`:** Relaxed atomics can still be optimized (dead store elimination, etc.). They guarantee atomicity, not "every access hits RAM."
- **After `join()`:** All memory operations from a joined thread are visible, regardless of memory order used during the thread's execution.
- **Profile first:** Don't use relaxed ordering "for performance" without measurement. On x86, there's usually zero difference from seq_cst.
- Compile with `-std=c++11 -O2 -pthread`.
