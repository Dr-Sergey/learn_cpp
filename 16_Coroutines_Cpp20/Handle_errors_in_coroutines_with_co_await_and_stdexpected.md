# Handle Errors in Coroutines with co_await and std::expected

**Category:** Coroutines Cpp20  
**Standard:** C++20, C++23 (`std::expected`)  
**Reference:** [cppreference — Coroutines](https://en.cppreference.com/w/cpp/language/coroutines), [cppreference — std::expected](https://en.cppreference.com/w/cpp/utility/expected)  

---

## Topic Overview

Exception handling in coroutines is deceptively complex. When an exception escapes the coroutine body, the compiler-generated code calls `promise.unhandled_exception()` - typically storing `std::current_exception()` into an `std::exception_ptr`. The exception is then rethrown when the caller inspects the result (e.g., `.get()` or `co_await`). This two-phase model means **exceptions cross suspension boundaries**, which has performance and design implications.

The reason this trips people up is the timing. In a normal function, the exception propagates immediately up the call stack. In a coroutine, the exception is frozen as an `exception_ptr`, the coroutine frame is destroyed, and only later - when the caller accesses the result - does the exception surface again. This means the throw site and the catch site can be on completely different threads and at completely different points in time.

C++23 `std::expected<T, E>` offers a value-or-error type that avoids exception overhead entirely. By making coroutines return `std::expected` and providing an `operator co_await` that **short-circuits on error**, you get monadic error propagation akin to Rust's `?` operator. If the awaited expected holds an error, the coroutine immediately `co_return`s that error without throwing.

There are **three layers** of error handling in coroutines, and choosing between them is mostly about frequency and cost:

| Layer | Mechanism | When to Use |
| --- | --- | --- |
| Exceptions | `unhandled_exception()` + `exception_ptr` | Truly exceptional failures (OOM, logic errors) |
| `std::expected` short-circuit | `co_await expected_val` propagates error | Domain errors (validation, I/O, parsing) |
| Error codes in promise | `yield_value` / `return_value` carry error | High-throughput paths where even `expected` wrapper cost matters |

The flow for the `std::expected` short-circuit pattern is:

```cpp
co_await some_expected_value
    │
    ├── has_value() == true  ──>  await_resume() returns T  ──> continue
    │
    └── has_value() == false ──>  await_suspend() does co_return error
                                  (short-circuit the coroutine)
```

For production systems, prefer `std::expected` for domain errors and reserve `unhandled_exception` for genuinely unexpected failures. This avoids the overhead of exception unwinding across coroutine frames, which involves destroying the promise, deallocating the frame, and rethrowing.

---

## Self-Assessment

### Q1: Implement `unhandled_exception()` patterns — store, rethrow on access, and demonstrate `exception_ptr` propagation across a suspension point

The most important rule here is that `unhandled_exception()` must not rethrow - it must only *store* the exception. The coroutine frame is still being cleaned up when this function runs, and rethrowing would immediately hit `std::terminate`. You store the exception as an `exception_ptr` and rethrow it later when the caller calls `.get()`:

```cpp
#include <coroutine>
#include <exception>
#include <iostream>
#include <stdexcept>
#include <utility>

template <typename T>
class Task {
public:
    struct promise_type {
        T value{};
        std::exception_ptr eptr{};

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T v) { value = std::move(v); }

        void unhandled_exception() {
            eptr = std::current_exception();  // capture — don't rethrow here
        }
    };

    // Resume and extract result — rethrows if the coroutine threw
    T get() {
        handle_.resume();
        auto& p = handle_.promise();
        if (p.eptr) {
            std::rethrow_exception(p.eptr);  // propagate to caller
        }
        return std::move(p.value);
    }

    explicit Task(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~Task() { if (handle_) handle_.destroy(); }
    Task(Task&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    std::coroutine_handle<promise_type> handle_;
};

Task<int> might_fail(bool do_fail) {
    if (do_fail) throw std::runtime_error("coroutine exploded");
    co_return 42;
}

int main() {
    try {
        auto ok = might_fail(false);
        std::cout << "ok = " << ok.get() << '\n';          // 42

        auto bad = might_fail(true);
        std::cout << "bad = " << bad.get() << '\n';         // throws
    } catch (std::exception const& ex) {
        std::cout << "caught: " << ex.what() << '\n';       // coroutine exploded
    }
}
```

The critical rule bears repeating: never call `std::rethrow_exception` inside `unhandled_exception()` - the coroutine is already unwinding. Store the pointer and rethrow when the caller accesses the result.

---

### Q2: Build a monadic `co_await` for `std::expected<T, E>` that short-circuits on error — the coroutine `?` operator

This is the most elegant error handling pattern in modern C++ coroutines. The key mechanism is `await_transform` in the promise type - it intercepts every `co_await` expression inside the coroutine and wraps it in our custom `ExpectedAwaiter`. When the expected holds an error, the awaiter stores that error in the promise and destroys the coroutine handle - the body never resumes. From the outside, the coroutine simply returns an error-containing `expected`:

```cpp
#include <coroutine>
#include <expected>
#include <iostream>
#include <string>
#include <utility>

// Forward-declare the coroutine return type
template <typename T, typename E>
class ExpectedTask;

// Awaiter: co_await an expected<T,E> — short-circuit on error
template <typename T, typename E, typename Promise>
struct ExpectedAwaiter {
    std::expected<T, E> exp;

    bool await_ready() noexcept { return exp.has_value(); }

    // On error: store the error in the promise and destroy the coroutine
    void await_suspend(std::coroutine_handle<Promise> h) noexcept {
        h.promise().result = std::unexpected(std::move(exp.error()));
        h.destroy();
    }

    T await_resume() { return std::move(*exp); }
};

template <typename T, typename E>
class ExpectedTask {
public:
    struct promise_type {
        std::expected<T, E> result;

        ExpectedTask get_return_object() {
            return ExpectedTask{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T v) { result = std::move(v); }
        void unhandled_exception() { std::terminate(); }

        // Enable co_await on expected<U,E> inside this coroutine
        template <typename U>
        ExpectedAwaiter<U, E, promise_type> await_transform(std::expected<U, E> exp) {
            return {std::move(exp)};
        }
    };

    std::expected<T, E> const& value() const { return handle_.promise().result; }

    explicit ExpectedTask(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~ExpectedTask() { if (handle_) handle_.destroy(); }
    ExpectedTask(ExpectedTask&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    std::coroutine_handle<promise_type> handle_;
};

// --- Domain functions returning expected ---
using Error = std::string;

std::expected<int, Error> parse_int(std::string_view s) {
    if (s == "42") return 42;
    return std::unexpected(Error{"parse error: " + std::string(s)});
}

std::expected<int, Error> safe_divide(int a, int b) {
    if (b == 0) return std::unexpected(Error{"division by zero"});
    return a / b;
}

// --- Coroutine with monadic short-circuit ---
ExpectedTask<int, Error> compute(std::string_view input, int divisor) {
    int num = co_await parse_int(input);       // short-circuits on error
    int result = co_await safe_divide(num, divisor); // short-circuits on error
    co_return result * 10;
}

int main() {
    auto r1 = compute("42", 7);
    if (r1.value()) std::cout << "r1 = " << *r1.value() << '\n';  // 60

    auto r2 = compute("bad", 7);
    if (!r2.value()) std::cout << "r2 error: " << r2.value().error() << '\n';

    auto r3 = compute("42", 0);
    if (!r3.value()) std::cout << "r3 error: " << r3.value().error() << '\n';
}
// Output:
// r1 = 60
// r2 error: parse error: bad
// r3 error: division by zero
```

`await_transform` intercepts every `co_await` expression inside the coroutine. When the expected holds an error, `await_suspend` stores the error in the promise and **destroys the coroutine handle** - the body never resumes. This is legal and well-defined: destroying a coroutine handle inside `await_suspend` is explicitly allowed by the standard.

---

### Q3: Combine exception-based and expected-based error handling — catch exceptions from legacy code and convert to `std::expected` inside a coroutine

The pattern here is a thin adapter function `try_call` that calls a potentially-throwing function and converts any exception into an `std::unexpected`. This lets you use the short-circuit `co_await` pattern even when you're calling legacy APIs that still throw. The boundary is explicit and localized:

```cpp
#include <coroutine>
#include <expected>
#include <iostream>
#include <stdexcept>
#include <string>
#include <exception>
#include <utility>

using Error = std::string;

// Legacy function that throws
int legacy_sqrt(int v) {
    if (v < 0) throw std::domain_error("negative input");
    // naive integer sqrt
    int r = 0;
    while (r * r <= v) ++r;
    return r - 1;
}

// Wrapper: catch exception -> expected
template <typename F, typename... Args>
auto try_call(F&& f, Args&&... args)
    -> std::expected<decltype(f(args...)), Error>
{
    try {
        return f(std::forward<Args>(args)...);
    } catch (std::exception const& ex) {
        return std::unexpected(Error{ex.what()});
    }
}

// Reuses ExpectedTask and ExpectedAwaiter from Q2
// (omitted for brevity — same definitions)

ExpectedTask<int, Error> safe_computation(int input) {
    // Convert throwing call into expected
    int root = co_await try_call(legacy_sqrt, input);
    int doubled = co_await try_call(legacy_sqrt, root * root - 1);
    co_return root + doubled;
}

int main() {
    auto r1 = safe_computation(25);
    if (r1.value()) std::cout << *r1.value() << '\n';  // 5 + 4 = 9

    auto r2 = safe_computation(-1);
    if (!r2.value()) std::cout << "error: " << r2.value().error() << '\n';
    // error: negative input
}
```

Here is a summary of the three strategies and when to pick each one:

| Strategy | Overhead | Use Case |
| --- | --- | --- |
| `unhandled_exception` + `exception_ptr` | High (unwind + alloc) | Unexpected failures |
| `co_await expected` short-circuit | Near-zero | Domain / recoverable errors |
| `try_call` wrapper | One try-catch at boundary | Adapting legacy throwing APIs |

---

## Notes

- `unhandled_exception()` is called inside a catch block - `std::current_exception()` is guaranteed to be valid. Calling `throw;` inside it re-throws and immediately hits `std::terminate` because the coroutine frame is still unwinding.
- Destroying a coroutine handle inside `await_suspend` is legal and well-defined. This is the key enabler for the short-circuit pattern - the coroutine frame is deallocated, and control returns to the caller.
- `await_transform` in the promise hijacks all `co_await` expressions. If you also need to `co_await` non-expected awaitables in the same coroutine, overload `await_transform` for those types too.
- `std::expected` is available in C++23. For C++20, use `tl::expected` or a similar backport.
- In coroutine chains (coroutine A `co_await`s coroutine B which `co_await`s C), exceptions propagate through `exception_ptr` at each level. This creates **N copies** of the exception - potentially expensive. The short-circuit pattern avoids this entirely.
- Mixing exceptions and `expected` in the same coroutine body works, but establishes a confusing contract. Choose one strategy per coroutine and document it.
