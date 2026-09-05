# Use pack expansion in lambda init-captures (C++20)

**Category:** Lambda and Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

### The Problem Before C++20

Before C++20, you couldn't capture a parameter pack by move directly into a lambda using init-captures. Every element of the pack lived outside the lambda, and the only way to bring the whole pack inside was to bundle it into a `std::tuple` first. That works, but it's awkward because you then need `std::apply` just to get the values back out again.

Here's what that workaround looked like:

```cpp
// C++17 workaround - clunky
template<typename... Args>
auto make_deferred_call_17(Args&&... args) {
    return [tup = std::make_tuple(std::forward<Args>(args)...)]() mutable {
        std::apply([](auto&... a) { /* use a... */ }, tup);
    };
}
```

The indirection through `std::tuple` and `std::apply` is a detour that only exists because the language didn't have a cleaner option yet.

### C++20: Pack Expansion in Init-Captures

C++20 fixes this directly. You can now write `...name = expr` inside the capture list, and the compiler creates one captured variable per element in the pack. No tuple, no `apply`, no workaround.

```cpp
template<typename... Args>
auto make_deferred_call(Args&&... args) {
    // Each element of the pack gets its own init-capture
    return [...captured = std::forward<Args>(args)]() mutable {
        // Use captured... as a pack inside the lambda
        (std::cout << ... << captured) << "\n";
    };
}
```

The syntax `...name = expr` creates one capture per pack element. Inside the lambda body you use `name...` as a pack just like you would outside.

### How It Works

To make this concrete, here's a simple `bind_front` built entirely with pack init-captures:

```cpp
template<typename F, typename... Args>
auto bind_front_simple(F f, Args&&... args) {
    // ...bound = std::forward<Args>(args) captures each arg by move/copy
    return [f = std::move(f), ...bound = std::forward<Args>(args)]
           (auto&&... remaining) mutable {
        return f(bound..., std::forward<decltype(remaining)>(remaining)...);
    };
}

int add(int a, int b, int c) { return a + b + c; }

int main() {
    auto add_10_20 = bind_front_simple(add, 10, 20);
    std::cout << add_10_20(30) << "\n";  // 60
    return 0;
}
```

When this is instantiated with `Args = {int, int}`, the capture `...bound = {10, 20}` effectively becomes two individual captures, `bound_0 = 10` and `bound_1 = 20`. Inside the body, `bound...` expands back to the full pack and gets prepended to the remaining arguments. The whole thing is zero-overhead - no heap allocation, no type erasure.

### Practical Uses

**1. Perfect-forwarding capture for deferred execution:**

This is the motivating use case. You want to capture a callable and its arguments and fire them later, without copying everything through a tuple.

```cpp
template<typename F, typename... Args>
auto defer(F&& f, Args&&... args) {
    return [f = std::forward<F>(f),
            ...captured_args = std::forward<Args>(args)]() mutable {
        return std::invoke(std::move(f), std::move(captured_args)...);
    };
}
```

**2. Capturing a pack for a coroutine:**

This matters a lot with coroutines. If a coroutine suspends, any references to local variables of the caller become dangling. Capturing by move before the first suspension point keeps the values alive.

```cpp
template<typename... Args>
task<void> log_all(Args&&... args) {
    // Capture by move before the first suspension point
    auto logger = [...vals = std::forward<Args>(args)]() {
        (std::cout << ... << vals) << "\n";
    };
    co_await some_async_op();
    logger();
}
```

**3. Building variadic event payloads:**

```cpp
template<typename... Payload>
auto make_event(std::string name, Payload&&... data) {
    return [name = std::move(name),
            ...payload = std::forward<Payload>(data)]() {
        std::cout << "Event: " << name << " with "
                  << sizeof...(payload) << " args\n";
    };
}
```

---

## Self-Assessment

### Q1: Move-capture a parameter pack into a lambda and invoke it

```cpp
#include <iostream>
#include <string>
#include <utility>

template<typename... Args>
auto capture_and_print(Args&&... args) {
    return [...vals = std::forward<Args>(args)]() {
        ((std::cout << vals << " "), ...) ;
        std::cout << "\n";
    };
}

int main() {
    auto fn = capture_and_print(1, 3.14, std::string("hello"));
    fn();  // Output: 1 3.14 hello
    return 0;
}
```

### Q2: Compare the C++17 tuple workaround with C++20 pack init-captures

Both approaches produce identical behavior - the C++20 version is just a lot less ceremony to read and write.

```cpp
#include <iostream>
#include <tuple>
#include <utility>

// C++17: wrap in tuple, unwrap with std::apply
template<typename... Args>
auto old_way(Args&&... args) {
    return [tup = std::make_tuple(std::forward<Args>(args)...)]() {
        std::apply([](const auto&... a) {
            ((std::cout << a << " "), ...);
            std::cout << "\n";
        }, tup);
    };
}

// C++20: direct pack capture
template<typename... Args>
auto new_way(Args&&... args) {
    return [...vals = std::forward<Args>(args)]() {
        ((std::cout << vals << " "), ...);
        std::cout << "\n";
    };
}

int main() {
    old_way(1, 2, 3)();  // 1 2 3
    new_way(1, 2, 3)();  // 1 2 3 - cleaner, no tuple overhead
    return 0;
}
```

### Q3: Implement a simple `bind_front` using pack init-captures

Notice how clean the implementation is - the captured front arguments and the incoming back arguments get forwarded in one expression, with no intermediate storage.

```cpp
#include <functional>
#include <iostream>
#include <utility>

template<typename F, typename... BoundArgs>
auto my_bind_front(F&& f, BoundArgs&&... bound) {
    return [f = std::forward<F>(f),
            ...front = std::forward<BoundArgs>(bound)]
           (auto&&... back) mutable
           -> decltype(auto) {
        return std::invoke(f, front...,
                           std::forward<decltype(back)>(back)...);
    };
}

void report(int id, const std::string& name, double score) {
    std::cout << "ID=" << id << " Name=" << name
              << " Score=" << score << "\n";
}

int main() {
    auto report_42 = my_bind_front(report, 42, std::string("Alice"));
    report_42(98.5);  // ID=42 Name=Alice Score=98.5
    return 0;
}
```

---

## Notes

- The syntax is `[...name = init-expr]` - the `...` precedes the capture name.
- Pack init-captures work with `mutable` lambdas and `constexpr` lambdas.
- This replaces the `std::tuple` + `std::apply` workaround from C++17.
- Be careful with capture-by-reference packs - lifetime issues apply just like single-variable captures.
