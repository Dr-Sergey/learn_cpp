# Implement adaptive spinning strategies: spin-then-yield-then-sleep

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/thread/yield>  

---

## Topic Overview

### The Waiting Problem

When a thread must wait for a condition (lock release, data arrival, flag set), there are three strategies with different latency/CPU tradeoffs. No single strategy is best in all situations - the right choice depends on how long you expect to wait.

| Strategy | Latency | CPU cost | Best for |
| --- | --- | --- | --- |
| **Spin** (`pause` loop) | Nanoseconds | 100% core usage | Ultra-low latency, short waits |
| **Yield** (`std::this_thread::yield`) | Microseconds | Context switch cost | Medium waits |
| **Sleep** (`futex`, condition variable) | ~50-100 µs | Minimal | Long/unknown waits |

The key insight is that each strategy trades CPU time for wake-up responsiveness. Spinning is maximally responsive but burns CPU continuously. Sleeping is CPU-friendly but adds scheduler overhead before your thread can run again. Adaptive spinning threads start with the fastest strategy and degrade gracefully if the wait turns out to be longer than expected.

### Pure Spinning - Maximum Responsiveness

Here's the foundation: a spinlock that never gives up the CPU. It uses `_mm_pause()` to be a good citizen while spinning, but it never yields the thread.

```cpp
#include <atomic>
#include <immintrin.h>  // _mm_pause

class SpinLock {
    std::atomic<bool> locked_{false};

public:
    void lock() {
        while (true) {
            // Try to acquire
            if (!locked_.exchange(true, std::memory_order_acquire)) {
                return;  // Got the lock
            }

            // Spin on read (no bus traffic) until released
            while (locked_.load(std::memory_order_relaxed)) {
                _mm_pause();  // Reduce power, avoid memory order violations
                              // ~10 cycles on Intel, hint to CPU
            }
        }
    }

    void unlock() {
        locked_.store(false, std::memory_order_release);
    }
};
```

The inner `while (locked_.load(...))` loop uses a read-only load before attempting the expensive `exchange`. This is the test-and-test-and-set (TTAS) pattern - it avoids generating write bus traffic while the lock is held by someone else.

Problems with pure spinning:

- Wastes 100% of a CPU core
- Causes **cache line bouncing** in contended scenarios
- Steals resources from the thread you're waiting on

### Adaptive Spinning - The Hybrid Approach

The adaptive version adds two additional phases after the spin phase. Rather than spinning forever, it progressively gives the OS more control.

```cpp
#include <atomic>
#include <thread>
#include <immintrin.h>

template<unsigned SpinCount = 1000,
         unsigned YieldCount = 100>
class AdaptiveSpinLock {
    std::atomic<bool> locked_{false};

public:
    void lock() {
        // Phase 1: Spin with pause (nanosecond latency)
        for (unsigned i = 0; i < SpinCount; ++i) {
            if (!locked_.exchange(true, std::memory_order_acquire)) {
                return;
            }
            _mm_pause();
        }

        // Phase 2: Yield (microsecond latency, give up timeslice)
        for (unsigned i = 0; i < YieldCount; ++i) {
            if (!locked_.exchange(true, std::memory_order_acquire)) {
                return;
            }
            std::this_thread::yield();
        }

        // Phase 3: Sleep (give up entirely, OS wakes us)
        while (locked_.exchange(true, std::memory_order_acquire)) {
            // Use futex or condition variable for OS-level sleep
            std::this_thread::sleep_for(std::chrono::microseconds(1));
        }
    }

    void unlock() {
        locked_.store(false, std::memory_order_release);
    }
};
```

The `SpinCount` and `YieldCount` template parameters let you tune the thresholds for your workload. A common heuristic is to set `SpinCount` so the spin phase lasts about as long as a context switch (roughly 1-10 µs), and `YieldCount` to cover the expected tail of medium-length waits.

### Production-Quality Adaptive Wait

This class extracts the adaptive waiting logic so it can be applied to any predicate, not just lock acquisition. It also adds exponential backoff in the sleep phase and a diagnostic for detecting contented scenarios.

```cpp
#include <atomic>
#include <chrono>
#include <thread>

class AdaptiveWaiter {
    // Configuration: tune based on workload profiling
    static constexpr int SPIN_ITERATIONS  = 4000;  // ~4-8 µs on modern CPUs
    static constexpr int YIELD_ITERATIONS = 50;     // ~50-100 µs
    static constexpr auto MIN_SLEEP = std::chrono::microseconds(10);
    static constexpr auto MAX_SLEEP = std::chrono::milliseconds(1);

    int consecutive_sleeps_ = 0;  // Track sleep frequency for back-off

public:
    // Wait until predicate returns true
    template<typename Predicate>
    void wait(Predicate&& pred) {
        // Phase 1: Tight spin with pause
        for (int i = 0; i < SPIN_ITERATIONS; ++i) {
            if (pred()) {
                consecutive_sleeps_ = 0;
                return;
            }
            spin_pause();
        }

        // Phase 2: Yield
        for (int i = 0; i < YIELD_ITERATIONS; ++i) {
            if (pred()) {
                consecutive_sleeps_ = 0;
                return;
            }
            std::this_thread::yield();
        }

        // Phase 3: Exponential back-off sleep
        auto sleep_time = MIN_SLEEP;
        while (!pred()) {
            std::this_thread::sleep_for(sleep_time);
            ++consecutive_sleeps_;
            sleep_time = std::min(sleep_time * 2, MAX_SLEEP);
        }
    }

    void reset() { consecutive_sleeps_ = 0; }

    // Useful for dynamically adjusting spin counts
    [[nodiscard]] bool is_contended() const { return consecutive_sleeps_ > 10; }

private:
    static void spin_pause() {
#if defined(__x86_64__) || defined(_M_X64)
        _mm_pause();
#elif defined(__aarch64__)
        asm volatile("yield" ::: "memory");
#else
        // Fallback: compiler barrier
        std::atomic_signal_fence(std::memory_order_seq_cst);
#endif
    }
};
```

The `is_contended()` method is particularly useful in production. If your waiter is frequently reaching the sleep phase, it's a signal that your wait durations are longer than your spin budget - you might want to increase `MAX_SLEEP` or investigate what's holding up the condition.

### Using Adaptive Wait in a Lock-Free Queue

Here's the adaptive waiter integrated into a queue where the producer and consumer may run at different rates. The producer waits when the queue is full; the consumer waits when it's empty.

```cpp
template<typename T, size_t N>
class WaitableQueue {
    std::array<std::atomic<T>, N> buffer_;
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};

    mutable AdaptiveWaiter producer_wait_;
    mutable AdaptiveWaiter consumer_wait_;

public:
    void push(const T& value) {
        producer_wait_.wait([&] {
            size_t tail = tail_.load(std::memory_order_relaxed);
            size_t next = (tail + 1) % N;
            return next != head_.load(std::memory_order_acquire);
        });

        size_t tail = tail_.load(std::memory_order_relaxed);
        buffer_[tail].store(value, std::memory_order_relaxed);
        tail_.store((tail + 1) % N, std::memory_order_release);
    }

    T pop() {
        consumer_wait_.wait([&] {
            return head_.load(std::memory_order_relaxed) !=
                   tail_.load(std::memory_order_acquire);
        });

        size_t head = head_.load(std::memory_order_relaxed);
        T value = buffer_[head].load(std::memory_order_relaxed);
        head_.store((head + 1) % N, std::memory_order_release);
        return value;
    }
};
```

The `alignas(64)` on `head_` and `tail_` ensures they live on separate cache lines. Without this, a producer updating `tail_` and a consumer reading `tail_` would cause false sharing, degrading performance even though those accesses don't conflict logically.

### Linux futex - OS-Level Efficient Wait

When you need the OS to manage the wait queue efficiently, a futex gives you kernel-assisted blocking with the minimum possible overhead. This is essentially what `std::atomic::wait` uses under the hood on Linux.

```cpp
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <atomic>

class FutexLock {
    std::atomic<int> state_{0};  // 0=unlocked, 1=locked-no-waiters, 2=locked-with-waiters

public:
    void lock() {
        int expected = 0;
        if (state_.compare_exchange_strong(expected, 1, std::memory_order_acquire)) {
            return;  // Fast path: uncontended
        }

        // Slow path: contended
        while (true) {
            // Set state to 2 (waiters present) and sleep
            if (expected == 2 ||
                state_.compare_exchange_strong(expected, 2, std::memory_order_acquire)) {
                // Sleep until state changes (kernel manages the wait queue)
                syscall(SYS_futex, &state_, FUTEX_WAIT, 2, nullptr, nullptr, 0);
            }
            // Retry after wakeup
            expected = 0;
            if (state_.compare_exchange_strong(expected, 2, std::memory_order_acquire)) {
                return;
            }
        }
    }

    void unlock() {
        if (state_.exchange(0, std::memory_order_release) == 2) {
            // Had waiters — wake one
            syscall(SYS_futex, &state_, FUTEX_WAKE, 1, nullptr, nullptr, 0);
        }
    }
};
```

The three-state design (`0`, `1`, `2`) is an important optimization. When there are no waiters, `unlock` just does a single atomic store - it never calls into the kernel. The kernel wake-up syscall only happens when the state is `2`, meaning at least one thread is actually sleeping. This is why futex-based locks have low overhead in the uncontended case.

---

## Self-Assessment

### Q1: Why not just always spin

Spinning wastes CPU cycles that could be used by other threads - including the thread you're waiting on. If the producer shares a CPU with the consumer and both spin, neither makes progress (priority inversion). Spinning also consumes power and generates heat. Pure spinning is only appropriate when: (1) the wait is expected to be very short (< 1 µs), (2) the waiting thread has a dedicated CPU core, and (3) latency is more important than throughput.

### Q2: What does `_mm_pause()` actually do

`_mm_pause()` is an x86 intrinsic (compiles to the `PAUSE` instruction) that:

1. **Signals to the CPU** that this is a spin loop - allows power-saving optimizations
2. **Adds a small delay** (~10 cycles) - prevents the spin loop from flooding the memory bus
3. **Avoids memory order violations** on Intel CPUs with speculative execution - without it, the CPU may speculatively read stale values, causing a costly pipeline flush when the value changes

On ARM, the equivalent is the `YIELD` instruction.

### Q3: When should you transition from spinning to sleeping

The transition point depends on the expected wait time. A common heuristic:

- Spin for ~1-10 µs (the cost of a context switch on most systems)
- If still waiting, yield for ~10-100 µs
- If still waiting, sleep with exponential back-off

Profile your specific workload: measure the typical wait duration and set the spin count just above the 90th percentile. If most waits complete during the spin phase, you get spin-level latency. If a wait exceeds the spin budget, the thread sleeps efficiently.

---

## Notes

- `std::this_thread::yield()` maps to `sched_yield()` on Linux - it relinquishes the timeslice
- `std::atomic::wait()` (C++20) provides a portable futex-like wait mechanism
- On Windows, use `WaitOnAddress` / `WakeByAddressSingle` as the futex equivalent
- LMAX Disruptor (a famous low-latency library) uses configurable wait strategies
- Always benchmark with realistic contention levels - microbenchmarks often mislead
