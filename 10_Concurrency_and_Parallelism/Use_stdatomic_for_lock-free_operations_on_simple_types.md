# Use std::atomic for lock-free operations on simple types

**Category:** Concurrency and Parallelism  
**Standard:** C++11, C++20, C++23 (core)  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic>  

---

## Topic Overview

`std::atomic<T>` wraps a simple type and provides **lock-free** read-modify-write operations that are safe to use from multiple threads without a mutex. The reason you need it is that even a simple `int` increment (`counter++`) is not atomic at the hardware level - it's a read, a modify, and a write, and another thread can interrupt between any of those steps.

### Core Operations

| Operation | Example | Description |
| --- | --- | --- |
| `load(order)` | `int v = x.load();` | Atomic read |
| `store(val, order)` | `x.store(42);` | Atomic write |
| `exchange(val, order)` | `int old = x.exchange(5);` | Atomic swap |
| `fetch_add(val, order)` | `int old = x.fetch_add(1);` | Atomic increment, returns old |
| `fetch_sub(val, order)` | `int old = x.fetch_sub(1);` | Atomic decrement, returns old |
| `compare_exchange_weak` | `x.compare_exchange_weak(exp, des);` | CAS (may spuriously fail) |
| `compare_exchange_strong` | `x.compare_exchange_strong(exp, des);` | CAS (no spurious failure) |

### Memory Orders

Think of these as a dial from "cheap but minimal guarantees" to "expensive but total guarantees":

```cpp
Weakest                                          Strongest
---------------------------------------------------------
relaxed          acquire/release          seq_cst
                 consume (deprecated)

relaxed:   no ordering - only atomicity guaranteed
acquire:   reads after this see writes from the releasing thread
release:   writes before this are visible to the acquiring thread
seq_cst:   total global order - all threads agree on sequence
```

---

## Self-Assessment

### Q1: Implement a lock-free reference counter using std::atomic\<int\> with fetch_add

**Answer:**

Reference counting is one of the classic uses of atomics. The key asymmetry is that `add_ref()` only needs relaxed ordering (you're just bumping a number), but `release()` needs `acq_rel` because the last release must be sure it sees all previous modifications to the object before deleting it:

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

// Intrusive reference-counted object
class RefCounted {
    std::atomic<int> ref_count_{1}; // starts at 1 (creator holds a ref)

public:
    void add_ref() {
        // relaxed: only need atomicity, not ordering
        // Other threads don't need to see our data yet
        int old = ref_count_.fetch_add(1, std::memory_order_relaxed);
        // old = previous count (must be > 0 if we hold a ref)
    }

    void release() {
        // fetch_sub with acq_rel:
        //   - release: ensure all writes TO the object happen before
        //     the ref count drops (so the deleting thread sees them)
        //   - acquire (on the thread that gets 1): ensure we see all
        //     previous writes before we delete
        int old = ref_count_.fetch_sub(1, std::memory_order_acq_rel);

        if (old == 1) {
            // We were the last reference (old was 1, now it's 0)
            // acq_rel ensures all other threads' writes are visible
            delete this;
        }
    }

    int use_count() const {
        return ref_count_.load(std::memory_order_relaxed);
    }
};

int main() {
    auto* obj = new RefCounted();
    std::cout << "Initial ref count: " << obj->use_count() << "\n";

    std::vector<std::thread> threads;
    std::atomic<int> work_done{0};

    for (int i = 0; i < 4; ++i) {
        obj->add_ref(); // increment BEFORE spawning thread
        threads.emplace_back([obj, &work_done] {
            // Use the object...
            work_done.fetch_add(1, std::memory_order_relaxed);
            // Release our reference
            obj->release();
        });
    }

    std::cout << "Ref count after spawning: " << obj->use_count() << "\n";

    for (auto& t : threads) t.join();

    std::cout << "Work done: " << work_done.load() << "\n";
    // Release the original reference
    obj->release(); // may delete obj here

    // Output:
    // Initial ref count: 1
    // Ref count after spawning: 5
    // Work done: 4
}
```

**Key insight:** `fetch_add` returns the **previous** value and atomically increments. For reference counting, the critical path is `release()`: `fetch_sub` returns the old value, and if it was 1, the count just hit 0 - safe to delete.

### Q2: Explain the difference between std::memory_order_seq_cst, relaxed, and acquire/release

**Answer:**

Here's a side-by-side walkthrough of all three. The relaxed example shows the most surprising behavior - two threads can each see stale values simultaneously, which is legal with relaxed ordering:

```cpp
MEMORY ORDER COMPARISON
=======================

relaxed:
  Thread 1:              Thread 2:
  x.store(1, relaxed);   y.store(1, relaxed);
  int a = y.load(relax); int b = x.load(relax);

  Possible: a==0 && b==0 <- BOTH threads see stale values!
  Only guarantee: atomicity (no torn reads/writes)
  Use for: counters, statistics - where ordering doesn't matter

acquire/release:
  Thread 1 (producer):          Thread 2 (consumer):
  data = 42;                    while(!flag.load(acquire)) {}
  flag.store(true, release);    assert(data == 42); // GUARANTEED!
  --- release ---                --- acquire ----
       all writes before              all reads after
       release are visible            acquire see those writes

  Pairing: store(release) -> load(acquire) creates happens-before
  Use for: producer/consumer, publish patterns

seq_cst (default):
  Thread 1:               Thread 2:
  x.store(1);             y.store(1);

  Thread 3:               Thread 4:
  if(x.load()==1          if(y.load()==1
     && y.load()==0)         && x.load()==0)

  IMPOSSIBLE for T3 && T4 to both observe their condition as true
  seq_cst gives a SINGLE TOTAL ORDER that all threads agree on
  Use for: when multiple atomics must appear ordered to all observers
  Cost: most expensive - often requires MFENCE on x86, DMB on ARM
```

Here's the producer/consumer pattern in code to make acquire/release concrete:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <cassert>

// Demonstration: acquire/release synchronizes data
int main() {
    int data = 0;                          // non-atomic
    std::atomic<bool> ready{false};        // synchronization flag

    std::thread producer([&] {
        data = 42;                         // write data first
        ready.store(true, std::memory_order_release);
        // release: flushes data=42 to be visible
    });

    std::thread consumer([&] {
        while (!ready.load(std::memory_order_acquire)) {}
        // acquire: "imports" all writes before the matching release
        assert(data == 42); // GUARANTEED: acquire sees the released data
        std::cout << "data = " << data << "\n";
    });

    producer.join();
    consumer.join();
    // Output: data = 42

    // If we used relaxed instead of acquire/release:
    //   data might be read as 0 (data race -> undefined behavior)
}
```

### Q3: Show a bug where two relaxed atomic stores cause unexpected read ordering on another thread

**Answer:**

This is the canonical "store buffer reordering" bug. Program order says `x` is stored before `y`, but relaxed ordering means another thread is allowed to observe `y==1` before `x==1`. On x86 this particular pattern doesn't manifest because of x86's strong memory model, but it's still undefined behavior and *will* fail on ARM:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

// THE BUG: Relaxed stores can be observed in ANY order by other threads

std::atomic<int> x{0}, y{0};

// Thread 1: stores x then y
void writer() {
    x.store(1, std::memory_order_relaxed);
    y.store(1, std::memory_order_relaxed);
    // With relaxed: the CPU/compiler may reorder these stores
    // Other threads might see y==1 BEFORE x==1
}

// Thread 2: reads y then x
void reader() {
    while (y.load(std::memory_order_relaxed) != 1) {}
    // y is 1 - but can x still be 0?
    int xval = x.load(std::memory_order_relaxed);

    if (xval == 0) {
        // BUG! We see y=1 (set after x=1 in program order)
        // but x=0 (the store to x hasn't propagated yet)
        std::cout << "BUG: y=1 but x=" << xval << "!\n";
    } else {
        std::cout << "OK: y=1 and x=" << xval << "\n";
    }
}

int main() {
    // Run many times to trigger the reordering
    for (int trial = 0; trial < 100'000; ++trial) {
        x.store(0, std::memory_order_relaxed);
        y.store(0, std::memory_order_relaxed);

        std::thread t1(writer);
        std::thread t2(reader);
        t1.join();
        t2.join();
    }
    // On ARM/POWER, you may see "BUG" lines
    // On x86, stores are ordered (TSO), so the bug won't manifest
    // but the code is STILL INCORRECT per the C++ standard

    // === FIX: use release/acquire ===
    // writer:  x.store(1, release); y.store(1, release);
    // reader:  while(y.load(acquire)!=1){} -> x.load(acquire) guaranteed 1
    //
    // Or simpler: use seq_cst (the default)
}
```

**Why this happens:**

- `relaxed` only guarantees atomicity (no torn reads), NOT ordering between different atomics.
- On weakly-ordered CPUs (ARM, POWER), stores to `x` and `y` can become visible to other cores in any order.
- The fix is `release` on the writer's stores and `acquire` on the reader's loads - this creates a happens-before chain.

---

## Notes

- **Lock-free check:** `std::atomic<T>::is_always_lock_free` (compile-time) or `x.is_lock_free()` (runtime). On x86-64, types up to 8 bytes are typically lock-free; 16-byte atomics may require `cmpxchg16b`.
- **Padding:** `std::atomic<T>` may be larger than `T` due to alignment requirements.
- **`fetch_add` vs `++`:** `counter++` on an atomic uses `seq_cst` by default. `counter.fetch_add(1, relaxed)` is cheaper if you don't need ordering.
- **`compare_exchange_weak` vs `_strong`:** `weak` can fail spuriously (returns false even when value matches expected). Use `weak` in CAS loops (slightly faster), `strong` in single-attempt checks.
- **Don't mix atomic and non-atomic access:** Accessing `x.load()` and `*(int*)&x` is a data race - undefined behavior.
- Compile with `-std=c++17 -O2 -pthread`.
