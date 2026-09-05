# Understand `std::conditional_t` for Selecting Types at Compile Time

**Category:** Type System and Deduction  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/types/conditional>  

---

## Topic Overview

### What Is `std::conditional_t`

`std::conditional_t<B, T, F>` is a compile-time type selector - it yields type `T` if the boolean `B` is `true`, or type `F` if `B` is `false`. It's the type-level equivalent of the ternary operator `?:`.

```cpp
#include <type_traits>

// conditional_t<true,  int, double> = int
// conditional_t<false, int, double> = double

static_assert(std::is_same_v<std::conditional_t<true, int, double>, int>);
static_assert(std::is_same_v<std::conditional_t<false, int, double>, double>);
```

### How It Works Internally

The implementation is elegantly simple - just a primary template and one partial specialization:

```cpp
// Simplified implementation:
template<bool B, typename T, typename F>
struct conditional { using type = T; };

template<typename T, typename F>
struct conditional<false, T, F> { using type = F; };

template<bool B, typename T, typename F>
using conditional_t = typename conditional<B, T, F>::type;
```

The key insight: partial specialization for `false` selects `F`, the primary template selects `T`.

### Common Use Cases

| Pattern | Expression |
| --- | --- |
| Cheap parameter passing | `conditional_t<sizeof(T) <= 8, T, const T&>` |
| Signed/unsigned selection | `conditional_t<std::is_signed_v<T>, int, unsigned>` |
| Const propagation | `conditional_t<IsConst, const T*, T*>` |
| Platform-specific types | `conditional_t<sizeof(void*) == 8, uint64_t, uint32_t>` |
| Enable/disable wrapper | `conditional_t<NeedsMutex, std::mutex, NullMutex>` |

### `cheap_param_t` - The Classic Example

When passing parameters, small types should be passed **by value** (fast copy), while large types should be passed **by const reference** (avoid copy). `conditional_t` lets you encode that rule as a type alias:

```cpp
template<typename T>
using cheap_param_t = std::conditional_t<
    (sizeof(T) <= 2 * sizeof(void*))   // Fits in 1-2 registers?
    && std::is_trivially_copyable_v<T>, // Safe to copy bitwise?
    T,                                   // YES: pass by value
    const T&                             // NO: pass by const reference
>;

// Usage:
template<typename T>
void process(cheap_param_t<T> value) { /* ... */ }
// int -> passed by value
// std::string -> passed by const reference
```

### Nesting `conditional_t` (and When Not To)

You can chain conditions for multi-way selection, but deep nesting quickly becomes unreadable. Here's the problem and the two cleaner alternatives:

```cpp
// Multi-way type selection with nesting
template<typename T>
using numeric_storage_t = std::conditional_t<
    std::is_integral_v<T>,
    long long,                              // integral -> long long
    std::conditional_t<
        std::is_floating_point_v<T>,
        double,                              // floating -> double
        T                                    // other -> keep as-is
    >
>;
```

But deeply nested `conditional_t` becomes unreadable. Alternatives for multi-way selection:

```cpp
// Alternative 1: Template specialization
template<typename T, typename = void>
struct storage_type { using type = T; };

template<typename T>
struct storage_type<T, std::enable_if_t<std::is_integral_v<T>>> {
    using type = long long;
};

template<typename T>
struct storage_type<T, std::enable_if_t<std::is_floating_point_v<T>>> {
    using type = double;
};

template<typename T>
using storage_type_t = typename storage_type<T>::type;

// Alternative 2: if constexpr with lambda (C++20)
template<typename T>
using storage_v2 = decltype([]{
    if constexpr (std::is_integral_v<T>) return long long{};
    else if constexpr (std::is_floating_point_v<T>) return double{};
    else return T{};
}());
```

### Does `conditional_t` Evaluate Both Branches

This is the most important gotcha, and the reason this trips people up is that the name "conditional" implies only one branch matters. In reality, **both branch types must be valid (well-formed)**, but only the chosen branch becomes the result. This is different from `if constexpr`, which truly discards the unchosen branch:

```cpp
// BOTH int and double must be valid types — they always are
std::conditional_t<true, int, double>  // OK: yields int, but double must be valid too

// Potential problem:
template<typename T>
using maybe_deref = std::conditional_t<
    std::is_pointer_v<T>,
    std::remove_pointer_t<T>,  // This ALWAYS gets instantiated!
    T
>;
// This works fine because remove_pointer_t<T> is always valid (returns T for non-pointers)

// But this would fail:
template<typename T>
using broken = std::conditional_t<
    std::is_pointer_v<T>,
    typename T::element_type,  // ERROR if T has no element_type!
    T
>;
// Both branches are instantiated, so T::element_type must exist even when B=false
```

**Solution:** Use `if constexpr`, template specialization, or `std::enable_if` to truly avoid instantiating a branch.

---

## Self-Assessment

### Q1: Use `std::conditional_t<sizeof(T) <= 8, T, const T&>` to define a `cheap_param` type

The `show_value` function here prints which path was chosen at compile time, so you can see the rule in action for each type:

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <array>

// cheap_param_t: pass small trivially-copyable types by value, others by const ref
template<typename T>
using cheap_param_t = std::conditional_t<
    (sizeof(T) <= sizeof(void*) * 2) && std::is_trivially_copyable_v<T>,
    T,
    const T&
>;

// A function template using cheap_param_t
template<typename T>
void show_value(cheap_param_t<T> value) {
    if constexpr (std::is_same_v<cheap_param_t<T>, T>) {
        std::cout << "  [by value]     ";
    } else {
        std::cout << "  [by const ref] ";
    }
    std::cout << "sizeof(T)=" << sizeof(T) << "\n";
}

// Real-world usage: a comparison function
template<typename T>
bool equals(cheap_param_t<T> a, cheap_param_t<T> b) {
    return a == b;
}

struct BigStruct {
    char data[256];
    bool operator==(const BigStruct& other) const {
        return std::equal(std::begin(data), std::end(data), std::begin(other.data));
    }
};

int main() {
    // Verify deduced parameter types
    static_assert(std::is_same_v<cheap_param_t<int>, int>);           // 4 bytes, trivial -> by value
    static_assert(std::is_same_v<cheap_param_t<double>, double>);     // 8 bytes, trivial -> by value
    static_assert(std::is_same_v<cheap_param_t<std::string>, const std::string&>);  // large -> by ref
    static_assert(std::is_same_v<cheap_param_t<BigStruct>, const BigStruct&>);      // 256 bytes -> by ref

    // A pointer-size struct passes by value
    struct Pair { int a, b; };
    static_assert(sizeof(Pair) <= 2 * sizeof(void*));
    static_assert(std::is_same_v<cheap_param_t<Pair>, Pair>);

    // Show in action
    std::cout << "Parameter passing strategy:\n";
    show_value<int>(42);
    show_value<double>(3.14);
    show_value<std::string>(std::string("hello"));
    show_value<BigStruct>(BigStruct{});

    // Comparison
    std::cout << "\nComparisons:\n";
    std::cout << "  equals<int>(3, 3) = " << std::boolalpha << equals<int>(3, 3) << "\n";
    std::cout << "  equals<int>(3, 4) = " << equals<int>(3, 4) << "\n";

    return 0;
}
```

**Output:**

```text
Parameter passing strategy:
  [by value]     sizeof(T)=4
  [by value]     sizeof(T)=8
  [by const ref] sizeof(T)=...
  [by const ref] sizeof(T)=256

Comparisons:
  equals<int>(3, 3) = true
  equals<int>(3, 4) = false
```

**How this works:**

- `conditional_t` selects `T` (by value) when the type is small AND trivially copyable
- For large types or types with non-trivial copy (like `std::string`), it selects `const T&`
- The threshold `2 * sizeof(void*)` = 16 bytes on 64-bit platforms, which covers most primitive types and small structs
- `is_trivially_copyable_v` ensures we only pass by value when copying is genuinely cheap (no constructor calls)

### Q2: Show a case where nested `conditional_t` can be replaced with a specialization for clarity

All three approaches produce the same result. The point is to compare how each reads as the number of cases grows:

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// === MESSY: Nested conditional_t (hard to read, extend) ===
template<typename T>
using storage_messy = std::conditional_t<
    std::is_integral_v<T>,
    long long,
    std::conditional_t<
        std::is_floating_point_v<T>,
        long double,
        std::conditional_t<
            std::is_pointer_v<T>,
            void*,
            T  // default
        >
    >
>;

// === CLEAN: Template specialization ===
template<typename T, typename = void>
struct storage_selector { using type = T; };  // default

template<typename T>
struct storage_selector<T, std::enable_if_t<std::is_integral_v<T>>> {
    using type = long long;
};

template<typename T>
struct storage_selector<T, std::enable_if_t<std::is_floating_point_v<T>>> {
    using type = long double;
};

template<typename T>
struct storage_selector<T, std::enable_if_t<std::is_pointer_v<T>>> {
    using type = void*;
};

template<typename T>
using storage_clean = typename storage_selector<T>::type;

// === CLEANEST (C++20): Concepts with consteval ===
template<typename T>
struct storage_concept {
    using type = T;  // default
};

template<std::integral T>
struct storage_concept<T> {
    using type = long long;
};

template<std::floating_point T>
struct storage_concept<T> {
    using type = long double;
};

// (pointer concept requires custom, shown for completeness)

int main() {
    // All three approaches produce the same results:

    // Integral -> long long
    static_assert(std::is_same_v<storage_messy<int>, long long>);
    static_assert(std::is_same_v<storage_clean<int>, long long>);
    static_assert(std::is_same_v<storage_concept<int>::type, long long>);

    // Floating -> long double
    static_assert(std::is_same_v<storage_messy<float>, long double>);
    static_assert(std::is_same_v<storage_clean<float>, long double>);
    static_assert(std::is_same_v<storage_concept<float>::type, long double>);

    // Default -> T
    static_assert(std::is_same_v<storage_messy<std::string>, std::string>);
    static_assert(std::is_same_v<storage_clean<std::string>, std::string>);

    std::cout << "All static_asserts passed!\n";
    std::cout << "\nWhen to use each approach:\n";
    std::cout << "  conditional_t: 2-way selection (simple if/else on types)\n";
    std::cout << "  specialization: multi-way selection, extensible, clear\n";
    std::cout << "  concepts (C++20): cleanest, auto-disambiguates\n";

    return 0;
}
```

**Output:**

```text
All static_asserts passed!

When to use each approach:
  conditional_t: 2-way selection (simple if/else on types)
  specialization: multi-way selection, extensible, clear
  concepts (C++20): cleanest, auto-disambiguates
```

**Key advantage of specialization:**

- Each "branch" is a separate, self-contained specialization - easy to add, remove, or modify
- No nesting depth limit
- Each case is independently testable
- With concepts (C++20), the specializations are even cleaner and handle ambiguity via subsumption

### Q3: Explain how `conditional_t` avoids evaluating the unchosen branch's instantiation

**Short answer: It does NOT avoid evaluating the unchosen branch.**

Both `T` and `F` in `conditional_t<B, T, F>` must be **valid, well-formed types**. The only thing that's conditional is which one becomes the result. This is different from `if constexpr`, which truly discards the unchosen branch.

The "lazy conditional" pattern is the workaround - you select between wrapper types, and only access `::type` on the selected wrapper:

```cpp
#include <iostream>
#include <type_traits>

// Demonstration: both branches must be valid types

// This WORKS because both are always valid:
template<typename T>
using safe = std::conditional_t<
    std::is_integral_v<T>,
    int,
    double
>;
// int and double are always valid, regardless of T

// This ALSO works because remove_pointer_t is always well-defined:
template<typename T>
using also_safe = std::conditional_t<
    std::is_pointer_v<T>,
    std::remove_pointer_t<T>,   // Always valid: returns T if not a pointer
    T
>;

// This would FAIL for non-class types:
// template<typename T>
// using broken = std::conditional_t<
//     std::is_class_v<T>,
//     typename T::value_type,    // ERROR for int: int::value_type doesn't exist
//     T
// >;
// broken<int>  // Compile error! T::value_type instantiated even though B=false

// SOLUTION: defer instantiation with an intermediate template
template<typename T, typename = void>
struct safe_value_type { using type = T; };

template<typename T>
struct safe_value_type<T, std::void_t<typename T::value_type>> {
    using type = typename T::value_type;
};

template<typename T>
using safe_value_type_t = typename safe_value_type<T>::type;

// ALTERNATIVE SOLUTION: wrap in a lazy template
template<typename T>
struct identity_type { using type = T; };

template<typename T>
struct deref_type { using type = typename T::value_type; };

// Now conditional_t selects between WRAPPERS, not the types directly:
template<typename T>
using lazy_safe = typename std::conditional_t<
    std::is_class_v<T>,
    deref_type<T>,     // Not instantiated directly — just the wrapper
    identity_type<T>   // Only the SELECTED wrapper's ::type is accessed
>::type;
// This is the "lazy conditional" pattern!

#include <vector>
#include <string>

int main() {
    // safe: trivially works
    static_assert(std::is_same_v<safe<int>, int>);
    static_assert(std::is_same_v<safe<double>, double>);

    // also_safe: remove_pointer_t always valid
    static_assert(std::is_same_v<also_safe<int*>, int>);
    static_assert(std::is_same_v<also_safe<int>, int>);

    // safe_value_type_t: SFINAE-based solution
    static_assert(std::is_same_v<safe_value_type_t<std::vector<int>>, int>);
    static_assert(std::is_same_v<safe_value_type_t<int>, int>);

    // Lazy conditional: deferred instantiation
    static_assert(std::is_same_v<lazy_safe<std::vector<int>>, int>);
    // lazy_safe<int> would fail because deref_type<int>::type doesn't exist
    // But the lazy pattern ensures it's never instantiated for non-class types

    std::cout << "Key insight:\n";
    std::cout << "  conditional_t<B, T, F> — both T and F must be valid types\n";
    std::cout << "  To defer: use conditional_t to select WRAPPERS, then access ::type\n";
    std::cout << "  Or use SFINAE/concepts for truly conditional type computation\n";

    return 0;
}
```

**Summary:**

| Approach | Both branches evaluated? | True lazy? |
| --- | --- | --- |
| `conditional_t<B, T, F>` | Yes - both T and F must be valid | No |
| `conditional_t<B, WrapperT, WrapperF>::type` | Wrappers: yes, `::type`: only selected | Partially |
| `if constexpr` | Only chosen branch | Yes |
| Template specialization | Only matching specialization | Yes |
| Concepts + constrained templates | Only matched constraints | Yes |

---

## Notes

- `std::conditional_t` is the foundation of many metaprogramming patterns. Master it before moving to more complex type-level programming.
- The "lazy conditional" pattern (`conditional_t<B, Wrapper1<T>, Wrapper2<T>>::type`) is a standard technique in Boost and the standard library itself.
- For C++20 code, prefer `if constexpr` with `decltype(auto)` for multi-way type selection when possible - it's more readable than nested `conditional_t`.
- `conditional_t` is zero-cost - it's purely a compile-time alias, generating no runtime code.
- Common mistake: using `conditional_t` where `enable_if_t` is needed. `conditional_t` selects between two types; `enable_if_t` enables/disables a template entirely.
