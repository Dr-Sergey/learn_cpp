# Know how to use std::common_reference_t in range iterator requirements

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/types/common_reference>  

---

## Topic Overview

`std::common_reference_t` computes a type to which two types can both be implicitly converted, preserving reference semantics where possible. It's a key building block in C++20 ranges for defining iterator concepts. This is one of those traits that most people never use directly, but understanding what it does explains why certain iterators work (or don't work) with range algorithms.

### common_reference_t vs common_type_t

The two traits sound similar but have meaningfully different goals. `common_type_t` always gives you a value type - it strips references first, then finds the common type. `common_reference_t` tries to keep references alive when both sides are references to the same underlying type.

| Trait | Purpose | Example |
| --- | --- | --- |
| `std::common_type_t<A, B>` | Common **value** type (strips references) | `common_type_t<int&, double&>` -> `double` |
| `std::common_reference_t<A, B>` | Common type preserving **reference-ness** | `common_reference_t<int&, double&>` -> `double` (no ref, but tries to keep refs when possible) |

Here's the difference in concrete `static_assert` form:

```cpp
#include <type_traits>
#include <concepts>

// common_type: always produces a value type
static_assert(std::is_same_v<std::common_type_t<int&, int&>, int>);     // stripped ref
static_assert(std::is_same_v<std::common_type_t<int, double>, double>);

// common_reference: preserves references when both are references to the same type
static_assert(std::is_same_v<std::common_reference_t<int&, int&>, int&>);         // kept ref
static_assert(std::is_same_v<std::common_reference_t<int&, const int&>, const int&>);
static_assert(std::is_same_v<std::common_reference_t<int&, int&&>, const int&>);   // both bindable
static_assert(std::is_same_v<std::common_reference_t<int, double>, double>);       // value types - same as common_type
```

The reason this trips people up is that `common_reference_t<int&, int&&>` gives `const int&` - not `int&` or `int&&`. The idea is: what single reference type can both an lvalue ref and an rvalue ref bind to? A `const lvalue&` is the answer, because both can bind to it.

### Why Ranges Need common_reference

The `std::indirectly_readable` concept (which iterators must satisfy) requires:

```cpp
// Simplified version of the requirement:
template<class I>
concept indirectly_readable = requires {
    // The iterator's reference type and value_type& must share a common_reference
    typename std::common_reference_t<
        std::iter_reference_t<I>,          // what *it returns
        std::iter_value_t<I>&              // value_type&
    >;
};
```

This ensures that algorithms can work with both the reference returned by `*it` and copies of elements interchangeably. Without this guarantee, a sort algorithm couldn't safely compare `*it` against a temporary copy of an element.

### Proxy Iterators

The real power of `common_reference` shows with **proxy iterators** like `std::vector<bool>::iterator`. The reason this matters is that `vector<bool>` doesn't actually store `bool` values - it packs bits and returns a proxy object from `operator*`, not a real `bool&`:

```cpp
// vector<bool> returns a proxy object, not bool&
// *it returns vector<bool>::reference (a proxy), not bool&
// Without common_reference, this proxy iterator wouldn't satisfy indirectly_readable

// common_reference_t<vector<bool>::reference, bool&> -> bool
// Both the proxy and a bool& can convert to bool - algorithms work
```

---

## Self-Assessment

### Q1: Explain what `common_reference_t` computes and how it differs from `common_type_t`

Let's look at concrete outputs side by side. The key observation is the last two lines - `common_type_t<int&, int&>` strips the reference and gives `int`, while `common_reference_t<int&, int&>` keeps it as `int&`:

```cpp
#include <iostream>
#include <type_traits>
#include <concepts>

template<typename T>
constexpr const char* type_name();

template<> constexpr const char* type_name<int>() { return "int"; }
template<> constexpr const char* type_name<int&>() { return "int&"; }
template<> constexpr const char* type_name<const int&>() { return "const int&"; }
template<> constexpr const char* type_name<double>() { return "double"; }
template<> constexpr const char* type_name<long>() { return "long"; }

int main() {
    std::cout << "=== common_type_t (always strips references) ===\n";
    // common_type: applies std::decay - removes references, const, volatile
    using CT1 = std::common_type_t<int&, int&>;
    std::cout << "common_type_t<int&, int&> = " << type_name<CT1>() << "\n";  // int

    using CT2 = std::common_type_t<int, double>;
    std::cout << "common_type_t<int, double> = " << type_name<CT2>() << "\n"; // double

    std::cout << "\n=== common_reference_t (preserves references when possible) ===\n";
    // common_reference: keeps references when both inputs are references
    using CR1 = std::common_reference_t<int&, int&>;
    std::cout << "common_reference_t<int&, int&> = " << type_name<CR1>() << "\n"; // int&

    using CR2 = std::common_reference_t<int&, const int&>;
    std::cout << "common_reference_t<int&, const int&> = " << type_name<CR2>() << "\n"; // const int&

    using CR3 = std::common_reference_t<int, double>;
    std::cout << "common_reference_t<int, double> = " << type_name<CR3>() << "\n"; // double

    // Key difference: common_type ALWAYS strips refs
    // common_reference TRIES to preserve refs
    static_assert(std::is_same_v<std::common_type_t<int&, int&>, int>);         // no ref
    static_assert(std::is_same_v<std::common_reference_t<int&, int&>, int&>);  // ref preserved!
}
```

`common_type_t` applies `std::decay` to both types first - removing references, const, and volatile - then finds a common type. `common_reference_t` tries to find a type that both can bind to, preserving reference semantics. When both inputs are `int&`, `common_reference_t` returns `int&` (both bind to it). `common_type_t` returns `int` (stripped ref).

### Q2: Show why `ranges::indirectly_readable` requires a common reference

Here's what the concept is actually checking, and why it matters for algorithms like `std::ranges::sort`:

```cpp
#include <iostream>
#include <vector>
#include <concepts>
#include <iterator>
#include <type_traits>
#include <ranges>

// The indirectly_readable concept needs this relationship:
// common_reference_t<iter_reference_t<I>, iter_value_t<I>&> must exist
//
// WHY? Because algorithms need to compare/assign between:
// - *it (the reference returned by dereferencing the iterator)
// - copies of elements (value_type objects)

// For a normal vector<int>::iterator:
// iter_reference_t = int& (what *it returns)
// iter_value_t     = int  (the value type)
// common_reference_t<int&, int&> = int& - OK

// For vector<bool>::iterator (proxy):
// iter_reference_t = vector<bool>::reference (proxy object)
// iter_value_t     = bool
// common_reference_t<vector<bool>::reference, bool&> = bool - OK

int main() {
    // Verify standard iterators satisfy indirectly_readable
    static_assert(std::indirectly_readable<std::vector<int>::iterator>);
    static_assert(std::indirectly_readable<std::vector<double>::const_iterator>);
    static_assert(std::indirectly_readable<int*>);  // raw pointers too

    // Show the actual types involved
    using It = std::vector<int>::iterator;
    using Ref = std::iter_reference_t<It>;     // int&
    using Val = std::iter_value_t<It>;          // int
    using CR = std::common_reference_t<Ref, Val&>;  // int&

    static_assert(std::is_same_v<Ref, int&>);
    static_assert(std::is_same_v<Val, int>);
    static_assert(std::is_same_v<CR, int&>);

    std::cout << "All iterator concepts satisfied!\n";

    // Why this matters for algorithms:
    std::vector<int> v{3, 1, 4, 1, 5};
    // std::ranges::sort needs to swap *it1 with *it2
    // and compare *it with a value_type - requires common_reference
    std::ranges::sort(v);
    for (int x : v) std::cout << x << " ";  // 1 1 3 4 5
    std::cout << "\n";
}
```

### Q3: Write a custom iterator and verify it satisfies `std::indirectly_readable`

This custom iterator returns its values by value (not by reference), which changes what `common_reference_t` computes. Notice the `static_assert` that checks the relationship before asserting the concept is satisfied:

```cpp
#include <iostream>
#include <iterator>
#include <concepts>
#include <type_traits>

// Custom iterator over a C-style array that doubles values on read
template<typename T>
struct DoubleIterator {
    using value_type = T;
    using difference_type = std::ptrdiff_t;

    const T* ptr_ = nullptr;

    // Returns T by value (doubled) - not a reference
    T operator*() const { return *ptr_ * 2; }

    DoubleIterator& operator++() { ++ptr_; return *this; }
    DoubleIterator operator++(int) { auto tmp = *this; ++ptr_; return tmp; }

    bool operator==(const DoubleIterator& other) const { return ptr_ == other.ptr_; }
};

// Verify the common_reference requirement:
// iter_reference_t<DoubleIterator<int>> = int (operator* returns by value)
// iter_value_t<DoubleIterator<int>> = int
// common_reference_t<int, int&> = int - OK
static_assert(std::is_same_v<
    std::common_reference_t<int, int&>,
    int
>);

// Verify the concept is satisfied
static_assert(std::indirectly_readable<DoubleIterator<int>>);
static_assert(std::input_iterator<DoubleIterator<int>>);

int main() {
    int data[] = {1, 2, 3, 4, 5};

    DoubleIterator<int> begin{data};
    DoubleIterator<int> end{data + 5};

    std::cout << "Doubled: ";
    for (auto it = begin; it != end; ++it) {
        std::cout << *it << " ";  // 2 4 6 8 10
    }
    std::cout << "\n";

    // Can use with range algorithms:
    // (Would need to also provide sentinel/subrange adapter for full ranges support)
}
```

---

## Notes

- `common_reference_t` is primarily used internally by the ranges library - most users rarely need it directly.
- Customization point: specialize `std::basic_common_reference` to define common references for your own proxy types.
- If `common_reference_t<A, B>` doesn't exist (SFINAE-friendly), the types have no common reference - your iterator won't satisfy `indirectly_readable`.
- `std::common_reference_with<T, U>` is a concept that checks both the existence of the common reference and convertibility from both types.
