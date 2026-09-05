# Write Function Templates with Multiple Type Parameters

**Category:** Templates and Generic Programming  
**Standard:** C++11, C++17 (basic)  
**Reference:** <https://en.cppreference.com/w/cpp/language/function_template>  

---

## Topic Overview

### Function Templates with Multiple Type Parameters

A function template can have multiple type parameters, enabling it to work with combinations of types. The classic use case is arithmetic on mixed types - adding an `int` and a `double`, for instance - where you want the return type to reflect the actual result type rather than forcing a conversion.

```cpp
template <typename T, typename U>
auto add(T a, U b) -> decltype(a + b) {
    return a + b;
}
// add(1, 2.5) -> T=int, U=double, returns double
```

### Template Argument Deduction

The compiler deduces template arguments from the function arguments you pass. You don't need to write the types explicitly most of the time - the compiler figures them out for you:

```cpp
template <typename T, typename U>
void func(T a, U b);

func(42, 3.14);  // T=int, U=double - deduced from arguments
```

### Deducible vs Non-Deducible Parameters

Not every template parameter can be deduced. The rule of thumb is that if a parameter appears directly in the function parameter list, it can be deduced. If it only appears in the return type, inside a nested type, or as a non-type template parameter that isn't in the signature, it cannot.

| Position | Deducible? | Example |
| --- | --- | --- |
| Function parameter type | Yes | `void f(T x)` - T deduced from x |
| Return type (C++11) | No | `T f(int x)` - T not deducible |
| Return type (C++14 auto) | Yes (from return statement) | `auto f(int x) { return x; }` |
| Nested type | No | `void f(typename T::type x)` - T not deducible |
| Template parameter used only in default | No | `template<typename T = int>` |

### Explicit Template Arguments

When deduction doesn't work or gives the wrong answer, you can specify template arguments explicitly. The rule is that explicit arguments are applied left-to-right, and any remaining parameters are still deduced:

```cpp
template <typename R, typename T, typename U>
R convert(T a, U b);

convert<double>(1, 2);    // R=double (explicit), T=int, U=int (deduced)
convert<double, float>(1, 2);  // R=double, T=float (explicit), U=int (deduced)
```

**Rule:** Non-deducible parameters should come first in the template parameter list.

---

## Self-Assessment

### Q1: Write a `min` function template that works with any comparable type without using `std::min`

The examples below build up from a single-type `min` to a variadic one. Notice that `min_mixed` uses `std::common_type_t<T, U>` to figure out the right return type - that is the idiomatic way to handle operations on mixed types without losing precision.

```cpp
#include <iostream>
#include <string>
#include <concepts>

// === Basic: single-type min ===
template <typename T>
const T& my_min(const T& a, const T& b) {
    return (b < a) ? b : a;
}

// === Multi-type min: returns common type ===
template <typename T, typename U>
auto min_mixed(const T& a, const U& b) -> std::common_type_t<T, U> {
    return (b < a) ? b : a;
}

// === Variadic min: works with any number of arguments ===
template <typename T>
const T& min_variadic(const T& a) {
    return a;  // base case: single argument
}

template <typename T, typename... Args>
const T& min_variadic(const T& first, const T& second, const Args&... rest) {
    if (second < first)
        return min_variadic(second, rest...);
    else
        return min_variadic(first, rest...);
}

// === Constrained min (C++20): requires comparable ===
template <std::totally_ordered T>
const T& safe_min(const T& a, const T& b) {
    return (b < a) ? b : a;
}

// === Min with custom comparator ===
template <typename T, typename Compare>
const T& min_cmp(const T& a, const T& b, Compare comp) {
    return comp(b, a) ? b : a;
}

int main() {
    // Basic usage
    std::cout << "min(3, 7) = " << my_min(3, 7) << "\n";          // 3
    std::cout << "min(3.14, 2.71) = " << my_min(3.14, 2.71) << "\n";  // 2.71

    // String comparison (lexicographic)
    std::string s1 = "banana", s2 = "apple";
    std::cout << "min(\"banana\", \"apple\") = " << my_min(s1, s2) << "\n";  // apple

    // Mixed types
    std::cout << "min_mixed(3, 2.5) = " << min_mixed(3, 2.5) << "\n";  // 2.5

    // Variadic
    std::cout << "min_variadic(5,3,8,1,7) = " << min_variadic(5, 3, 8, 1, 7) << "\n";  // 1

    // Custom comparator (compare by absolute value)
    std::cout << "min_cmp(-10, 3, abs_less) = "
              << min_cmp(-10, 3, [](int a, int b) { return std::abs(a) < std::abs(b); })
              << "\n";  // 3 (|3| < |-10|)

    return 0;
}
```

**Expected output:**

```text
min(3, 7) = 3
min(3.14, 2.71) = 2.71
min("banana", "apple") = apple
min_mixed(3, 2.5) = 2.5
min_variadic(5,3,8,1,7) = 1
min_cmp(-10, 3, abs_less) = 3
```

### Q2: Explain template argument deduction rules for a function template with non-deducible parameters

The reason non-deducible parameters trip people up is that they look just like deducible ones in the function signature - the distinction is about context, not syntax. The key rule is that anything appearing in a "nested type" position (like `Container::value_type`) is non-deducible because the compiler would have to reverse-engineer the outer type from the inner one, which is generally impossible.

```cpp
#include <iostream>
#include <vector>
#include <type_traits>
#include <array>

// === Case 1: Return type is non-deducible (C++11) ===
template <typename R, typename T>
R explicit_return(T value) {
    return static_cast<R>(value);
}
// R cannot be deduced from arguments - must be specified explicitly
// explicit_return<double>(42) -> T=int (deduced), R=double (explicit)

// === Case 2: Nested type is non-deducible ===
template <typename Container>
void fill_container(typename Container::value_type val, Container& c) {
    c.push_back(val);
}
// Container cannot be deduced from `val` because typename Container::value_type
// is a "non-deduced context"

// === Case 3: Default template argument for non-deducible ===
template <typename R = double, typename T>
R convert(T value) {
    return static_cast<R>(value);
}
// R has a default - can be omitted: convert(42) -> R=double (default), T=int (deduced)

// === Case 4: Multiple non-deducible parameters ===
template <int N, typename T>
std::array<T, N> make_filled(T value) {
    std::array<T, N> arr;
    arr.fill(value);
    return arr;
}
// N is non-deducible (not a function parameter) - must be explicit
// make_filled<5>(3.14) -> N=5 (explicit), T=double (deduced)

int main() {
    // Case 1: Must specify non-deducible return type
    double d = explicit_return<double>(42);
    std::cout << "explicit_return<double>(42) = " << d << "\n";

    // Case 2: Must specify Container (non-deduced context)
    std::vector<int> v;
    fill_container<std::vector<int>>(42, v);
    // Or let the compiler figure it out from the second parameter:
    // fill_container(42, v);  // Container=vector<int> from `v`, but val param blocks it

    // Case 3: Default template argument
    auto r1 = convert(42);         // R=double (default), T=int
    auto r2 = convert<float>(42);  // R=float (explicit), T=int
    std::cout << "convert(42) = " << r1 << " (double by default)\n";
    std::cout << "convert<float>(42) = " << r2 << "\n";

    // Case 4: NTTP must be explicit
    auto arr = make_filled<5>(3.14);
    std::cout << "make_filled<5>(3.14): ";
    for (auto x : arr) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "\n=== Deduction Rules Summary ===\n";
    std::cout << "1. Types in function params: deducible\n";
    std::cout << "2. Return type (C++11): NOT deducible\n";
    std::cout << "3. Nested types (T::type): NOT deducible\n";
    std::cout << "4. NTTPs not in signature: NOT deducible\n";
    std::cout << "5. Put non-deducible params FIRST in the list\n";

    return 0;
}
```

### Q3: Show how explicit template arguments override deduction in a function call

Explicit arguments give you two things: they resolve ambiguity when the compiler can't deduce (like `add(1, 2.5)` where `T` would need to be both `int` and `double`), and they let you force conversions that deduction would reject. Be aware that forced conversions can cause narrowing - `add<int>(1, 2.5)` silently truncates the `double`.

```cpp
#include <iostream>
#include <string>
#include <type_traits>

template <typename T, typename U>
void show_types(T a, U b) {
    std::cout << "  T = " << typeid(T).name()
              << ", U = " << typeid(U).name() << "\n";
    std::cout << "  a = " << a << ", b = " << b << "\n";
}

template <typename T>
T add(T a, T b) {
    return a + b;
}

int main() {
    // === Deduction ===
    std::cout << "=== Deduced ===\n";
    show_types(42, 3.14);          // T=int, U=double
    show_types("hello", 'c');      // T=const char*, U=char

    // === Explicit overrides deduction ===
    std::cout << "\n=== Explicit (overriding deduction) ===\n";
    show_types<double>(42, 3.14);          // T=double (explicit), U=double (deduced)
    show_types<double, int>(42, 3.14);     // T=double (explicit), U=int (explicit)
    // 42 is converted to double, 3.14 is converted to int!

    // === Resolving ambiguity with explicit arguments ===
    std::cout << "\n=== Resolving ambiguity ===\n";
    // add(1, 2.5);  // ERROR: T can't be both int and double
    auto r1 = add<double>(1, 2.5);  // T=double, 1 converts to 1.0
    auto r2 = add<int>(1, 2.5);     // T=int, 2.5 truncates to 2
    std::cout << "add<double>(1, 2.5) = " << r1 << "\n";  // 3.5
    std::cout << "add<int>(1, 2.5) = " << r2 << "\n";      // 3

    // === Forcing specific instantiation ===
    std::cout << "\n=== Forcing specific types ===\n";
    std::string s1 = "hello";
    std::string s2 = " world";
    auto s3 = add<std::string>(s1, s2);  // works
    std::cout << "add<string>: " << s3 << "\n";

    // Without explicit: add("hello", " world") -> T=const char* -> ERROR (no + for pointers)
    // With explicit:    add<std::string>("hello", " world") -> converts both to string

    std::cout << "\n=== Rules ===\n";
    std::cout << "1. Explicit args are applied left-to-right\n";
    std::cout << "2. Remaining params are deduced from function args\n";
    std::cout << "3. Explicit args can force conversions\n";
    std::cout << "4. Useful for resolving ambiguity when T appears multiple times\n";

    return 0;
}
```

**Expected output:**

```text
=== Deduced ===
  T = i, U = d
  a = 42, b = 3.14
  T = PKc, U = c
  a = hello, b = c

=== Explicit (overriding deduction) ===
  T = d, U = d
  a = 42, b = 3.14
  T = d, U = i
  a = 42, b = 3

=== Resolving ambiguity ===
add<double>(1, 2.5) = 3.5
add<int>(1, 2.5) = 3

=== Forcing specific types ===
add<string>: hello world

=== Rules ===
1. Explicit args are applied left-to-right
2. Remaining params are deduced from function args
3. Explicit args can force conversions
4. Useful for resolving ambiguity when T appears multiple times
```

---

## Notes

- Place **non-deducible** template parameters first in the parameter list so users specify only what's needed.
- In C++14, `auto` return type makes the return type deducible from the function body.
- In C++20, `auto` parameters create **abbreviated function templates**: `auto add(auto a, auto b)`.
- `std::common_type_t<T, U>` is the idiomatic way to find a common return type for mixed-type operations.
- Explicit template arguments are applied left-to-right; trailing parameters can still be deduced.
- Be aware: explicit arguments can cause **implicit conversions** that deduction would reject.
