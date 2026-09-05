# Understand co_return in coroutines and the final_suspend protocol

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

In C++20 coroutines, `co_return` is the statement that completes a coroutine and optionally
delivers a result. How the coroutine behaves **after** `co_return` - whether it stays alive
long enough for you to read that result, or destroys itself immediately - is controlled by
the `final_suspend()` method of the promise type. Getting this wrong leads to use-after-free.

### The Coroutine Lifecycle with co_return

Here's the step-by-step sequence that happens when a coroutine hits `co_return`:

```cpp
1. Caller creates coroutine -> initial_suspend()
2. Coroutine body runs -> may co_yield / co_await
3. co_return expr; ->

   a. promise.return_value(expr)    [or return_void() for co_return;]
   b. Destroy local variables (in reverse order)
   c. promise.final_suspend()

4. If final_suspend suspends  -> coroutine frame is preserved, caller destroys it

   If final_suspend doesn't suspend -> coroutine destroys itself immediately
```

### The Promise Type Requirements

Your promise type must provide exactly one of `return_value` or `return_void`, depending
on whether your coroutine returns a value. Having both is ill-formed:

```cpp
struct promise_type {
    // Called when co_return expr; is executed
    void return_value(T value);   // for co_return with a value
    // OR
    void return_void();           // for co_return; (no value)

    // Called after return_value/return_void - controls post-completion behavior
    auto final_suspend() noexcept;
    // Must return an awaitable (suspend_always, suspend_never, or custom)
};
```

### final_suspend: suspend_always vs suspend_never

This choice is the central design decision of any Task-like type. Get it wrong and you
have a dangling frame or a double-destroy:

| Choice | Behavior | Use When |
| --- | --- | --- |
| `suspend_always{}` | Coroutine suspends at end - frame stays alive | You need to read the result after completion |
| `suspend_never{}` | Coroutine destroys itself immediately | Fire-and-forget; nothing reads the result |

### Complete Task<T> Implementation

Here's a minimal but complete `Task<T>` that puts all the pieces together. Notice that
`final_suspend` returns `suspend_always` so the `get()` method can safely access the result:

```cpp
#include <coroutine>
#include <iostream>
#include <optional>

template<typename T>
struct Task {
    struct promise_type {
        std::optional<T> result;

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }

        // Suspend at final point so the caller can retrieve the result
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T value) {
            result = std::move(value);
        }

        void unhandled_exception() {
            std::terminate();
        }
    };

    std::coroutine_handle<promise_type> handle;

    explicit Task(std::coroutine_handle<promise_type> h) : handle(h) {}
    ~Task() { if (handle) handle.destroy(); }

    // Non-copyable, movable
    Task(const Task&) = delete;
    Task(Task&& other) noexcept : handle(other.handle) { other.handle = nullptr; }

    T get() {
        if (!handle.done()) handle.resume();
        return *handle.promise().result;
    }
};
```

---

## Self-Assessment

### Q1: Implement a task coroutine whose final_suspend returns the result to an awaiting coroutine

Let's put the `Task<T>` to use with two simple coroutines. The key thing to observe is how
`get()` drives the lazy coroutine and then reads the result from the promise:

```cpp
#include <coroutine>
#include <iostream>
#include <optional>

template<typename T>
struct Task {
    struct promise_type {
        std::optional<T> result;

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T val) { result = std::move(val); }
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;

    explicit Task(std::coroutine_handle<promise_type> h) : handle(h) {}
    ~Task() { if (handle) handle.destroy(); }
    Task(Task&& o) noexcept : handle(o.handle) { o.handle = nullptr; }
    Task(const Task&) = delete;

    T get() {
        if (!handle.done()) handle.resume();
        return *handle.promise().result;
    }
};

Task<int> compute_answer() {
    co_return 42;
}

Task<std::string> greet(std::string name) {
    co_return "Hello, " + name + "!";
}

int main() {
    auto t1 = compute_answer();
    std::cout << "Answer: " << t1.get() << "\n";   // 42

    auto t2 = greet("World");
    std::cout << t2.get() << "\n";   // Hello, World!
}
```

**How it works:**

- `co_return 42` calls `promise.return_value(42)`, storing the result in `std::optional<int>`.
- `final_suspend()` returns `suspend_always` - the coroutine frame stays alive so `get()` can read `result`.
- The caller calls `get()`, which resumes the lazy coroutine, and reads the result from the promise.
- The destructor calls `handle.destroy()` to free the coroutine frame.

### Q2: Show the sequence of calls: return_value -> final_suspend -> destroy

Here's a traced version of `Task` that prints each lifecycle step as it happens - useful
for building a mental model of what the coroutine machinery is doing:

```cpp
#include <coroutine>
#include <iostream>

struct TracedTask {
    struct promise_type {
        int result = 0;

        TracedTask get_return_object() {
            std::cout << "[1] get_return_object()\n";
            return TracedTask{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept {
            std::cout << "[2] initial_suspend()\n";
            return {};
        }

        void return_value(int v) {
            std::cout << "[4] return_value(" << v << ")\n";
            result = v;
        }

        std::suspend_always final_suspend() noexcept {
            std::cout << "[5] final_suspend()\n";
            return {};
        }

        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;
    explicit TracedTask(std::coroutine_handle<promise_type> h) : handle(h) {}
    ~TracedTask() {
        if (handle) {
            std::cout << "[6] handle.destroy()\n";
            handle.destroy();
        }
    }
    TracedTask(TracedTask&& o) noexcept : handle(o.handle) { o.handle = nullptr; }
    TracedTask(const TracedTask&) = delete;
};

TracedTask example() {
    std::cout << "[3] coroutine body running\n";
    co_return 99;
    // After co_return:
    // -> return_value(99) is called
    // -> local variables destroyed
    // -> final_suspend() is called
}

int main() {
    auto task = example();    // [1] get_return_object, [2] initial_suspend
    task.handle.resume();     // [3] body, [4] return_value(99), [5] final_suspend
    std::cout << "Result: " << task.handle.promise().result << "\n";
    // [6] handle.destroy() in destructor
}
// Output:
// [1] get_return_object()
// [2] initial_suspend()
// [3] coroutine body running
// [4] return_value(99)
// [5] final_suspend()
// Result: 99
// [6] handle.destroy()
```

### Q3: Explain why final_suspend should typically not be suspend_never if the coroutine result must be retrieved

**Answer:**

If `final_suspend()` returns `suspend_never`, the coroutine **destroys itself immediately**
after `return_value()` completes. This means:

1. **The coroutine frame (including the promise) is freed** before the caller can read the result.
2. Any attempt to access `handle.promise().result` is **use-after-free** - undefined behavior.
3. Calling `handle.destroy()` in the destructor is **double-free** - also undefined behavior.
4. Even checking `handle.done()` is UB because the handle is dangling.

```cpp
// DANGEROUS: suspend_never at final_suspend
struct BadPromise {
    int result;
    // ...
    std::suspend_never final_suspend() noexcept { return {}; }
    void return_value(int v) { result = v; }
};

// After co_return, the coroutine self-destructs.
// auto task = bad_coroutine();
// task.handle.resume();
// task.handle.promise().result;   // UB! Frame already destroyed
// ~Task() -> handle.destroy();     // UB! Double destroy
```

**When suspend_never IS safe:**

- Fire-and-forget coroutines where no one ever reads the result.
- The coroutine handle is nulled after resume so the destructor doesn't double-destroy.
- The coroutine is managed by an external scheduler that doesn't need the return value.

**Best practice:** Default to `suspend_always` for `final_suspend()` and have the owner call `handle.destroy()`.

---

## Notes

- `final_suspend()` must be `noexcept` - throwing from it is undefined behavior.
- A coroutine that uses `co_return expr;` must have `return_value()` in its promise. One that uses `co_return;` (or falls off the end) must have `return_void()`. Having both is ill-formed.
- `co_return` can co-exist with `co_await` and `co_yield` in the same coroutine body.
- Libraries like cppcoro, folly::coro, and libunifex provide production-ready `Task<T>` types - don't reinvent them for real projects.
