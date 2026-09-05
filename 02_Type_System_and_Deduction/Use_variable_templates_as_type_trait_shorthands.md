# Use Variable Templates as Type Trait Shorthands

**Category:** Type System and Deduction  
**Standard:** C++14  
**Reference:** <https://en.cppreference.com/w/cpp/language/variable_template>  

---

## Topic Overview

### What Are Variable Templates

C++14 introduced **variable templates** - templates that define a family of variables rather than a family of types or functions. The most visible use in day-to-day C++ is the `_v` shorthands for type traits:

```cpp
// Standard library pattern:
template<typename T>
constexpr bool is_integral_v = std::is_integral<T>::value;

// So instead of:
std::is_integral<int>::value   // verbose

// You write:
std::is_integral_v<int>        // clean
```

It's a small thing, but in template-heavy code you'll write these checks constantly, and the `::value` form adds real noise.

### Standard Library `_v` Suffixes

Since C++17, the standard library provides `_v` variable templates for all boolean type traits:

| Verbose Form | Shorthand |
| --- | --- |
| `std::is_integral<T>::value` | `std::is_integral_v<T>` |
| `std::is_pointer<T>::value` | `std::is_pointer_v<T>` |
| `std::is_same<T, U>::value` | `std::is_same_v<T, U>` |
| `std::is_base_of<B, D>::value` | `std::is_base_of_v<B, D>` |

### Beyond Boolean Traits

Variable templates aren't limited to `bool`. They can hold any compile-time constant:

```cpp
// A compile-time constant (not just bool)
template<typename T>
constexpr size_t alignment_of_v = alignof(T);

// A compile-time string (C++20 with constexpr string)
template<typename T>
constexpr const char* type_name_v = "unknown";

template<> constexpr const char* type_name_v<int> = "int";
template<> constexpr const char* type_name_v<double> = "double";
```

### Variable Templates Can Be Specialized

Just like class templates, variable templates support full and partial specialization. This is what makes them useful for compile-time configuration:

```cpp
template<typename T>
constexpr bool is_serializable_v = false;  // default: not serializable

template<>
constexpr bool is_serializable_v<int> = true;  // int is serializable

template<>
constexpr bool is_serializable_v<double> = true;  // double too
```

---

## Self-Assessment

### Q1: Define `template<typename T> constexpr bool is_pointer_v = std::is_pointer<T>::value;`

Defining your own `_v` shorthand is literally one line - this is exactly what the standard library does internally. Here's the pattern applied to several traits at once:

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// Define our own _v shorthand (this is exactly what the standard library does)
template<typename T>
constexpr bool my_is_pointer_v = std::is_pointer<T>::value;

// More examples of _v shorthands
template<typename T>
constexpr bool my_is_const_v = std::is_const<T>::value;

template<typename T>
constexpr bool my_is_reference_v = std::is_reference<T>::value;

template<typename T, typename U>
constexpr bool my_is_same_v = std::is_same<T, U>::value;

// Custom variable template with non-bool value
template<typename T>
constexpr size_t my_sizeof_v = sizeof(T);

int main() {
    std::cout << std::boolalpha;

    // is_pointer_v
    static_assert(my_is_pointer_v<int*>);
    static_assert(!my_is_pointer_v<int>);
    static_assert(my_is_pointer_v<const char*>);
    static_assert(!my_is_pointer_v<int&>);  // references are not pointers

    std::cout << "=== my_is_pointer_v ===\n";
    std::cout << "int*:        " << my_is_pointer_v<int*> << "\n";
    std::cout << "int:         " << my_is_pointer_v<int> << "\n";
    std::cout << "const char*: " << my_is_pointer_v<const char*> << "\n";
    std::cout << "int&:        " << my_is_pointer_v<int&> << "\n";

    // is_const_v
    std::cout << "\n=== my_is_const_v ===\n";
    std::cout << "int:       " << my_is_const_v<int> << "\n";
    std::cout << "const int: " << my_is_const_v<const int> << "\n";

    // is_same_v
    std::cout << "\n=== my_is_same_v ===\n";
    std::cout << "int, int:    " << my_is_same_v<int, int> << "\n";
    std::cout << "int, double: " << my_is_same_v<int, double> << "\n";

    // sizeof_v
    std::cout << "\n=== my_sizeof_v ===\n";
    std::cout << "char:   " << my_sizeof_v<char> << "\n";
    std::cout << "int:    " << my_sizeof_v<int> << "\n";
    std::cout << "double: " << my_sizeof_v<double> << "\n";

    return 0;
}
```

The `static_assert` lines at the top verify correctness at compile time - there's no runtime cost. Everything you see in the output was already known to the compiler before the program ran.

**Output:**

```text
=== my_is_pointer_v ===
int*:        true
int:         false
const char*: true
int&:        false

=== my_is_const_v ===
int:       false
const int: true

=== my_is_same_v ===
int, int:    true
int, double: false

=== my_sizeof_v ===
char:   1
int:    4
double: 8
```

### Q2: Show that variable templates can be specialized just like class templates

Specialization is where variable templates get really useful. You can set per-type policies at compile time with very little code:

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// Primary template: default behavior
template<typename T>
constexpr bool is_serializable_v = std::is_trivially_copyable_v<T>;

// Explicit specialization for std::string (custom rule)
template<>
constexpr bool is_serializable_v<std::string> = true;  // we support string serialization

// Explicit specialization for vector (not supported)
template<>
constexpr bool is_serializable_v<std::vector<int>> = false;

// A type trait with a default value
template<typename T>
constexpr int type_priority_v = 0;  // lowest priority by default

template<> constexpr int type_priority_v<int> = 10;
template<> constexpr int type_priority_v<double> = 8;
template<> constexpr int type_priority_v<std::string> = 5;
template<> constexpr int type_priority_v<bool> = 2;

// Partial specialization for pointers (works with variable templates!)
template<typename T>
constexpr bool is_serializable_v<T*> = false;  // raw pointers -> not serializable

// Partial specialization: all const types inherit from base trait
// (Variable template partial spec works since C++14)
template<typename T>
constexpr int type_priority_v<const T> = type_priority_v<T>;  // const T same as T

int main() {
    std::cout << std::boolalpha;

    // Specializations of is_serializable_v
    std::cout << "=== is_serializable_v ===\n";
    std::cout << "int:         " << is_serializable_v<int> << "\n";        // trivially copyable
    std::cout << "double:      " << is_serializable_v<double> << "\n";     // trivially copyable
    std::cout << "string:      " << is_serializable_v<std::string> << "\n"; // specialized
    std::cout << "vector<int>: " << is_serializable_v<std::vector<int>> << "\n";  // specialized
    std::cout << "int*:        " << is_serializable_v<int*> << "\n";        // partial spec

    // Specializations of type_priority_v
    std::cout << "\n=== type_priority_v ===\n";
    std::cout << "int:    " << type_priority_v<int> << "\n";
    std::cout << "double: " << type_priority_v<double> << "\n";
    std::cout << "string: " << type_priority_v<std::string> << "\n";
    std::cout << "bool:   " << type_priority_v<bool> << "\n";
    std::cout << "char:   " << type_priority_v<char> << "\n";  // default: 0

    // Partial specialization: const T
    std::cout << "\nconst int priority: " << type_priority_v<const int> << "\n";

    // Use in compile-time logic
    static_assert(is_serializable_v<int>);
    static_assert(is_serializable_v<std::string>);
    static_assert(!is_serializable_v<int*>);

    return 0;
}
```

The partial specialization `is_serializable_v<T*>` is worth noticing - it applies to any raw pointer type, not just `int*`. That's the same partial specialization mechanism you'd use with class templates, just with a much shorter definition.

**Output:**

```text
=== is_serializable_v ===
int:         true
double:      true
string:      true
vector<int>: false
int*:        false

=== type_priority_v ===
int:    10
double: 8
string: 5
bool:   2
char:   0

const int priority: 10
```

### Q3: Use a variable template as a shorthand and verify it produces the correct value

If you're ever unsure whether your `_v` shorthand is correct, `static_assert` is your friend. This example shows the pattern for verifying a custom shorthand against both the standard `_v` form and the raw `::value` form:

```cpp
#include <iostream>
#include <type_traits>
#include <cstdint>

// Define _v shorthands and verify they match the class template ::value
template<typename T>
constexpr bool my_is_integral_v = std::is_integral<T>::value;

template<typename T>
constexpr bool my_is_floating_v = std::is_floating_point<T>::value;

template<typename T>
constexpr bool my_is_signed_v = std::is_signed<T>::value;

template<typename T>
constexpr bool my_is_arithmetic_v = std::is_arithmetic<T>::value;

// Verify at compile time that our _v matches the standard _v and ::value
template<typename T>
constexpr void verify_traits() {
    // Our shorthand matches the standard _v
    static_assert(my_is_integral_v<T> == std::is_integral_v<T>);
    static_assert(my_is_floating_v<T> == std::is_floating_point_v<T>);
    static_assert(my_is_signed_v<T> == std::is_signed_v<T>);
    static_assert(my_is_arithmetic_v<T> == std::is_arithmetic_v<T>);

    // Standard _v matches ::value
    static_assert(std::is_integral_v<T> == std::is_integral<T>::value);
}

// Custom compound trait using variable templates
template<typename T>
constexpr bool is_numeric_v = my_is_integral_v<T> || my_is_floating_v<T>;

template<typename T>
constexpr bool is_signed_integer_v = my_is_integral_v<T> && my_is_signed_v<T>;

int main() {
    // Compile-time verification for all types
    verify_traits<int>();
    verify_traits<double>();
    verify_traits<char>();
    verify_traits<bool>();
    verify_traits<uint64_t>();
    verify_traits<void*>();

    std::cout << std::boolalpha;

    // Runtime display
    std::cout << "=== Trait checks ===\n";
    std::cout << "int:      integral=" << my_is_integral_v<int>
              << " floating=" << my_is_floating_v<int>
              << " numeric=" << is_numeric_v<int>
              << " signed_int=" << is_signed_integer_v<int> << "\n";

    std::cout << "double:   integral=" << my_is_integral_v<double>
              << " floating=" << my_is_floating_v<double>
              << " numeric=" << is_numeric_v<double>
              << " signed_int=" << is_signed_integer_v<double> << "\n";

    std::cout << "uint32_t: integral=" << my_is_integral_v<uint32_t>
              << " floating=" << my_is_floating_v<uint32_t>
              << " numeric=" << is_numeric_v<uint32_t>
              << " signed_int=" << is_signed_integer_v<uint32_t> << "\n";

    std::cout << "void*:    integral=" << my_is_integral_v<void*>
              << " floating=" << my_is_floating_v<void*>
              << " numeric=" << is_numeric_v<void*>
              << " signed_int=" << is_signed_integer_v<void*> << "\n";

    std::cout << "\nAll static_asserts passed — _v shorthands produce correct values!\n";

    return 0;
}
```

The compound traits `is_numeric_v` and `is_signed_integer_v` show how you can build higher-level checks by composing simpler variable template traits - the same way you'd combine booleans, just at compile time.

**Output:**

```text
=== Trait checks ===
int:      integral=true floating=false numeric=true signed_int=true
double:   integral=false floating=true numeric=true signed_int=false
uint32_t: integral=true floating=false numeric=true signed_int=false
void*:    integral=false floating=false numeric=false signed_int=false

All static_asserts passed — _v shorthands produce correct values!
```

---

## Notes

- Variable templates were introduced in **C++14**, but the `_v` suffixed standard library traits were added in **C++17**.
- Variable templates support **full and partial specialization** - making them powerful for compile-time configuration.
- Use `inline constexpr` for variable templates in headers (since C++17) to avoid ODR violations.
- The `_t` suffix convention is for type aliases (`remove_pointer_t`, `decay_t`), while `_v` is for value shorthands (`is_same_v`, `is_const_v`).
