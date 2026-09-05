# Implement an awaitable timer that suspends and resumes after a delay

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

An **awaitable timer** suspends a coroutine and schedules its resumption after a specified delay. This combines coroutine machinery with an event loop or thread pool. It is also one of the clearest illustrations of what the awaitable protocol actually does - the three methods `await_ready`, `await_suspend`, and `await_resume` map directly onto "should we bother suspending?", "what do we do with the handle while suspended?", and "what value does this expression produce?".

### Awaitable Timer Flow

Here is the sequence of events when a coroutine hits `co_await timer(500ms)`. The coroutine suspends, hands its handle to the event loop, and has no further presence on any thread until the event loop calls `handle.resume()` after the delay:

```cpp
Coroutine                    Timer / Event Loop
   │                              │
   ├─ co_await timer(500ms)       │
   │   await_ready() = false      │
   │   await_suspend(handle) ────>│── schedule(handle, 500ms)
   │   [SUSPENDED]                │
   │                              │── ... 500ms pass ...
   │                              │── handle.resume()
   │   await_resume()        <────│
   ├─ continues execution         │
```

### Awaitable Protocol

| Method | Purpose | Return type |
| --- | --- | --- |
| `await_ready()` | Skip suspension if result already available | `bool` |
| `await_suspend(h)` | Schedule resumption (store/dispatch handle) | `void`, `bool`, or `coroutine_handle<>` |
| `await_resume()` | Return the co_await result | Any type |

---

## Self-Assessment

### Q1: Write an awaitable that suspends the coroutine and schedules its resumption via a timer

The simplest possible timer implementation spawns a thread, sleeps for the requested duration, then calls `handle.resume()`. This works for demonstration but is wasteful in production (a real timer uses OS-level facilities like `epoll` or `IOCP`). The important thing to notice is that after `await_suspend` returns, the coroutine is suspended on a thread-pool thread - it does not resume on the original thread:

```cpp
#include <chrono>
#include <coroutine>
#include <iostream>
#include <thread>

// Simple task coroutine type for demonstration
struct Task {
    struct promise_type {
        Task get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
};

// Awaitable timer: suspends the coroutine, resumes after a delay
struct TimerAwaitable {
    std::chrono::milliseconds duration;

    bool await_ready() const noexcept {
        return duration.count() <= 0;  // don't suspend for zero delay
    }

    void await_suspend(std::coroutine_handle<> handle) const {
        // Schedule resumption on a new thread after the delay
        std::thread([handle, d = duration]() {
            std::this_thread::sleep_for(d);
            handle.resume();  // resume the coroutine
        }).detach();
    }

    void await_resume() const noexcept {
        // Nothing to return — timer just delays
    }
};

// Helper function for nice syntax
TimerAwaitable async_sleep(std::chrono::milliseconds ms) {
    return TimerAwaitable{ms};
}

Task example() {
    using namespace std::chrono_literals;

    std::cout << "Start\n";
    co_await async_sleep(200ms);   // suspend for 200ms
    std::cout << "After 200ms\n";
    co_await async_sleep(300ms);   // suspend for 300ms more
    std::cout << "After 300ms more\n";
}

int main() {
    example();
    std::this_thread::sleep_for(std::chrono::seconds(1));  // wait for coroutine
}
// Expected output (with delays between lines):
// Start
// After 200ms
// After 300ms more
```

`await_ready()` returns `false` for non-zero delays, causing suspension. `await_suspend()` spawns a thread that sleeps then calls `handle.resume()`. The coroutine resumes on the timer thread, not the original thread. In production, use an event loop or thread pool instead of spawning threads per timer.

### Q2: Show the symmetric transfer technique for resuming coroutines without stack overflow

Without symmetric transfer, every `handle.resume()` call pushes a new stack frame. If you have a chain of a thousand coroutines each resuming the next, you get a thousand frames deep - stack overflow. Symmetric transfer solves this by making `await_suspend` return a `coroutine_handle<>`. The runtime then does a tail-call to resume that handle, so the chain of coroutines uses O(1) stack space no matter how long it is:

```cpp
#include <coroutine>
#include <iostream>

// Symmetric transfer: await_suspend returns a coroutine_handle
// instead of void. The compiler does a tail-call to resume the
// returned handle, avoiding stack buildup.

struct PingPong {
    struct promise_type {
        std::coroutine_handle<> next;  // the coroutine to resume

        PingPong get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }

        // Symmetric transfer at final_suspend:
        // instead of returning suspend_always, return the next handle
        auto final_suspend() noexcept {
            struct TransferAwaitable {
                std::coroutine_handle<> next;
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<>) noexcept {
                    return next ? next : std::noop_coroutine();
                }
                void await_resume() noexcept {}
            };
            return TransferAwaitable{next};
        }

        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;

    void set_next(PingPong& other) {
        handle.promise().next = other.handle;
    }
};

PingPong ping(int& counter) {
    while (counter < 5) {
        std::cout << "ping " << counter << '\n';
        ++counter;
        co_return;  // symmetric transfer to 'next'
    }
}

int main() {
    int counter = 0;

    auto p = ping(counter);
    p.handle.resume();  // prints: ping 0

    std::cout << "Counter: " << counter << '\n';
    p.handle.destroy();
}
// Expected output:
// ping 0
// Counter: 1
```

To summarise why this matters: without symmetric transfer, `handle.resume()` calls build up the stack (coroutine A resumes B resumes C -> stack overflow). With symmetric transfer, `await_suspend` returns a `coroutine_handle<>`, and the compiler does a **tail-call** - no stack growth. This pattern enables chains of thousands of coroutines.

### Q3: Explain the executor model: how resumption is dispatched to a thread pool

The executor model is the standard way to decouple coroutines from threads in production systems. The key idea is that `await_suspend` does not call `handle.resume()` directly - it pushes the handle into a queue, and a worker thread from the pool eventually picks it up and calls `resume()`. This means the coroutine resumes on a pool thread, not the thread that originally suspended it:

```cpp
Coroutine suspends
       │
       │ await_suspend(handle)
       ▼
┌──────────────────┐
│   Executor /       │
│   Event Loop       │  <- stores handle in a queue
│                    │
│  queue.push(handle)│
└─────────┬────────┘
          │
          │ When ready (timer, IO, event)
          ▼
┌──────────────────┐
│   Thread Pool      │
│                    │
│  handle.resume()   │  <- picks a worker thread to resume
└──────────────────┘
```

Here is a simplified thread pool executor that implements this pattern. The `ScheduleOn` awaitable is what a coroutine uses to hop onto the thread pool - it always suspends and immediately enqueues the handle:

```cpp
#include <condition_variable>
#include <coroutine>
#include <functional>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

class ThreadPoolExecutor {
    std::vector<std::thread> workers_;
    std::queue<std::coroutine_handle<>> queue_;
    std::mutex mutex_;
    std::condition_variable cv_;
    bool stop_ = false;

public:
    explicit ThreadPoolExecutor(int num_threads) {
        for (int i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    std::coroutine_handle<> handle;
                    {
                        std::unique_lock lock(mutex_);
                        cv_.wait(lock, [this] { return stop_ || !queue_.empty(); });
                        if (stop_ && queue_.empty()) return;
                        handle = queue_.front();
                        queue_.pop();
                    }
                    handle.resume();  // resume coroutine on this worker thread
                }
            });
        }
    }

    void schedule(std::coroutine_handle<> handle) {
        {
            std::lock_guard lock(mutex_);
            queue_.push(handle);
        }
        cv_.notify_one();
    }

    ~ThreadPoolExecutor() {
        { std::lock_guard lock(mutex_); stop_ = true; }
        cv_.notify_all();
        for (auto& w : workers_) w.join();
    }
};

// Awaitable that schedules resumption on the executor
struct ScheduleOn {
    ThreadPoolExecutor& executor;

    bool await_ready() { return false; }  // always suspend
    void await_suspend(std::coroutine_handle<> h) {
        executor.schedule(h);  // enqueue for thread pool
    }
    void await_resume() {}  // no result
};
```

The model in a nutshell: `await_suspend(handle)` stores the handle in the executor's queue (not immediately resumed). A worker thread picks up the handle and calls `handle.resume()`. The coroutine resumes on whichever thread the executor chooses. This decouples the coroutine from any specific thread.

---

## Notes

- **Never** call `handle.resume()` from inside `await_suspend()` unless you understand reentrancy. Use the executor pattern instead.
- Symmetric transfer (`await_suspend` returning `coroutine_handle<>`) is critical for preventing stack overflow in coroutine chains.
- `std::noop_coroutine()` is the "null" handle for symmetric transfer - returns control to the caller.
- Real-world timer implementations use OS-level timers (epoll, IOCP, kqueue) instead of `sleep_for`.
