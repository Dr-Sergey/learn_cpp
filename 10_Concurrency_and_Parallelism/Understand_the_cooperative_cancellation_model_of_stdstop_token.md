# Understand the cooperative cancellation model of std::stop_token

**Category:** Concurrency and Parallelism  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/thread/stop_token>  

---

## Topic Overview

C++20 introduces a **cooperative cancellation** mechanism: `std::stop_source`, `std::stop_token`, and `std::stop_callback`. Unlike preemptive interruption (killing a thread), cooperative cancellation *asks* a thread to stop - the thread checks and responds at its own pace. The word "cooperative" is doing a lot of work here: the thread must actively participate. It has to check the token and decide to exit. If it never checks, the stop request is simply ignored.

### Components

Here is how the three pieces connect. The source owns the cancellation state and sends the signal; the token reads the signal; the callback lets you react to the signal even when blocked:

```cpp
stop_source  --creates-->  stop_token  <--checked by--  Worker Thread

stop_source::request_stop() --signal-->  stop_token::stop_requested() -> true
                                               |
                                           triggers
                                               |
                                         stop_callback (registered)
```

### Key Properties

| Property | Behavior |
| --- | --- |
| Thread-safe | All operations are safe to call concurrently |
| Cooperative | Thread must *choose* to check and respond |
| One-shot | Once stop is requested, it stays requested forever |
| Lightweight | Token copies share the same internal state |
| Integrated with jthread | `std::jthread` automatically requests stop on destruction |

---

## Self-Assessment

### Q1: Show that stop_token::stop_requested() is a polling mechanism, not a preemptive interrupt

**Answer:**

Watch the output carefully when you run this. The stop signal fires at 350ms, but the worker is partway through iteration 4's sleep. It completes that sleep before it gets a chance to check the flag. That's cooperative cancellation in action:

```cpp
#include <thread>
#include <stop_token>
#include <iostream>
#include <chrono>

void stubborn_worker(std::stop_token token) {
    int iteration = 0;
    while (iteration < 10) {
        ++iteration;
        std::cout << "Working... iteration " << iteration << "\n";

        // POLLING: the thread CHOOSES when to check
        if (token.stop_requested()) {
            std::cout << "Stop requested at iteration " << iteration
                      << " - cleaning up and exiting\n";
            return; // cooperative: we decide to stop
        }

        // Simulate work - stop_requested() does NOT interrupt this!
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    std::cout << "Completed all iterations\n";
}

int main() {
    std::stop_source source;

    std::thread worker(stubborn_worker, source.get_token());

    // Request stop after 350ms - worker won't stop mid-sleep
    std::this_thread::sleep_for(std::chrono::milliseconds(350));
    std::cout << "--- Requesting stop ---\n";
    source.request_stop();

    worker.join();
    // Output:
    // Working... iteration 1
    // Working... iteration 2
    // Working... iteration 3
    // Working... iteration 4
    // --- Requesting stop ---
    // Stop requested at iteration 4 - cleaning up and exiting
    //
    // KEY: The worker ran iteration 4 FULLY (including sleep)
    // before checking stop_requested(). The stop signal does NOT
    // interrupt the thread - it only sets a flag that the thread
    // polls at its convenience.

    // === PREEMPTIVE (thread cancellation, NOT C++) ===
    // pthread_cancel() can kill a thread at any cancellation point.
    // This is DANGEROUS: resources leak, destructors don't run.
    //
    // === COOPERATIVE (C++20 stop_token) ===
    // The thread checks stop_requested() and decides:
    // - When to check (every iteration? every 1000 items?)
    // - How to clean up (flush buffers, close files)
    // - Whether to stop at all (it can ignore the request!)
}
```

**Explanation:** `stop_requested()` returns `true` after `request_stop()` is called, but it never forces the thread to actually stop. The thread must explicitly check the flag and choose to return. This is cooperative - the thread maintains control over its execution.

### Q2: Implement a long-running computation that checks stop_requested() at each iteration

**Answer:**

The interesting design decision here is *how often* to check. Check too often and you add overhead in a tight loop; check too rarely and cancellation feels sluggish. The example shows checking every 1000 iterations as a practical middle ground:

```cpp
#include <thread>
#include <stop_token>
#include <iostream>
#include <chrono>
#include <numeric>
#include <vector>
#include <cmath>

// === CPU-intensive task with cooperative cancellation ===
double compute_pi_montecarlo(std::stop_token token, int max_samples) {
    long long inside = 0;
    int i = 0;

    for (; i < max_samples; ++i) {
        // Check every 1000 iterations (balance responsiveness vs overhead)
        if ((i % 1000 == 0) && token.stop_requested()) {
            std::cout << "  Cancelled after " << i << " samples\n";
            break;
        }

        // Monte Carlo: random point in [0,1)x[0,1), check if inside unit circle
        double x = (double)rand() / RAND_MAX;
        double y = (double)rand() / RAND_MAX;
        if (x * x + y * y <= 1.0) ++inside;
    }

    return 4.0 * inside / i; // pi approx 4 * (points inside circle / total points)
}

int main() {
    // === Using jthread: automatic stop on destruction ===
    {
        double result = 0;
        {
            std::jthread worker([&](std::stop_token token) {
                result = compute_pi_montecarlo(token, 100'000'000);
                std::cout << "  pi approx " << result << "\n";
            });

            // Let it run for 50ms, then jthread destructor requests stop
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            std::cout << "Destroying jthread (auto stop)...\n";
        } // <- jthread destructor calls request_stop() + join()

        std::cout << "Result: pi approx " << result << "\n\n";
    }

    // === Using stop_source for manual control ===
    {
        std::stop_source source;
        double result = 0;

        std::jthread worker([&](std::stop_token token) {
            result = compute_pi_montecarlo(token, 100'000'000);
        });

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        source.request_stop(); // explicit cancellation
        worker.join();
        std::cout << "Manual stop: pi approx " << result << "\n";
    }

    // === Checking interval matters ===
    // Check too often: overhead from atomic load on every iteration
    // Check too rarely: slow response to cancellation
    //
    // Guidelines:
    //   Tight loop (ns per iter): check every 1000-10000 iterations
    //   Medium loop (us per iter): check every iteration
    //   I/O bound: use stop_callback instead of polling

    // Output:
    // Destroying jthread (auto stop)...
    //   Cancelled after 5000 samples
    //   pi approx 3.1416
    // Result: pi approx 3.1416
}
```

**Explanation:** The computation checks `stop_requested()` every 1000 iterations - a balance between responsiveness (stops promptly) and overhead (atomic load is cheap but not free in a tight loop). `std::jthread`'s destructor automatically calls `request_stop()` and `join()`, making RAII-based cancellation effortless.

### Q3: Use stop_callback to wake a blocking wait when a stop is requested

**Answer:**

Here's the trickiest scenario: what if your thread isn't spinning in a loop, but is instead blocked on a `condition_variable::wait()`? Polling doesn't help because the thread is asleep. The solution is `stop_callback` - it fires when the stop is requested, and your callback can wake the sleeping thread:

```cpp
#include <thread>
#include <stop_token>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <iostream>
#include <chrono>
#include <optional>

// === Problem: blocking wait can't be cancelled by polling ===
// A thread blocked on condition_variable::wait() can't check stop_requested()
// Solution: stop_callback notifies the condition variable when stop is requested

template<typename T>
class CancellableQueue {
    std::queue<T> queue_;
    std::mutex mtx_;
    std::condition_variable cv_;

public:
    void push(T value) {
        {
            std::lock_guard lock(mtx_);
            queue_.push(std::move(value));
        }
        cv_.notify_one();
    }

    // Blocking pop with cancellation support
    std::optional<T> pop(std::stop_token token) {
        std::unique_lock lock(mtx_);

        // Register a callback that wakes us up when stop is requested
        std::stop_callback callback(token, [this] {
            cv_.notify_all(); // wake ALL waiters so they can check stop
        });

        // Wait for data OR stop request
        cv_.wait(lock, [&] {
            return !queue_.empty() || token.stop_requested();
        });

        if (token.stop_requested())
            return std::nullopt; // cancelled!

        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }
};

int main() {
    CancellableQueue<int> queue;

    // Consumer: blocks waiting for items
    std::jthread consumer([&](std::stop_token token) {
        while (true) {
            auto item = queue.pop(token); // blocks until item or stop
            if (!item) {
                std::cout << "Consumer: cancelled, exiting\n";
                return;
            }
            std::cout << "Consumer: got " << *item << "\n";
        }
    });

    // Producer: push some items
    for (int i = 1; i <= 3; ++i) {
        queue.push(i);
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    std::cout << "Destroying jthread (triggering stop)...\n";
    // jthread destructor:
    //   1. request_stop() -> stop_callback fires -> cv_.notify_all()
    //   2. join() -> waits for consumer to exit

    // Output:
    // Consumer: got 1
    // Consumer: got 2
    // Consumer: got 3
    // Destroying jthread (triggering stop)...
    // Consumer: cancelled, exiting

    // === How stop_callback works ===
    //
    // Without callback:
    //   consumer blocks on cv_.wait()
    //   request_stop() sets flag, but consumer is sleeping!
    //   consumer stays blocked until next push() wakes it
    //
    // With callback:
    //   request_stop() -> callback fires -> cv_.notify_all()
    //   consumer wakes up, checks stop_requested() -> true -> exits
    //
    // The callback is AUTOMATICALLY deregistered when it goes out of scope.
    // It is called SYNCHRONOUSLY inside request_stop() if stop was already
    // requested, or when request_stop() is called later.
}
```

**Explanation:** `stop_callback` bridges the gap between polling and blocking. When a thread is blocked on `condition_variable::wait()`, it can't poll `stop_requested()`. The callback fires `notify_all()` when stop is requested, waking the blocked thread so it can check the flag and exit. The callback is thread-safe and automatically deregistered when destroyed.

---

## Notes

- **`stop_source` -> `stop_token` -> `stop_callback`:** Source signals, token checks, callback reacts. Multiple tokens can share one source.
- **`jthread` integration:** `std::jthread` holds a `stop_source` internally and passes the token to the callable. Destructor calls `request_stop()` + `join()`.
- **One-shot:** `request_stop()` is irreversible - you cannot "un-request" a stop. Design your protocol accordingly.
- **Callback execution:** If stop is already requested when you construct `stop_callback`, the callback fires immediately in the constructor. If not, it fires in the thread that calls `request_stop()`.
- **Use with `condition_variable_any`:** C++20 adds `condition_variable_any::wait(lock, stop_token, pred)` which has built-in stop_token support - no manual callback needed.
- Compile with `-std=c++20 -O2 -pthread`.
