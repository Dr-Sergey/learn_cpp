# Use Tag Dispatch for Selecting Overloads Based on Type Properties

**Category:** Templates and Generic Programming  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/iterator/advance>  

---

## Topic Overview

### What Is Tag Dispatch

Tag dispatch uses **empty tag types** (like `std::true_type`, `std::false_type`, or iterator category tags) as function parameters to select the correct overload at compile time. The tag carries no data - its only purpose is to route the call to the right overload.

Here is the basic pattern:

```cpp
// Tag types (empty structs used only for overload selection)
struct fast_tag {};
struct slow_tag {};

void process(int x, fast_tag) { /* O(1) algorithm */ }
void process(int x, slow_tag) { /* O(n) algorithm */ }

// Dispatch based on a compile-time property
template <typename T>
void process(T x) {
    if constexpr (sizeof(T) <= 8)
        process(x, fast_tag{});   // small -> fast path
    else
        process(x, slow_tag{});   // large -> slow path
}
```

### How `std::advance` Uses Tag Dispatch

The standard library's `std::advance` is the textbook example. It picks a completely different algorithm depending on what the iterator can do:

```cpp
random_access_iterator_tag  ->  it += n        (O(1))
bidirectional_iterator_tag  ->  ++it / --it    (O(n))
input_iterator_tag          ->  ++it only      (O(n), forward only)
```

### Tag Dispatch vs Alternatives

| Technique | When to Use |
| --- | --- |
| **Tag dispatch** | Pre-C++17, or when overloads need different signatures |
| **`if constexpr`** | C++17+, single function body, all branches in one place |
| **Concepts/requires** | C++20+, cleanest syntax, best error messages |
| **SFINAE/enable_if** | Pre-C++20, complex constraints |

---

## Self-Assessment

### Q1: Implement `advance()` using tag dispatch on `iterator_category` (random vs bidirectional vs input)

The dispatcher extracts the iterator category tag type from `std::iterator_traits` and constructs a temporary instance of it. That instance is passed to one of the three `advance_impl` overloads, and normal overload resolution picks the right one at compile time.

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <forward_list>
#include <iterator>

// === Tag dispatch: different overloads for different iterator categories ===

namespace my {

// Random access: O(1)
template <typename It>
void advance_impl(It& it, int n, std::random_access_iterator_tag) {
    std::cout << "  [random_access] it += " << n << "\n";
    it += n;
}

// Bidirectional: O(n), can go forward and backward
template <typename It>
void advance_impl(It& it, int n, std::bidirectional_iterator_tag) {
    std::cout << "  [bidirectional] ";
    if (n >= 0) {
        std::cout << "++it x" << n << "\n";
        while (n-- > 0) ++it;
    } else {
        std::cout << "--it x" << -n << "\n";
        while (n++ < 0) --it;
    }
}

// Input/forward: O(n), forward only
template <typename It>
void advance_impl(It& it, int n, std::input_iterator_tag) {
    std::cout << "  [input/forward] ++it x" << n << "\n";
    while (n-- > 0) ++it;
    // Cannot go backwards!
}

// Main function: dispatches based on iterator_category
template <typename It>
void advance(It& it, int n) {
    // The tag is an empty struct — exists only for overload selection
    advance_impl(it, n,
        typename std::iterator_traits<It>::iterator_category{});
}

}  // namespace my

int main() {
    std::cout << "=== Vector (random access) ===\n";
    std::vector<int> v = {10, 20, 30, 40, 50};
    auto vit = v.begin();
    my::advance(vit, 3);
    std::cout << "  Value: " << *vit << "\n";  // 40

    std::cout << "\n=== List (bidirectional) ===\n";
    std::list<int> l = {10, 20, 30, 40, 50};
    auto lit = l.begin();
    my::advance(lit, 2);
    std::cout << "  Value: " << *lit << "\n";  // 30
    my::advance(lit, -1);
    std::cout << "  Value: " << *lit << "\n";  // 20

    std::cout << "\n=== Forward list (forward only) ===\n";
    std::forward_list<int> fl = {10, 20, 30, 40, 50};
    auto flit = fl.begin();
    my::advance(flit, 4);
    std::cout << "  Value: " << *flit << "\n";  // 50

    return 0;
}
```

Notice that `random_access_iterator_tag` inherits from `bidirectional_iterator_tag` in the standard hierarchy, so the most derived matching overload wins. That is why the vector iterator correctly calls the `random_access` version even though a `bidirectional` overload also exists.

### Q2: Compare tag dispatch with `if constexpr` for the same selection and explain when each is better

Both approaches produce identical runtime behavior here - the choice is about readability, extensibility, and which C++ version you are targeting.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === Method 1: Tag Dispatch ===
struct trivial_tag {};
struct nontrivial_tag {};

template <typename T>
void copy_impl(const T* src, T* dst, size_t n, trivial_tag) {
    std::cout << "  [tag] memcpy (trivial)\n";
    std::memcpy(dst, src, n * sizeof(T));
}

template <typename T>
void copy_impl(const T* src, T* dst, size_t n, nontrivial_tag) {
    std::cout << "  [tag] element-wise copy (non-trivial)\n";
    for (size_t i = 0; i < n; ++i)
        dst[i] = src[i];
}

template <typename T>
void copy_v1(const T* src, T* dst, size_t n) {
    using tag = std::conditional_t<
        std::is_trivially_copyable_v<T>,
        trivial_tag,
        nontrivial_tag>;
    copy_impl(src, dst, n, tag{});
}

// === Method 2: if constexpr ===
template <typename T>
void copy_v2(const T* src, T* dst, size_t n) {
    if constexpr (std::is_trivially_copyable_v<T>) {
        std::cout << "  [if constexpr] memcpy (trivial)\n";
        std::memcpy(dst, src, n * sizeof(T));
    } else {
        std::cout << "  [if constexpr] element-wise copy (non-trivial)\n";
        for (size_t i = 0; i < n; ++i)
            dst[i] = src[i];
    }
}

int main() {
    int src_i[] = {1, 2, 3};
    int dst_i[3];

    std::string src_s[] = {"a", "b", "c"};
    std::string dst_s[3];

    std::cout << "=== Tag dispatch ===\n";
    copy_v1(src_i, dst_i, 3);  // trivial -> memcpy
    copy_v1(src_s, dst_s, 3);  // non-trivial -> loop

    std::cout << "\n=== if constexpr ===\n";
    copy_v2(src_i, dst_i, 3);  // trivial -> memcpy
    copy_v2(src_s, dst_s, 3);  // non-trivial -> loop

    std::cout << "\n=== When to use which? ===\n";
    std::cout << "Tag dispatch:\n";
    std::cout << "  + Works in C++11/14\n";
    std::cout << "  + Each overload is a separate function (cleaner for many cases)\n";
    std::cout << "  + Naturally extensible (add more tag types)\n";
    std::cout << "  + Iterator category dispatch is classic example\n";
    std::cout << "  - More boilerplate\n";

    std::cout << "\nif constexpr:\n";
    std::cout << "  + Requires C++17\n";
    std::cout << "  + Less code, all logic in one function\n";
    std::cout << "  + Easier to read for simple binary choices\n";
    std::cout << "  - Harder to extend to many categories\n";
    std::cout << "  - All code paths visible in one function\n";

    return 0;
}
```

The reason tag dispatch still has a place even in C++17 codebases: when you have more than two categories (like the three iterator categories above), `if constexpr` chains become a ladder of `else if`. Tag dispatch gives you naturally separate, independently readable functions instead.

### Q3: Show how `std::true_type` and `std::false_type` are used as dispatch tags

`std::true_type` and `std::false_type` are themselves empty structs with a `value` member. Standard type traits inherit from them, which means you can pass a trait instantiation directly as a tag - no intermediate conversion needed.

```cpp
#include <iostream>
#include <type_traits>
#include <cstring>
#include <string>
#include <vector>

// std::true_type and std::false_type are the simplest dispatch tags:
// struct true_type  { static constexpr bool value = true; };
// struct false_type { static constexpr bool value = false; };

// === Example 1: Optimize destruction ===
template <typename T>
void destroy_impl(T* ptr, size_t n, std::true_type /*trivially_destructible*/) {
    std::cout << "  Trivially destructible -> no-op\n";
    // Nothing to do!
}

template <typename T>
void destroy_impl(T* ptr, size_t n, std::false_type /*trivially_destructible*/) {
    std::cout << "  Non-trivial -> calling destructors\n";
    for (size_t i = 0; i < n; ++i)
        ptr[i].~T();
}

template <typename T>
void destroy(T* ptr, size_t n) {
    // std::is_trivially_destructible<T> inherits from true_type or false_type
    destroy_impl(ptr, n, std::is_trivially_destructible<T>{});
}

// === Example 2: Serialize differently for POD vs complex ===
template <typename T>
void serialize_impl(const T& val, std::true_type /*trivially_copyable*/) {
    std::cout << "  Binary serialize: " << sizeof(T) << " bytes (memcpy-safe)\n";
}

template <typename T>
void serialize_impl(const T& val, std::false_type /*trivially_copyable*/) {
    std::cout << "  Custom serialize required (non-trivial type)\n";
}

template <typename T>
void serialize(const T& val) {
    serialize_impl(val, std::is_trivially_copyable<T>{});
}

// === Example 3: Dispatch on custom predicate ===
template <typename T>
using is_small = std::bool_constant<(sizeof(T) <= sizeof(void*))>;

template <typename T>
void pass_strategy(const T&, std::true_type /*small*/) {
    std::cout << "  Pass by value (small: " << sizeof(T) << " bytes)\n";
}

template <typename T>
void pass_strategy(const T&, std::false_type /*small*/) {
    std::cout << "  Pass by const ref (large: " << sizeof(T) << " bytes)\n";
}

template <typename T>
void analyze_passing(const T& val) {
    pass_strategy(val, is_small<T>{});
}

int main() {
    std::cout << "=== Destruction dispatch ===\n";
    int arr_int[3] = {1, 2, 3};
    destroy(arr_int, 3);  // trivial -> no-op

    std::string arr_str[2] = {"a", "b"};
    destroy(arr_str, 0);  // non-trivial -> destructors (n=0 here for safety)

    std::cout << "\n=== Serialization dispatch ===\n";
    int x = 42;
    serialize(x);                      // trivial -> binary
    std::string s = "hello";
    serialize(s);                      // non-trivial -> custom

    std::cout << "\n=== Custom bool_constant dispatch ===\n";
    analyze_passing(42);               // small
    analyze_passing(3.14);             // small (8 bytes, depends on pointer size)
    struct Big { char data[128]; };
    Big b{};
    analyze_passing(b);                // large

    return 0;
}
```

The `is_small` alias shows how `std::bool_constant<expr>` lets you turn any compile-time boolean expression into a tag. This is the bridge between a trait check and the dispatch mechanism.

---

## Notes

- **Tag dispatch** uses empty types as function parameters to select overloads at compile time.
- Standard tags: `std::true_type`, `std::false_type`, `std::random_access_iterator_tag`, etc.
- `std::bool_constant<expr>` creates custom true/false tags from compile-time expressions.
- **Advantage**: pre-C++17, naturally extensible, separate function bodies.
- **Disadvantage**: more boilerplate than `if constexpr` or concepts.
- In modern C++ (20+), prefer **concepts** for new code - tag dispatch is still useful in library code.
