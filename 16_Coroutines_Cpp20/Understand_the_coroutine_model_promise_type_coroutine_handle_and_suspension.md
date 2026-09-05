# Understand the coroutine model: promise_type, coroutine_handle, and suspension

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

C++20 coroutines are **stackless** - they suspend by returning to the caller and resume later. The compiler transforms a coroutine function into a state machine. If you have used coroutines in Python or JavaScript, the concept of suspending and resuming will feel familiar; the difference in C++ is that you have explicit control over every piece of the machinery.

### Three Pillars

Every coroutine in C++ is built from three collaborating pieces. You need to understand what each one is responsible for before the bigger picture makes sense:

```cpp
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  promise_type      │    │ coroutine_handle │    │  Awaitable       │
│                    │    │                  │    │                  │
│ Controls behavior: │    │ Raw handle to    │    │ Controls each    │
│ • initial_suspend  │    │ coroutine frame: │    │ co_await point:  │
│ • final_suspend    │    │ • resume()       │    │ • await_ready    │
│ • return_value     │    │ • destroy()      │    │ • await_suspend  │
│ • yield_value      │    │ • done()         │    │ • await_resume   │
│ • unhandled_exc    │    │ • promise()      │    │                  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

The `promise_type` is the coroutine's policy object - it decides how the coroutine starts, how it ends, and how values flow in and out. The `coroutine_handle` is a raw pointer-like token that lets you resume or destroy the coroutine from outside. The `Awaitable` is the protocol that any type must implement to be used with `co_await`.

### Coroutine Keywords

Three keywords can turn an ordinary function into a coroutine. The presence of any one of them is enough:

| Keyword | Triggers | Promise method called |
| --- | --- | --- |
| `co_await expr` | Suspension point | `await_transform(expr)` (optional) |
| `co_yield value` | Yield + suspend | `yield_value(value)` |
| `co_return value` | Complete with value | `return_value(value)` |
| `co_return;` | Complete (void) | `return_void()` |

A function is a coroutine if it contains **any** of these keywords.

---

## Self-Assessment

### Q1: Explain the lifecycle of a coroutine: initial_suspend, resumption, final_suspend, and destroy

Understanding the lifecycle is the most important step. Everything else - generators, task types, symmetric transfer - is built on top of these five stages. Here is what happens in order, from the moment a caller invokes the coroutine function to the moment the frame is freed:

```cpp
Coroutine Lifecycle:

[1] Caller calls coroutine function
    └── Compiler allocates coroutine frame (heap or HALO)
    └── Constructs promise_type
    └── Calls promise.get_return_object() -> return value

[2] co_await promise.initial_suspend()
    ├── suspend_always: coroutine suspends, caller gets return value (LAZY)
    └── suspend_never:  coroutine starts running immediately (EAGER)

[3] Coroutine body executes
    ├── co_await: may suspend
    ├── co_yield: yield value + suspend
    └── co_return: set result

[4] co_await promise.final_suspend()
    ├── suspend_always: coroutine suspended at end (must be destroyed manually)
    └── suspend_never:  coroutine auto-destroys (DANGEROUS: can't read result!)

[5] Destruction
    └── handle.destroy() -> destroys locals, promise, frees frame
```

The demonstration below instruments each step so you can see exactly when each phase fires. Notice that `initial_suspend` returning `suspend_always` makes the coroutine lazy - the body does not run until the first explicit `resume()`:

```cpp
#include <coroutine>
#include <iostream>

struct LifecycleDemo {
    struct promise_type {
        LifecycleDemo get_return_object() {
            std::cout << "[1] get_return_object\n";
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() {
            std::cout << "[2] initial_suspend (lazy)\n";
            return {};
        }
        std::suspend_always final_suspend() noexcept {
            std::cout << "[4] final_suspend\n";
            return {};
        }
        void return_void() {
            std::cout << "[3c] return_void (co_return)\n";
        }
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;
    ~LifecycleDemo() {
        if (handle) {
            std::cout << "[5] destroy\n";
            handle.destroy();
        }
    }
};

LifecycleDemo demo_coroutine() {
    std::cout << "[3a] body start\n";
    std::cout << "[3b] body end\n";
    co_return;
}

int main() {
    std::cout << "--- Create ---\n";
    auto d = demo_coroutine();
    std::cout << "--- Resume ---\n";
    d.handle.resume();  // runs body to completion
    std::cout << "--- Scope exit ---\n";
}
// Expected output:
// --- Create ---
// [1] get_return_object
// [2] initial_suspend (lazy)
// --- Resume ---
// [3a] body start
// [3b] body end
// [3c] return_void (co_return)
// [4] final_suspend
// --- Scope exit ---
// [5] destroy
```

Step `[5]` fires last because `final_suspend` returns `suspend_always`, so the coroutine is still "alive" (but done) when `main` returns. The destructor of `LifecycleDemo` calls `handle.destroy()` to clean up.

### Q2: Write a minimal generator coroutine that yields values one at a time

A generator is one of the simplest and most useful coroutine patterns. The idea is that the coroutine body is like a function that can pause and hand back a value mid-execution, then pick up exactly where it left off when the caller asks for the next value. `co_yield` is the keyword that does both: it stores the value in the promise and suspends the coroutine.

```cpp
#include <coroutine>
#include <iostream>

template<typename T>
class Generator {
public:
    struct promise_type {
        T current_value;

        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }  // lazy
        std::suspend_always final_suspend() noexcept { return {}; }  // stay alive for cleanup

        std::suspend_always yield_value(T value) {
            current_value = value;
            return {};  // suspend after yielding
        }

        void return_void() {}  // co_return; or end of function
        void unhandled_exception() { std::terminate(); }
    };

private:
    std::coroutine_handle<promise_type> handle_;

    explicit Generator(std::coroutine_handle<promise_type> h) : handle_(h) {}

public:
    Generator(Generator&& o) noexcept : handle_(std::exchange(o.handle_, {})) {}
    ~Generator() { if (handle_) handle_.destroy(); }

    bool next() {
        handle_.resume();
        return !handle_.done();
    }

    T value() const { return handle_.promise().current_value; }
};

// Fibonacci generator
Generator<long long> fibonacci() {
    long long a = 0, b = 1;
    while (true) {
        co_yield a;
        auto temp = a + b;
        a = b;
        b = temp;
    }
}

// Range generator
Generator<int> range(int start, int end) {
    for (int i = start; i < end; ++i)
        co_yield i;
}

int main() {
    std::cout << "Fibonacci: ";
    auto fib = fibonacci();
    for (int i = 0; i < 10 && fib.next(); ++i)
        std::cout << fib.value() << ' ';
    std::cout << '\n';

    std::cout << "Range 5..10: ";
    auto r = range(5, 10);
    while (r.next())
        std::cout << r.value() << ' ';
    std::cout << '\n';
}
// Expected output:
// Fibonacci: 0 1 1 2 3 5 8 13 21 34
// Range 5..10: 5 6 7 8 9
```

Each call to `next()` resumes the coroutine and runs it until the next `co_yield`. The value lands in `current_value` inside the promise, and the caller reads it back through `value()`. Because `final_suspend` returns `suspend_always`, the frame stays alive between calls and is only freed when the `Generator` destructor runs.

### Q3: Show how `co_await`, `co_yield`, and `co_return` differ in their semantics

All three keywords relate to suspension, but they serve different purposes. The demo below uses all three in one coroutine so you can see exactly what each one does to control flow:

```cpp
#include <coroutine>
#include <iostream>
#include <string>

struct Demo {
    struct promise_type {
        std::string last_yield;
        int result = 0;

        Demo get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        // Called by co_yield
        std::suspend_always yield_value(std::string s) {
            last_yield = std::move(s);
            return {};  // co_yield always suspends
        }

        // Called by co_return
        void return_value(int v) {
            result = v;  // co_return sets final value, no more resumption
        }

        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;
    ~Demo() { if (handle) handle.destroy(); }
};

Demo show_keywords() {
    // co_await: suspend and wait for something
    // std::suspend_always has await_ready()=false
    co_await std::suspend_always{};  // explicit suspension point
    std::cout << "Resumed after co_await\n";

    // co_yield: produce a value and suspend
    // Equivalent to: co_await promise.yield_value("Hello")
    co_yield std::string("Hello");
    std::cout << "Resumed after co_yield\n";

    co_yield std::string("World");
    std::cout << "Resumed after second co_yield\n";

    // co_return: set final value and go to final_suspend
    // No more code runs after co_return
    co_return 42;
    // Unreachable code here!
}

int main() {
    auto d = show_keywords();
    auto& p = d.handle.promise();

    d.handle.resume();   // initial_suspend -> co_await suspend_always
    d.handle.resume();   // co_await -> prints "Resumed after co_await"
    std::cout << "Yielded: " << p.last_yield << '\n';

    d.handle.resume();   // first co_yield -> prints "Resumed after co_yield"
    std::cout << "Yielded: " << p.last_yield << '\n';

    d.handle.resume();   // second co_yield -> prints, then co_return
    std::cout << "Final: " << p.result << '\n';
    std::cout << "Done: " << d.handle.done() << '\n';
}
// Expected output:
// Resumed after co_await
// Yielded: Hello
// Resumed after co_yield
// Yielded: World
// Resumed after second co_yield
// Final: 42
// Done: 1
```

The important distinction to take away is this: `co_await` waits for something outside the coroutine (whatever the awaitable says to do), `co_yield` produces a value to the caller and suspends, and `co_return` terminates the coroutine permanently with an optional final value. After `co_return`, `handle.done()` becomes `true` and calling `resume()` again would be undefined behaviour.

**Comparison:**

| Keyword | Purpose | Suspends? | Promise method |
| --- | --- | --- | --- |
| `co_await expr` | Wait for async result | Depends on `await_ready()` | (uses awaitable protocol) |
| `co_yield value` | Produce value + suspend | Always | `yield_value(value)` |
| `co_return value` | Set final result + finish | Goes to `final_suspend` | `return_value(value)` |
| `co_return;` | Finish (void) | Goes to `final_suspend` | `return_void()` |

---

## Notes

- A function becomes a coroutine by containing `co_await`, `co_yield`, or `co_return`. You cannot have a regular `return` in a coroutine.
- `co_yield x` is syntactic sugar for `co_await promise.yield_value(x)`.
- The return type's `promise_type` nested type is found by the compiler automatically.
- `coroutine_handle<promise_type>` is typed; `coroutine_handle<>` (= `coroutine_handle<void>`) is type-erased.
- `handle.done()` returns `true` after `final_suspend` - never call `resume()` on a done handle.
