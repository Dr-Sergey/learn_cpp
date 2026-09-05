# Use std::future::wait_for for polling with a timeout

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/future/wait_for>  

---

## Topic Overview

`std::future::wait_for()` checks whether a future's result is ready, waiting for at most a specified duration. It returns a `std::future_status` enum, which tells you one of three things: the result is ready, the timeout elapsed without a result, or the task was deferred and will never become ready until you call `get()`.

| Status | Meaning |
| --- | --- |
| `future_status::ready` | Result or exception is available - `get()` won't block |
| `future_status::timeout` | Duration elapsed, result not yet ready |
| `future_status::deferred` | Task was deferred (won't run until `get()` is called) |

### Signature

```cpp
template<class Rep, class Period>
std::future_status wait_for(const std::chrono::duration<Rep,Period>& timeout);
```

### Key Pattern

The non-blocking poll pattern is straightforward: keep calling `wait_for` with a zero or short duration until the status is `ready`, then call `get()` knowing it will return immediately.

```cpp
// Non-blocking poll loop:
while (fut.wait_for(0ms) != std::future_status::ready) {
    // do other work...
}
auto result = fut.get(); // guaranteed immediate
```

---

## Self-Assessment

### Q1: Poll a future every 100ms with wait_for and show the std::future_status values

**Answer:**

Watch the status values change as we poll a task that takes about 350ms to complete. This illustrates all three possible return values in a realistic scenario.

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <chrono>

using namespace std::chrono_literals;

int heavy_computation() {
    std::this_thread::sleep_for(350ms); // simulate work
    return 42;
}

int main() {
    auto fut = std::async(std::launch::async, heavy_computation);

    int poll_count = 0;
    while (true) {
        auto status = fut.wait_for(100ms);

        switch (status) {
            case std::future_status::ready:
                std::cout << "Poll " << ++poll_count
                          << ": READY\n";
                break;
            case std::future_status::timeout:
                std::cout << "Poll " << ++poll_count
                          << ": TIMEOUT (still working...)\n";
                break;
            case std::future_status::deferred:
                std::cout << "Poll " << ++poll_count
                          << ": DEFERRED (won't run until get())\n";
                // Must call get() to run a deferred task
                break;
        }

        if (status == std::future_status::ready) {
            break;
        }
    }

    int result = fut.get(); // immediate - already ready
    std::cout << "Result: " << result << "\n";

    // Output:
    // Poll 1: TIMEOUT (still working...)
    // Poll 2: TIMEOUT (still working...)
    // Poll 3: TIMEOUT (still working...)
    // Poll 4: READY
    // Result: 42
    //
    // (3 timeouts * 100ms = 300ms, then ready at ~350ms)

    // === GOTCHA: wait_for on deferred future ===
    auto deferred = std::async(std::launch::deferred, [] { return 99; });
    auto st = deferred.wait_for(1s);
    std::cout << "\nDeferred future status: "
              << (st == std::future_status::deferred ? "deferred" : "other") << "\n";
    // deferred futures ALWAYS return future_status::deferred from wait_for
    // They NEVER become ready until you call get()
    std::cout << "Deferred result: " << deferred.get() << "\n";

    // Output:
    // Deferred future status: deferred
    // Deferred result: 99
}
```

### Q2: Use wait_for in a UI event loop to remain responsive while waiting for a background task

**Answer:**

The key insight here is that a UI cannot afford to block. By polling with a short timeout equal to one frame's worth of time, the UI loop stays alive and responsive while the background task runs independently.

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <chrono>
#include <string>
#include <functional>

using namespace std::chrono_literals;

// Simulated UI framework
class SimpleUI {
    int frame_count_ = 0;
public:
    void process_events() {
        // In a real UI: handle mouse, keyboard, window events
        ++frame_count_;
    }

    void render() {
        // Show a spinner or progress indicator
        const char spinner[] = {'|', '/', '-', '\\'};
        std::cout << "\r  [" << spinner[frame_count_ % 4]
                  << "] Loading... (frame " << frame_count_ << ")  " << std::flush;
    }

    void show_result(const std::string& result) {
        std::cout << "\r  Result: " << result << "                  \n";
    }

    void show_error(const std::string& msg) {
        std::cout << "\r  ERROR: " << msg << "                  \n";
    }
};

// Background task: expensive computation
std::string fetch_data() {
    std::this_thread::sleep_for(800ms);
    return "Data loaded (42 records)";
}

int main() {
    SimpleUI ui;

    // Launch background task
    auto fut = std::async(std::launch::async, fetch_data);

    // UI event loop - remains responsive
    while (true) {
        ui.process_events();
        ui.render();

        // Poll future with short timeout (16ms ~= 60fps)
        auto status = fut.wait_for(16ms);

        if (status == std::future_status::ready) {
            try {
                auto result = fut.get(); // immediate, won't block
                ui.show_result(result);
            } catch (const std::exception& e) {
                ui.show_error(e.what());
            }
            break;
        }
        // status == timeout -> continue event loop
    }

    // Output (animated):
    //   [|] Loading... (frame 1)
    //   [/] Loading... (frame 2)
    //   [-] Loading... (frame 3)
    //   ... (~50 frames at 16ms each)
    //   Result: Data loaded (42 records)
}
```

The UI loop polls with `wait_for(16ms)`, keeping the frame rate at ~60fps. When the background task finishes, the next poll returns `ready` and we retrieve the result. The UI never freezes because `wait_for` never blocks longer than 16ms.

### Q3: Show a timeout-based retry pattern using wait_for and promise/future pairs

**Answer:**

`wait_for` shines in retry logic because it lets you abandon a slow attempt and move on without blocking indefinitely. The promise-based variant shows a more flexible design where each attempt runs on a fully independent detached thread.

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <chrono>
#include <stdexcept>
#include <functional>

using namespace std::chrono_literals;

// Simulate an unreliable network call
int unreliable_fetch(int attempt) {
    // Simulate variable response time
    auto delay = std::chrono::milliseconds(100 * attempt + 50);
    std::this_thread::sleep_for(delay);

    if (attempt < 3) {
        throw std::runtime_error("Connection timeout");
    }
    return 200; // success on 3rd attempt
}

// Retry with timeout per attempt
template<typename F>
auto retry_with_timeout(F func, int max_retries,
                        std::chrono::milliseconds per_attempt_timeout)
    -> decltype(func(0))
{
    for (int attempt = 1; attempt <= max_retries; ++attempt) {
        // Launch each attempt with std::async
        auto fut = std::async(std::launch::async, func, attempt);

        auto status = fut.wait_for(per_attempt_timeout);

        if (status == std::future_status::ready) {
            try {
                auto result = fut.get(); // may throw
                std::cout << "  Attempt " << attempt << ": SUCCESS\n";
                return result;
            } catch (const std::exception& e) {
                std::cout << "  Attempt " << attempt << ": FAILED ("
                          << e.what() << ")\n";
                // Continue to next attempt
            }
        } else {
            std::cout << "  Attempt " << attempt << ": TIMED OUT\n";
            // Future destructor will block until async completes
            // In production, you'd want a cancellable task
        }
    }
    throw std::runtime_error("All retries exhausted");
}

// Promise/future pattern: cancel abandoned attempts
void promise_based_retry() {
    std::cout << "\n--- Promise-based retry ---\n";

    for (int attempt = 1; attempt <= 3; ++attempt) {
        std::promise<int> prom;
        auto fut = prom.get_future();

        std::thread worker([p = std::move(prom), attempt]() mutable {
            try {
                auto result = unreliable_fetch(attempt);
                p.set_value(result);
            } catch (...) {
                p.set_exception(std::current_exception());
            }
        });
        worker.detach(); // detach so we can abandon on timeout

        auto status = fut.wait_for(250ms);

        if (status == std::future_status::ready) {
            try {
                int result = fut.get();
                std::cout << "  Attempt " << attempt
                          << ": got " << result << "\n";
                return;
            } catch (const std::exception& e) {
                std::cout << "  Attempt " << attempt
                          << ": failed (" << e.what() << ")\n";
            }
        } else {
            std::cout << "  Attempt " << attempt << ": timed out\n";
        }
    }
}

int main() {
    std::cout << "--- Async retry ---\n";
    try {
        int result = retry_with_timeout(unreliable_fetch, 5, 500ms);
        std::cout << "Final result: " << result << "\n";
    } catch (const std::exception& e) {
        std::cout << "Failed: " << e.what() << "\n";
    }

    promise_based_retry();

    // Output:
    // --- Async retry ---
    //   Attempt 1: FAILED (Connection timeout)
    //   Attempt 2: FAILED (Connection timeout)
    //   Attempt 3: SUCCESS
    // Final result: 200
    //
    // --- Promise-based retry ---
    //   Attempt 1: failed (Connection timeout)
    //   Attempt 2: failed (Connection timeout)
    //   Attempt 3: got 200
}
```

`wait_for` returns `timeout` without consuming the result, so you can abandon slow attempts and retry. Using `promise/future` pairs with detached threads allows each attempt to be independent - if an attempt times out, the detached thread eventually completes and the promise is destroyed harmlessly.

---

## Notes

- **`wait_for(0ms)`:** Instant, non-blocking check. Returns `ready` if available, `timeout` otherwise. Perfect for tight polling loops.
- **Deferred trap:** `wait_for` on a deferred future ALWAYS returns `deferred` - it will never become `ready` until `get()` is called. Always use `std::launch::async` if you want real parallelism.
- **`wait_for` vs `wait_until`:** `wait_for` takes a duration (relative), `wait_until` takes a time_point (absolute). Use `wait_until` for deadline-based timeouts.
- **Exception propagation:** If the async task throws, `get()` re-throws the exception in the calling thread. `wait_for` does NOT throw - it just reports status.
- **Single-use:** `get()` can only be called once. Second call is UB. Use `shared_future` if multiple threads need the result.
- Compile with `-std=c++17 -O2 -pthread`.
