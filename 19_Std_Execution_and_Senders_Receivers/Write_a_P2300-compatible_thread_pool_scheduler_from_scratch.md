# Write a P2300-compatible thread pool scheduler from scratch

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

Writing your own scheduler is the best way to internalize how P2300 actually works under the hood. The whole machinery - `scheduler`, `sender`, `operation_state`, `receiver` - becomes much less abstract once you have built a minimal but working version.

A P2300 scheduler must provide `schedule()` returning a sender. The sender, when connected to a receiver and started, enqueues work on the thread pool and notifies the receiver upon completion. The flow looks like this:

```cpp
Scheduler architecture:

  scheduler::schedule()
      |
      v
  pool_sender
      |
  connect(receiver) -> operation_state
      |
  start(op_state):
      |
      +- check stop_token
      |     +- stopped? -> receiver.set_stopped()
      |     +- not stopped:
      |
      +- enqueue work on thread pool
      |
      +- pool thread runs:
            receiver.set_value()
```

---

## Self-Assessment

### Q1: Implement a scheduler whose `schedule()` returns a sender

Let's build this bottom-up. First comes a simple thread pool using a mutex-protected queue, then the P2300 glue types that wrap it. Read through the three pieces in order - `thread_pool`, `pool_sender`, and `pool_scheduler` - and notice how each layer only needs to know about the one below it:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <vector>
#include <iostream>

// Simple thread pool:
class thread_pool {
public:
    explicit thread_pool(int num_threads) {
        for (int i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] { worker_loop(); });
        }
    }

    ~thread_pool() {
        {
            std::lock_guard lk(mtx_);
            shutdown_ = true;
        }
        cv_.notify_all();
        for (auto& t : workers_) t.join();
    }

    void enqueue(std::function<void()> task) {
        {
            std::lock_guard lk(mtx_);
            tasks_.push(std::move(task));
        }
        cv_.notify_one();
    }

private:
    void worker_loop() {
        while (true) {
            std::function<void()> task;
            {
                std::unique_lock lk(mtx_);
                cv_.wait(lk, [&] { return shutdown_ || !tasks_.empty(); });
                if (shutdown_ && tasks_.empty()) return;
                task = std::move(tasks_.front());
                tasks_.pop();
            }
            task();
        }
    }

    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool shutdown_ = false;
};

// Forward declarations:
class pool_scheduler;

// The sender returned by schedule():
class pool_sender {
public:
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(),
        stdexec::set_stopped_t()
    >;

    explicit pool_sender(thread_pool* pool) : pool_(pool) {}

    // connect: creates operation_state
    template <class Receiver>
    class operation {
    public:
        operation(thread_pool* pool, Receiver recv)
            : pool_(pool), recv_(std::move(recv)) {}

        void start() noexcept {
            pool_->enqueue([this]() {
                stdexec::set_value(std::move(recv_));
            });
        }

    private:
        thread_pool* pool_;
        Receiver recv_;
    };

    template <class Receiver>
    auto connect(Receiver recv) {
        return operation<Receiver>{pool_, std::move(recv)};
    }

private:
    thread_pool* pool_;
};

// The scheduler:
class pool_scheduler {
public:
    using scheduler_concept = stdexec::scheduler_t;

    explicit pool_scheduler(thread_pool* pool) : pool_(pool) {}

    pool_sender schedule() const noexcept {
        return pool_sender{pool_};
    }

    bool operator==(const pool_scheduler&) const = default;

private:
    thread_pool* pool_;
};

int main() {
    thread_pool pool(4);
    pool_scheduler sched(&pool);

    // Use our custom scheduler with stdexec:
    auto pipeline = stdexec::schedule(sched)
        | stdexec::then([]() {
            std::cout << "Running on pool thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 42
}
```

The fact that `stdexec::schedule(sched) | stdexec::then(...)` just works with our custom scheduler is the payoff. We satisfied the concept requirements (`sender_concept`, `completion_signatures`, `connect`), and the standard library machinery treated our type as a first-class scheduler.

### Q2: `operation_state` - `start()` enqueues, receiver notified on completion

The `operation_state` is the piece that actually owns the in-flight work. Understanding its lifecycle is essential because the rules around it are strict and breaking them causes hard-to-debug bugs. Here is the lifecycle annotated, followed by a more complete implementation that handles errors from the enqueue operation:

```cpp
// The operation_state lifecycle:
//
// 1. connect(sender, receiver) -> operation_state
//    - Stores sender's data + receiver
//    - No work happens yet
//
// 2. start(operation_state)
//    - Enqueues a task on the thread pool
//    - The task calls receiver.set_value() when done
//
// 3. Pool thread picks up the task:
//    - Calls receiver.set_value() (or set_error/set_stopped)
//    - Receiver processes the completion

template <class Receiver>
class pool_operation {
public:
    using operation_state_concept = stdexec::operation_state_t;

    pool_operation(thread_pool* pool, Receiver recv)
        : pool_(pool), recv_(std::move(recv)) {}

    // start() MUST NOT throw (noexcept):
    void start() noexcept {
        try {
            pool_->enqueue([this]() {
                // Notify the receiver on the pool thread:
                stdexec::set_value(std::move(recv_));
            });
        } catch (...) {
            // If enqueue fails, signal error:
            stdexec::set_error(std::move(recv_),
                std::current_exception());
        }
    }

private:
    thread_pool* pool_;
    Receiver recv_;
};

// Key requirements:
// - operation_state is NOT movable after start()
// - start() is called exactly once
// - Receiver must be notified exactly once (value OR error OR stopped)
// - The operation_state must remain alive until the receiver is notified
```

The "exactly once" rule for receiver notifications is the most important constraint. If you call `set_value` and then also `set_error`, the behavior is undefined. If you never call any of them, the caller will block forever.

### Q3: Cancellation support via stop token

Cancellation in P2300 works through stop tokens. The receiver carries a stop token, and the operation can query it to decide whether to actually do the work or short-circuit with `set_stopped`. The key question is when to check: before enqueuing (cheap) and again on the pool thread (correct for timing-sensitive cases):

```cpp
#include <stop_token>

template <class Receiver>
class cancellable_operation {
public:
    using operation_state_concept = stdexec::operation_state_t;

    cancellable_operation(thread_pool* pool, Receiver recv)
        : pool_(pool), recv_(std::move(recv)) {}

    void start() noexcept {
        // Check the stop token BEFORE enqueueing:
        auto stop_token = stdexec::get_stop_token(
            stdexec::get_env(recv_));

        if (stop_token.stop_requested()) {
            // Already cancelled, don't enqueue:
            stdexec::set_stopped(std::move(recv_));
            return;
        }

        // Register a stop callback to handle cancellation
        // while waiting in the queue:
        pool_->enqueue([this, stop_token]() {
            // Check again on the pool thread:
            if (stop_token.stop_requested()) {
                stdexec::set_stopped(std::move(recv_));
            } else {
                stdexec::set_value(std::move(recv_));
            }
        });
    }

private:
    thread_pool* pool_;
    Receiver recv_;
};

// Updated sender with cancellation in completion_signatures:
class cancellable_pool_sender {
public:
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(),        // normal completion
        stdexec::set_error_t(std::exception_ptr),  // error
        stdexec::set_stopped_t()       // cancellation
    >;
    // ...
};

// Usage:
// auto pipeline = stdexec::schedule(sched)
//     | stdexec::then([]() { return 42; });
//
// If a stop is requested before the work runs,
// the pipeline completes with set_stopped() instead of set_value().
```

The double-check pattern - once before enqueue and once on the thread - is a common real-world technique. The first check avoids wasting a queue slot for work that is already cancelled. The second check handles cancellation that arrives while the task is sitting in the queue waiting for a thread.

---

## Notes

- A scheduler must satisfy the `scheduler` concept: has `schedule()` returning a sender.
- `operation_state` must be non-movable after `start()` (it's pinned in memory).
- `start()` must be `noexcept` - errors go through `set_error`, not exceptions.
- The receiver is notified exactly once with exactly one of: value, error, or stopped.
- Real implementations use lock-free queues and work-stealing for performance.
- The `get_env` / `get_stop_token` protocol connects receivers to cancellation.
