# Understand the ranges concept hierarchy: range, view, viewable_range

**Category:** Ranges Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges>  

---

## Topic Overview

C++20 defines key **range concepts** that classify types by their capabilities. The most important distinction to internalize is the difference between a *range* (anything you can iterate over) and a *view* (a range that's cheap enough to pass around by value through a pipeline).

```cpp
    range                <- has begin() and end()
      +- borrowed_range   <- safe to use after the range object dies
      +- sized_range      <- O(1) size()
      +- viewable_range   <- can be piped into view adaptors
      +- view             <- O(1) copy/move, non-owning
```

The reason the `view` concept requires O(1) copy/move is that view adaptors pass views by value through the pipeline. If copying a view were O(n), every `|` would silently copy your entire data set. By requiring O(1), the standard guarantees that composing views with `|` is essentially free.

| Concept | Key Requirement | Example YES | Example NO |
| --- | --- | --- | --- |
| `range` | Has `begin()` + `end()` | `vector`, `string`, `span` | `int` |
| `view` | O(1) move, non-owning | `string_view`, `span`, `filter_view` | `vector` (O(n) copy) |
| `viewable_range` | Can be passed to `\|` | `vector&`, `string_view`, views | rvalue `vector` (C++20) |
| `borrowed_range` | Iterators outlive range | `span`, `string_view`, `subrange` | `vector` |

---

## Self-Assessment

### Q1: Explain what a `view` is and why O(1) copy/move is required

The intuition here is that a pipeline like `v | filter(p) | transform(f) | take(5)` creates view objects at each stage. If creating those objects were O(n), you'd pay the cost of the entire range just to build the pipeline description - before even iterating it. The O(1) requirement ensures the pipeline is built in constant time and all the actual work happens during iteration.

```cpp
#include <iostream>
#include <ranges>
#include <vector>
#include <string_view>
#include <span>
#include <concepts>

int main() {
    // A VIEW is a range that:
    // 1. Is O(1) move-constructible
    // 2. Is O(1) move-assignable
    // 3. Is O(1) destructible
    // 4. Does NOT own the elements (non-owning)

    // Views are views:
    static_assert(std::ranges::view<std::string_view>);
    static_assert(std::ranges::view<std::span<int>>);

    std::vector<int> v = {1, 2, 3, 4, 5};
    auto filt = v | std::views::filter([](int x) { return x > 2; });
    static_assert(std::ranges::view<decltype(filt)>);

    // Containers are NOT views:
    static_assert(!std::ranges::view<std::vector<int>>);
    static_assert(!std::ranges::view<std::string>);

    // WHY O(1) copy/move is required:
    // View adaptors (|) pass views by VALUE through the pipeline.
    // If copying a view were O(n), a pipeline like:
    //   v | filter(p) | transform(f) | take(5)
    // would create copies at each stage -> O(n) per stage!
    // With O(1) views, the whole pipeline is essentially free.

    std::cout << "string_view is view: "
              << std::ranges::view<std::string_view> << "\n";
    std::cout << "vector is view: "
              << std::ranges::view<std::vector<int>> << "\n";
    std::cout << "filter_view is view: "
              << std::ranges::view<decltype(filt)> << "\n";

    // View sizes:
    std::cout << "\nsizeof(string_view): " << sizeof(std::string_view) << "\n";
    std::cout << "sizeof(span<int>):   " << sizeof(std::span<int>) << "\n";
    std::cout << "sizeof(filter_view): " << sizeof(filt) << "\n";
}
// Expected output:
//   string_view is view: 1
//   vector is view: 0
//   filter_view is view: 1
//
//   sizeof(string_view): 16
//   sizeof(span<int>):   16
//   sizeof(filter_view): <small, depends on lambda size>
```

Look at those `sizeof` values - `string_view` and `span<int>` are just two pointers (or pointer + size). A `filter_view` is similarly tiny: a pointer to the source range plus the stored predicate. Nothing proportional to the number of elements.

---

### Q2: Show that `vector` is a range but not a view, while `string_view` is both

This `classify` helper makes it easy to see the four properties side by side. The pattern you'll notice is that owning containers (`vector`, `string`) are ranges but not views and not borrowed, while non-owning wrappers (`string_view`, `span`) are all four.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <string_view>
#include <span>
#include <ranges>
#include <concepts>

template <typename T>
void classify(const char* name) {
    std::cout << name << ":\n";
    std::cout << "  range:       " << std::boolalpha << std::ranges::range<T> << "\n";
    std::cout << "  view:        " << std::ranges::view<T> << "\n";
    std::cout << "  sized_range: " << std::ranges::sized_range<T> << "\n";
    std::cout << "  borrowed:    " << std::ranges::borrowed_range<T> << "\n";
}

int main() {
    classify<std::vector<int>>("vector<int>");
    classify<std::string>("string");
    classify<std::string_view>("string_view");
    classify<std::span<int>>("span<int>");

    // vector: range YES, view NO (O(n) copy, owns elements)
    // string_view: range YES, view YES (O(1) copy, non-owning)
    // span: range YES, view YES, borrowed_range YES

    // Practical consequence:
    std::vector<int> v = {1, 2, 3};

    // This works because | takes a reference to v:
    auto r1 = v | std::views::transform([](int x) { return x * 2; });
    for (int x : r1) std::cout << x << " ";
    std::cout << "\n";

    // std::views::all wraps a container reference into a view:
    auto all_v = std::views::all(v);  // ref_view<vector<int>>
    static_assert(std::ranges::view<decltype(all_v)>);
    std::cout << "views::all(vector) is view: true\n";
}
// Expected output:
//   vector<int>:
//     range:       true
//     view:        false
//     sized_range: true
//     borrowed:    false
//
//   string:
//     range:       true
//     view:        false
//     sized_range: true
//     borrowed:    false
//
//   string_view:
//     range:       true
//     view:        true
//     sized_range: true
//     borrowed:    true
//
//   span<int>:
//     range:       true
//     view:        true
//     sized_range: true
//     borrowed:    true
//
//   2 4 6
//   views::all(vector) is view: true
```

The `views::all(v)` call at the end is what happens implicitly when you write `v | some_adaptor`. The pipe operator wraps the lvalue container in a `ref_view` (a view over a reference), so the pipeline sees a proper view. This is transparent to you in normal use, but understanding it explains why `v | transform(...)` works even though `vector` is not a view.

---

### Q3: Write a custom range type that satisfies `std::ranges::range`

All you need to satisfy `std::ranges::range` is a working `begin()` and `end()`. But to get meaningful behavior from algorithms, your iterator needs the right operations and type aliases. This example starts from scratch and shows the minimum required.

```cpp
#include <iostream>
#include <ranges>
#include <concepts>
#include <algorithm>
#include <cstddef>

// Minimal custom range: generates integers in [start, end)
struct IntRange {
    int start_val;
    int end_val;

    // begin() and end() are ALL that's needed for std::ranges::range
    int* begin() { return nullptr; }  // won't work - need proper iterators

    // Let's use a proper iterator:
    struct Iterator {
        int current;

        using value_type = int;
        using difference_type = std::ptrdiff_t;

        int operator*() const { return current; }
        Iterator& operator++() { ++current; return *this; }
        Iterator operator++(int) { auto tmp = *this; ++current; return tmp; }

        bool operator==(const Iterator&) const = default;
    };

    Iterator begin() const { return {start_val}; }
    Iterator end() const { return {end_val}; }
};

// Verify it's a range!
static_assert(std::ranges::range<IntRange>);
static_assert(std::ranges::forward_range<IntRange>);  // multi-pass!

int main() {
    IntRange r{1, 6};

    // Works with range-based for:
    std::cout << "IntRange: ";
    for (int x : r) std::cout << x << " ";
    std::cout << "\n";

    // Works with range algorithms:
    auto sum = 0;
    std::ranges::for_each(r, [&sum](int x) { sum += x; });
    std::cout << "Sum: " << sum << "\n";

    // Works with views:
    std::cout << "Squared: ";
    for (int x : r | std::views::transform([](int x) { return x * x; })) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    // Check concept satisfaction:
    std::cout << std::boolalpha;
    std::cout << "range: " << std::ranges::range<IntRange> << "\n";
    std::cout << "view:  " << std::ranges::view<IntRange> << "\n";
}
// Expected output:
//   IntRange: 1 2 3 4 5
//   Sum: 15
//   Squared: 1 4 9 16 25
//   range: true
//   view:  false
```

`IntRange` is a `range` and even a `forward_range` (because `Iterator` supports multi-pass), but it's not a `view` - it doesn't have the O(1) copy guarantee, and it doesn't opt in via `view_interface`. If you wanted to use it directly in a pipe with `|`, you'd need to either wrap it with `views::all` or inherit from `view_interface<IntRange>`.

---

## Notes

- **`viewable_range`:** A range that can be converted to a view (via `views::all`). Lvalue containers qualify. Rvalue containers don't in C++20 but do in C++23 via `owning_view`.
- **`views::all`:** Wraps an lvalue container in `ref_view<C>` or returns a view as-is.
- **`view_interface<D>`:** CRTP base that provides default `front()`, `back()`, `empty()`, `size()`, `operator[]`, `operator bool` from `begin()`/`end()`.
- **`enable_view<T>`:** Opt-in trait to mark a type as a view. Inherit from `view_interface` or specialize `enable_view`.
- **`borrowed_range`:** Iterators from a `borrowed_range` remain valid even after the range object is destroyed. `span`, `string_view`, and `subrange` are borrowed.
