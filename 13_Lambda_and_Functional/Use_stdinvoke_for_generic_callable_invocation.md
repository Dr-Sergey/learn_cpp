# Use std::invoke for generic callable invocation

**Category:** Lambda and Functional  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/invoke>  

---

## Topic Overview

`std::invoke` gives you a **uniform syntax** for calling any callable: free functions, member functions, member data pointers, lambdas, and function objects. Without it, each callable category needs its own call syntax, which makes writing generic code that works with all of them surprisingly painful.

The problem is that member function pointers use a completely different call syntax from everything else:

```cpp
// Without std::invoke - different call syntax for each:
free_func(42);                  // free function
(obj.*member_func)(42);         // member function pointer
(ptr->*member_func)(42);        // member function pointer via pointer
obj.*member_data;               // member data pointer
lambda(42);                     // lambda/functor

// With std::invoke - ONE syntax for ALL:
std::invoke(free_func, 42);
std::invoke(member_func, obj, 42);
std::invoke(member_func, ptr, 42);
std::invoke(member_data, obj);
std::invoke(lambda, 42);
```

The reason this matters is that template code often needs to call something it received as a parameter, and that something could be any of the above. Without `std::invoke`, you'd need a bunch of `if constexpr` branches or trait specializations to handle member pointers versus everything else. With `std::invoke`, you write the call once and it handles all cases.

---

## Self-Assessment

### Q1: Show that `std::invoke` works uniformly with free functions, member function pointers, and lambdas

Here you can see all the callable categories in one place. Pay attention to how member functions and member data pointers are handled - `std::invoke` puts the object immediately after the callable, which reads naturally as "call this, on that object, with these args."

```cpp
#include <iostream>
#include <functional>
#include <string>

int add(int a, int b) { return a + b; }

struct Widget {
    std::string name;
    int compute(int x) const { return x * 2; }
};

int main() {
    // 1. Free function
    auto r1 = std::invoke(add, 3, 4);
    std::cout << "free function: " << r1 << "\n";

    // 2. Lambda
    auto square = [](int x) { return x * x; };
    auto r2 = std::invoke(square, 5);
    std::cout << "lambda: " << r2 << "\n";

    // 3. Member function pointer
    Widget w{"Gadget"};
    auto r3 = std::invoke(&Widget::compute, w, 10);
    std::cout << "member function: " << r3 << "\n";

    // 4. Member function pointer via pointer
    Widget* pw = &w;
    auto r4 = std::invoke(&Widget::compute, pw, 7);
    std::cout << "member func (ptr): " << r4 << "\n";

    // 5. Member data pointer
    auto r5 = std::invoke(&Widget::name, w);
    std::cout << "member data: " << r5 << "\n";

    // 6. std::function
    std::function<int(int)> fn = [](int x) { return x + 100; };
    auto r6 = std::invoke(fn, 5);
    std::cout << "std::function: " << r6 << "\n";
}
// Expected output:
//   free function: 7
//   lambda: 25
//   member function: 20
//   member func (ptr): 14
//   member data: Gadget
//   std::function: 105
```

---

### Q2: Implement a higher-order function `apply` that uses `std::invoke` to call any callable

Once you have `std::invoke`, building higher-order utilities that accept any callable is straightforward. Here's a plain `apply` wrapper and a practical `timed_apply` that measures how long any callable takes - both work equally well with free functions, member functions, and lambdas.

```cpp
#include <iostream>
#include <functional>
#include <string>
#include <chrono>

// Generic apply - works with ANY callable (free, member, lambda, etc.)
template <typename F, typename... Args>
auto apply(F&& f, Args&&... args) {
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}

// Practical: timed_apply - measures execution time of any callable
template <typename F, typename... Args>
auto timed_apply(F&& f, Args&&... args) {
    auto start = std::chrono::steady_clock::now();
    auto result = std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
    auto elapsed = std::chrono::steady_clock::now() - start;
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();
    std::cout << "  Elapsed: " << us << " us\n";
    return result;
}

int multiply(int a, int b) { return a * b; }

struct Calculator {
    int base;
    int add(int x) const { return base + x; }
};

int main() {
    // apply with free function
    std::cout << "multiply: " << apply(multiply, 6, 7) << "\n";

    // apply with lambda
    auto sq = [](int x) { return x * x; };
    std::cout << "square: " << apply(sq, 8) << "\n";

    // apply with member function
    Calculator calc{100};
    std::cout << "calc.add: " << apply(&Calculator::add, calc, 42) << "\n";

    // timed_apply
    std::cout << "timed multiply:\n";
    auto result = timed_apply(multiply, 1000, 2000);
    std::cout << "  result: " << result << "\n";
}
// Expected output:
//   multiply: 42
//   square: 64
//   calc.add: 142
//   timed multiply:
//     Elapsed: <N> us
//     result: 2000000
```

---

### Q3: Explain why `std::is_invocable_v` is used to constrain callable template parameters

When you write a template that accepts a callable, you're making an implicit promise about what kind of callable you expect. Without an explicit constraint, passing the wrong thing produces an error deep inside `std::invoke`, which is hard to diagnose. Constraining the template at the boundary gives you a clear, early error message and also enables correct SFINAE behavior.

```cpp
#include <iostream>
#include <functional>
#include <type_traits>
#include <concepts>
#include <string>
#include <utility>

// Without constraint: terrible error messages
template <typename F>
auto call_unconstrained(F&& f) {
    return std::invoke(std::forward<F>(f), 42);
    // If F can't be called with (int), error is DEEP inside std::invoke
}

// With is_invocable_v: clear constraint
template <typename F>
    requires std::is_invocable_v<F, int>
auto call_constrained(F&& f) {
    return std::invoke(std::forward<F>(f), 42);
}

// With concept (even cleaner)
template <std::invocable<int> F>
auto call_concept(F&& f) {
    return std::invoke(std::forward<F>(f), 42);
}

// Checking return type too
template <typename F>
    requires std::is_invocable_r_v<double, F, int>  // must return double
auto call_returning_double(F&& f) {
    return std::invoke(std::forward<F>(f), 42);
}

int main() {
    auto square = [](int x) { return x * x; };
    auto to_str = [](int x) { return std::to_string(x); };

    // All work:
    std::cout << call_constrained(square) << "\n";  // 1764
    std::cout << call_concept(square) << "\n";       // 1764

    // Compile-time trait checks:
    std::cout << std::boolalpha;
    std::cout << "square(int) invocable? "
              << std::is_invocable_v<decltype(square), int> << "\n";
    std::cout << "square(string) invocable? "
              << std::is_invocable_v<decltype(square), std::string> << "\n";
    std::cout << "square(int)->double? "
              << std::is_invocable_r_v<double, decltype(square), int> << "\n";
    std::cout << "to_str(int)->double? "
              << std::is_invocable_r_v<double, decltype(to_str), int> << "\n";
}
// Expected output:
//   1764
//   1764
//   square(int) invocable? true
//   square(string) invocable? false
//   square(int)->double? true
//   to_str(int)->double? false
```

**Why constrain with `is_invocable_v`:**

1. **Better error messages** - error fires at the constraint, not deep inside implementation
2. **SFINAE-friendly** - enables overload sets that select based on callability
3. **Self-documenting** - the signature shows exactly what's required
4. **`is_invocable_r_v<R, F, Args...>`** - also checks that the return type is convertible to `R`

---

## Notes

- **`std::invoke_r<R>(f, args...)` (C++23):** Invokes and casts the result to `R` in one call.
- **Inside the standard library:** `std::invoke` is used internally by `std::thread`, `std::async`, `std::apply`, `std::visit`, and more.
- **`std::apply`:** Like `std::invoke` but unpacks a `tuple` as arguments: `std::apply(f, tuple)`.
- **Concepts shorthand:** The `std::invocable<F, Args...>` concept is equivalent to `std::is_invocable_v<F, Args...>` but fits more naturally in requires clauses.
- **Performance:** `std::invoke` is a zero-overhead abstraction - it compiles to the same code as a direct call.
