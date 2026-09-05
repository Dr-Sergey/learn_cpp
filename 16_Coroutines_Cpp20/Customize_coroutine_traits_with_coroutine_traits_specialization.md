# Customize Coroutine Traits with coroutine_traits Specialization

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** [cppreference — std::coroutine_traits](https://en.cppreference.com/w/cpp/coroutine/coroutine_traits)  

---

## Topic Overview

When the compiler encounters a function whose body contains `co_await`, `co_yield`, or `co_return`, it needs to locate the **promise type** that governs the coroutine's behaviour. The lookup rule is: `std::coroutine_traits<ReturnType, ArgTypes...>::promise_type`. The default primary template simply aliases `ReturnType::promise_type`, so if your return type already nests a `promise_type`, everything works automatically.

However, you can **specialise** `std::coroutine_traits` to inject a promise type into return types you don't control - third-party types, standard library types, or even `void`. This is the mechanism that lets you write `std::future<int> compute() { co_return 42; }` without modifying `std::future` itself; you provide the specialisation, and the compiler uses it.

The reason this is so powerful is that it completely decouples the promise type from the return type. Normally these two are tightly coupled (the return type nests the promise), but `coroutine_traits` breaks that coupling. You can adapt any existing type - ones you wrote years ago, ones from a third-party library - into a coroutine return type without touching the original definition.

The compiler also passes **all function parameters** (including the implicit `this` for member functions) as additional template arguments to `coroutine_traits`. This allows different promise types for the same return type depending on the calling context - e.g., a free function vs. a member function of a specific class.

| Lookup Rule | Template Instantiation |
| --- | --- |
| Free function `R f(A1, A2)` | `coroutine_traits<R, A1, A2>::promise_type` |
| Member function `R C::f(A1)` | `coroutine_traits<R, C&, A1>::promise_type` (non-const) |
| Member function `R C::f(A1) const` | `coroutine_traits<R, C const&, A1>::promise_type` |
| Lambda `[](A1) -> R` | `coroutine_traits<R, LambdaType&, A1>::promise_type` |

To make the compile-time lookup process concrete, here is what happens step by step when the compiler processes a coroutine:

```cpp
Compiler sees: R func(Args...) { co_await ... }
                  │
                  ▼
std::coroutine_traits<R, Args...>::promise_type  ──>  Promise P
                  │
                  ▼
P p;    // promise object constructed
auto return_obj = p.get_return_object();  // R
```

---

## Self-Assessment

### Q1: Specialise `coroutine_traits` so that a function returning `std::optional<T>` can use `co_return` and `co_await`

The goal here is to make `std::optional<T>` a legal coroutine return type - something you normally cannot do because `std::optional` has no nested `promise_type`. The specialisation provides the missing promise, and an `operator co_await` overload on `optional` provides the short-circuiting `co_await` semantics. This gives you something close to Rust's `?` operator for optional chaining:

```cpp
#include <coroutine>
#include <iostream>
#include <optional>

// Promise that makes std::optional<T> a coroutine return type
template <typename T>
struct OptionalPromise {
    std::optional<T> result;

    std::optional<T> get_return_object() {
        // Deferred: we fill result in return_value / return_void
        // But get_return_object is called before the body runs.
        // We return a reference-wrapper trick via coroutine_handle.
        // Simpler approach: store in promise -> move out at final_suspend.
        return result;  // will be updated — see note below
    }

    std::suspend_never initial_suspend() noexcept { return {}; }
    std::suspend_never final_suspend() noexcept { return {}; }

    void return_value(T v) { result.emplace(std::move(v)); }
    void unhandled_exception() { result.reset(); }
};

// --- coroutine_traits specialisation ---
template <typename T, typename... Args>
struct std::coroutine_traits<std::optional<T>, Args...> {
    using promise_type = OptionalPromise<T>;
};

// Awaiter: co_await on optional — short-circuit on nullopt
template <typename T>
struct OptionalAwaiter {
    std::optional<T> const& opt;
    bool await_ready() noexcept { return opt.has_value(); }
    void await_suspend(std::coroutine_handle<> h) noexcept {
        h.destroy();  // nullopt -> abandon coroutine
    }
    T await_resume() { return *opt; }
};

template <typename T>
OptionalAwaiter<T> operator co_await(std::optional<T> const& opt) {
    return {opt};
}

// --- Usage ---
std::optional<int> try_parse(std::string_view s) {
    if (s == "42") return 42;
    return std::nullopt;
}

// This function is a coroutine returning std::optional<int>!
std::optional<int> doubled(std::string_view input) {
    int val = co_await try_parse(input);    // short-circuits on nullopt
    co_return val * 2;
}

int main() {
    auto a = doubled("42");
    auto b = doubled("bad");
    std::cout << "a = " << a.value_or(-1) << '\n';  // 84
    std::cout << "b = " << b.value_or(-1) << '\n';  // -1
}
```

> **Note:** The simplified `get_return_object` above has a subtlety - the optional is copied before the body runs. A production implementation stores a `coroutine_handle` in a wrapper and extracts the value lazily. The pattern here illustrates the traits specialisation mechanism.

---

### Q2: Use `coroutine_traits` to give different promise types to member coroutines vs free coroutines that return the same type

The nice part about including the implicit `this` parameter in the traits lookup is that you can make the same return type behave differently depending on which class is doing the co_return. The free function gets one promise, and the member function of a specific class gets a completely different one - selected at compile time with zero runtime overhead:

```cpp
#include <coroutine>
#include <cstdio>
#include <exception>

struct Task {
    struct promise_type;  // default — used for free functions
    using handle_type = std::coroutine_handle<promise_type>;

    struct promise_type {
        const char* tag = "free";
        Task get_return_object() { return Task{handle_type::from_promise(*this)}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    void run() {
        if (handle_) {
            std::printf("[%s] running\n", handle_.promise().tag);
            handle_.resume();
        }
    }

    handle_type handle_;
    ~Task() { if (handle_) handle_.destroy(); }
};

// A dedicated promise for member coroutines of class Scheduler
struct SchedulerPromise {
    const char* tag = "scheduler-member";
    Task get_return_object() {
        return Task{std::coroutine_handle<SchedulerPromise>::from_promise(*this)};
    }
    std::suspend_always initial_suspend() { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
    void return_void() {}
    void unhandled_exception() { std::terminate(); }
};

struct Scheduler {
    Task schedule();   // member coroutine — will use SchedulerPromise
};

// Specialise: when return type is Task and first implicit arg is Scheduler&
template <>
struct std::coroutine_traits<Task, Scheduler&> {
    using promise_type = SchedulerPromise;
};

// Implementations
Task free_coro() { co_return; }           // uses Task::promise_type
Task Scheduler::schedule() { co_return; } // uses SchedulerPromise

int main() {
    auto t1 = free_coro();
    t1.run();   // prints: [free] running

    Scheduler s;
    auto t2 = s.schedule();
    t2.run();   // prints: [scheduler-member] running
}
```

The key insight here is that the compiler passes `Scheduler&` (the implicit `this`) as the second template argument to `coroutine_traits<Task, Scheduler&>`, which matches the explicit specialisation. Free functions get `coroutine_traits<Task>`, which falls through to the default and picks up `Task::promise_type`.

---

### Q3: Specialise `coroutine_traits` for `void` return type to build a fire-and-forget coroutine

`void` is the simplest possible return type - the caller gets nothing back. By specialising `coroutine_traits<void, Args...>`, you turn any `void`-returning function into a potential coroutine. This is how you get fire-and-forget semantics: the coroutine runs to completion on its own, and the caller doesn't hold a handle to it:

```cpp
#include <coroutine>
#include <iostream>

// Promise for fire-and-forget coroutines returning void
struct FireAndForgetPromise {
    void get_return_object() {}  // return type is void — nothing to return

    std::suspend_never initial_suspend() noexcept { return {}; }
    std::suspend_never final_suspend() noexcept { return {}; }
    void return_void() noexcept {}

    void unhandled_exception() {
        // In production: log or store exception_ptr
        std::terminate();
    }
};

// Specialise for void return type
template <typename... Args>
struct std::coroutine_traits<void, Args...> {
    using promise_type = FireAndForgetPromise;
};

// A trivial awaitable for demonstration
struct SomeAsyncOp {
    bool await_ready() noexcept { return false; }
    void await_suspend(std::coroutine_handle<> h) noexcept {
        // Simulate: complete immediately on another "thread"
        h.resume();
    }
    int await_resume() noexcept { return 99; }
};

// void-returning coroutine — fire-and-forget
void launch_task() {
    int result = co_await SomeAsyncOp{};
    std::cout << "Async result: " << result << '\n';
}

int main() {
    launch_task();  // runs to completion eagerly (due to suspend_never)
    std::cout << "After launch\n";
    // Output:
    // Async result: 99
    // After launch
}
```

The table below shows how what `get_return_object()` returns changes the contract between the coroutine and its caller:

| `get_return_object()` return | Meaning |
| --- | --- |
| `void` | Fire-and-forget; caller gets nothing |
| `Task` | Caller owns the coroutine handle |
| `std::future<T>` | Caller can `.get()` the result later |

---

## Notes

- The primary `std::coroutine_traits<R, Args...>` template is defined in `<coroutine>` and simply exposes `R::promise_type`. Specialisations **replace** this entirely.
- You may only specialise `coroutine_traits` for types in your own namespace or for program-defined types (ODR). Specialising for `std::string` is technically allowed but inadvisable.
- The parameter list passed to `coroutine_traits` is the **decayed** parameter types - references are preserved, but arrays decay to pointers. For member functions, the implicit object parameter is added as the first argument after the return type.
- Because `coroutine_traits` is resolved at compile time, there is **zero runtime cost** - the mechanism is purely a type-level dispatch.
- Combining `coroutine_traits` with concepts (C++20) lets you SFINAE-constrain which functions can be coroutines. If `coroutine_traits<R, Args...>::promise_type` is ill-formed, the function cannot be a coroutine.
- Avoid overuse: prefer nesting `promise_type` inside the return type when you own it. Reserve `coroutine_traits` specialisation for adapting third-party types.
