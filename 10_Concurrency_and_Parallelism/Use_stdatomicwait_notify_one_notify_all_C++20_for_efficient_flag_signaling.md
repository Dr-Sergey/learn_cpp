# Use std::atomic::wait, notify_one, notify_all (C++20) for efficient flag signaling

**Category:** Concurrency and Parallelism  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic/wait>  

---

## Topic Overview

This file focuses on **practical flag-signaling patterns** using C++20 atomic wait/notify - multi-phase gates, state machines, and replacing polling loops. Think of it as the hands-on companion to the latency-comparison file: here we look at how these primitives actually slot into real coordination patterns.

### Flag Signaling Patterns

Atomic wait/notify is ideal for signaling discrete state transitions. The producer stores a new value and fires a notification; the consumer blocks until the value no longer matches what it expected:

```cpp
Thread A (producer):                Thread B (consumer):
do_work();                          state.wait(IDLE);    // block
state.store(READY);                 // ... wakes when state != IDLE
state.notify_all();                 process_result();
```

### Multi-Value State Machine

Unlike `condition_variable`, `atomic::wait(old)` can discriminate on **specific values**. You can have a thread that waits only for a particular phase, not just for "any change":

```cpp
enum class Phase { Init, Ready, Running, Done };
std::atomic<Phase> phase{Phase::Init};

// Wait for a SPECIFIC phase:
phase.wait(Phase::Init);    // blocks only while phase == Init
phase.wait(Phase::Ready);   // blocks only while phase == Ready
```

This makes multi-phase coordination code extremely readable - each wait call says exactly what the thread is blocked on.

---

## Self-Assessment

### Q1: Use atomic::wait to block a thread until the atomic value changes, without a condition variable

**Answer:**

The worker below drives through a three-phase pipeline by blocking on each current phase in turn. Notice how clean the state machine reads compared to a condition variable version that would need a mutex, a predicate, and a separate bool per phase.

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <chrono>

// A multi-phase pipeline using atomic flag signaling

enum class State : int { Idle = 0, Phase1 = 1, Phase2 = 2, Done = 3 };

int main() {
    std::atomic<State> state{State::Idle};
    int shared_data = 0;

    // Worker waits for each phase transition
    std::thread worker([&] {
        // Wait for Phase1
        state.wait(State::Idle);
        std::cout << "Worker: entered Phase1, data = " << shared_data << "\n";
        shared_data *= 2;

        // Wait for Phase2
        state.wait(State::Phase1);
        std::cout << "Worker: entered Phase2, data = " << shared_data << "\n";
        shared_data += 100;

        // Wait for Done
        state.wait(State::Phase2);
        std::cout << "Worker: Done, final data = " << shared_data << "\n";
    });

    // Controller drives state transitions
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    shared_data = 21;
    state.store(State::Phase1);
    state.notify_one(); // worker unblocks from wait(Idle)

    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    state.store(State::Phase2);
    state.notify_one(); // worker unblocks from wait(Phase1)

    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    state.store(State::Done);
    state.notify_one(); // worker unblocks from wait(Phase2)

    worker.join();

    // Output:
    // Worker: entered Phase1, data = 21
    // Worker: entered Phase2, data = 42
    // Worker: Done, final data = 142
}
```

`state.wait(State::Idle)` blocks until `state` is no longer `Idle`. When the controller stores `Phase1` and calls `notify_one()`, the worker unblocks. Each subsequent `wait()` targets the current phase, creating a clean state machine without any mutex or condition variable.

### Q2: Replace a polling loop with atomic::wait for flag signaling

**Answer:**

The polling approach is tempting because it is simple - just check a flag in a loop. The problem is that it wastes CPU between checks and adds latency proportional to the sleep interval. The batch gate at the end shows how `notify_all` can release many threads simultaneously, which polling cannot do efficiently.

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

// BAD: Polling loop (wastes CPU and has ~1ms latency gap)
void polling_approach() {
    std::atomic<bool> flag{false};
    auto t0 = std::chrono::steady_clock::now();

    std::thread signaler([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        flag.store(true, std::memory_order_release);
    });

    // Polls every 1ms - wastes CPU + up to 1ms late
    while (!flag.load(std::memory_order_acquire)) {
        std::this_thread::sleep_for(std::chrono::milliseconds(1)); // problematic
    }

    auto dt = std::chrono::steady_clock::now() - t0;
    std::cout << "Polling: woke after "
              << std::chrono::duration_cast<std::chrono::microseconds>(dt).count()
              << " us\n";
    signaler.join();
}

// GOOD: atomic::wait (zero CPU, instant wake)
void wait_approach() {
    std::atomic<bool> flag{false};
    auto t0 = std::chrono::steady_clock::now();

    std::thread signaler([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        flag.store(true, std::memory_order_release);
        flag.notify_one(); // wake the waiter immediately
    });

    flag.wait(false); // blocks with 0% CPU until flag != false

    auto dt = std::chrono::steady_clock::now() - t0;
    std::cout << "Wait:    woke after "
              << std::chrono::duration_cast<std::chrono::microseconds>(dt).count()
              << " us\n";
    signaler.join();
}

// === Batch flag: signal N workers to start ===
void batch_start_gate() {
    constexpr int N = 8;
    std::atomic<int> go{0}; // 0 = not started, 1 = start!
    std::atomic<int> finished{0};

    std::vector<std::thread> workers;
    for (int i = 0; i < N; ++i) {
        workers.emplace_back([&, i] {
            go.wait(0);  // all workers block here
            // ... do work ...
            finished.fetch_add(1, std::memory_order_relaxed);
        });
    }

    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    std::cout << "Releasing " << N << " workers...\n";
    go.store(1);
    go.notify_all(); // wake all workers simultaneously

    for (auto& w : workers) w.join();
    std::cout << "All " << finished.load() << " workers done\n";
}

int main() {
    polling_approach();
    wait_approach();
    batch_start_gate();

    // Output:
    // Polling: woke after 51200 us   (up to 1ms late)
    // Wait:    woke after 50100 us   (nearly instant wake)
    // Releasing 8 workers...
    // All 8 workers done
}
```

The polling loop sleeps in 1ms intervals, introducing up to 1ms of latency after the flag is set and burning cycles for each check. `atomic::wait` blocks efficiently (using OS futex/WaitOnAddress) and wakes within microseconds of the notification. The batch start gate pattern shows `notify_all()` releasing multiple workers simultaneously.

### Q3: Explain how atomic::wait is more efficient than a mutex+condition_variable for simple flags

**Answer:**

The overhead breakdown below shows where the extra cost of `condition_variable` comes from. It is not that `condition_variable` is poorly implemented - it is that it carries the cost of mutex operations in addition to the OS sleep/wake, and the mutex machinery adds up.

```cpp
OVERHEAD BREAKDOWN: mutex + condition_variable vs atomic::wait

mutex + condition_variable (signal one flag):
  Notifier side:                   Waiter side:

  1. Lock mutex         ~20-40ns   1. Lock mutex        ~20-40ns
  2. Set flag            ~1ns      2. Check predicate    ~1ns
  3. Unlock mutex       ~20-40ns   3. Unlock + sleep    ~100ns
  4. cv.notify_one()   ~200-500ns  4. OS wake           ~1-5us
                                   5. Re-lock mutex     ~20-40ns

  Total notifier: ~250-580ns       Total waiter: ~1.2-5.1us

  Extra objects: std::mutex (40-64 bytes) + std::condition_variable (48-72 bytes)
  Syscalls: lock/unlock + futex_wake + futex_wait

atomic::wait (signal one flag):
  Notifier side:                   Waiter side:

  1. Atomic store        ~5ns      1. Compare value      ~5ns
  2. notify_one()     ~100-300ns   2. OS sleep          ~100ns
                                   3. OS wake           ~1-3us

  Total notifier: ~105-305ns       Total waiter: ~1-3us

  Extra objects: just the atomic itself (1-8 bytes)
  Syscalls: futex_wake + futex_wait (no lock/unlock)
```

```cpp
#include <atomic>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <chrono>

// Side-by-side: code complexity comparison

// === condition_variable approach (verbose) ===
class FlagCV {
    std::mutex mtx_;
    std::condition_variable cv_;
    bool ready_ = false;
public:
    void set() {
        {
            std::lock_guard lock(mtx_);
            ready_ = true;
        } // must unlock before notify for efficiency
        cv_.notify_one();
    }
    void wait_for_flag() {
        std::unique_lock lock(mtx_);
        cv_.wait(lock, [&] { return ready_; });
    }
    void reset() {
        std::lock_guard lock(mtx_);
        ready_ = false;
    }
    // sizeof: 40 (mutex) + 48 (cv) + 1 (bool) + padding ~128 bytes
};

// === atomic::wait approach (minimal) ===
class FlagAtomic {
    std::atomic<bool> ready_{false};
public:
    void set() {
        ready_.store(true, std::memory_order_release);
        ready_.notify_one();
    }
    void wait_for_flag() {
        ready_.wait(false, std::memory_order_acquire);
    }
    void reset() {
        ready_.store(false, std::memory_order_relaxed);
    }
    // sizeof: 1 byte (just atomic<bool>)
};

int main() {
    constexpr int ROUNDS = 500'000;

    // Benchmark: ping-pong between two threads
    auto bench = [&](auto& flag_a, auto& flag_b, const char* name) {
        auto start = std::chrono::steady_clock::now();

        std::thread t([&] {
            for (int i = 0; i < ROUNDS; ++i) {
                flag_a.wait_for_flag();
                flag_a.reset();
                flag_b.set();
            }
        });

        for (int i = 0; i < ROUNDS; ++i) {
            flag_a.set();
            flag_b.wait_for_flag();
            flag_b.reset();
        }
        t.join();

        auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << name << ": " << ns / ROUNDS << " ns/round-trip\n";
    };

    FlagCV cv_a, cv_b;
    bench(cv_a, cv_b, "condition_variable");

    FlagAtomic at_a, at_b;
    bench(at_a, at_b, "atomic::wait      ");

    // Typical output:
    // condition_variable: 4200 ns/round-trip
    // atomic::wait      : 1800 ns/round-trip
    //
    // atomic::wait is ~2.3x faster because:
    // - No mutex lock/unlock overhead
    // - Fewer syscalls (no lock contention path)
    // - Smaller memory footprint (cache-friendly)
}
```

The ~2.3x speedup comes from four sources: no mutex lock/unlock overhead, fewer cache lines touched (1 byte vs ~128 bytes), fewer syscalls, and simpler code paths with less surface area for bugs. The condition variable is not worse by design - it is just carrying extra machinery that you do not need when the only thing you are guarding is the flag itself.

**When condition_variable is still better:** complex predicates that depend on multiple variables, situations where the same lock already protects shared data you need during the wait, or when you need `condition_variable_any` with a stop token.

---

## Notes

- **Ordering on wait/notify:** `wait()` accepts a `memory_order` argument (default: `seq_cst`). For flag signaling, `acquire` on the waiter and `release` on the store is sufficient.
- **notify without store is useless:** `notify_one()` doesn't change the value - if you notify without storing a new value, the waiter checks, sees the old value, and goes back to sleep.
- **atomic::wait on any type:** Works with `atomic<int>`, `atomic<enum>`, `atomic<bool>`, etc. The value comparison is bitwise (`memcmp`-style).
- **No lost wakes:** If `store()` + `notify_one()` happens before the waiter calls `wait()`, the waiter checks the value immediately and doesn't block (it never sees the old value).
- Compile with `-std=c++20 -O2 -pthread`.
