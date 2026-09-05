# Understand DRF-SC: Data-Race Freedom Implies Sequential Consistency

**Category:** Concurrency and Parallelism  
**Standard:** C++11 and later (Memory Model §6.9.2 [intro.races])  
**Reference:** [cppreference - Memory model](https://en.cppreference.com/w/cpp/language/memory_model)  

---

## Topic Overview

The C++ memory model is intentionally weak: it allows compilers and hardware to reorder memory accesses aggressively. This creates a vast space of possible behaviors that most programmers cannot reason about. The **DRF-SC theorem** is the escape hatch: **if your program has no data races, then every execution behaves as if all operations were sequentially consistent.** You get the simplest possible mental model for free - provided you eliminate data races.

A **data race** in C++ is defined precisely: two memory accesses to the same location by different threads, where at least one is a write and they are not ordered by a *happens-before* relationship. This is **undefined behavior** - not just non-determinism, but UB that permits the compiler to assume it never happens and optimize accordingly. The DRF-SC guarantee therefore says: eliminate UB, and the remaining nondeterminism (interleaving order) is the only complexity you face.

| Concept | Definition | How to Achieve |
| --- | --- | --- |
| **Happens-before** | Transitive closure of *sequenced-before* and *synchronizes-with* | Use mutexes, atomics, `std::call_once`, etc. |
| **Synchronizes-with** | A release operation that pairs with an acquire on the same atomic | `store(release)` paired with `load(acquire)`, `unlock` paired with `lock` |
| **Data race** | Conflicting, unordered accesses (at least one write) to a non-atomic | Bug: UB |
| **DRF-SC** | No data races -> program behaves as-if sequentially consistent | All shared mutable state accessed under proper synchronization |

Here is a diagram showing the C++ memory model hierarchy and where DRF programs sit within it:

```cpp
                    ┌─────────────────────────────────┐
                    │     C++ Memory Model Hierarchy   │
                    └─────────────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                     ▼
     Sequentially            Acquire/Release         Relaxed
      Consistent              Ordering              Ordering
    (memory_order_          (memory_order_         (memory_order_
       seq_cst)          acquire / release)          relaxed)
              │                    │                     │
              └────────┬───────────┘                     │
                       ▼                                 │
                  DRF Programs                           │
              (behave as seq_cst)                        │
                                                        ▼
                                              Expert-only: requires
                                              manual reasoning about
                                              happens-before graphs
```

DRF programs - that is, programs using mutexes, condition variables, and `seq_cst` atomics consistently - all fall into the "behaves as seq_cst" bucket regardless of which ordering they use, because they have no races. Relaxed ordering without careful proof of correctness is expert territory, and bugs there are nearly impossible to catch without dedicated tooling.

The practical implication is a **two-tier approach**: write the vast majority of your concurrent code using mutexes, condition variables, and `seq_cst` atomics - all of which establish happens-before edges and prevent data races. Only descend to weaker orderings (`acquire`/`release`, `relaxed`) in performance-critical sections where you can formally prove correctness. DRF-SC guarantees that the simple code path is correct without requiring you to reason about hardware memory models.

Why does this matter for real code? Because **compilers exploit the absence of data races**. If the compiler can prove a non-atomic variable is not accessed concurrently, it can cache it in a register, reorder it across calls, or eliminate it entirely. A data race doesn't just give you a "wrong value" - it can cause the compiler to generate code that loops infinitely, skips branches, or corrupts unrelated memory. The reason this trips people up is that on their particular machine, with their particular compiler version and optimization flags, the racy code appears to work fine. The UB is latent, waiting for a new compiler version or a slightly different runtime pattern to surface it.

---

## Self-Assessment

### Q1: Demonstrate a data race and show how adding proper synchronization restores sequentially-consistent behavior

The example below has two versions side-by-side: a racy version using plain non-atomic integers, and a fixed version using `seq_cst` atomics. The racy version is UB, so the comment describing what "can happen" is actually understating it - the compiler is free to produce any behavior at all. The safe version obeys DRF-SC.

```cpp
// Compile: g++ -std=c++20 -pthread -fsanitize=thread q1_drf_sc.cpp -o q1
// Run with ThreadSanitizer to detect the race in the first version.
#include <atomic>
#include <iostream>
#include <thread>

// ============ VERSION 1: DATA RACE (UB) ============
namespace racy {
    int x = 0;       // non-atomic, shared
    int y = 0;       // non-atomic, shared
    int r1 = 0, r2 = 0;

    void thread_a() {
        x = 1;        // W(x)
        r1 = y;       // R(y)
    }
    void thread_b() {
        y = 1;        // W(y)
        r2 = x;       // R(x)
    }
    // Both x and y have conflicting accesses with no happens-before.
    // Result r1 == 0 && r2 == 0 is possible under UB - and compilers
    // may actually produce it via store-buffer forwarding or reordering.
}

// ============ VERSION 2: DRF via seq_cst atomics ============
namespace drf {
    std::atomic<int> x{0};
    std::atomic<int> y{0};
    int r1 = 0, r2 = 0;

    void thread_a() {
        x.store(1, std::memory_order_seq_cst);    // W(x)
        r1 = y.load(std::memory_order_seq_cst);   // R(y)
    }
    void thread_b() {
        y.store(1, std::memory_order_seq_cst);     // W(y)
        r2 = x.load(std::memory_order_seq_cst);    // R(x)
    }
    // DRF-SC guarantees: at least one of r1, r2 must be 1.
    // The outcome r1 == 0 && r2 == 0 is impossible.
}

int main() {
    // Run the DRF version many times to confirm the invariant
    int both_zero = 0;
    constexpr int ITERS = 1'000'000;

    for (int i = 0; i < ITERS; ++i) {
        drf::x = 0;
        drf::y = 0;
        drf::r1 = 0;
        drf::r2 = 0;

        std::thread a(drf::thread_a);
        std::thread b(drf::thread_b);
        a.join();
        b.join();

        if (drf::r1 == 0 && drf::r2 == 0)
            ++both_zero;
    }

    std::cout << "DRF version: r1==0 && r2==0 occurred "
              << both_zero << " / " << ITERS << " times\n";
    // Expected: 0 / 1000000
    return 0;
}
```

**Key insight:** The racy version has UB - the compiler and hardware may produce any result. The DRF version uses `seq_cst` atomics, which establish happens-before edges. DRF-SC guarantees the program behaves as if all operations execute in some single total order, making `r1 == 0 && r2 == 0` impossible.

---

### Q2: Show that `acquire`/`release` atomics also provide DRF-SC for the protected non-atomic data - not just for the atomics themselves

You do not need `seq_cst` everywhere to get DRF-SC. Any mechanism that creates a happens-before edge is enough. Here, a single `acquire`/`release` pair on a boolean flag protects a whole struct of non-atomic data:

```cpp
// Compile: g++ -std=c++20 -pthread q2_acq_rel_drf.cpp -o q2
#include <atomic>
#include <cassert>
#include <iostream>
#include <thread>

struct Payload {
    int a = 0;
    int b = 0;
    int c = 0;
};

Payload data;                          // non-atomic shared state
std::atomic<bool> ready{false};        // synchronization flag

void producer() {
    // These writes to non-atomic `data` happen-before the release store
    data.a = 42;
    data.b = 100;
    data.c = data.a + data.b;
    ready.store(true, std::memory_order_release);  // <- release
}

void consumer() {
    while (!ready.load(std::memory_order_acquire))  // <- acquire
        ;  // spin

    // The acquire-load synchronizes-with the release-store.
    // All writes sequenced-before the release in the producer
    // happen-before all reads sequenced-after the acquire here.
    // Therefore, these reads of non-atomic data are NOT a data race.
    assert(data.a == 42);
    assert(data.b == 100);
    assert(data.c == 142);
    std::cout << "All assertions passed: data is consistent.\n";
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```

**Key insight:** DRF-SC does not require `seq_cst` everywhere. Any mechanism that creates happens-before edges - including `acquire`/`release` - suffices to make non-atomic accesses race-free. Once race-free, the program behaves sequentially consistently. The weaker ordering only affects how atomics interact with *each other* when multiple atomics are in play (there is no single total order guarantee with acquire/release alone).

---

### Q3: Illustrate a subtle case where `relaxed` atomics do NOT cause a data race but still permit non-sequentially-consistent outcomes for the atomics themselves

This is perhaps the most subtle point in the whole DRF-SC story. Atomic accesses can never be data races by definition - they are always well-defined. But "no data race" is not the same as "sequentially consistent." With relaxed atomics, the outcome you see can still violate any single total ordering you might imagine. DRF-SC only protects the non-atomic world.

```cpp
// Compile: g++ -std=c++20 -pthread q3_relaxed_non_sc.cpp -o q3
#include <atomic>
#include <iostream>
#include <thread>

// Store-buffer litmus test with relaxed atomics
std::atomic<int> x{0}, y{0};
int r1 = 0, r2 = 0;

void thread_a() {
    x.store(1, std::memory_order_relaxed);
    r1 = y.load(std::memory_order_relaxed);
}

void thread_b() {
    y.store(1, std::memory_order_relaxed);
    r2 = x.load(std::memory_order_relaxed);
}

int main() {
    int both_zero = 0;
    constexpr int ITERS = 1'000'000;

    for (int i = 0; i < ITERS; ++i) {
        x.store(0, std::memory_order_relaxed);
        y.store(0, std::memory_order_relaxed);
        r1 = 0;
        r2 = 0;

        std::thread a(thread_a);
        std::thread b(thread_b);
        a.join();
        b.join();

        if (r1 == 0 && r2 == 0)
            ++both_zero;
    }

    std::cout << "Relaxed atomics: r1==0 && r2==0 occurred "
              << both_zero << " / " << ITERS << " times\n";
    // On x86: likely 0 (TSO hides this). On ARM/POWER: likely > 0.
    // This is NOT a data race (atomics are never races), but it IS
    // a non-sequentially-consistent outcome. DRF-SC does not apply
    // to the ordering of atomic operations themselves under relaxed.

    std::cout << "\nNote: No data race exists (all accesses are atomic).\n"
              << "But DRF-SC guarantees SC behaviour for NON-ATOMIC data\n"
              << "only. Atomic operations under relaxed ordering can still\n"
              << "exhibit non-SC interleavings.\n";

    return 0;
}
```

**Key insight:** DRF-SC says "if there are no data races on non-atomic objects, the program behaves as-if sequentially consistent." Atomic accesses **by definition** cannot be data races (they are always well-defined). Therefore, DRF-SC does not constrain the ordering of relaxed atomic operations - they can exhibit non-SC outcomes. The guarantee applies to the *non-atomic* world: if you protect all non-atomics properly, those reads and writes behave as-if SC.

---

## Notes

- **DRF-SC is a contract:** you eliminate data races, and the compiler/hardware gives you SC semantics for non-atomic accesses. Break the contract (introduce a race) and you get UB, not just weak ordering.
- **ThreadSanitizer** (`-fsanitize=thread`) is the best practical tool for verifying DRF. It dynamically detects happens-before violations. Use it in CI.
- **`volatile` does not prevent data races.** In C++, `volatile` has no threading semantics. Only `std::atomic` and synchronization primitives create happens-before edges.
- **Benign races don't exist in C++.** Unlike Java (which has a weaker guarantee), C++ treats every data race as UB. Even "benign" patterns like racy counters or double-checked locking without atomics are formally UB and can miscompile.
- **DRF-SC across standards:** The guarantee has been part of C++ since C++11. C++20 and C++23 added new synchronization primitives (barriers, latches, `osyncstream`) but the fundamental DRF-SC theorem is unchanged.
- **Compiler fences** (`std::atomic_thread_fence`) also establish happens-before edges and can be used to achieve DRF without per-variable atomic types, though this is an advanced technique.
