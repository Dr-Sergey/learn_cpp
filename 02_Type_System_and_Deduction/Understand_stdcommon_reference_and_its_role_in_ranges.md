# Understand `std::common_reference` and Its Role in Ranges

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/types/common_reference>  

---

## Topic Overview

### What Is `std::common_reference`

`std::common_reference_t<T, U>` computes a type to which both `T` and `U` can be **implicitly converted as references**. It's a generalization of `std::common_type` that preserves reference semantics.

The key difference from `common_type` is that it tries to keep references alive when possible, rather than stripping them:

```cpp
#include <type_traits>
#include <concepts>

// common_type strips references first, then finds a common type:
static_assert(std::is_same_v<std::common_type_t<int&, int>, int>);

// common_reference can preserve reference-ness:
static_assert(std::is_same_v<std::common_reference_t<int&, int>, int>);
static_assert(std::is_same_v<std::common_reference_t<int&, const int&>, const int&>);
static_assert(std::is_same_v<std::common_reference_t<int&, int&&>, const int&>);
```

Notice the last case: `int&` and `int&&` can both bind to `const int&`, so that becomes the common reference.

### `std::common_reference` vs `std::common_type`

| Feature | `std::common_type_t<T, U>` | `std::common_reference_t<T, U>` |
| --- | --- | --- |
| Reference handling | Strips all references first | Preserves references when possible |
| Primary use | Value types | Reference-compatible types |
| Used by | Arithmetic, comparisons | Ranges/iterators |
| `int& + int` | `int` | `int` |
| `int& + const int&` | `int` | `const int&` (reference preserved) |
| `int& + int&&` | `int` | `const int&` (safe common reference) |

### Why Ranges Need `common_reference`

This is where the concept gets practically important. The Ranges library requires that iterators and their reference types are **comparable** and **interconvertible**. `common_reference` provides the type that makes this possible.

Key concepts that rely on `common_reference`:

```cpp
std::indirectly_readable<I>
  requires:
    common_reference_with<iter_value_t<I>&, iter_reference_t<I>>

std::input_iterator<I>
  requires indirectly_readable

std::sortable<I>
  requires indirect_strict_weak_order
  which requires common_reference between iterator reference types

std::mergeable<I1, I2, O>
  requires common references between two iterator types and output
```

In short: if your iterator's value type and reference type don't have a common reference, none of the ranges algorithms will work with it.

### The Problem `common_reference` Solves

Consider a proxy iterator (like `std::vector<bool>::iterator`). This is the motivating case - ordinary iterators are trivial, but proxy iterators are where things get interesting:

```cpp
// For std::vector<int>:
//   iter_value_t<It>      = int
//   iter_reference_t<It>  = int&
//   Common reference: int& (trivial: int& and int& share int&)

// For std::vector<bool>:
//   iter_value_t<It>      = bool
//   iter_reference_t<It>  = std::vector<bool>::reference (a proxy!)
//   Common reference: ???
//   common_reference_t<bool&, vector<bool>::reference> must be well-defined
//   for the iterator to model std::input_iterator
```

Without `common_reference`, proxy iterators would not satisfy iterator concepts. `common_reference` bridges the gap between the value type and the proxy reference type.

### How `common_reference` Is Computed

The algorithm works through a prioritized fallback chain:

```cpp
1. If T and U are both reference types:

     Apply reference collapsing rules to find a common ref
     e.g., int& and const int& -> const int&

2. If step 1 fails, check basic_common_reference<T, U> specialization

     (customization point for proxy types)

3. If step 2 fails, try common_type_t<T, U>

4. If all fail -> SFINAE (no common reference exists)
```

### Customizing `common_reference` for Proxy Types

If you write a proxy reference type, you need to tell the system how it relates to the actual value type. Here's the template specialization pattern to do that:

```cpp
#include <type_traits>
#include <concepts>

// A proxy reference type
struct BoolProxy {
    bool& ref;
    operator bool() const { return ref; }
    BoolProxy& operator=(bool b) { ref = b; return *this; }
};

// Teach common_reference about our proxy
template<template<class> class TQual, template<class> class UQual>
struct std::basic_common_reference<bool, BoolProxy, TQual, UQual> {
    using type = bool;
};

template<template<class> class TQual, template<class> class UQual>
struct std::basic_common_reference<BoolProxy, bool, TQual, UQual> {
    using type = bool;
};

// Now common_reference_t<bool&, BoolProxy> = bool
```

Both orderings of the specialization are needed because `common_reference` may query them in either order.

---

## Self-Assessment

### Q1: Explain why `std::common_reference_t<int&, int>` is `int` and how that differs from `common_type`

The step-by-step reasoning in the comments matters here - the algorithm tries to preserve a reference but can only do so when both types can bind to it:

```cpp
#include <iostream>
#include <type_traits>

int main() {
    // --- common_type: always strips references, then finds common type ---
    // Step: remove_reference(int&) -> int, remove_reference(int) -> int
    // Result: common_type_t<int, int> -> int
    static_assert(std::is_same_v<std::common_type_t<int&, int>, int>);

    // --- common_reference: tries to preserve references ---
    // Input: int& and int
    // Can int be bound to int&? No (rvalue can't bind to non-const lvalue ref)
    // Can int& be converted to int? Yes (lvalue-to-rvalue conversion)
    // Can int be converted to int? Yes (identity)
    // Best common reference: int (both can convert to int)
    static_assert(std::is_same_v<std::common_reference_t<int&, int>, int>);

    // --- Where they DIFFER ---
    // Case: int& and const int&
    // common_type: strips refs -> common_type_t<int, int> -> int (no reference!)
    static_assert(std::is_same_v<std::common_type_t<int&, const int&>, int>);

    // common_reference: int& can bind to const int&, const int& is common ref
    static_assert(std::is_same_v<std::common_reference_t<int&, const int&>, const int&>);
    // ^^^ This is the KEY difference: common_reference preserves the reference!

    // More examples:
    // int& and int&& -> const int& (safe binding for both)
    static_assert(std::is_same_v<std::common_reference_t<int&, int&&>, const int&>);

    // common_type would give: int (no reference)
    static_assert(std::is_same_v<std::common_type_t<int&, int&&>, int>);

    // double& and int& -> no common reference as references
    // Falls back to common_type_t<double, int> -> double
    static_assert(std::is_same_v<std::common_reference_t<double&, int&>, double>);

    std::cout << "All assertions passed.\n";
    std::cout << "\nSummary:\n";
    std::cout << "  common_type always produces a value type (no references)\n";
    std::cout << "  common_reference preserves references when possible\n";
    std::cout << "  Both agree when references can't be preserved\n";

    return 0;
}
```

**Output:**

```text
All assertions passed.

Summary:
  common_type always produces a value type (no references)
  common_reference preserves references when possible
  Both agree when references can't be preserved
```

### Q2: Show where ranges algorithms use `common_reference` to determine iterator/sentinel compatibility

This example makes the abstract concept concrete by checking the actual requirements against real iterators. The `check_iterator_common_ref` function prints what the ranges machinery is actually checking under the hood:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <ranges>
#include <concepts>
#include <type_traits>
#include <iterator>

// The std::indirectly_readable concept requires:
//   common_reference_with<iter_value_t<I>&, iter_reference_t<I>>
// This is checked for every iterator used with ranges algorithms.

// Let's verify it for standard iterators:
template<typename It>
void check_iterator_common_ref() {
    using Val = std::iter_value_t<It>;
    using Ref = std::iter_reference_t<It>;

    std::cout << "  value_type:     " << typeid(Val).name() << "\n";
    std::cout << "  reference_type: " << typeid(Ref).name() << "\n";

    // This is what ranges requires:
    using CR = std::common_reference_t<Val&, Ref>;
    std::cout << "  common_ref:     " << typeid(CR).name() << "\n";

    // Verify the concept
    static_assert(std::common_reference_with<Val&, Ref>,
                  "Iterator does not satisfy common_reference requirement!");
    std::cout << "  common_reference_with: satisfied\n\n";
}

// Custom iterator with proxy reference
struct ProxyRef {
    int* ptr;
    operator int() const { return *ptr; }
    ProxyRef& operator=(int v) { *ptr = v; return *this; }
};

int main() {
    std::cout << "=== vector<int>::iterator ===\n";
    check_iterator_common_ref<std::vector<int>::iterator>();

    std::cout << "=== const vector<int>::iterator ===\n";
    check_iterator_common_ref<std::vector<int>::const_iterator>();

    // Ranges algorithms use common_reference internally:
    std::vector<int> v = {5, 3, 1, 4, 2};

    // std::ranges::sort requires std::sortable<I>
    // which requires std::indirect_strict_weak_order<ranges::less, I>
    // which requires common_reference between projected types
    std::ranges::sort(v);

    std::cout << "=== Sorted vector ===\n";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n\n";

    // ranges::equal requires indirectly_comparable
    // which requires common_reference between the two iterator reference types
    std::vector<int> v2 = {1, 2, 3, 4, 5};
    bool eq = std::ranges::equal(v, v2);
    std::cout << "Vectors equal: " << std::boolalpha << eq << "\n";

    return 0;
}
```

**Output:**

```text
=== vector<int>::iterator ===
  value_type:     int
  reference_type: int&
  common_ref:     int&
  common_reference_with: satisfied

=== const vector<int>::iterator ===
  value_type:     int
  reference_type: const int&
  common_ref:     const int&
  common_reference_with: satisfied

=== Sorted vector ===
1 2 3 4 5

Vectors equal: true
```

**How ranges use common_reference:**

- Every ranges algorithm constrains its iterators with concepts like `std::input_iterator`, `std::sortable`, etc.
- These concepts transitively require `std::indirectly_readable`, which checks `common_reference_with<value_type&, reference_type>`
- This ensures that the iterator's value type and reference type are compatible - critical for proxy iterators
- If the common_reference requirement fails, the algorithm is SFINAE'd out with a clear concept error

### Q3: Write a concept that requires two types to share a common reference

Building on the concepts in the standard library, you can compose them to express exactly the constraint you need:

```cpp
#include <iostream>
#include <type_traits>
#include <concepts>
#include <string>
#include <string_view>

// Concept: two types share a common reference
template<typename T, typename U>
concept SharesCommonReference = std::common_reference_with<T, U>;

// Extended concept: common reference must be a specific type
template<typename T, typename U, typename Expected>
concept CommonReferenceIs =
    std::common_reference_with<T, U> &&
    std::same_as<std::common_reference_t<T, U>, Expected>;

// Concept: all types in a pack share pairwise common references
template<typename T, typename... Us>
concept AllShareCommonRef = (std::common_reference_with<T, Us> && ...);

// Usage: function constrained by common reference
template<typename T, typename U>
    requires SharesCommonReference<T, U>
auto safe_compare(const T& a, const U& b) -> bool {
    using Common = std::common_reference_t<const T&, const U&>;
    return static_cast<Common>(a) == static_cast<Common>(b);
}

// Usage: generic merge that requires common reference
template<typename T, typename U>
    requires SharesCommonReference<T&, U&>
void print_pair(const T& a, const U& b) {
    using Common = std::common_reference_t<const T&, const U&>;
    std::cout << "  a as common: " << static_cast<Common>(a) << "\n";
    std::cout << "  b as common: " << static_cast<Common>(b) << "\n";
}

int main() {
    // int& and double& share common reference (double)
    static_assert(SharesCommonReference<int&, double&>);

    // int& and const int& share common reference (const int&)
    static_assert(SharesCommonReference<int&, const int&>);

    // Verify expected common reference types
    static_assert(CommonReferenceIs<int&, const int&, const int&>);
    static_assert(CommonReferenceIs<int&, int&&, const int&>);

    // All numeric types share common refs
    static_assert(AllShareCommonRef<int, double, float, long>);

    // Use in constrained function
    std::cout << "=== safe_compare ===\n";
    std::cout << "3 == 3.0: " << std::boolalpha << safe_compare(3, 3.0) << "\n";
    std::cout << "3 == 4.0: " << safe_compare(3, 4.0) << "\n";

    std::cout << "\n=== print_pair ===\n";
    print_pair(42, 3.14);

    // string + string_view share common reference
    std::string s = "hello";
    std::string_view sv = "hello";
    static_assert(SharesCommonReference<std::string&, std::string_view&>);
    std::cout << "\n=== string vs string_view ===\n";
    std::cout << "equal: " << safe_compare(s, sv) << "\n";

    return 0;
}
```

**Output:**

```text
=== safe_compare ===
3 == 3.0: true
3 == 4.0: false

=== print_pair ===
  a as common: 42
  b as common: 3.14

=== string vs string_view ===
equal: true
```

**How this works:**

- `SharesCommonReference<T, U>` wraps `std::common_reference_with` - the standard concept that checks if a common reference type exists and both types are convertible to it
- `CommonReferenceIs` adds an additional constraint on what the common reference type must be
- `AllShareCommonRef` uses fold expressions to check pairwise common references across a parameter pack
- These concepts are particularly useful when writing generic algorithms that must work with both value types and proxy references

---

## Notes

- `std::common_reference` is the backbone of the C++20 Ranges concept hierarchy. Understanding it is essential for writing custom iterators that work with ranges.
- If you define a proxy reference type, you likely need to specialize `std::basic_common_reference` to make your iterator satisfy `std::indirectly_readable`.
- `common_reference_with<T, U>` checks not just that the type exists, but that both `T` and `U` are **convertible** to it: `std::convertible_to<T, common_reference_t<T, U>>`.
- The difference between `common_type` and `common_reference` matters most for reference types - for pure value types, they often agree.
