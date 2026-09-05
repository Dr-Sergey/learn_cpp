# Use `std::remove_pointer` and `std::add_pointer` in Pointer Type Transformations

**Category:** Type System and Deduction  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/types/remove_pointer>  

---

## Topic Overview

### What Are These Traits

`std::remove_pointer` and `std::add_pointer` are type transformation traits that manipulate pointer types at compile time. They are the pointer-level counterparts to the reference traits you've seen elsewhere:

| Trait | Input | Output |
| --- | --- | --- |
| `remove_pointer_t<int*>` | `int*` | `int` |
| `remove_pointer_t<int**>` | `int**` | `int*` (one level only) |
| `remove_pointer_t<int>` | `int` | `int` (no change) |
| `add_pointer_t<int>` | `int` | `int*` |
| `add_pointer_t<int&>` | `int&` | `int*` (removes ref, adds ptr) |
| `add_pointer_t<void>` | `void` | `void*` |

### Core Syntax

Let's see the basic transformations in code:

```cpp
#include <type_traits>

// Remove one pointer level
using T1 = std::remove_pointer_t<int*>;        // int
using T2 = std::remove_pointer_t<const int*>;  // const int
using T3 = std::remove_pointer_t<int**>;       // int*

// Add one pointer level
using T4 = std::add_pointer_t<int>;    // int*
using T5 = std::add_pointer_t<int&>;   // int*  (reference removed first)
using T6 = std::add_pointer_t<void>;   // void*
```

Notice that `add_pointer_t` on a reference type first removes the reference and then adds the pointer. That matches what `decltype(&x)` gives you when `x` is an lvalue.

### `std::add_lvalue_reference` and `void`

One subtle design point worth knowing: `std::add_lvalue_reference<void>` is valid and returns `void` (not `void&`, which would be illegal). This is achieved via SFINAE internally - the trait gracefully handles types that can't form references:

```cpp
static_assert(std::is_same_v<std::add_lvalue_reference_t<int>, int&>);
static_assert(std::is_same_v<std::add_lvalue_reference_t<void>, void>);  // no error!
```

### Related Traits

Here is the full family of pointer and reference manipulation traits for reference:

| Trait | Purpose |
| --- | --- |
| `remove_pointer` | Strip `*` from pointer type |
| `add_pointer` | Add `*` to type (handles refs) |
| `remove_reference` | Strip `&`/`&&` |
| `remove_cv` | Strip `const`/`volatile` |
| `remove_cvref` (C++20) | Strip cv-qualifiers and references |
| `decay` | Mimics pass-by-value transformations |

---

## Self-Assessment

### Q1: Use `std::remove_pointer_t` to strip a pointer from a type in a template metafunction

This example builds a `pointee_type` metafunction that works for both raw pointers and smart pointers via specialization:

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <memory>

// A metafunction that extracts the element type from a pointer
template<typename T>
struct pointee_type {
    // For raw pointers: remove the pointer
    using type = std::remove_pointer_t<T>;
};

// Specialization for smart pointers
template<typename T>
struct pointee_type<std::unique_ptr<T>> {
    using type = T;
};

template<typename T>
struct pointee_type<std::shared_ptr<T>> {
    using type = T;
};

template<typename T>
using pointee_type_t = typename pointee_type<T>::type;

// A function that dereferences any pointer-like thing
template<typename T>
auto safe_deref(T&& ptr) -> std::remove_pointer_t<std::decay_t<T>>& {
    return *ptr;
}

// Print the pointed-to type name (simplified)
template<typename T>
void describe_pointer() {
    using Pointed = std::remove_pointer_t<T>;
    std::cout << "  is_pointer: " << std::is_pointer_v<T> << "\n";
    std::cout << "  pointed-to type same as original: "
              << std::is_same_v<T, Pointed> << "\n";
}

int main() {
    std::cout << std::boolalpha;

    // Basic remove_pointer usage
    static_assert(std::is_same_v<std::remove_pointer_t<int*>, int>);
    static_assert(std::is_same_v<std::remove_pointer_t<const double*>, const double>);
    static_assert(std::is_same_v<std::remove_pointer_t<char**>, char*>);
    static_assert(std::is_same_v<std::remove_pointer_t<int>, int>);  // no change

    // pointee_type metafunction
    static_assert(std::is_same_v<pointee_type_t<int*>, int>);
    static_assert(std::is_same_v<pointee_type_t<std::string*>, std::string>);
    static_assert(std::is_same_v<pointee_type_t<std::unique_ptr<double>>, double>);
    static_assert(std::is_same_v<pointee_type_t<std::shared_ptr<int>>, int>);

    std::cout << "=== describe_pointer ===\n";
    std::cout << "int*:\n";  describe_pointer<int*>();
    std::cout << "int:\n";   describe_pointer<int>();
    std::cout << "int**:\n"; describe_pointer<int**>();

    // safe_deref with raw pointer
    int x = 42;
    int* p = &x;
    safe_deref(p) = 100;
    std::cout << "\nx after safe_deref(p)=100: " << x << "\n";

    return 0;
}
```

**Output:**

```text
=== describe_pointer ===
int*:
  is_pointer: true
  pointed-to type same as original: false
int:
  is_pointer: false
  pointed-to type same as original: true
int**:
  is_pointer: true
  pointed-to type same as original: false

x after safe_deref(p)=100: 100
```

### Q2: Build a type trait that repeatedly removes all pointer levels from `T****` to `T`

`remove_pointer` only strips one level at a time. If you need to strip all levels, you write a recursive trait. The recursion bottoms out at the non-pointer case:

```cpp
#include <iostream>
#include <type_traits>

// Recursive type trait: strip ALL pointer levels
template<typename T>
struct remove_all_pointers {
    using type = T;  // base case: not a pointer
};

template<typename T>
struct remove_all_pointers<T*> {
    using type = typename remove_all_pointers<T>::type;  // recurse
};

// Also handle cv-qualified pointers
template<typename T>
struct remove_all_pointers<T* const> {
    using type = typename remove_all_pointers<T>::type;
};

template<typename T>
struct remove_all_pointers<T* volatile> {
    using type = typename remove_all_pointers<T>::type;
};

template<typename T>
struct remove_all_pointers<T* const volatile> {
    using type = typename remove_all_pointers<T>::type;
};

template<typename T>
using remove_all_pointers_t = typename remove_all_pointers<T>::type;

// Count pointer depth
template<typename T>
struct pointer_depth {
    static constexpr int value = 0;
};

template<typename T>
struct pointer_depth<T*> {
    static constexpr int value = 1 + pointer_depth<T>::value;
};

template<typename T>
constexpr int pointer_depth_v = pointer_depth<T>::value;

int main() {
    std::cout << std::boolalpha;

    // Test removing all pointer levels
    static_assert(std::is_same_v<remove_all_pointers_t<int>, int>);
    static_assert(std::is_same_v<remove_all_pointers_t<int*>, int>);
    static_assert(std::is_same_v<remove_all_pointers_t<int**>, int>);
    static_assert(std::is_same_v<remove_all_pointers_t<int***>, int>);
    static_assert(std::is_same_v<remove_all_pointers_t<int****>, int>);
    static_assert(std::is_same_v<remove_all_pointers_t<const double***>, const double>);

    // Pointer depth
    static_assert(pointer_depth_v<int> == 0);
    static_assert(pointer_depth_v<int*> == 1);
    static_assert(pointer_depth_v<int**> == 2);
    static_assert(pointer_depth_v<int****> == 4);

    std::cout << "=== Pointer Depth ===\n";
    std::cout << "int:      " << pointer_depth_v<int> << "\n";
    std::cout << "int*:     " << pointer_depth_v<int*> << "\n";
    std::cout << "int**:    " << pointer_depth_v<int**> << "\n";
    std::cout << "int****:  " << pointer_depth_v<int****> << "\n";

    // Contrast: remove_pointer only strips one level
    static_assert(std::is_same_v<std::remove_pointer_t<int****>, int***>);
    // But remove_all_pointers strips all:
    static_assert(std::is_same_v<remove_all_pointers_t<int****>, int>);

    std::cout << "\nAll static_asserts passed!\n";

    return 0;
}
```

**Output:**

```text
=== Pointer Depth ===
int:      0
int*:     1
int**:    2
int****:  4

All static_asserts passed!
```

### Q3: Explain how `std::add_lvalue_reference` handles `void` without error

The reason `add_lvalue_reference<void>` doesn't blow up is SFINAE. The implementation tries to form `T&`, and if that expression is ill-formed (as it is for `void`), it falls back to returning `T` unchanged. Here is a simplified version of how that works:

```cpp
#include <iostream>
#include <type_traits>

// The key insight: void& is illegal in C++.
// But std::add_lvalue_reference<void> doesn't try to form void&.
// Instead, it uses SFINAE internally:

// Simplified implementation:
namespace detail {
    template<typename T>
    auto try_add_lref(int) -> std::type_identity<T&>;  // preferred if T& is valid

    template<typename T>
    auto try_add_lref(...) -> std::type_identity<T>;    // fallback for void, etc.
}

template<typename T>
using my_add_lvalue_reference_t = typename decltype(detail::try_add_lref<T>(0))::type;

int main() {
    std::cout << std::boolalpha;

    // Standard behavior:
    static_assert(std::is_same_v<std::add_lvalue_reference_t<int>, int&>);
    static_assert(std::is_same_v<std::add_lvalue_reference_t<int&>, int&>);   // already ref
    static_assert(std::is_same_v<std::add_lvalue_reference_t<int&&>, int&>);  // && -> &
    static_assert(std::is_same_v<std::add_lvalue_reference_t<void>, void>);   // no error!

    // Our simplified version matches:
    static_assert(std::is_same_v<my_add_lvalue_reference_t<int>, int&>);
    static_assert(std::is_same_v<my_add_lvalue_reference_t<void>, void>);

    // Same applies to add_rvalue_reference:
    static_assert(std::is_same_v<std::add_rvalue_reference_t<int>, int&&>);
    static_assert(std::is_same_v<std::add_rvalue_reference_t<void>, void>);

    // And add_pointer - it handles references gracefully:
    static_assert(std::is_same_v<std::add_pointer_t<int>, int*>);
    static_assert(std::is_same_v<std::add_pointer_t<int&>, int*>);    // ref stripped
    static_assert(std::is_same_v<std::add_pointer_t<void>, void*>);   // void* is legal

    // Why this matters: std::declval<T>() depends on add_rvalue_reference
    // working for void. Since add_rvalue_reference_t<void> is just void,
    // declval<void>() has return type void - which is valid in C++.

    std::cout << "=== add_lvalue_reference results ===\n";
    std::cout << "int -> int&:    " << std::is_same_v<std::add_lvalue_reference_t<int>, int&> << "\n";
    std::cout << "void -> void:   " << std::is_same_v<std::add_lvalue_reference_t<void>, void> << "\n";
    std::cout << "int& -> int&:   " << std::is_same_v<std::add_lvalue_reference_t<int&>, int&> << "\n";
    std::cout << "int&& -> int&:  " << std::is_same_v<std::add_lvalue_reference_t<int&&>, int&> << "\n";

    return 0;
}
```

**Output:**

```text
=== add_lvalue_reference results ===
int -> int&:    true
void -> void:   true
int& -> int&:   true
int&& -> int&:  true
```

The trait uses SFINAE - it attempts to form `T&`, and if that fails (as for `void`), it falls back to returning `T` unchanged. This is why these traits are preferred over manually writing `T&` in template code.

---

## Notes

- **`remove_pointer`** only strips one level. For multi-level pointers, you need a recursive trait (as shown in Q2).
- **`add_pointer_t<T&>`** first removes the reference, then adds a pointer. This matches the behavior of `decltype(&x)` where `x` is an lvalue.
- Use `std::decay_t` when you want array-to-pointer decay, function-to-pointer decay, and cv-removal all at once.
- These traits are essential building blocks in template metaprogramming and are used internally by many standard library components.
