# Use std::atomic_flag as the simplest lock-free primitive

**Category:** Concurrency and Parallelism  
**Standard:** C++11, C++20 (base)  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic_flag>  

---

## Topic Overview

`std::atomic_flag` is the **only** type the C++ standard guarantees to be lock-free. It holds a single boolean state (set/clear) and provides test-and-set as its core operation. Every other atomic type, including `std::atomic<bool>`, has a *probably* lock-free implementation on modern hardware - but the standard only makes the absolute guarantee for `atomic_flag`.

### API Summary

| Operation | Standard | Description |
| --- | --- | --- |
| `ATOMIC_FLAG_INIT` | C++11 | Static initializer (clear state) |
| `atomic_flag flag{}` | C++20 | Value-initialized to clear |
| `flag.test_and_set(order)` | C++11 | Set flag, return **previous** value |
| `flag.clear(order)` | C++11 | Clear the flag |
| `flag.test(order)` | C++20 | Read without modifying |
| `flag.wait(old, order)` | C++20 | Block while `flag.test() == old` |
| `flag.notify_one()` | C++20 | Wake one waiting thread |
| `flag.notify_all()` | C++20 | Wake all waiting threads |

### Key Properties

The reason `atomic_flag` is always lock-free while `atomic<bool>` is not guaranteed is that `atomic_flag` exposes the absolute minimum interface - just test-and-set and clear, which map directly to single hardware instructions that every CPU supports natively:

```cpp
atomic_flag vs atomic<bool>:
----------------------------
atomic_flag:
  Always lock-free (guaranteed by the standard)
  Minimal interface -> minimal misuse
  No load/store (until C++20 test())
  No compare_exchange

atomic<bool>:
  MAY use a mutex internally (is_lock_free() can be false)
  Full atomic interface: load, store, compare_exchange
  More flexible for general boolean flags
```

---

## Self-Assessment

### Q1: Implement a spinlock using std::atomic_flag::test_and_set with acquire/release

**Answer:**

The acquire/release pairing here is not optional. The acquire on `lock()` ensures that the protected code sees all memory written before the previous `unlock()`. The release on `unlock()` ensures that what you wrote inside the critical section is visible to the next thread that acquires the lock:

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

class SpinLock {
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT; // C++11: clear state
    //       or:  std::atomic_flag flag_{};     // C++20: value-initialized (clear)

public:
    void lock() {
        // test_and_set returns the PREVIOUS value:
        //   false -> wasn't locked, now IS locked (we acquired it)
        //   true  -> was already locked, spin again
        while (flag_.test_and_set(std::memory_order_acquire)) {
            // === Spin strategies (from worst to best) ===
            // 1. Bare spin: while (flag_.test_and_set(acquire)) {}
            //    -> Burns CPU, causes cache-line bouncing
            //
            // 2. PAUSE hint: __builtin_ia32_pause() / _mm_pause()
            //    -> Reduces pipeline flush penalty on x86
            //
            // 3. Test-then-TAS (TTAS):
            //    -> Read-only test first to avoid cache-line invalidation
            #if defined(__cpp_lib_atomic_flag_test) // C++20
            while (flag_.test(std::memory_order_relaxed)) {
                // Spin on read-only test (shared cache line)
                // Only attempt test_and_set when we see it's clear
            }
            #endif
        }
    }

    void unlock() {
        flag_.clear(std::memory_order_release);
        // release ensures all writes before unlock() are visible
        // to the next thread that acquires the lock
    }
};

// Demonstration: protect a shared counter
int main() {
    SpinLock spin;
    int counter = 0;
    constexpr int N_THREADS = 4;
    constexpr int N_ITERS = 100'000;

    std::vector<std::thread> threads;
    for (int t = 0; t < N_THREADS; ++t) {
        threads.emplace_back([&] {
            for (int i = 0; i < N_ITERS; ++i) {
                spin.lock();
                ++counter; // protected by spinlock
                spin.unlock();
            }
        });
    }
    for (auto& t : threads) t.join();

    std::cout << "Counter: " << counter
              << " (expected " << N_THREADS * N_ITERS << ")\n";

    // Output:
    // Counter: 400000 (expected 400000)
}
```

**Why acquire/release?**

- `lock()` uses `acquire`: ensures all reads/writes after lock() see data written before the previous `unlock()`.
- `unlock()` uses `release`: ensures all writes before unlock() are flushed before the lock becomes available.
- Together they form a **happens-before** chain: `unlock()` -> `lock()` guarantees visibility.

### Q2: Show why std::atomic_flag is always lock-free while std::atomic\<bool\> may not be

**Answer:**

The short explanation is that `atomic_flag`'s interface was deliberately designed to match what hardware provides unconditionally. Here's the full reasoning:

```cpp
#include <atomic>
#include <iostream>

int main() {
    // === atomic_flag: ALWAYS lock-free ===
    // The C++ standard [atomics.flag] states:
    //   "The atomic_flag type provides the classic test-and-set functionality.
    //    It has two states, set and clear.
    //    Operations on an object of type atomic_flag shall be lock-free."
    //
    // This is a NORMATIVE REQUIREMENT - not optional.

    std::atomic_flag flag{};
    // Note: atomic_flag has no is_lock_free() member, and does not need one.
    // Being lock-free is part of the type's definition, not a runtime property
    // you query - unlike std::atomic<T>, which does provide is_lock_free().
    std::cout << "atomic_flag is always lock-free: true\n";

    // === atomic<bool>: MIGHT NOT be lock-free ===
    // The standard makes NO lock-free guarantee for atomic<bool>.
    // On exotic platforms, atomic<bool> could use a mutex internally.

    std::atomic<bool> abool{false};
    std::cout << "atomic<bool> is lock-free: "
              << abool.is_lock_free() << "\n"; // usually 1, but NOT guaranteed

    // Compile-time check:
    std::cout << "atomic<bool> is_always_lock_free: "
              << std::atomic<bool>::is_always_lock_free << "\n";
    // On x86/ARM this is typically true, but on some embedded
    // platforms it could be false.

    // === WHY the difference? ===
    //
    // atomic_flag has an intentionally MINIMAL interface:
    //   - test_and_set() -> hardware TAS/XCHG instruction
    //   - clear()        -> hardware store instruction
    //
    // Every CPU ever made supports these two operations natively.
    // There is NO platform where test-and-set requires a mutex.
    //
    // atomic<bool> has a RICHER interface:
    //   - load, store, exchange, compare_exchange_weak/strong
    //   - These all map to lock-free instructions on modern CPUs
    //   - But the standard can't guarantee this for ALL architectures
    //
    // Example architecture where atomic<bool> might not be lock-free:
    //   - A hypothetical 16-bit microcontroller without CAS instruction
    //   - atomic<bool> compare_exchange needs CAS -> falls back to mutex
    //   - atomic_flag test_and_set only needs TAS -> always native

    // === Size comparison ===
    std::cout << "\nsizeof(atomic_flag):  " << sizeof(std::atomic_flag) << "\n";
    std::cout << "sizeof(atomic<bool>): " << sizeof(std::atomic<bool>) << "\n";

    // Output:
    // atomic_flag is always lock-free: 1
    // atomic<bool> is lock-free: 1
    // atomic<bool> is_always_lock_free: 1   (on x86/ARM)
    //
    // sizeof(atomic_flag):  1   (or 4 with padding)
    // sizeof(atomic<bool>): 1
}
```

### Q3: Use atomic_flag for a one-shot 'signal has fired' notification between threads

**Answer:**

`test_and_set()` has perfect "exactly once" semantics: it atomically reads the old value and sets the flag, so only one thread can ever observe the transition from clear to set. That makes it a natural primitive for one-shot events and once-flags. The C++20 `wait()`/`notify()` API adds efficient blocking on top of this:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <chrono>
#include <vector>

// === Pattern 1: One-shot signal (C++11 compatible) ===
void one_shot_signal_cpp11() {
    std::atomic_flag signal = ATOMIC_FLAG_INIT; // starts CLEAR

    std::thread producer([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << "Producer: firing signal\n";
        signal.test_and_set(std::memory_order_release); // CLEAR -> SET
    });

    // C++11: must poll (no test() or wait())
    while (!signal.test_and_set(std::memory_order_acquire)) {
        signal.clear(std::memory_order_relaxed); // wasn't set yet, clear our TAS
        std::this_thread::yield();
    }
    // Now signal is SET -> producer has fired
    std::cout << "Consumer: signal received!\n";
    producer.join();
}

// === Pattern 2: One-shot signal (C++20 - efficient) ===
void one_shot_signal_cpp20() {
    std::atomic_flag signal{}; // starts clear

    std::thread producer([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << "Producer: firing signal\n";
        signal.test_and_set(std::memory_order_release);
        signal.notify_all(); // wake all waiters
    });

    // C++20: efficient blocking wait
    signal.wait(false); // blocks while flag.test() == false (i.e., flag is clear)
    std::cout << "Consumer: signal received!\n";
    producer.join();
}

// === Pattern 3: Once-flag - ensure work runs exactly once ===
class OnceFlag {
    std::atomic_flag done_ = ATOMIC_FLAG_INIT;
public:
    // Returns true for exactly ONE caller, false for all others
    bool try_claim() {
        return !done_.test_and_set(std::memory_order_acq_rel);
        // First caller: test_and_set returns false (wasn't set) -> !false = true
        // All others:   test_and_set returns true (was set)     -> !true  = false
    }
};

void once_flag_demo() {
    OnceFlag init_flag;
    std::atomic<int> init_count{0};

    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i) {
        threads.emplace_back([&, i] {
            if (init_flag.try_claim()) {
                // Exactly one thread executes this
                init_count.fetch_add(1, std::memory_order_relaxed);
                std::cout << "Thread " << i << " performed initialization\n";
            }
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "Init ran " << init_count.load() << " time(s)\n";
}

int main() {
    std::cout << "--- C++11 one-shot ---\n";
    one_shot_signal_cpp11();

    std::cout << "\n--- C++20 one-shot ---\n";
    one_shot_signal_cpp20();

    std::cout << "\n--- Once flag ---\n";
    once_flag_demo();

    // Output:
    // --- C++11 one-shot ---
    // Producer: firing signal
    // Consumer: signal received!
    //
    // --- C++20 one-shot ---
    // Producer: firing signal
    // Consumer: signal received!
    //
    // --- Once flag ---
    // Thread 3 performed initialization
    // Init ran 1 time(s)
}
```

**Key insight:** `test_and_set()` is the perfect primitive for "exactly once" semantics - it atomically reads the old value and sets the new value, so only one thread can ever see the transition from clear to set.

---

## Notes

- **Initialization:** In C++11/14/17, use `ATOMIC_FLAG_INIT` for clear state. In C++20, value initialization `{}` gives clear state. A default-constructed `atomic_flag` without initializer has an **indeterminate** state in C++11.
- **No `operator=`:** You cannot assign to an `atomic_flag`. Use `test_and_set()` and `clear()` only.
- **Spinlock vs std::mutex:** Spinlocks are appropriate when contention is very low and critical sections are tiny (< 100ns). For anything else, `std::mutex` is better because it yields the CPU to the OS scheduler.
- **TTAS optimization:** The "test-then-test-and-set" pattern (C++20 `test()`) avoids cache-line invalidation during spinning. Read-only `test()` keeps the line in shared state; only `test_and_set()` forces exclusive ownership.
- **C++20 `wait()`/`notify()`** on `atomic_flag` makes it a lightweight event primitive - more efficient than `condition_variable` for simple signals.
- Compile with `-std=c++20 -O2 -pthread`.
