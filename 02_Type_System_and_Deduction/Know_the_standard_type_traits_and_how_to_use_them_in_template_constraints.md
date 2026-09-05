# Know the standard type traits and how to use them in template constraints

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/header/type_traits>  

---

## Topic Overview

Type traits are compile-time predicates and transformations on types. C++20 lets you use them directly in **concepts** and **requires clauses** to constrain templates. Think of them as questions you can ask about a type at compile time: "Is this an integer?", "Does it have a trivial copy constructor?", "What happens if I strip the reference off this?"

### Categories of Type Traits

| Category | Examples | Purpose |
| --- | --- | --- |
| **Primary classification** | `is_integral_v`, `is_floating_point_v`, `is_class_v`, `is_enum_v` | Check what kind of type it is |
| **Composite classification** | `is_arithmetic_v`, `is_scalar_v`, `is_object_v` | Combine primary categories |
| **Type properties** | `is_const_v`, `is_trivially_copyable_v`, `is_nothrow_move_constructible_v` | Check type properties |
| **Type relationships** | `is_same_v`, `is_base_of_v`, `is_convertible_v` | Compare two types |
| **Type transformations** | `remove_const_t`, `decay_t`, `remove_reference_t`, `add_pointer_t` | Transform types |

### Using Traits to Constrain Templates

C++ has given us progressively cleaner ways to gate a template on type properties. Here's how the same constraint looks in each era:

```cpp
#include <type_traits>
#include <concepts>

// C++11 style: SFINAE with enable_if
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
T add_sfinae(T a, T b) { return a + b; }

// C++20 style: requires clause
template<typename T>
    requires std::is_integral_v<T>
T add_requires(T a, T b) { return a + b; }

// C++20 style: concept
template<std::integral T>
T add_concept(T a, T b) { return a + b; }
```

For new code, prefer the C++20 style. The `enable_if` version is worth knowing because you'll encounter it in older codebases and in the standard library itself.

### std::decay_t vs std::remove_cvref_t (C++20)

These two are often confused. The difference is that `decay_t` also converts arrays to pointers and function types to function pointers - it mimics what happens when you pass something by value to a function:

| Trait | Removes | Also does |
| --- | --- | --- |
| `remove_reference_t<T>` | References (`&`, `&&`) | Nothing else |
| `remove_cv_t<T>` | `const`, `volatile` | Nothing else |
| `remove_cvref_t<T>` (C++20) | References + const/volatile | Nothing else |
| `decay_t<T>` | References + const/volatile | **Array->pointer**, **function->function pointer** |

```cpp
// With an array type:
using T = const int[5];
// remove_cvref_t<T> -> int[5]  (still an array!)
// decay_t<T>         -> int*   (array decayed to pointer)

// With a function type:
using F = void(int);
// remove_cvref_t<F> -> void(int)   (still a function type)
// decay_t<F>         -> void(*)(int) (function pointer)
```

---

## Self-Assessment

### Q1: Use `is_integral_v`, `is_same_v`, and `is_base_of_v` to gate a function template

Three different traits, three different use cases. Notice the `remove_cvref_t` in the second example - that's needed so `count_chars` accepts `const std::string&` and `std::string&&` too, not just `std::string`:

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// 1. Gate with is_integral_v - only works with integer types
template<typename T>
    requires std::is_integral_v<T>
T safe_divide(T numerator, T denominator) {
    if (denominator == 0) return 0;  // avoid division by zero
    return numerator / denominator;
}

// 2. Gate with is_same_v - only works with specific type
template<typename T>
    requires std::is_same_v<std::remove_cvref_t<T>, std::string>
size_t count_chars(T&& s) {
    return s.size();
}

// 3. Gate with is_base_of_v - only works with derived classes
struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

struct Circle : Shape {
    double radius;
    Circle(double r) : radius(r) {}
    double area() const override { return 3.14159 * radius * radius; }
};

struct Square : Shape {
    double side;
    Square(double s) : side(s) {}
    double area() const override { return side * side; }
};

template<typename T>
    requires std::is_base_of_v<Shape, T>
void print_area(const T& shape) {
    std::cout << "Area: " << shape.area() << "\n";
}

int main() {
    // is_integral_v:
    std::cout << safe_divide(10, 3) << "\n";    // 3
    std::cout << safe_divide(100, 0) << "\n";   // 0
    // safe_divide(3.14, 2.0);  // ERROR: double is not integral

    // is_same_v:
    std::cout << count_chars(std::string("hello")) << "\n";  // 5
    // count_chars(42);  // ERROR: int is not string

    // is_base_of_v:
    Circle c(5.0);
    Square s(3.0);
    print_area(c);  // Area: 78.5398
    print_area(s);  // Area: 9
    // print_area(42);  // ERROR: int does not derive from Shape
}
```

### Q2: Explain the difference between `std::decay_t` and `std::remove_cvref_t`

The two traits diverge when arrays and function types are involved. For plain references and cv-qualifiers they do the same thing, but `decay_t` goes further. The `typeid(...).name()` output is implementation-defined but still shows the difference:

```cpp
#include <iostream>
#include <type_traits>

template<typename T>
void show_decay() {
    using Decayed = std::decay_t<T>;
    using Stripped = std::remove_cvref_t<T>;

    std::cout << "  decay_t:        " << typeid(Decayed).name() << "\n";
    std::cout << "  remove_cvref_t: " << typeid(Stripped).name() << "\n";
    std::cout << "  Same? " << std::boolalpha
              << std::is_same_v<Decayed, Stripped> << "\n\n";
}

int main() {
    // Case 1: Simple reference - both strip it
    std::cout << "const int& :\n";
    show_decay<const int&>();
    // decay_t -> int, remove_cvref_t -> int  (SAME)

    // Case 2: Array - decay converts to pointer!
    std::cout << "int[5] :\n";
    show_decay<int[5]>();
    // decay_t -> int*, remove_cvref_t -> int[5]  (DIFFERENT!)

    // Case 3: Function - decay converts to function pointer!
    std::cout << "void(int) :\n";
    show_decay<void(int)>();
    // decay_t -> void(*)(int), remove_cvref_t -> void(int)  (DIFFERENT!)

    // Case 4: Pointer - both leave it alone
    std::cout << "const int* :\n";
    show_decay<const int*>();
    // decay_t -> const int*, remove_cvref_t -> const int*  (SAME)

    // Rule of thumb:
    // - Use decay_t when you want what "pass-by-value" gives you
    // - Use remove_cvref_t when you just want to strip qualifiers/refs
}
```

`decay_t` mimics what happens when you pass an argument by value to a template - arrays become pointers, functions become function pointers, and cv-qualifiers/references are stripped. `remove_cvref_t` only strips `const`, `volatile`, and references - it preserves array and function types.

### Q3: Write a custom type trait that detects if a type has a `.size()` member

This is the classic SFINAE detection pattern, shown in both the C++11/14 style and the cleaner C++20 concept form. The `std::void_t<...>` trick works by substituting the expression into a template - if the expression is ill-formed (no `.size()`), substitution fails and the `false_type` specialization is selected:

```cpp
#include <iostream>
#include <type_traits>
#include <vector>
#include <string>
#include <array>
#include <concepts>
#include <cstddef>
#include <utility>

// Method 1: SFINAE-based detection (C++11/14)
template<typename T, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>>
    : std::true_type {};

template<typename T>
inline constexpr bool has_size_v = has_size<T>::value;

// Method 2: Concept-based detection (C++20)
template<typename T>
concept Sizable = requires(T t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

// Use the trait to constrain a function
template<typename T>
    requires has_size_v<T>
auto safe_empty_check(const T& container) -> bool {
    return container.size() == 0;
}

// Or use the concept directly
template<Sizable T>
void print_size(const T& container) {
    std::cout << "Size: " << container.size() << "\n";
}

int main() {
    // Test the trait
    static_assert(has_size_v<std::vector<int>>);       // has .size()
    static_assert(has_size_v<std::string>);             // has .size()
    static_assert(!has_size_v<int>);                    // no .size()
    static_assert(!has_size_v<int*>);                   // no .size()
    static_assert(has_size_v<std::array<int, 5>>);     // has .size()

    // Test the concept
    static_assert(Sizable<std::vector<int>>);
    static_assert(!Sizable<int>);

    // Use constrained functions
    std::vector<int> v{1, 2, 3};
    print_size(v);                  // Size: 3
    std::cout << safe_empty_check(v) << "\n";  // 0 (false)

    // print_size(42);  // ERROR: int doesn't satisfy Sizable
}
```

---

## Notes

- Always prefer `_v` variable templates (`is_integral_v<T>`) over `::value` - shorter and clearer.
- Always prefer `_t` alias templates (`remove_const_t<T>`) over `::type`.
- C++20 concepts are the modern replacement for SFINAE-based `enable_if` - prefer concepts when targeting C++20.
- `std::void_t<...>` (C++17) is the key trick for SFINAE detection traits - it maps any valid expression to `void`.
- Custom traits follow the convention: struct name inherits from `std::true_type` or `std::false_type`, with a `_v` variable template shorthand.
