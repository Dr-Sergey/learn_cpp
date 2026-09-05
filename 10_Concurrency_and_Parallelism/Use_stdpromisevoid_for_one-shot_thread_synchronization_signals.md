# Use std::promise<void> for one-shot thread synchronization signals

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/promise>  

---

## Topic Overview

`std::promise<void>` paired with `std::future<void>` creates a one-shot signal - no data is transferred, just a "go" or "done" notification from one thread to another. It's the simplest cross-thread synchronization primitive that doesn't require a condition variable, a mutex, or any shared flag.

### Core Idea

The pattern is straightforward. Thread A holds the `promise` and calls `set_value()` when it has finished some initialization or reached a checkpoint. Thread B holds the `future` and calls `get()`, which blocks until `set_value()` is called. Here is what that looks like side by side:

```cpp
Thread A:                          Thread B:
─────────                          ─────────
std::promise<void> ready;          auto fut = ready.get_future();
// ... initialize ...              fut.get(); // blocks until set_value()
ready.set_value();                 // guaranteed: init is complete
```

The reason this is so useful is the memory ordering guarantee: everything Thread A did before calling `set_value()` is visible to Thread B after `get()` returns. You don't need an explicit memory fence or extra synchronization.

### When to Use

Each of the mechanisms in the table below has a different sweet spot. If you just need a one-shot "we're ready" signal in C++11 code, `promise<void>` is the cleanest option.

| Mechanism | Use case |
| --- | --- |
| `promise<void>` | One-shot signal, C++11, no mutex needed |
| `condition_variable` | Repeated signals, complex predicates |
| `binary_semaphore` | One-shot or repeated, C++20 |
| `latch(1)` | One-shot signal, C++20, simpler syntax |
| `atomic::wait` | One-shot or repeated, C++20, no mutex |

---

## Self-Assessment

### Q1: Use promise<void> and its future to signal a thread that initialization is complete

This is the classic use case. A server starts up in a background thread, does some initialization, and then signals the main thread that it is ready to accept connections. The main thread can't proceed safely until that signal arrives.

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <chrono>

class Server {
    int port_ = 0;
    bool running_ = false;

public:
    void start(std::promise<void> init_done) {
        // Simulate initialization
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        port_ = 8080;
        running_ = true;

        // Signal: initialization is complete
        init_done.set_value();
        // ^^^ After this, any thread calling fut.get() will proceed

        // Continue running server...
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }

    int port() const { return port_; }
};

int main() {
    Server server;
    std::promise<void> init_promise;
    auto init_future = init_promise.get_future();

    // Start server in background
    std::thread server_thread(&Server::start, &server,
                              std::move(init_promise));

    // Wait for initialization to complete
    std::cout << "Waiting for server to initialize...\n";
    init_future.get(); // blocks until set_value() is called
    // Everything server did BEFORE set_value() is now visible

    std::cout << "Server ready on port " << server.port() << "\n";

    server_thread.join();

    // Output:
    // Waiting for server to initialize...
    // Server ready on port 8080
}
```

Notice that `port_` is accessed safely after `init_future.get()` returns, even though it's not atomic and no mutex is involved. The promise-future pair provides the necessary happens-before relationship.

### Q2: Show the difference between promise<void>::set_value() and notify_one/notify_all

The fundamental difference between `promise<void>` and `condition_variable` is that the promise remembers its signal. If you call `set_value()` before anyone calls `get()`, the signal is not lost - the next `get()` returns immediately. A raw `notify_one()` with no flag is lost if nobody is waiting yet.

```cpp
#include <future>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <iostream>
#include <vector>

int main() {
    // === promise<void>: exactly ONE signal, remembered ===
    {
        std::cout << "--- promise<void> ---\n";
        std::promise<void> prom;
        auto fut = prom.get_future();

        // Signal BEFORE anyone waits - NOT lost!
        prom.set_value();

        // Wait AFTER signal - returns immediately
        fut.get();
        std::cout << "Signal was remembered (not lost)\n";

        // Can't signal again:
        // prom.set_value(); // THROWS: promise_already_satisfied

        // Can't wait again:
        // fut.get(); // UB: future already retrieved (no shared state)
    }

    // === condition_variable: repeated, but signal can be lost ===
    {
        std::cout << "\n--- condition_variable ---\n";
        std::mutex mtx;
        std::condition_variable cv;
        bool ready = false;

        // Without the flag, signal can be LOST:
        // cv.notify_one(); // if no one is waiting -> notification gone!

        // With the flag (correct pattern):
        {
            std::lock_guard lock(mtx);
            ready = true;
        }
        cv.notify_one();

        // Can wait and check the flag (works because flag persists):
        {
            std::unique_lock lock(mtx);
            cv.wait(lock, [&] { return ready; }); // immediate: ready is true
        }
        std::cout << "Works because the flag persists\n";

        // Can signal AGAIN (reusable):
        ready = false;
        {
            std::lock_guard lock(mtx);
            ready = true;
        }
        cv.notify_one();
    }

    // === Key differences ===
    // promise<void>:
    //   Signal is STORED - never lost
    //   No mutex needed
    //   ONE TIME only (set_value once, get once)
    //   Cannot signal multiple waiters (unless shared_future)
    //
    // condition_variable:
    //   Signal can be lost (need a flag)
    //   Needs a mutex
    //   Reusable (can notify many times)
    //   notify_all wakes all waiters

    // === Multiple waiters with shared_future ===
    {
        std::cout << "\n--- shared_future<void> (broadcast) ---\n";
        std::promise<void> prom;
        auto shared_fut = prom.get_future().share();
        // ^^^ shared_future: multiple threads can call get()

        std::vector<std::thread> waiters;
        for (int i = 0; i < 4; ++i) {
            waiters.emplace_back([shared_fut, i] {
                shared_fut.get(); // blocks until set_value
                std::cout << "Waiter " << i << " released\n";
            });
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        prom.set_value(); // release ALL waiters at once

        for (auto& t : waiters) t.join();
    }

    // Output:
    // --- promise<void> ---
    // Signal was remembered (not lost)
    //
    // --- condition_variable ---
    // Works because the flag persists
    //
    // --- shared_future<void> (broadcast) ---
    // Waiter 0 released
    // Waiter 2 released
    // Waiter 1 released
    // Waiter 3 released
}
```

The `shared_future` section at the bottom shows how to broadcast to multiple threads. A regular `future` can only be `get()`-ted once. Calling `.share()` upgrades it to a `shared_future<void>` that any number of threads can wait on - all of them wake up when `set_value()` is called.

### Q3: Demonstrate using promise<void> to implement a one-time event without a condition variable

Three practical patterns in one example: a start gate, error propagation, and a simple pipeline. These come up regularly in real concurrent code.

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <chrono>
#include <vector>

// === Pattern 1: Start gate - all threads start simultaneously ===
void start_gate_pattern() {
    std::cout << "--- Start gate ---\n";
    std::promise<void> go_signal;
    auto shared = go_signal.get_future().share();

    std::vector<std::thread> racers;
    for (int i = 0; i < 4; ++i) {
        racers.emplace_back([shared, i] {
            shared.get(); // all block here until go_signal fires
            auto now = std::chrono::steady_clock::now();
            std::cout << "Racer " << i << " started at "
                      << std::chrono::duration_cast<std::chrono::microseconds>(
                             now.time_since_epoch()).count() % 1'000'000
                      << " us\n";
        });
    }

    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::cout << "GO!\n";
    go_signal.set_value(); // release all racers simultaneously

    for (auto& r : racers) r.join();
}

// === Pattern 2: Error propagation as signal ===
void error_signal_pattern() {
    std::cout << "\n--- Error signal ---\n";
    std::promise<void> work_done;
    auto fut = work_done.get_future();

    std::thread worker([prom = std::move(work_done)]() mutable {
        try {
            // Simulate work that might fail
            throw std::runtime_error("disk full");
            prom.set_value(); // never reached
        } catch (...) {
            prom.set_exception(std::current_exception());
            // Now fut.get() will RETHROW this exception
        }
    });

    try {
        fut.get(); // rethrows the exception from the worker
    } catch (const std::runtime_error& e) {
        std::cout << "Caught from worker: " << e.what() << "\n";
    }

    worker.join();
}

// === Pattern 3: Pipeline stages connected by promise<void> ===
void pipeline_pattern() {
    std::cout << "\n--- Pipeline ---\n";
    std::promise<void> stage1_done, stage2_done;
    auto fut1 = stage1_done.get_future();
    auto fut2 = stage2_done.get_future();
    int data = 0;

    std::thread s1([&data, p = std::move(stage1_done)]() mutable {
        data = 10;
        std::cout << "Stage 1: data = " << data << "\n";
        p.set_value();
    });

    std::thread s2([&data, &fut1, p = std::move(stage2_done)]() mutable {
        fut1.get(); // wait for stage 1
        data *= 2;
        std::cout << "Stage 2: data = " << data << "\n";
        p.set_value();
    });

    std::thread s3([&data, &fut2] {
        fut2.get(); // wait for stage 2
        data += 5;
        std::cout << "Stage 3: data = " << data << "\n";
    });

    s1.join(); s2.join(); s3.join();
}

int main() {
    start_gate_pattern();
    error_signal_pattern();
    pipeline_pattern();

    // Output:
    // --- Start gate ---
    // GO!
    // Racer 0 started at 123456 us
    // Racer 2 started at 123458 us
    // Racer 1 started at 123460 us
    // Racer 3 started at 123462 us
    //
    // --- Error signal ---
    // Caught from worker: disk full
    //
    // --- Pipeline ---
    // Stage 1: data = 10
    // Stage 2: data = 20
    // Stage 3: data = 25
}
```

The error propagation pattern is particularly elegant: `set_exception(std::current_exception())` stores the exception in the shared state, and calling `get()` on the future rethrows it in the waiting thread. You get exception propagation across thread boundaries for free.

---

## Notes

- `promise<void>` vs boolean: `set_value()` takes no argument. It purely signals "done." The future doesn't carry data - just the event.
- No lost signals: unlike `condition_variable`, `set_value` is remembered. If `get()` is called after `set_value`, it returns immediately.
- One-shot only: `set_value` can be called exactly once. A second call throws `future_error(promise_already_satisfied)`.
- `shared_future`: call `fut.share()` to get a `shared_future<void>` that multiple threads can `get()` from. Essential for broadcast patterns.
- Broken promise: if a `promise` is destroyed without calling `set_value` or `set_exception`, `fut.get()` throws `future_error(broken_promise)`.
- C++20 alternatives: for one-shot signals, `std::latch(1)` and `std::binary_semaphore(0)` are more ergonomic.
- Compile with `-std=c++11 -O2 -pthread`.
