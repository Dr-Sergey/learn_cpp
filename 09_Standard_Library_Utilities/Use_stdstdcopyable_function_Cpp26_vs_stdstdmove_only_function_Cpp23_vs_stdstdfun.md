# Use std::copyable_function (C++26) vs std::move_only_function (C++23) vs std::function

**Category:** Standard Library Utilities  
**Standard:** C++23, C++26  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/move_only_function>  

---

## Topic Overview

C++ has three type-erased callable wrappers, each with different ownership semantics. Understanding when to use which one mostly comes down to two questions: does the callable need to be copied, and does the signature need to be const-correct?

### Comparison Table

| Feature | `std::function` | `move_only_function` | `copyable_function` |
| --- | --- | --- | --- |
| Standard | C++11 | C++23 | C++26 |
| Copyable | Yes (required) | No | Yes |
| Move-only callables | No | Yes | No |
| const-correctness | Bug (`const` calls non-const) | Correct | Correct |
| noexcept in signature | No | Yes | Yes |

### The const-correctness Problem

The const-correctness bug in `std::function` is one of those design mistakes that has to stay in the standard for compatibility reasons. The problem is that a `const std::function<void()>` will happily call a mutable lambda - the `const` on the wrapper doesn't propagate to the wrapped callable. `move_only_function` and `copyable_function` fix this by encoding constness in the function signature itself.

```cpp
#include <functional>

// std::function has a design bug:
void bug_example() {
    int counter = 0;
    const std::function<void()> f = [counter]() mutable { ++counter; };
    f();  // Compiles! const function calling mutable lambda
    // This violates const-correctness
}

// C++23 move_only_function fixes this:
// std::move_only_function<void() const> f = ...;
// f() on a const reference only works if the callable is const-invocable
```

### Usage Patterns

Here's the practical guide for which wrapper to reach for. If you have a lambda capturing a `unique_ptr`, `std::function` simply won't compile - that's when `move_only_function` becomes necessary.

```cpp
#include <functional>

// std::function: copyable callbacks (event handlers, shared callbacks)
std::vector<std::function<void(int)>> handlers;
handlers.push_back([](int x) { std::cout << x; });

// move_only_function: move-only callables (unique_ptr captures)
std::move_only_function<void()> task = [p = std::make_unique<int>(42)]() {
    std::cout << *p;
};
// Can't copy task — but can store in a vector and move it

// copyable_function (C++26): like function but with correct const behavior
std::copyable_function<void() const> safe_cb = []() { std::cout << "safe"; };
```

---

## Self-Assessment

### Q1: Why does std::function require copyability

`std::function` was designed before move semantics were widespread. Its API guarantees copyability, which means the wrapped callable must be copyable too. This prevents wrapping lambdas that capture `unique_ptr` or other move-only types - the wrapper would need to copy them when it copies itself, and it can't. `move_only_function` drops the copyability requirement entirely, which is why it can hold move-only callables.

### Q2: When should you use each

- `std::function`: legacy code, or any API that already expects copyable callbacks and you can't change it.
- `std::move_only_function`: C++23 code with move-only callables, task queues, thread pools where tasks need unique ownership.
- `std::copyable_function`: C++26 replacement for `std::function` in new code where you want correct const semantics and copyability.

### Q3: What is `std::function_ref` (C++26)

A non-owning reference to a callable - the callable equivalent of `std::string_view`. It carries zero allocation and no ownership, just a pointer to whatever callable you pass in. Use it when you need to accept a callback for the duration of a function call but don't need to store it anywhere. It's the lightest option in the family.

---

## Notes

- `std::move_only_function` supports `noexcept` in the signature: `move_only_function<void() noexcept>`.
- `std::function_ref` (C++26) is the lightest option - no heap allocation, no ownership.
- All three support `const`, `noexcept`, and ref-qualification in the signature (except `std::function`).
- Prefer `move_only_function` for task queues and thread pools.
