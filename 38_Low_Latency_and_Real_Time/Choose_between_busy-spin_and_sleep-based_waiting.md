# Choose Between Busy-Spin and Sleep-Based Waiting

**Category:** Low Latency and Real Time  
**Standard:** C++20, C++23  
**Reference:** [Intel PAUSE instruction](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/), [C++20 atomic wait/notify](https://en.cppreference.com/w/cpp/atomic/atomic/wait)  

---

## Topic Overview

When a thread waits for an event - a message arrival, a flag change, or a condition - the two extremes are **busy-spinning** (polling in a tight loop) and **sleep-based waiting** (blocking via OS primitives). The choice fundamentally trades CPU resources for wake-up latency: spinning provides sub-microsecond response but wastes an entire core; sleeping frees the core but adds 1-15µs of wake-up latency from the OS scheduler.

The reason the trade-off is so stark is physics. A sleeping thread has handed control to the kernel. To wake it up, the kernel has to notice the event, find the thread in its wait queue, mark it runnable, and schedule it onto a CPU. That sequence takes time - time you can't shorten no matter how fast your hardware is. A spinning thread, by contrast, notices the event in the very next loop iteration because it never stopped looking.

The **PAUSE** instruction (x86 `_mm_pause()`) is critical for spin loops. Without it, the CPU's speculative execution pipeline fills with useless iterations, wasting power and starving sibling hyperthreads. PAUSE inserts a short delay (~5ns on Skylake, ~140ns on pre-Skylake) and signals the CPU that this is a spin loop. On ARM, `__yield()` serves a similar purpose.

**Hybrid spinning** combines the best of both: spin for a short duration (typically 1-100µs), then fall back to OS sleep. This captures the common case (event arrives quickly) with low latency while avoiding burning a core during prolonged waits. C++20 `std::atomic::wait/notify` implements a platform-optimized hybrid - spinning briefly, then using futex/WaitOnAddress.

| Strategy | Wake Latency | CPU Usage | Best For |
| --- | --- | --- | --- |
| Busy-spin (tight loop) | < 100 ns | 100% of core | Ultra-low latency, dedicated core |
| Busy-spin + PAUSE | < 200 ns | ~80% of core | Spin on shared core |
| Hybrid (spin then sleep) | 200 ns - 5 µs | Low average | General low-latency |
| `std::atomic::wait` | 500 ns - 5 µs | Minimal | Portable, system-optimized |
| `condition_variable::wait` | 5-15 µs | Minimal | Non-critical latency |
| `std::this_thread::sleep_for` | 1-15 ms (!) | Minimal | Background tasks only |

Here's how CPU usage and wake latency relate across the strategies. The point is that you're always trading one for the other:

```cpp
LATENCY vs CPU TRADE-OFF:

CPU Usage  │ ▓▓▓▓▓ Busy-spin
  100%     │ ▓▓▓▓  PAUSE spin
           │ ▓▓    Hybrid
           │ ▓     atomic::wait
           │ ░     condition_variable
           │──────────────────
           0     5    10    15  µs  Wake Latency
```

---

## Self-Assessment

### Q1: Implement three waiting strategies - busy-spin with PAUSE, hybrid spin-then-sleep, and `std::atomic::wait` - and benchmark their wake-up latency

The benchmark measures round-trip wake latency: how long from when the producer sets the flag to when the consumer thread observes it. Pay attention to how the strategies compare and think about what your hot path actually needs.

```cpp
#include <atomic>
#include <chrono>
#include <thread>
#include <cstdio>
#include <cstdint>
#include <immintrin.h>  // _mm_pause

std::atomic<bool> flag{false};

// Strategy 1: Pure busy-spin with PAUSE
void busy_spin_wait() {
    while (!flag.load(std::memory_order_acquire)) {
        _mm_pause();
    }
}

// Strategy 2: Hybrid — spin for N iterations, then sleep
void hybrid_wait(int spin_iters = 1000) {
    for (int i = 0; i < spin_iters; ++i) {
        if (flag.load(std::memory_order_acquire)) return;
        _mm_pause();
    }
    // Fall back to OS-assisted wait
    while (!flag.load(std::memory_order_acquire)) {
        std::this_thread::yield();
    }
}

// Strategy 3: C++20 atomic wait (kernel-assisted)
void atomic_wait() {
    flag.wait(false, std::memory_order_acquire);
}

template <typename WaitFn>
double measure_latency(WaitFn wait_fn, const char* label) {
    constexpr int kTrials = 10'000;
    double total_ns = 0;

    for (int trial = 0; trial < kTrials; ++trial) {
        flag.store(false, std::memory_order_relaxed);

        std::thread waiter([&] { wait_fn(); });

        // Small delay to ensure waiter is spinning
        for (int i = 0; i < 100; ++i) _mm_pause();

        auto t0 = std::chrono::high_resolution_clock::now();
        flag.store(true, std::memory_order_release);
        flag.notify_one();  // for atomic_wait variant
        waiter.join();
        auto t1 = std::chrono::high_resolution_clock::now();

        total_ns += std::chrono::duration<double, std::nano>(t1 - t0).count();
    }

    double avg_ns = total_ns / kTrials;
    std::printf("%-25s avg wake: %8.1f ns\n", label, avg_ns);
    return avg_ns;
}

int main() {
    measure_latency(busy_spin_wait, "Busy-spin + PAUSE");
    measure_latency(hybrid_wait,    "Hybrid (1000 spins)");
    measure_latency(atomic_wait,    "std::atomic::wait");
}
```

On a modern x86 machine you'll typically see the busy-spin strategy win by a factor of 5-10x on wake latency. But remember that "winning" here means burning a whole CPU core. If you only have one isolated core for this thread, that's fine. If not, you need to think carefully.

### Q2: Build a `SpinLock` with PAUSE that degrades gracefully under contention by tracking spin counts and adapting

A real production spin lock doesn't just spin blindly. It uses the test-and-test-and-set (TTAS) pattern: spin on a read-only load (which doesn't generate write traffic on the cache bus), and only attempt the expensive exchange when the lock looks free. Under high contention this makes a significant difference.

```cpp
#include <atomic>
#include <cstdint>
#include <immintrin.h>
#include <thread>
#include <vector>
#include <cstdio>
#include <chrono>

class AdaptiveSpinLock {
    std::atomic<bool> locked_{false};
    std::atomic<uint64_t> total_spins_{0};
    std::atomic<uint64_t> acquisitions_{0};

public:
    void lock() noexcept {
        uint64_t spins = 0;
        // Phase 1: try immediate acquire
        if (!locked_.exchange(true, std::memory_order_acquire)) {
            acquisitions_.fetch_add(1, std::memory_order_relaxed);
            return;
        }

        // Phase 2: spin with PAUSE, test-and-test-and-set
        while (true) {
            // Test (read-only) — doesn't cause cache-line bouncing
            while (locked_.load(std::memory_order_relaxed)) {
                _mm_pause();
                ++spins;

                // Phase 3: back off if spinning too long
                if (spins > 10'000) {
                    std::this_thread::yield();  // give up time slice
                }
            }
            // Test-and-set (write) — only when likely unlocked
            if (!locked_.exchange(true, std::memory_order_acquire)) {
                total_spins_.fetch_add(spins, std::memory_order_relaxed);
                acquisitions_.fetch_add(1, std::memory_order_relaxed);
                return;
            }
        }
    }

    void unlock() noexcept {
        locked_.store(false, std::memory_order_release);
    }

    // Diagnostics
    double avg_spins() const {
        uint64_t acq = acquisitions_.load();
        return acq > 0 ? double(total_spins_.load()) / acq : 0.0;
    }
};

int main() {
    AdaptiveSpinLock lock;
    int64_t counter = 0;
    constexpr int kThreads = 8;
    constexpr int kOps = 100'000;

    auto t0 = std::chrono::high_resolution_clock::now();
    std::vector<std::thread> threads;
    for (int i = 0; i < kThreads; ++i) {
        threads.emplace_back([&] {
            for (int j = 0; j < kOps; ++j) {
                lock.lock();
                ++counter;
                lock.unlock();
            }
        });
    }
    for (auto& t : threads) t.join();
    auto t1 = std::chrono::high_resolution_clock::now();

    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    std::printf("Counter: %lld (expected %d)\n",
                (long long)counter, kThreads * kOps);
    std::printf("Time: %.2f ms, avg spins/acquisition: %.1f\n",
                ms, lock.avg_spins());
}
```

The `avg_spins()` diagnostic is valuable in production. If it's consistently high, your critical section is too long or you have too many threads competing. If it's near zero, you're barely contending at all - which is the ideal state.

### Q3: Compare `std::condition_variable` vs `std::atomic::wait` vs a manual futex call for producer-consumer wake-up latency

This benchmark gives you a concrete feel for the OS-level cost of each signaling mechanism. The condition variable is the most familiar, but it's also the heaviest because it always involves mutex locking and kernel transitions. The raw futex call is what the kernel actually provides - `std::atomic::wait` is essentially a well-behaved wrapper around it.

```cpp
#include <atomic>
#include <condition_variable>
#include <mutex>
#include <thread>
#include <chrono>
#include <cstdio>

#ifdef __linux__
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#endif

struct CondVarChannel {
    std::mutex mtx;
    std::condition_variable cv;
    bool ready = false;

    void signal() {
        { std::lock_guard lk(mtx); ready = true; }
        cv.notify_one();
    }
    void wait() {
        std::unique_lock lk(mtx);
        cv.wait(lk, [this] { return ready; });
    }
    void reset() { ready = false; }
};

struct AtomicWaitChannel {
    std::atomic<int> value{0};

    void signal() {
        value.store(1, std::memory_order_release);
        value.notify_one();
    }
    void wait() {
        value.wait(0, std::memory_order_acquire);
    }
    void reset() { value.store(0, std::memory_order_relaxed); }
};

#ifdef __linux__
struct FutexChannel {
    std::atomic<int> value{0};

    void signal() {
        value.store(1, std::memory_order_release);
        syscall(SYS_futex, &value, FUTEX_WAKE, 1, nullptr, nullptr, 0);
    }
    void wait() {
        while (value.load(std::memory_order_acquire) == 0) {
            syscall(SYS_futex, &value, FUTEX_WAIT, 0, nullptr, nullptr, 0);
        }
    }
    void reset() { value.store(0, std::memory_order_relaxed); }
};
#endif

template <typename Channel>
double benchmark_channel(const char* name) {
    Channel ch;
    constexpr int kTrials = 50'000;
    double total_ns = 0;

    for (int i = 0; i < kTrials; ++i) {
        ch.reset();
        std::thread consumer([&] { ch.wait(); });

        std::this_thread::yield();  // let consumer begin waiting

        auto t0 = std::chrono::high_resolution_clock::now();
        ch.signal();
        consumer.join();
        auto t1 = std::chrono::high_resolution_clock::now();

        total_ns += std::chrono::duration<double, std::nano>(t1 - t0).count();
    }

    double avg = total_ns / kTrials;
    std::printf("%-25s avg: %8.1f ns\n", name, avg);
    return avg;
}

int main() {
    benchmark_channel<CondVarChannel>("condition_variable");
    benchmark_channel<AtomicWaitChannel>("atomic::wait");
#ifdef __linux__
    benchmark_channel<FutexChannel>("raw futex");
#endif
}
```

You'll typically see `atomic::wait` and the raw futex very close to each other, and both noticeably faster than `condition_variable`. The condition variable's extra mutex operations are what slow it down. For a background task or UI event, the difference doesn't matter. For a hot path that fires 100,000 times per second, it absolutely does.

---

## Notes

- **PAUSE on hyperthreaded cores**: Without PAUSE, a spinning thread saturates the execution port, starving its sibling. PAUSE yields resources and saves ~10-20% power.
- **`std::atomic::wait`** (C++20) uses futex on Linux, `WaitOnAddress` on Windows, and `__ulock_wait` on macOS - all kernel-assisted, efficient wake.
- **Never** use `sleep_for(1ms)` for low-latency waiting - timer resolution is often 1-15ms due to OS timer granularity.
- **Test-and-test-and-set** pattern: spin on a read-only load (no bus traffic), then attempt exchange only when the lock appears free - reduces cache-line bouncing.
- Measure with `perf stat -e context-switches,cpu-migrations` to verify spinning avoids context switches.
- For dedicated cores (isolated with `isolcpus`), pure busy-spin is optimal - the core has nothing else to do.
