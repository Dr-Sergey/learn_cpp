# Implement a task type with structured concurrency semantics

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

**Structured concurrency** ensures that child tasks cannot outlive their parent. When a parent coroutine completes or is cancelled, all child tasks are joined or cancelled first. This is a design constraint, not a language feature - C++20 gives you the tools to enforce it, but it is up to the task type's implementation to make the guarantee hold.

The reason this matters is that unstructured concurrency is a major source of use-after-free bugs and resource leaks in async code. If a child coroutine holds a reference to a local variable in the parent, and the parent exits while the child is still running, the child accesses freed stack memory. Structured concurrency eliminates this class of bug by construction.

### Unstructured vs Structured Concurrency

The difference is stark. In the unstructured model, the parent simply fires off children and exits. In the structured model, the parent must join all children before it returns:

```cpp
UNSTRUCTURED:                    STRUCTURED:
  parent()                          parent()
    ├─ spawn(child_a)                  ├─ auto a = child_a()
    ├─ spawn(child_b)                  ├─ auto b = child_b()
    ├─ return           <- parent     ├─ co_await a          <- joins
    │                     exits!     ├─ co_await b          <- joins
    │  child_a still running!         └─ return              <- safe!
    │  child_b still running!
    ├─ dangling references,           All children done before parent exits.
    │  leaked resources               No dangling references.
```

### Key Properties

| Property | Description |
| --- | --- |
| **Lifetime guarantee** | Child coroutine lifetime is contained within parent lifetime |
| **Exception propagation** | Child exceptions propagate to parent on await |
| **Cancellation** | Destroying parent destroys (cancels) children |
| **No fire-and-forget** | Every task must be awaited or explicitly cancelled |

---

## Self-Assessment

### Q1: Write a `Task<T>` coroutine that supports `co_await` and propagates exceptions to the awaiter

The exception propagation mechanism is what makes this different from a basic `task<T>`. When a child coroutine throws, the exception is captured in the promise via `unhandled_exception()`. When the parent `co_await`s the child and calls `await_resume()`, that stored exception is rethrown in the parent's context. This means the parent can use a normal `try/catch` around `co_await` to handle child failures:

```cpp
#include <coroutine>
#include <exception>
#include <iostream>
#include <optional>
#include <stdexcept>
#include <string>
#include <utility>

template<typename T>
class Task {
public:
    struct promise_type {
        std::optional<T> value;
        std::exception_ptr exception;
        std::coroutine_handle<> continuation;

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }

        auto final_suspend() noexcept {
            struct Awaiter {
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> h) noexcept {
                    return h.promise().continuation
                        ? h.promise().continuation : std::noop_coroutine();
                }
                void await_resume() noexcept {}
            };
            return Awaiter{};
        }

        void return_value(T val) { value = std::move(val); }
        void unhandled_exception() { exception = std::current_exception(); }
    };

private:
    std::coroutine_handle<promise_type> handle_;
    explicit Task(std::coroutine_handle<promise_type> h) : handle_(h) {}

public:
    Task(Task&& o) noexcept : handle_(std::exchange(o.handle_, {})) {}
    ~Task() { if (handle_) handle_.destroy(); }
    Task(const Task&) = delete;

    // Awaitable interface
    bool await_ready() const noexcept { return false; }
    std::coroutine_handle<> await_suspend(std::coroutine_handle<> caller) noexcept {
        handle_.promise().continuation = caller;
        return handle_;
    }
    T await_resume() {
        if (handle_.promise().exception)
            std::rethrow_exception(handle_.promise().exception);
        return std::move(*handle_.promise().value);
    }

    T get() {
        handle_.resume();
        if (handle_.promise().exception)
            std::rethrow_exception(handle_.promise().exception);
        return std::move(*handle_.promise().value);
    }
};

// Coroutine that may throw
Task<int> might_fail(bool fail) {
    if (fail)
        throw std::runtime_error("computation failed!");
    co_return 42;
}

Task<std::string> caller() {
    try {
        int val = co_await might_fail(true);  // exception propagated here
        co_return "Got: " + std::to_string(val);
    } catch (const std::exception& e) {
        co_return std::string("Error: ") + e.what();
    }
}

int main() {
    auto result = caller();
    std::cout << result.get() << '\n';
}
// Expected output:
// Error: computation failed!
```

`unhandled_exception()` captures the exception in the promise via `std::current_exception()`. `await_resume()` checks for the stored exception and rethrows it in the **caller's** context. The caller can use `try/catch` around `co_await` to handle child failures just like a normal function call would.

### Q2: Implement cancellation: destroy the task's `coroutine_handle` before completion

The beauty of the RAII-based task type is that cancellation is almost free - it falls out naturally from the destructor. If you never `co_await` a task and let it go out of scope, the destructor destroys the frame and it is as if the coroutine never ran. For tasks that have already started, cancellation is safe only at suspension points - never while the coroutine is actively executing:

```cpp
// Using Task<T> from Q1

Task<int> long_computation() {
    std::cout << "Step 1\n";
    co_return 100;  // would complete normally
}

void cancellation_example() {
    {
        auto t = long_computation();  // lazy: not started yet
        // t is never co_awaited or .get() called
        // t goes out of scope here
    }  // ~Task() calls handle_.destroy()
       // The coroutine frame is freed
       // No resources leak

    std::cout << "Task cancelled (never ran)\n";
}

void cancel_after_start() {
    // More realistic: cancel a started coroutine
    struct CancellableTask {
        std::coroutine_handle<> handle;
        bool cancelled = false;

        void cancel() {
            if (handle && !handle.done()) {
                cancelled = true;
                handle.destroy();  // destroy at suspension point
                handle = {};       // null out to prevent double-destroy
            }
        }

        ~CancellableTask() {
            if (handle) handle.destroy();
        }
    };

    std::cout << "Cancellation is safe when coroutine is suspended.\n";
    std::cout << "NEVER destroy a running coroutine!\n";
}

int main() {
    cancellation_example();
    cancel_after_start();
}
// Expected output:
// Task cancelled (never ran)
// Cancellation is safe when coroutine is suspended.
// NEVER destroy a running coroutine!
```

The cancellation rules are worth memorising because violating them is undefined behaviour:

1. **Only destroy at suspension points.** A coroutine can be destroyed when it's suspended (at `initial_suspend`, `co_await`, or `final_suspend`).
2. **Never destroy a running coroutine.** This is undefined behavior.
3. **Lazy tasks are easy to cancel** - just destroy without ever resuming.
4. **RAII handles cancellation naturally** - `~Task()` destroys the handle.

### Q3: Show how structured concurrency (all children joined before parent) prevents resource leaks

This example shows the structured pattern in action: the parent creates three child tasks and awaits each one before returning. If the parent is cancelled or throws an exception before awaiting all children, the `~Task()` destructors fire in reverse order and destroy the unawaited children cleanly:

```cpp
// Using Task<T> from Q1
#include <iostream>
#include <string>
#include <vector>

Task<int> child_task(int id) {
    std::cout << "  Child " << id << " running\n";
    co_return id * 10;
}

Task<int> parent_task() {
    std::cout << "Parent starts\n";

    // Structured: create child tasks
    auto a = child_task(1);
    auto b = child_task(2);
    auto c = child_task(3);

    // Must await all children before returning
    int ra = co_await a;  // child 1 runs and completes
    int rb = co_await b;  // child 2 runs and completes
    int rc = co_await c;  // child 3 runs and completes

    std::cout << "Parent: all children done\n";
    co_return ra + rb + rc;
}

int main() {
    auto result = parent_task();
    std::cout << "Total: " << result.get() << '\n';
}
// Expected output:
// Parent starts
//   Child 1 running
//   Child 2 running
//   Child 3 running
// Parent: all children done
// Total: 60
```

The structured concurrency guarantee works through the RAII ownership chain:

```cpp
With structured concurrency:
  parent owns child tasks (RAII)
    │
    ├─ If parent co_awaits all children: clean completion
    ├─ If parent throws before awaiting: ~Task destroys unawaited children
    ├─ If parent is cancelled: ~Task destroys children first
    │
    └─ Children CANNOT outlive parent -> no dangling refs, no leaks

Without structured concurrency (fire-and-forget):

  - spawn(child) runs independently
  - Parent exits, child still runs
  - Child may access destroyed parent's locals -> UB
  - Child may hold resources forever -> leak
```

---

## Notes

- Structured concurrency is **not built into C++20** - it's a design pattern enforced by the task type.
- The P2300 `std::execution` proposal formalizes structured concurrency for C++26.
- Libraries implementing structured concurrency: cppcoro, libunifex, stdexec.
- Key invariant: every `task<T>` must be either awaited or destroyed, never leaked.
- For parallel children, use a `when_all()` combinator that awaits multiple tasks concurrently.
