# Combine Coroutines with `std::execution` for Structured Async Concurrency

**Category:** Concurrency and Parallelism  
**Standard:** C++20, C++26, C++23 (Coroutines)  
**Reference:** [P2300 - std::execution](https://wg21.link/P2300), [cppreference - Coroutines](https://en.cppreference.com/w/cpp/language/coroutines)  

---

## Topic Overview

C++20 coroutines and the `std::execution` sender/receiver framework (P2300, targeting C++26) are two complementary approaches to asynchronous programming. They solve different parts of the problem, and they work beautifully together.

Coroutines give you a **sequential-looking syntax** for async control flow - you write code that reads top-to-bottom even when the actual execution jumps between threads and suspends at arbitrary points. Senders/receivers give you a **compositional, value-oriented graph** of async operations with structured concurrency guarantees baked in. Combining them yields code that is both readable and architecturally sound.

The core insight is that a **sender** represents a description of work that *will* produce a value - it is lazy, it has not started yet. A **coroutine** provides a natural syntax to *await* that value. The bridge is an `operator co_await` for senders: when a coroutine `co_await`s a sender, it suspends until the sender completes, then resumes with the result. This lets you write code like this:

```cpp
task<int> compute() {
    int a = co_await on(thread_pool, just(42) | then([](int x) { return x * 2; }));
    int b = co_await on(thread_pool, just(10) | then([](int x) { return x + 5; }));
    co_return a + b;
}
```

The coroutine reads sequentially, but each `co_await` hands off to the sender framework, which may run the work on a thread pool and resume the coroutine when the result is ready.

| Aspect | Coroutines (C++20) | Senders/Receivers (P2300) | Combined |
| --- | --- | --- | --- |
| **Syntax** | `co_await`, `co_return` | Pipe operator, algorithm composition | Coroutine body with sender expressions |
| **Composition** | Sequential by default | Graph-based, supports `when_all` | Both: sequential flow + parallel fan-out |
| **Error handling** | Exceptions or return codes | `set_error` channel | Exceptions from sender errors |
| **Cancellation** | Via `stop_token` in promise | `set_stopped` channel, stop tokens | Structured: parent cancellation propagates |
| **Scheduling** | Manual (post to executor) | `on`, `transfer`, `schedule` | Coroutine resumes on sender's scheduler |
| **Structured concurrency** | Not inherent | Built-in (`when_all` scoping) | Enforced through sender lifetime |

The diagram below shows what happens when a coroutine `co_await`s a sender. The coroutine suspends, the sender gets connected to a receiver embedded in the coroutine's promise, the async work runs, and then the coroutine resumes with the result:

```cpp
+------------------ Coroutine Frame -----------------------+
|                                                           |
|  task<int> work() {                                       |
|      auto sender = just(42) | then([](int x){ return x; });|
|                    ^                                      |
|                    | sender describes work                |
|                    |                                      |
|      int val = co_await sender;                           |
|                    |                                      |
|           +-------+--------+                              |
|           |  Suspend       |  <- coroutine suspends        |
|           |  Connect       |  <- sender connected to       |
|           |  + Start       |    receiver in promise        |
|           +-------+--------+                              |
|                   | async work executes...                |
|                   |                                       |
|           +-------+--------+                              |
|           |  set_value(42) |  <- sender completes          |
|           |  Resume        |  <- coroutine resumes         |
|           +-------+--------+                              |
|                   v                                       |
|      co_return val;                                       |
|  }                                                        |
+-----------------------------------------------------------+
```

The benefit of this combination over raw coroutines is **structured concurrency**. The sender/receiver model guarantees that all child operations complete before the parent scope exits - even under cancellation. This eliminates dangling references and use-after-free bugs that plague "fire-and-forget" coroutine designs. The benefit over raw senders is **readability**: sequential async logic reads like synchronous code instead of deeply nested algorithm trees.

---

## Self-Assessment

### Q1: Implement a basic `task<T>` coroutine type that can `co_await` a simple sender-like object

This is a lot of boilerplate, but reading through it is worth the investment. The `async_value` type acts as a minimal stand-in for a real P2300 sender. Pay attention to how `await_suspend` is the seam where the coroutine hands control to the async world, and `await_resume` is where it picks up the result on the way back:

```cpp
// Compile: g++ -std=c++20 -pthread -fcoroutines q1_task_sender.cpp -o q1
#include <coroutine>
#include <functional>
#include <iostream>
#include <thread>
#include <utility>

// ---- Minimal sender: produces a value asynchronously ----
template <typename T>
struct async_value {
    T value_;
    explicit async_value(T v) : value_(std::move(v)) {}

    // Awaiter interface - bridges sender concept to co_await
    bool await_ready() const noexcept { return false; }

    void await_suspend(std::coroutine_handle<> h) const {
        // Simulate async: resume on a new thread
        std::thread([h, this]() {
            // In a real sender, this is where connect+start happens
            h.resume();
        }).detach();
    }

    T await_resume() const noexcept { return value_; }
};

// ---- Minimal task<T> coroutine ----
template <typename T>
struct task {
    struct promise_type {
        T result_;
        std::coroutine_handle<> continuation_;

        task get_return_object() {
            return task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }

        struct final_awaiter {
            bool await_ready() noexcept { return false; }
            void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
                if (h.promise().continuation_)
                    h.promise().continuation_.resume();
            }
            void await_resume() noexcept {}
        };

        final_awaiter final_suspend() noexcept { return {}; }
        void return_value(T v) { result_ = std::move(v); }
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle_;

    explicit task(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~task() { if (handle_) handle_.destroy(); }

    task(const task&) = delete;
    task(task&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}

    // Make task itself awaitable (for nesting)
    bool await_ready() const noexcept { return false; }
    void await_suspend(std::coroutine_handle<> caller) noexcept {
        handle_.promise().continuation_ = caller;
        handle_.resume();  // start the task
    }
    T await_resume() const noexcept { return handle_.promise().result_; }

    T sync_wait() {
        handle_.resume();  // start coroutine
        // Spin until done (production code uses a better mechanism)
        while (!handle_.done())
            std::this_thread::yield();
        return handle_.promise().result_;
    }
};

// ---- Usage: coroutine co_awaits sender-like objects ----
task<int> compute() {
    int a = co_await async_value<int>(42);
    int b = co_await async_value<int>(58);
    co_return a + b;
}

int main() {
    auto t = compute();
    int result = t.sync_wait();
    std::cout << "Result: " << result << "\n";  // 100
    return 0;
}
```

The `async_value` acts as a minimal sender proxy. Its `await_suspend` replaces the connect+start phase of a real sender, and `await_resume` delivers the value. In production senders (P2300), the `operator co_await` on a sender automatically builds a receiver that resumes the coroutine on `set_value`.

---

### Q2: Show structured concurrency by running multiple senders in parallel within a coroutine using a `when_all`-like primitive

The goal here is to fan out two tasks in parallel and then rejoin before the parent coroutine continues. The `when_all_awaitable` type is the key - notice how it uses an atomic counter to detect when both children have finished, and only then resumes the parent:

```cpp
// Compile: g++ -std=c++20 -pthread -fcoroutines q2_when_all.cpp -o q2
#include <coroutine>
#include <iostream>
#include <thread>
#include <tuple>
#include <atomic>
#include <utility>
#include <exception>
#include <string>

// Simplified task<T> (same as Q1, abbreviated)
template <typename T>
struct task {
    struct promise_type {
        T result_;
        std::coroutine_handle<> continuation_;
        task get_return_object() {
            return task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        struct final_awaiter {
            bool await_ready() noexcept { return false; }
            void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
                if (h.promise().continuation_) h.promise().continuation_.resume();
            }
            void await_resume() noexcept {}
        };
        final_awaiter final_suspend() noexcept { return {}; }
        void return_value(T v) { result_ = std::move(v); }
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle_;
    explicit task(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~task() { if (handle_) handle_.destroy(); }
    task(task&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}
    bool await_ready() const noexcept { return false; }
    void await_suspend(std::coroutine_handle<> c) noexcept {
        handle_.promise().continuation_ = c;
        handle_.resume();
    }
    T await_resume() const noexcept { return handle_.promise().result_; }
    T sync_wait() {
        handle_.resume();
        while (!handle_.done()) std::this_thread::yield();
        return handle_.promise().result_;
    }
};

// ---- when_all: awaitable that runs two tasks in parallel ----
template <typename A, typename B>
struct when_all_awaitable {
    task<A> ta_;
    task<B> tb_;
    A result_a_;
    B result_b_;
    std::atomic<int> remaining_{2};
    std::coroutine_handle<> parent_;

    when_all_awaitable(task<A>&& a, task<B>&& b)
        : ta_(std::move(a)), tb_(std::move(b)) {}

    bool await_ready() const noexcept { return false; }

    void await_suspend(std::coroutine_handle<> h) noexcept {
        parent_ = h;
        // Launch both tasks on separate threads
        std::thread([this]() {
            ta_.handle_.resume();
            while (!ta_.handle_.done()) std::this_thread::yield();
            result_a_ = ta_.handle_.promise().result_;
            if (remaining_.fetch_sub(1, std::memory_order_acq_rel) == 1)
                parent_.resume();
        }).detach();

        std::thread([this]() {
            tb_.handle_.resume();
            while (!tb_.handle_.done()) std::this_thread::yield();
            result_b_ = tb_.handle_.promise().result_;
            if (remaining_.fetch_sub(1, std::memory_order_acq_rel) == 1)
                parent_.resume();
        }).detach();
    }

    std::tuple<A, B> await_resume() const noexcept {
        return {result_a_, result_b_};
    }
};

template <typename A, typename B>
auto when_all(task<A>&& a, task<B>&& b) {
    return when_all_awaitable<A, B>(std::move(a), std::move(b));
}

// ---- Worker coroutines ----
task<int> fetch_from_db() {
    // Simulate async DB query
    co_return 42;
}

task<std::string> fetch_from_api() {
    co_return std::string("hello");
}

task<std::string> orchestrate() {
    // Structured concurrency: both tasks run in parallel,
    // both must complete before we proceed.
    auto [db_result, api_result] = co_await when_all(
        fetch_from_db(), fetch_from_api());

    co_return api_result + " " + std::to_string(db_result);
}

int main() {
    auto t = orchestrate();
    auto result = t.sync_wait();
    std::cout << "Combined: " << result << "\n";  // "hello 42"
    return 0;
}
```

`when_all` is the structured concurrency primitive. It guarantees both child tasks complete before the parent coroutine resumes. In P2300, `execution::when_all(sender1, sender2)` returns a sender that completes with a tuple of results. If either child fails or is cancelled, the other is cancelled too - preventing resource leaks. The simplified version above omits cancellation but demonstrates the core pattern.

---

### Q3: Show when you should prefer pure senders over coroutines, and vice versa

The two models have genuinely different strengths, and the real skill is knowing which to reach for. Think of it this way: senders are for building infrastructure, coroutines are for writing business logic that uses that infrastructure.

```cpp
// Compile: g++ -std=c++20 -pthread -fcoroutines q3_sender_vs_coro.cpp -o q3
#include <concepts>
#include <functional>
#include <iostream>
#include <thread>
#include <vector>
#include <coroutine>
#include <exception>
#include <type_traits>
#include <utility>

// ---- Pure sender style: composable pipeline (pseudo P2300) ----
// Senders are lazy descriptions of work. They compose without allocating
// coroutine frames, making them ideal for building reusable pipelines.

template <typename F>
struct then_sender {
    F func_;
    // In P2300: connect() produces an operation_state
    // start() begins execution, set_value() delivers result
    auto execute(int input) { return func_(input); }
};

template <typename F>
auto then(F&& f) { return then_sender<std::decay_t<F>>{std::forward<F>(f)}; }

// Senders compose naturally into DAGs:
void sender_pipeline_example() {
    // This is a DESCRIPTION of work, not execution.
    // No coroutine frame, no heap allocation.
    auto pipeline = then([](int x) { return x * 2; });
    auto pipeline2 = then([](int x) { return x + 10; });

    // Execute the pipeline (in P2300: sync_wait connects + starts)
    int result = pipeline2.execute(pipeline.execute(21));
    std::cout << "Sender pipeline: " << result << "\n";  // 52
}

// ---- Coroutine style: sequential async logic ----
// Coroutines excel at sequential control flow with branching,
// loops, and error handling.

struct simple_task {
    struct promise_type {
        int result_ = 0;
        simple_task get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
};

// Coroutines are best for complex control flow:
// - Retry loops with backoff
// - Conditional branching based on intermediate results
// - Resource cleanup (RAII in coroutine frame)

// ---- Decision matrix ----
void print_decision_matrix() {
    std::cout << "\n=== When to Use What ===\n\n";
    std::cout << "| Scenario                          | Prefer           | Why |\n";
    std::cout << "|-----------------------------------|------------------|-----|\n";
    std::cout << "| Reusable async pipeline           | Senders          | "
              << "Zero-alloc composition |\n";
    std::cout << "| Fan-out / fan-in parallelism      | Senders          | "
              << "when_all / when_any |\n";
    std::cout << "| Sequential async with branching   | Coroutines       | "
              << "Natural syntax |\n";
    std::cout << "| Retry loops with backoff          | Coroutines       | "
              << "Loop + co_await |\n";
    std::cout << "| Framework-level composition       | Senders          | "
              << "Type-erased pipelines |\n";
    std::cout << "| Application-level business logic  | Coroutines       | "
              << "Readability |\n";
    std::cout << "| Cancellation-critical code        | Both (combined)  | "
              << "Structured concurrency |\n";
}

int main() {
    sender_pipeline_example();
    print_decision_matrix();

    std::cout << "\nGuideline: Use senders to BUILD the async infrastructure.\n"
              << "Use coroutines to CONSUME it with readable business logic.\n"
              << "Bridge them with operator co_await on senders.\n";
    return 0;
}
```

Senders are optimal for building reusable, composable async infrastructure - schedulers, retry policies, timeout wrappers - because they compose at the type level without coroutine frame allocations. Coroutines are optimal for consuming that infrastructure in application code, providing sequential syntax for inherently sequential logic. The bridge (`co_await sender`) combines both: the infrastructure is a sender graph, and the business logic reads like synchronous code.

---

## Notes

- P2300 status: `std::execution` is targeting C++26. Implementations exist in stdexec (NVIDIA) and libunifex (Meta). Use these to experiment today.
- `operator co_await` for senders: P2300 defines `as_awaitable()` to adapt a sender into an awaiter. This is how `co_await some_sender` works - the sender is connected to a receiver embedded in the coroutine's promise type.
- Coroutine frame allocation: Each `co_await` in a coroutine does not allocate a new frame - the frame is allocated once at coroutine creation. However, nested coroutines each get their own frame. Senders avoid this by composing at compile time.
- Cancellation propagation: In P2300, stop tokens flow through the sender tree. When a parent is cancelled, `set_stopped()` propagates to children. Coroutines receive cancellation via `stop_token` in their promise type. The combined model makes this transparent.
- Error channels: Senders have `set_error(E)` alongside `set_value(T)` and `set_stopped()`. When a coroutine `co_await`s a sender that errors, the error is typically thrown as an exception in the coroutine. Use `let_error` to handle errors within the sender graph without exceptions.
- Do not mix and match blindly. Pure sender pipelines should own their lifetimes. Coroutines should `co_await` senders, not store sender handles across suspension points - that risks dangling references if the sender completes after the coroutine frame is destroyed.
