# Use `std::function_ref` (C++26) as a Lightweight Non-Owning Callable

**Category:** Templates and Generic Programming  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/function_ref>  

---

## Topic Overview

### What Is `std::function_ref`

`std::function_ref` is a **non-owning**, **lightweight** type-erased callable wrapper coming in C++26. Think of it as a "borrow" of a callable - it stores just two pointers and does zero heap allocation, but it does not own what it points to:

```cpp
void apply(std::function_ref<int(int)> f, int x) {
    std::cout << f(x);  // Calls through the reference
}
```

### Comparison with Other Callable Wrappers

| Property | `std::function` | `std::function_ref` | Template param `F` |
| --- | :---: | :---: | :---: |
| Type-erased | Yes | Yes | No |
| Heap allocation | Possible (SBO) | Never | Never |
| Owns callable | Yes (copies/moves) | No (reference only) | Depends |
| Nullable | Yes (`operator bool`) | No (always valid) | N/A |
| Size | ~32-64 bytes | ~16 bytes (2 pointers) | 0 (compile-time) |
| Can store | Yes | No - must not outlive callable | N/A |
| Copy | Copies callable | No copy - must not dangle | Copies callable |

### When to Use Each

Here is the short decision rule:

```cpp
Need to STORE callable for later?    -> std::function (owns it)
Need to PASS callable to a function? -> std::function_ref (cheapest type-erasure)
Need maximum performance?            -> template<typename F> (no type erasure)
```

---

## Self-Assessment

### Q1: Replace a `std::function` parameter with `std::function_ref` where ownership is not needed

The problem with `std::function` as a parameter type is that it does more than you need: it copies the callable, may heap-allocate, and checks for null on every call. If the callable only needs to live for the duration of one call, `function_ref` is the right tool.

Since `std::function_ref` is C++26, this example uses a manual implementation to demonstrate the concept:

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <string>
#include <algorithm>

// === BEFORE: std::function (unnecessary overhead for a parameter) ===
// void for_each_v1(const std::vector<int>& data,
//                  std::function<void(int)> callback) {
//     for (int v : data) callback(v);
// }
// Problem: std::function may allocate, copies the callable, checks for null

// === AFTER: std::function_ref (C++26) — lightweight, no allocation ===
// Note: std::function_ref is C++26 — we'll show the concept with a manual implementation

// Manual function_ref implementation (to demonstrate the concept)
template <typename Sig>
class function_ref;

template <typename R, typename... Args>
class function_ref<R(Args...)> {
    void* obj_;
    R (*callback_)(void*, Args...);

public:
    // Accept any callable
    template <typename F>
        requires (!std::is_same_v<std::decay_t<F>, function_ref>)
    function_ref(F&& f) noexcept
        : obj_(const_cast<void*>(static_cast<const void*>(std::addressof(f))))
        , callback_([](void* ptr, Args... args) -> R {
              return (*static_cast<std::add_pointer_t<F>>(ptr))(std::forward<Args>(args)...);
          }) {}

    R operator()(Args... args) const {
        return callback_(obj_, std::forward<Args>(args)...);
    }
};

// === Usage: pass-through callable (no ownership needed) ===
void for_each(const std::vector<int>& data,
              function_ref<void(int)> callback) {
    for (int v : data) callback(v);
}

int transform_reduce(const std::vector<int>& data,
                     function_ref<int(int)> transform,
                     int init) {
    int result = init;
    for (int v : data) result += transform(v);
    return result;
}

void apply_n_times(function_ref<void()> action, int n) {
    for (int i = 0; i < n; ++i) action();
}

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    std::cout << "=== function_ref with lambda ===\n";
    for_each(data, [](int v) {
        std::cout << "  " << v * 2 << "\n";
    });

    std::cout << "\n=== function_ref with stateful lambda ===\n";
    int sum = 0;
    for_each(data, [&sum](int v) { sum += v; });
    std::cout << "  Sum: " << sum << "\n";  // 15

    std::cout << "\n=== function_ref in transform_reduce ===\n";
    int total = transform_reduce(data, [](int v) { return v * v; }, 0);
    std::cout << "  Sum of squares: " << total << "\n";  // 55

    std::cout << "\n=== function_ref with free function ===\n";
    int counter = 0;
    auto increment = [&counter]() { ++counter; };
    apply_n_times(increment, 5);
    std::cout << "  Counter: " << counter << "\n";  // 5

    std::cout << "\n=== Size comparison ===\n";
    std::cout << "  sizeof(function_ref<void(int)>): " << sizeof(function_ref<void(int)>) << "\n";
    std::cout << "  sizeof(std::function<void(int)>): " << sizeof(std::function<void(int)>) << "\n";

    return 0;
}
```

### Q2: Show that `function_ref` has no heap allocation and a smaller size than `std::function`

The size difference follows directly from the design: `function_ref` is two pointers - one to the callable object and one to the dispatch function. `std::function` needs additional storage for small-buffer optimization, a deleter, and type metadata.

```cpp
#include <iostream>
#include <functional>
#include <cstddef>
#include <chrono>

// Reuse the function_ref implementation from Q1 (or use std::function_ref in C++26)
template <typename Sig> class function_ref;

template <typename R, typename... Args>
class function_ref<R(Args...)> {
    void* obj_;
    R (*callback_)(void*, Args...);
public:
    template <typename F>
        requires (!std::is_same_v<std::decay_t<F>, function_ref>)
    function_ref(F&& f) noexcept
        : obj_(const_cast<void*>(static_cast<const void*>(std::addressof(f))))
        , callback_([](void* ptr, Args... args) -> R {
              return (*static_cast<std::add_pointer_t<F>>(ptr))(std::forward<Args>(args)...);
          }) {}
    R operator()(Args... args) const {
        return callback_(obj_, std::forward<Args>(args)...);
    }
};

int main() {
    std::cout << "=== Size comparison ===\n";
    std::cout << "sizeof(function_ref<int(int)>):   "
              << sizeof(function_ref<int(int)>) << " bytes\n";
    // Typically 16 bytes (2 pointers)

    std::cout << "sizeof(std::function<int(int)>):  "
              << sizeof(std::function<int(int)>) << " bytes\n";
    // Typically 32-64 bytes (SBO buffer + vtable pointer + etc.)

    std::cout << "\n=== No heap allocation proof ===\n";
    // function_ref stores only 2 pointers — always fits inline
    // No dynamic memory needed for ANY callable, no matter the size

    // Large capture that forces std::function to heap-allocate:
    int arr[100]{};
    auto big_lambda = [arr](int x) { return x + arr[0]; };

    std::cout << "sizeof(big_lambda): " << sizeof(big_lambda) << " bytes\n";
    std::cout << "function_ref still only: " << sizeof(function_ref<int(int)>) << " bytes\n";
    std::cout << "std::function with big lambda: " << sizeof(std::function<int(int)>)
              << " bytes (but may heap-allocate internally)\n";

    // Demonstrate it works
    function_ref<int(int)> ref = big_lambda;
    std::cout << "\nref(42) = " << ref(42) << "\n";  // 42

    std::cout << "\n=== Performance benefit ===\n";
    std::cout << "function_ref: no allocation -> no cache miss, no indirection to heap\n";
    std::cout << "std::function: may heap-allocate -> potential cache miss\n";
    std::cout << "Template param: zero overhead, but can't be stored in container\n";

    return 0;
}
```

### Q3: Explain the lifetime constraint: `function_ref` must not outlive the referenced callable

This is the one rule you absolutely must not forget. `function_ref` stores a raw pointer to whatever callable you give it. The moment that callable goes out of scope, your `function_ref` is dangling - and using it is undefined behavior with no diagnostic.

```cpp
#include <iostream>
#include <functional>

template <typename Sig> class function_ref;
template <typename R, typename... Args>
class function_ref<R(Args...)> {
    void* obj_;
    R (*callback_)(void*, Args...);
public:
    template <typename F>
        requires (!std::is_same_v<std::decay_t<F>, function_ref>)
    function_ref(F&& f) noexcept
        : obj_(const_cast<void*>(static_cast<const void*>(std::addressof(f))))
        , callback_([](void* ptr, Args... args) -> R {
              return (*static_cast<std::add_pointer_t<F>>(ptr))(std::forward<Args>(args)...);
          }) {}
    R operator()(Args... args) const {
        return callback_(obj_, std::forward<Args>(args)...);
    }
};

// === SAFE: callable outlives function_ref ===
void safe_example() {
    auto lambda = [](int x) { return x * 2; };
    function_ref<int(int)> ref = lambda;  // ref -> lambda (on stack)
    std::cout << "Safe: " << ref(21) << "\n";  // 42
    // lambda and ref destroyed together — no problem
}

// === DANGEROUS: function_ref outlives callable ===
// function_ref<int(int)> make_dangling() {
//     auto lambda = [](int x) { return x * 2; };
//     return function_ref<int(int)>(lambda);
//     // lambda destroyed here! Returned function_ref DANGLES!
// }

// === DANGEROUS: storing function_ref ===
// struct Callback {
//     function_ref<void()> ref;  // Non-owning!
// };
// Callback store_ref() {
//     auto f = []() { std::cout << "Hi\n"; };
//     return Callback{function_ref<void()>(f)};
//     // f destroyed -> Callback::ref dangles!
// }

// === SAFE pattern: pass-through only ===
void process(function_ref<void(int)> callback) {
    // callback is used immediately within this function
    callback(42);
    // No storing, no returning — safe
}

int main() {
    std::cout << "=== Lifetime rules ===\n";
    safe_example();

    process([](int v) { std::cout << "Processed: " << v << "\n"; });

    std::cout << "\n=== Guidelines ===\n";
    std::cout << "SAFE: function_ref as function parameter (pass-through)\n";
    std::cout << "DANGER: returning function_ref from function\n";
    std::cout << "DANGER: storing function_ref as member (callable may die)\n";
    std::cout << "-> If you need to STORE a callable, use std::function instead\n";

    return 0;
}
```

The mental model that helps here: treat `function_ref` exactly like a raw reference. You would never return a reference to a local variable - the same logic applies to `function_ref`.

---

## Notes

- `std::function_ref` (C++26) is a non-owning, lightweight callable wrapper - **2 pointers**, no heap.
- Use for **function parameters only** - where the callable outlives the call.
- **Never store** a `function_ref` - it doesn't own the callable and can dangle.
- Compared to `std::function`: no allocation, no null state, smaller size, but no ownership.
- Compared to template `F`: type-erased (can be virtual, stored in containers), but has indirection cost.
- If you need C++26 before it's available, libraries like `tl::function_ref` or `llvm::function_ref` provide equivalent functionality.
