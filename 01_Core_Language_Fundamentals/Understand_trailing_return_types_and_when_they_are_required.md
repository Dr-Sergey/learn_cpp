# Understand trailing return types and when they are required

**Category:** Core Language Fundamentals  
**Standard:** C++14  
**Reference:** <https://en.cppreference.com/w/cpp/language/function>  

---

## Topic Overview

A **trailing return type** places the return type after the parameter list using `-> Type`. This was introduced in C++11 and is sometimes the **only** way to express a return type.

### Syntax

The two forms below are equivalent for simple cases - the trailing style just moves the type to the right of the parameter list:

```cpp
// Traditional
int add(int a, int b) { return a + b; }

// Trailing return type (equivalent)
auto add(int a, int b) -> int { return a + b; }
```

### When Trailing Return Types Are Required

**1. Return type depends on parameter types (pre-C++14):**

In C++11, parameter names aren't in scope at the leading return-type position, but they ARE in scope after the parameter list - which is exactly why trailing return types exist:

```cpp
// C++11: Can't use 'a' and 'b' in the return type position (they aren't declared yet)
// auto add(auto a, auto b) -> decltype(a + b)  // Won't compile in C++11

// Solution: trailing return type
template<typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}
// Here a and b ARE in scope after the parameter list
```

**2. Lambda return types:**

When a lambda body has multiple return paths that would otherwise deduce different types, pinning the return type with `->` keeps things explicit:

```cpp
auto lambda = [](int a, int b) -> double {
    if (a > b) return a;     // returns double (converted)
    else return a / b;       // returns double (not int division!)
};
```

**3. Decltype on member functions:**

Sometimes the return type naturally refers to a member that isn't yet in scope in the leading position:

```cpp
struct Container {
    std::vector<int> data;

    // Return type depends on 'data' which isn't visible in leading position
    auto begin() -> decltype(data.begin()) { return data.begin(); }
};
```

### C++14 Made Many Cases Unnecessary

C++14 added automatic return type deduction, which eliminates the need for trailing return types in most everyday cases:

```cpp
// C++14: auto return type deduction - no trailing type needed
template<typename T, typename U>
auto add(T a, U b) {
    return a + b;  // Compiler deduces return type
}
```

---

## Self-Assessment

### Q1: Write a function whose return type depends on its parameters using a trailing return type

The key insight here is that `decltype(a * b)` in the trailing position can see `a` and `b` because they've already been declared - that's not true in the leading position:

```cpp
#include <iostream>
#include <string>
#include <type_traits>

// C++11 style: trailing return type is REQUIRED
// because a and b aren't in scope until after the parameter list
template<typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) {
    return a * b;
}

// More complex: conditional return type
template<typename T, typename U>
auto safe_divide(T a, U b) -> decltype(a / b) {
    if (b == 0) throw std::runtime_error("Division by zero");
    return a / b;
}

// Trailing return type with conditional expression
template<typename T>
auto clamp(T value, T lo, T hi) -> decltype(value < lo ? lo : value) {
    return value < lo ? lo : (value > hi ? hi : value);
}

int main() {
    auto r1 = multiply(3, 4.5);      // double (int * double -> double)
    auto r2 = multiply(2.0f, 3.0);   // double (float * double -> double)
    auto r3 = safe_divide(10, 3);    // int (int / int -> int)
    auto r4 = safe_divide(10.0, 3);  // double

    std::cout << "3 * 4.5 = " << r1 << "\n";     // 13.5
    std::cout << "2.0f * 3.0 = " << r2 << "\n";  // 6
    std::cout << "10 / 3 = " << r3 << "\n";       // 3
    std::cout << "10.0 / 3 = " << r4 << "\n";     // 3.33333

    static_assert(std::is_same_v<decltype(r1), double>);
    static_assert(std::is_same_v<decltype(r3), int>);
}
```

**How this works:**

- `-> decltype(a * b)` computes the return type from the expression `a * b`.
- Since `a` and `b` are declared in the parameter list, they're in scope for the trailing return type.
- In the traditional return type position (before the function name), the parameters aren't available yet.

### Q2: Show that `auto f() -> decltype(a+b)` was necessary before C++14 automatic return deduction

This example contrasts the C++11 and C++14 approaches side by side, and also shows cases where trailing return types remain useful even today:

```cpp
#include <iostream>
#include <vector>
#include <type_traits>

// ========= C++11: Trailing return type REQUIRED =========

// Without trailing return type - DOESN'T compile in C++11:
// template<typename T, typename U>
// decltype(a + b) add_v1(T a, U b) { return a + b; }  // ERROR: a, b not in scope!

// With trailing return type - WORKS in C++11:
template<typename T, typename U>
auto add_v2(T a, U b) -> decltype(a + b) { return a + b; }

// ========= C++14: Auto deduction - no trailing type needed =========

template<typename T, typename U>
auto add_v3(T a, U b) { return a + b; }  // C++14: compiler deduces return type

// ========= Cases where trailing return type is STILL useful in C++14+ =========

// 1. When deduction would give the wrong type:
auto bad_ref(std::vector<int>& v) { return v[0]; }      // Returns int (copy!)
auto good_ref(std::vector<int>& v) -> int& { return v[0]; }  // Returns int&

// 2. SFINAE with decltype (enable/disable based on expression validity):
template<typename T>
auto has_size(T& t) -> decltype(t.size(), void()) {
    std::cout << "Has size: " << t.size() << "\n";
}

int main() {
    auto r1 = add_v2(1, 2.5);   // C++11 style -> double
    auto r2 = add_v3(1, 2.5);   // C++14 style -> double

    std::cout << r1 << " " << r2 << "\n";

    std::vector<int> v{1, 2, 3};
    good_ref(v) = 99;  // Modifies v[0]
    std::cout << "v[0] = " << v[0] << "\n";  // 99

    has_size(v);  // "Has size: 3"
}
```

**How this works:**

- In C++11, `decltype(a + b)` in the trailing position was the only way to express a return type dependent on parameters.
- C++14 added automatic return type deduction (`auto` without trailing type), eliminating most uses.
- Trailing return types remain useful for: explicit reference returns, SFINAE expressions, and clarity.

### Q3: Use a trailing return type to return a lambda type from a function template

Lambda types are unique and unnameable, which makes trailing return types (and `auto` deduction) essential when you need to return them from functions:

```cpp
#include <iostream>
#include <functional>

// The lambda type is unique and unnameable - you need auto/trailing return
// In C++11, you MUST use trailing return type with decltype:

template<typename T>
auto make_adder(T offset) -> decltype([offset](T x) { return x + offset; }) {
    return [offset](T x) { return x + offset; };
}
// Note: This doesn't actually work in C++11 because lambdas weren't allowed in
// unevaluated contexts. It works from C++20.

// C++14+: auto deduction works perfectly for this
template<typename T>
auto make_multiplier(T factor) {
    return [factor](T x) { return x * factor; };
}

// When you need to specify the return type explicitly (e.g., for std::function):
template<typename T>
auto make_counter(T start) -> std::function<T()> {
    auto count = std::make_shared<T>(start);
    return [count]() { return (*count)++; };
}

// Trailing return type for clarity in complex cases:
template<typename Container>
auto first_element(Container& c)
    -> decltype(*c.begin())
{
    return *c.begin();
}

int main() {
    auto add5 = make_multiplier(5);
    std::cout << "5 * 3 = " << add5(3) << "\n";  // 15

    auto counter = make_counter(0);
    std::cout << counter() << "\n";  // 0
    std::cout << counter() << "\n";  // 1
    std::cout << counter() << "\n";  // 2

    std::vector<int> v{10, 20, 30};
    first_element(v) = 99;
    std::cout << "v[0] = " << v[0] << "\n";  // 99
}
```

**How this works:**

- Lambda types are unique - they can't be written explicitly.
- `auto` return type (C++14+) handles this automatically.
- Trailing return types are still useful when you need to ensure the return is a reference, use SFINAE, or explicitly specify `std::function<>`.

---

## Notes

- Trailing return types are required in C++11 when the return type depends on parameter types.
- C++14 auto return type deduction made most trailing return types optional.
- Trailing return types remain useful for: SFINAE (`-> decltype(expr)`), ensuring reference returns, and clarity in complex template metaprogramming.
- In lambdas, trailing return type prevents implicit conversion: `[](int x) -> double { return x; }`.
- Some coding styles use trailing return types consistently for all functions for visual alignment.
