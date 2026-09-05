# Understand coroutine interaction with exceptions and stack unwinding

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

### How Exceptions Work in Coroutines

Coroutines have a unique exception handling model compared to regular functions, and the reason this trips people up is that the exception doesn't propagate immediately to the caller the way it would in a normal function call. Instead, the compiler wraps your entire coroutine body in a try/catch, and what happens to the exception depends entirely on what your `promise_type::unhandled_exception()` does with it.

Here is the sequence when an exception escapes the coroutine body:

1. The exception is caught by the compiler-generated wrapper.
2. `promise_type::unhandled_exception()` is called with the exception active.
3. The exception can be stored (via `std::current_exception()`) and rethrown later.
4. Stack unwinding destroys local variables in the coroutine frame, but the frame itself stays alive until the handle is destroyed.

The critical point: the caller doesn't see the exception at the moment it's thrown. It only appears when the caller later `co_await`s the result and `await_resume()` rethrows the stored exception pointer.

### The Compiler-Generated Try/Catch

It helps to see the machine the compiler actually builds. Your clean coroutine function becomes something like this under the hood:

```cpp
// What you write:
task<int> compute() {
    auto data = co_await fetch();
    co_return process(data);
}

// What the compiler generates (conceptual):
task<int> compute() {
    auto& promise = get_promise();
    auto return_object = promise.get_return_object();

    try {
        co_await promise.initial_suspend();
        // --- your code ---
        auto data = co_await fetch();
        co_return process(data);
        // --- end your code ---
    } catch (...) {
        promise.unhandled_exception();  // <-- exception caught here
    }
    co_await promise.final_suspend();
}
```

Notice that `final_suspend` runs whether the body completed normally or threw. That's why `final_suspend` must be `noexcept` - if it could throw, you'd have an unhandled exception inside what is already an exception-handling context.

### Implementing `unhandled_exception()`

The standard approach stores the exception for later rethrowing. This keeps the coroutine's promise object as the single owner of the exception until the caller picks it up:

```cpp
struct task_promise {
    std::exception_ptr exception_;

    void unhandled_exception() {
        // Store the current exception
        exception_ = std::current_exception();
    }

    // When the caller awaits the result, rethrow if needed
    int get_result() {
        if (exception_)
            std::rethrow_exception(exception_);
        return value_;
    }
};
```

The caller calls `get_result()` (or the equivalent `await_resume()` in the awaiter), and that's where the exception surfaces. From the caller's perspective, `co_await inner_task` throws exactly as if the inner function had thrown directly - the machinery is invisible.

### Exception Safety at Suspension Points

Exceptions can happen at different stages of `co_await`, and they behave slightly differently depending on where they occur. Understanding this is important if you're writing custom awaitables:

```cpp
task<void> example() {
    // Stage 1: evaluating the awaitable expression -- exceptions propagate normally
    auto awaitable = get_awaitable();  // may throw

    // Stage 2: await_ready() -- may throw (before suspension)
    // Stage 3: await_suspend() -- if it throws, the coroutine is resumed immediately
    // Stage 4: await_resume() -- may throw (after resumption)

    auto result = co_await std::move(awaitable);
    // If await_resume() throws, exception enters the promise's unhandled_exception
}
```

### Critical: Exceptions in `await_suspend()`

If `await_suspend()` throws, **the coroutine is immediately resumed** and the exception propagates to the coroutine body's implicit try/catch - not to the caller of `resume()`. This is surprising behaviour that can cause subtle bugs. The practical rule: don't throw from `await_suspend()`. If you need to signal failure, store the error and let `await_resume()` throw it instead:

```cpp
struct dangerous_awaiter {
    bool await_ready() { return false; }

    void await_suspend(std::coroutine_handle<> h) {
        throw std::runtime_error("suspend failed");
        // The coroutine is resumed, and this exception propagates
        // to the coroutine body's implicit try/catch
    }

    void await_resume() {}
};
```

### Exception Propagation Across Coroutine Boundaries

Exceptions travel across `co_await` boundaries cleanly - from the perspective of the calling coroutine, it looks just like a regular function call throwing. The exception is stored in the inner coroutine's promise, then rethrown by `await_resume()` when the outer coroutine resumes and reads the result:

```cpp
task<int> inner() {
    throw std::runtime_error("inner error");
    co_return 42;  // never reached
}

task<void> outer() {
    try {
        int val = co_await inner();
        // inner()'s exception is rethrown here by await_resume()
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
        // Output: Caught: inner error
    }
}
```

### `noexcept` Coroutines

You **cannot** safely declare a coroutine as `noexcept` in most implementations. If you try, and an exception is thrown, `std::terminate()` is called. The correct approach is to control what `unhandled_exception()` does - either store and forward, or terminate explicitly if you know the coroutine must not throw:

```cpp
// Careful: if unhandled_exception() calls std::terminate(),
// an exception in the body terminates the program
struct terminate_on_exception_promise {
    void unhandled_exception() {
        std::terminate();  // hard crash
    }
};

// Better: log and store
struct safe_promise {
    void unhandled_exception() {
        std::cerr << "Coroutine exception caught\n";
        exception_ = std::current_exception();
    }
};
```

---

## Self-Assessment

### Q1: What happens when an exception escapes a coroutine body

Here is the exact step-by-step sequence. It's worth memorising this because it's the mental model you need whenever you're debugging coroutine exception behaviour:

1. The compiler-generated code catches the exception.
2. `promise.unhandled_exception()` is called - typically stores the exception via `std::current_exception()`.
3. Execution continues to `promise.final_suspend()`.
4. When the caller `co_await`s the result and calls `await_resume()`, the stored exception is rethrown.
5. Local variables in the coroutine frame are destroyed (stack unwinding of the coroutine frame).

### Q2: Demonstrate exception propagation from an inner coroutine to an outer one

Watch how the exception thrown inside `read_file` surfaces as a normal C++ exception at the `co_await` expression in `process`. No special syntax required - it just works, because `await_resume()` rethrows the stored `exception_ptr`:

```cpp
#include <iostream>
#include <stdexcept>

task<std::string> read_file(std::string path) {
    if (path.empty())
        throw std::invalid_argument("empty path");
    co_return "file contents";
}

task<void> process() {
    try {
        auto content = co_await read_file("");  // throws
        std::cout << content << "\n";           // never reached
    } catch (const std::invalid_argument& e) {
        std::cout << "Error: " << e.what() << "\n";
        // Output: Error: empty path
    }
}
```

### Q3: Why is `final_suspend` usually `noexcept`

`final_suspend()` is called after the coroutine body completes, including after `unhandled_exception()` has already run. If `final_suspend()` itself throws, you now have an exception propagating inside what is already the exception cleanup path - the behaviour is undefined. This is why the standard requires `final_suspend` to be `noexcept`:

```cpp
struct my_promise {
    // MUST be noexcept -- throwing here is UB
    auto final_suspend() noexcept {
        return std::suspend_always{};
    }
};
```

The `noexcept` on `final_suspend` is a standard requirement to avoid double-exception scenarios. Every promise type you write should have this. If your linter or compiler doesn't warn about a missing `noexcept` here, add it manually anyway.

---

## Notes

- Always implement `unhandled_exception()` - if you forget, exceptions in the coroutine body silently disappear with no error, which is extremely hard to debug.
- Store exceptions with `std::current_exception()` and rethrow in `await_resume()` - this is the standard pattern that makes coroutine exceptions behave like regular function exceptions from the caller's point of view.
- Exceptions in `await_suspend()` have surprising semantics - avoid throwing there. If your suspend logic can fail, surface the failure through `await_resume()` instead.
- `final_suspend()` must be `noexcept` - this is enforced by the standard and for good reason.
- Coroutine exception handling adds overhead via the exception pointer store/rethrow mechanism; consider `std::expected` for performance-critical paths where exceptions are an expected outcome rather than a true error.
