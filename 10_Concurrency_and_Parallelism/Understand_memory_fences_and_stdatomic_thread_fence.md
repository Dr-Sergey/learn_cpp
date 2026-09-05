# Understand memory fences and std::atomic_thread_fence

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence>  

---

## Topic Overview

A **memory fence** (also called a barrier) is an ordering constraint that applies to *all* memory operations across it, not just a single atomic variable. This is the key distinction from per-variable memory ordering: when you put `memory_order_release` on a single `store`, you are ordering *that one store* relative to prior operations. A release fence, by contrast, orders *all* prior stores relative to subsequent operations in the same thread.

`std::atomic_thread_fence` lets you place these ordering constraints without tying them to any specific atomic object.

### Fence vs Per-Variable Memory Order

Here is the essential contrast:

```cpp
Per-variable order:                    Fence:
  x.store(1, release);                  x.store(1, relaxed);
  // orders THIS store                  atomic_thread_fence(release);
  // relative to                        y.store(1, relaxed);
  // prior operations                   // orders ALL prior stores
                                        // relative to ALL subsequent stores
```

With per-variable ordering, each store is individually annotated. With a fence, one instruction covers everything above or below it in program order.

### Fence Types

| Fence | Effect | Use Case |
| --- | --- | --- |
| `acquire` | Prevents reads/writes AFTER the fence from moving BEFORE it | Consumer side: read flag, fence, read data |
| `release` | Prevents reads/writes BEFORE the fence from moving AFTER it | Producer side: write data, fence, write flag |
| `acq_rel` | Both acquire and release | Read-modify-write patterns |
| `seq_cst` | Total order across all threads | Rarely needed with fences |

### When Fences Are Useful

- **Batching:** One fence can order multiple relaxed operations (cheaper than per-operation ordering on some architectures)
- **Separation of concerns:** Data access code uses relaxed; synchronization logic adds fences separately
- **Legacy/hardware interop:** Maps directly to hardware fence instructions (`DMB` on ARM, `LWSYNC` on POWER)

---

## Self-Assessment

### Q1: Show that atomic_thread_fence(acquire) applies to all prior relaxed loads in the current thread

**Answer:**

The power of a fence is that it covers everything, not just one variable. In this example, a single release fence orders three separate non-atomic writes, and a single acquire fence on the reader side guarantees all three are visible. No per-variable annotation needed:

```cpp
#include <atomic>
#include <thread>
#include <cassert>
#include <iostream>

// A fence applies to ALL memory operations, not just one variable
std::atomic<int> flag{0};
int data1 = 0;  // non-atomic
int data2 = 0;  // non-atomic
int data3 = 0;  // non-atomic

void producer() {
    // Write multiple pieces of data
    data1 = 10;
    data2 = 20;
    data3 = 30;

    // Release fence: ensures ALL stores above are visible
    // before the flag store below
    std::atomic_thread_fence(std::memory_order_release);

    flag.store(1, std::memory_order_relaxed);
    //         ^^^^^^^ relaxed is sufficient - the fence does the ordering
}

void consumer() {
    // Spin until we see the flag
    while (flag.load(std::memory_order_relaxed) != 1)
        ;
    //   ^^^^^^^ relaxed - fence below handles ordering

    // Acquire fence: ensures ALL loads below see the stores
    // that happened before the producer's release fence
    std::atomic_thread_fence(std::memory_order_acquire);

    // ALL three non-atomic reads are guaranteed to see
    // the producer's writes (data1=10, data2=20, data3=30)
    assert(data1 == 10);  // guaranteed
    assert(data2 == 20);  // guaranteed
    assert(data3 == 30);  // guaranteed
    std::cout << "All data visible: " << data1 << ", "
              << data2 << ", " << data3 << "\n";
    // Output: All data visible: 10, 20, 30
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();

    // === KEY DIFFERENCE FROM PER-VARIABLE ORDER ===
    //
    // With per-variable order, you'd need:
    //   flag.store(1, memory_order_release);  // orders only stores BEFORE this store
    //   flag.load(memory_order_acquire);       // orders only loads AFTER this load
    //
    // This is EQUIVALENT for one flag variable. But with a FENCE:
    //   All relaxed stores before the release fence are ordered
    //   All relaxed loads after the acquire fence are ordered
    //   This includes stores/loads to OTHER atomic variables!
    //
    // Proof: if we had flag2, flag3, etc., a single fence
    // would cover ALL of them, whereas per-variable ordering
    // would need acquire/release on EACH variable.
}
```

**Explanation:** The release fence orders *all* prior stores (data1, data2, data3) relative to the subsequent relaxed store of the flag. The acquire fence orders *all* subsequent loads relative to the prior relaxed load of the flag. One fence covers multiple variables - this is the key advantage over per-variable memory ordering.

### Q2: Explain the difference between a fence and an acquire load on a single atomic variable

**Answer:**

The difference is scope. An acquire load on `x` only creates ordering with respect to `x` specifically - it does not say anything about other atomic variables. A fence, placed after any number of relaxed loads, creates ordering for all of them at once. The diagram below makes this concrete:

```cpp
Acquire LOAD on a variable:           Acquire FENCE:
─────────────────────────────         ─────────────────
auto val = x.load(acquire);           auto val = x.load(relaxed);
// Only operations AFTER this         auto y = flag.load(relaxed);
// load are ordered with              atomic_thread_fence(acquire);
// respect to THIS load of x.         // ALL operations below are
// Does NOT affect other              // ordered with respect to
// atomic variables.                  // ALL relaxed loads above.

┌─────────────────────┐               ┌─────────────────────┐
│ x.load(acquire)     │               │ x.load(relaxed)     │
│                     │               │ y.load(relaxed)      │
│ ══════barrier═══════│               │ z.load(relaxed)      │
│ ops after HERE      │               │ ══════barrier════════│
│   are ordered       │               │ ALL ops below        │
│   w.r.t. x only    │               │   are ordered w.r.t. │
└─────────────────────┘               │   ALL loads above    │
                                      └─────────────────────┘
```

**Key differences:**

| Aspect | Acquire load (`x.load(acquire)`) | Acquire fence (`atomic_thread_fence(acquire)`) |
| --- | --- | --- |
| Scope | Orders only w.r.t. `x` | Orders w.r.t. ALL prior relaxed loads |
| Performance | Often free on x86 (loads are acquire by default) | Explicit barrier instruction on ARM |
| Use case | Single synchronization variable | Multiple atomic variables need ordering |
| Composability | One per variable | One fence covers everything |

**When to use a fence:** When you have multiple atomic variables that all need acquire/release semantics, a single fence is simpler and potentially cheaper than putting `memory_order_acquire` on each load individually.

**When per-variable is better:** When you synchronize on just one variable (the common case), per-variable ordering is clearer and has zero extra cost on x86.

### Q3: Write a lock implementation using only fences and atomic_flag for educational purposes

**Answer:**

This spinlock shows exactly where each fence goes and why. The acquire fence after locking ensures the critical section sees the previous holder's writes. The release fence before unlocking ensures this holder's writes are visible to the next acquirer:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>

class FenceSpinLock {
    std::atomic_flag locked_ = ATOMIC_FLAG_INIT; // false initially

public:
    void lock() {
        // Spin until we successfully set the flag
        while (locked_.test_and_set(std::memory_order_relaxed)) {
            // busy wait - flag was already set (someone else holds the lock)
        }
        // We acquired the lock (test_and_set returned false = was unlocked)

        // ACQUIRE fence: ensures all memory operations AFTER this point
        // see the stores made by the previous lock holder before their
        // release fence
        std::atomic_thread_fence(std::memory_order_acquire);
    }

    void unlock() {
        // RELEASE fence: ensures all memory operations BEFORE this point
        // are visible to the next thread that acquires the lock
        std::atomic_thread_fence(std::memory_order_release);

        // Clear the flag - lock is now available
        locked_.clear(std::memory_order_relaxed);
        // relaxed is fine - the fence above already ordered everything
    }
};

// === Test: protect shared data ===
FenceSpinLock spin;
int shared_counter = 0; // non-atomic, protected by spinlock

void increment(int times) {
    for (int i = 0; i < times; ++i) {
        spin.lock();
        ++shared_counter; // safe: only one thread at a time
        spin.unlock();
    }
}

int main() {
    constexpr int THREADS = 8;
    constexpr int ITERS = 100'000;

    std::vector<std::thread> threads;
    for (int t = 0; t < THREADS; ++t)
        threads.emplace_back(increment, ITERS);
    for (auto& t : threads) t.join();

    std::cout << "Counter: " << shared_counter << "\n";
    std::cout << "Expected: " << THREADS * ITERS << "\n";
    // Output:
    // Counter: 800000
    // Expected: 800000

    // === EQUIVALENT without fences (per-operation ordering): ===
    // lock:   while (flag.test_and_set(memory_order_acquire)) {}
    // unlock: flag.clear(memory_order_release);
    //
    // The fence version separates the ordering concern from the
    // atomic operation, which can be useful when the lock protects
    // multiple data structures (the fence covers everything).
}
```

**Explanation:** The acquire fence after `test_and_set` ensures all subsequent reads in the critical section see the previous holder's writes. The release fence before `clear` ensures all writes in the critical section are visible to the next acquirer. Using fences with relaxed atomics is equivalent to using acquire/release on the atomic operations directly.

---

## Notes

- **On x86:** Fences often compile to no-ops because x86 has a strong memory model (loads are acquire, stores are release by default). Only `seq_cst` fences emit `MFENCE`.
- **On ARM/POWER:** Fences compile to `DMB` (ARM) or `LWSYNC`/`HWSYNC` (POWER) instructions - real cost.
- **`std::atomic_thread_fence` vs `std::atomic_signal_fence`:** Thread fences order between threads; signal fences order between a thread and a signal handler in the same thread.
- **Rule of thumb:** Prefer per-variable ordering (`load(acquire)`, `store(release)`) for clarity. Use fences when you need to batch ordering across multiple variables.
- Compile with `-std=c++11 -O2 -pthread`.
