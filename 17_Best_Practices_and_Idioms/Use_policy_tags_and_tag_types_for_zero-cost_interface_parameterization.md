# Use policy tags and tag types for zero-cost interface parameterization

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>  

---

## Topic Overview

**Tag types** are empty structs whose entire job is to select an overload or policy at compile time. Because they're empty, the compiler optimizes them away completely - you get different behavior at different call sites with zero runtime cost and, more importantly, with call sites that read like plain English rather than `func<true>(x)`.

### Standard Library Examples

The standard library uses this pattern extensively. You've probably seen these without thinking about why they exist:

```cpp
std::piecewise_construct_t   // tag for std::pair
std::in_place_t              // tag for std::optional, std::variant
std::allocator_arg_t         // tag for allocator-aware types
std::defer_lock_t            // tag for std::unique_lock
```

Each of these is an empty struct. Passing an instance of one as an argument tells the constructor which overload to use - compile-time dispatch with no runtime overhead.

---

## Self-Assessment

### Q1: Define policy tags and overload on them

Here's how to build your own version of the pattern. Two empty structs - `check_bounds_t` and `no_check_t` - let callers choose between a bounds-checking overload and an unchecked one. The tag argument disappears in optimized builds; all that's left is the right code for the chosen policy.

```cpp
#include <cstddef>
#include <iostream>
#include <stdexcept>
#include <vector>
#include <initializer_list>
#include <string>

// Policy tags (empty structs)
struct check_bounds_t {};
struct no_check_t {};

// Tag instances (inline constexpr for ODR safety)
inline constexpr check_bounds_t check_bounds{};
inline constexpr no_check_t no_check{};

template<typename T>
class Array {
    std::vector<T> data_;
public:
    Array(std::initializer_list<T> init) : data_(init) {}

    // Overload 1: with bounds checking
    T& operator()(size_t i, check_bounds_t) {
        if (i >= data_.size())
            throw std::out_of_range("index " + std::to_string(i));
        return data_[i];
    }

    // Overload 2: no bounds checking (zero overhead)
    T& operator()(size_t i, no_check_t) noexcept {
        return data_[i];
    }

    size_t size() const { return data_.size(); }
};

int main() {
    Array<int> arr{10, 20, 30, 40, 50};

    // Safe access (debug/testing)
    std::cout << arr(2, check_bounds) << '\n';  // 30

    // Fast access (hot path)
    std::cout << arr(4, no_check) << '\n';       // 50

    try {
        arr(10, check_bounds);  // throws!
    } catch (const std::out_of_range& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }
}
// Expected output:
// 30
// 50
// Caught: index 10
```

The two overloads are completely separate functions. There's no `if` statement branching at runtime; the decision is made when the compiler resolves the call. The `check_bounds` and `no_check` arguments are just the mechanism for making that choice readable.

### Q2: Show that policy tags generate separate code paths with zero runtime branching

This example makes the "zero runtime branching" point explicit by selecting entirely different algorithms - binary search vs. linear search - based on the tag. The caller's choice of `sorted` or `unsorted` is resolved at compile time.

```cpp
#include <iostream>

// Policy tags
struct sorted_t {};
struct unsorted_t {};
inline constexpr sorted_t sorted{};
inline constexpr unsorted_t unsorted{};

// Two completely different algorithms, selected at compile time
template<typename Iter>
bool find(Iter begin, Iter end, int value, sorted_t) {
    // Binary search for sorted data - O(log n)
    std::cout << "[binary search] ";
    auto it = std::lower_bound(begin, end, value);
    return it != end && *it == value;
}

template<typename Iter>
bool find(Iter begin, Iter end, int value, unsorted_t) {
    // Linear search for unsorted data - O(n)
    std::cout << "[linear search] ";
    return std::find(begin, end, value) != end;
}

int main() {
    int sorted_data[] = {1, 3, 5, 7, 9, 11};
    int unsorted_data[] = {7, 2, 9, 1, 5, 3};

    // Compiler generates DIFFERENT code for each call:
    bool r1 = find(std::begin(sorted_data), std::end(sorted_data), 7, sorted);
    std::cout << std::boolalpha << r1 << '\n';

    bool r2 = find(std::begin(unsorted_data), std::end(unsorted_data), 5, unsorted);
    std::cout << r2 << '\n';

    // NO runtime if/else branching! The tag selects the overload
    // at compile time, and the empty tag parameter is optimized away.
}
// Expected output:
// [binary search] true
// [linear search] true
```

Each `find` instantiation compiles to a completely different function body. The tag is not stored anywhere - it's an empty struct used purely as a compile-time signal. This is what "zero-cost" means here.

### Q3: Compare policy tags with template boolean parameters

The alternative to tag types for compile-time switching is a `template<bool>` parameter. Both achieve zero runtime cost, but the readability is quite different at the call site. `access_v1<true>(v, i)` tells you nothing without looking up the template; `access_v2(v, i, checked)` is self-explanatory.

```cpp
#include <iostream>
#include <vector>

// Approach 1: bool template parameter (bad readability)
template<bool CheckBounds>
int access_v1(const std::vector<int>& v, size_t i) {
    if constexpr (CheckBounds) {
        return v.at(i);
    } else {
        return v[i];
    }
}

// Approach 2: tag types (good readability)
struct checked_t {};
struct unchecked_t {};
inline constexpr checked_t checked{};
inline constexpr unchecked_t unchecked{};

int access_v2(const std::vector<int>& v, size_t i, checked_t) { return v.at(i); }
int access_v2(const std::vector<int>& v, size_t i, unchecked_t) { return v[i]; }

int main() {
    std::vector<int> v = {10, 20, 30};

    // BAD: what does true/false mean?
    access_v1<true>(v, 1);   // checked? sorted? reversed?
    access_v1<false>(v, 1);  // who knows without looking up the definition

    // GOOD: self-documenting
    access_v2(v, 1, checked);    // obviously: bounds-checked
    access_v2(v, 1, unchecked);  // obviously: no bounds check

    std::cout << access_v2(v, 1, checked) << '\n';
}
// Expected output:
// 20
```

There's also a practical extensibility advantage: if you later need a third option (say, `clamped_t` that clamps the index instead of throwing), you add a tag and an overload. With the boolean parameter you'd have to change the template to something more complex.

| Feature | `template<bool B>` | Tag types |
| --- | --- | --- |
| Readability | `func<true>(x)` - unclear | `func(x, checked)` - clear |
| Can mix with overloads | Awkward | Natural |
| Runtime overhead | Zero | Zero |
| Extensible | No (only true/false) | Yes (add new tags) |
| Error messages | Template errors | "no matching overload" |

---

## Notes

- Tag types cost zero bytes at runtime - they're empty structs, optimized away completely by the compiler.
- Use `inline constexpr` tag instances in headers for ODR safety when the tags are used across translation units.
- The STL uses tags extensively: `std::piecewise_construct`, `std::in_place`, `std::defer_lock` are all real examples you can study.
- Combine with concepts for even better error messages in C++20 - a `requires` clause on the overload tells the caller exactly what's expected.
