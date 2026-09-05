# Implement a task<T> coroutine type from scratch

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

`task<T>` is the fundamental coroutine return type for lazy, single-consumer, asynchronous computations. It starts suspended and only runs when awaited. Understanding how to build one from scratch is probably the most educational thing you can do with C++20 coroutines - it forces you to understand every piece of the machinery.

### task<T> Anatomy

The diagram below shows the sequence of events when a caller `co_await`s a `task<int>`. Notice that the initial suspend immediately hands control back to the caller, and it is only the `co_await` call that starts the task running. When the task finishes, it uses symmetric transfer to jump directly back to the caller without going through an intermediate stack frame:

```cpp
task<int> compute()          Caller
    │                           │
    ├─ initial_suspend:         │
    │   suspend_always      ────┤ co_await compute()
    │   [suspended]              │ await_suspend: stores caller handle
    │                            │ caller suspends too
    ├─ resume() called      <───┤
    │   runs body...             │
    ├─ co_return 42              │
    │   return_value(42)         │
    ├─ final_suspend:            │
    │   symmetric transfer  ────>| await_resume: returns 42
    │   [destroyed later]        │ caller continues
```

### Required Promise Members

Each member of the promise type has a specific job. If any of these are missing or wrong, the coroutine either won't compile or will corrupt memory:

| Member | Purpose |
| --- | --- |
| `get_return_object()` | Creates the `task<T>` from the promise |
| `initial_suspend()` | Returns `suspend_always` -> lazy start |
| `final_suspend()` | Returns symmetric transfer awaitable -> resumes awaiter |
| `return_value(T)` | Stores the co_return value |
| `unhandled_exception()` | Stores the exception for later rethrow |

---

## Self-Assessment

### Q1: Write a minimal `task<T>` with promise_type, get_return_object, initial_suspend, final_suspend, and return_value

Here is the full implementation. Pay particular attention to `final_suspend` - this is where symmetric transfer happens, and getting it wrong is the most common source of crashes and use-after-free bugs in coroutine code:

```cpp
#include <coroutine>
#include <exception>
#include <iostream>
#include <optional>
#include <utility>

template<typename T>
class task {
public:
    struct promise_type {
        std::optional<T> value;
        std::exception_ptr exception;
        std::coroutine_handle<> continuation;  // who's awaiting us

        task get_return_object() {
            return task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }  // lazy

        // Symmetric transfer: resume the awaiter when done
        auto final_suspend() noexcept {
            struct FinalAwaiter {
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> h) noexcept {
                    auto& cont = h.promise().continuation;
                    return cont ? cont : std::noop_coroutine();
                }
                void await_resume() noexcept {}
            };
            return FinalAwaiter{};
        }

        void return_value(T val) { value = std::move(val); }
        void unhandled_exception() { exception = std::current_exception(); }
    };

private:
    std::coroutine_handle<promise_type> handle_;

    explicit task(std::coroutine_handle<promise_type> h) : handle_(h) {}

public:
    task(task&& other) noexcept : handle_(std::exchange(other.handle_, {})) {}
    ~task() { if (handle_) handle_.destroy(); }

    task(const task&) = delete;
    task& operator=(const task&) = delete;

    // Make task itself awaitable
    bool await_ready() const noexcept { return false; }

    std::coroutine_handle<> await_suspend(std::coroutine_handle<> caller) noexcept {
        handle_.promise().continuation = caller;  // remember who to resume
        return handle_;  // symmetric transfer: start running this task
    }

    T await_resume() {
        auto& promise = handle_.promise();
        if (promise.exception)
            std::rethrow_exception(promise.exception);
        return std::move(*promise.value);
    }

    // Synchronous getter (for main)
    T get() {
        handle_.resume();
        auto& promise = handle_.promise();
        if (promise.exception)
            std::rethrow_exception(promise.exception);
        return std::move(*promise.value);
    }
};

// Usage
task<int> add(int a, int b) {
    co_return a + b;
}

task<int> multiply(int a, int b) {
    co_return a * b;
}

int main() {
    auto result = add(3, 4);
    std::cout << "3 + 4 = " << result.get() << '\n';

    auto result2 = multiply(5, 6);
    std::cout << "5 * 6 = " << result2.get() << '\n';
}
// Expected output:
// 3 + 4 = 7
// 5 * 6 = 30
```

Walking through the key design decisions: `initial_suspend()` returns `suspend_always`, so the coroutine doesn't run until `resume()` or `co_await`. `return_value()` stores the result in `std::optional<T>`. `final_suspend()` uses symmetric transfer to resume the continuation without growing the call stack. The destructor destroys the coroutine frame via `handle_.destroy()`.

### Q2: Add `co_await` support so `task<T>` can await another `task<U>`

Because `task<T>` already has `await_ready`, `await_suspend`, and `await_resume` defined as public members, it is already awaitable - you can `co_await` it directly from inside another coroutine. Here is what that looks like in practice, including multi-level chaining:

```cpp
// Using the task<T> from Q1:

task<int> compute_inner() {
    co_return 42;
}

task<std::string> compute_outer() {
    // co_await another task — this works because task<T> has
    // await_ready, await_suspend, await_resume
    int value = co_await compute_inner();  // suspends, runs inner, gets 42

    co_return "The answer is " + std::to_string(value);
}

task<int> chain_example() {
    // Multi-level chaining:
    task<int> a = add(10, 20);       // lazy — not running yet
    task<int> b = multiply(3, 4);    // lazy — not running yet

    int x = co_await a;  // runs add, gets 30
    int y = co_await b;  // runs multiply, gets 12

    co_return x + y;     // 42
}

// The key is the await_suspend method in task<T>:
//   std::coroutine_handle<> await_suspend(std::coroutine_handle<> caller) {
//       handle_.promise().continuation = caller;  // store who to resume
//       return handle_;  // symmetric transfer: run the awaited task
//   }
//
// Flow:
//   outer calls co_await inner
//   -> inner.await_suspend(outer_handle)
//   -> inner.handle_.promise().continuation = outer_handle  (remember outer)
//   -> return inner.handle_  (start inner via symmetric transfer)
//   -> inner runs, hits co_return
//   -> inner.final_suspend() -> symmetric transfer to continuation (= outer)
//   -> outer.await_resume() reads inner's value
```

### Q3: Explain the transfer of ownership between awaiter and awaitable in the task type

Ownership in a coroutine type is all about who holds the `coroutine_handle` and who is responsible for destroying the frame. Getting this wrong leads to double-frees or leaked frames. The `task<T>` uses RAII - the task object owns the handle, and the destructor calls `handle_.destroy()`. The two scenarios are:

```cpp
                    task<int> t = compute();
                         │
                         │ owns coroutine_handle (via RAII)
                         │ ~task() calls handle_.destroy()
                         │
  ┌──────────────────┴───────────────────┐
  │ Scenario A: Never awaited         │
  │                                    │
  │ task goes out of scope             │
  │ -> destructor destroys handle      │
  │ -> coroutine frame freed           │
  └───────────────────────────────────────┘

  ┌───────────────────────────────────────┐
  │ Scenario B: Awaited by caller      │
  │                                    │
  │ int x = co_await t;                │
  │ -> t.await_suspend sets continuation│
  │ -> symmetric transfer runs task     │
  │ -> task completes, resumes caller   │
  │ -> t goes out of scope, destroyed   │
  └───────────────────────────────────────┘
```

The four rules that follow from this ownership model are:

1. **Only one owner:** The `task<T>` object owns the coroutine handle. Move semantics transfer ownership.
2. **Single consumer:** A `task<T>` should be `co_await`ed at most once. The continuation is set once.
3. **Destruction timing:** The coroutine must be at a suspension point when destroyed. `final_suspend` returning `suspend_always` (or a transferring awaitable) ensures this.
4. **Move-only:** `task<T>` is non-copyable. Copying a coroutine handle would create two owners - double-free.

---

## Notes

- `task<T>` is always **lazy** (`initial_suspend = suspend_always`). Eager tasks use `suspend_never`.
- The `final_suspend` should **never** return `suspend_never` - that destroys the coroutine before the awaiter can read the result.
- Libraries like cppcoro, folly::coro, and libunifex provide production-quality `task<T>` implementations.
- `task<void>` needs `return_void()` instead of `return_value(T)`.
