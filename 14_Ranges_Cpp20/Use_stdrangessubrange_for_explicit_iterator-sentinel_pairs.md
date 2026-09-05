# Use std::ranges::subrange for explicit iterator-sentinel pairs

**Category:** Ranges Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/subrange>  

---

## Topic Overview

`std::ranges::subrange` is a lightweight, non-owning view that wraps an **iterator-sentinel** pair into a proper range. Its main job is to bridge the gap between legacy APIs that return pairs of iterators and the modern ranges world that expects a single range object.

Think of it as a named handle for a portion of a range. Instead of carrying around two separate iterators and remembering which is which, you wrap them into a `subrange` and get all the range operations for free.

### What subrange Provides

| Feature | Description |
| --- | --- |
| Construction | From `(iterator, sentinel)` or `(iterator, sentinel, size)` |
| Models | `ranges::view`, `ranges::borrowed_range` |
| Sized | If `sized_sentinel_for<S, I>` is satisfied |
| Category | Inherits the iterator category of `I` |

### Class Template Signature

```cpp
template<std::input_or_output_iterator I,
         std::sentinel_for<I> S = I,
         std::ranges::subrange_kind K = /* deduced */>
class subrange : public std::ranges::view_interface<subrange<I, S, K>>;
```

### Key Operations

```cpp
subrange sr(first, last);     // from iterator pair
sr.begin();                   // returns first
sr.end();                     // returns last
sr.size();                    // if sized
sr.advance(n);                // returns subrange with begin advanced by n
auto [b, e] = sr;             // structured bindings (C++17)
```

### When to Use subrange

1. **Legacy API returns two iterators** - wrap them to use with ranges algorithms.
2. **Pass a sub-portion of a container** to `ranges::sort`, `ranges::find`, etc.
3. **Store an iterator-sentinel pair** in a variable for reuse.
4. **Destructure with structured bindings** for cleaner code.

---

## Self-Assessment

### Q1: Construct a subrange from two iterators and pass it to a range algorithm

The key thing here is that a `subrange` over five elements of a vector behaves exactly like any other range. You can sort it, count into it, and iterate it - and all changes happen in-place in the original vector.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {9, 3, 7, 1, 5, 8, 2, 6, 4};

    // Sort only the middle portion [index 2, index 7)
    auto sr = std::ranges::subrange(data.begin() + 2, data.begin() + 7);
    std::ranges::sort(sr);

    std::cout << "After partial sort: ";
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';

    // Count elements > 5 in the subrange
    auto count = std::ranges::count_if(sr, [](int n) { return n > 5; });
    std::cout << "Elements > 5 in subrange: " << count << '\n';

    // Subrange from first 4 elements
    auto first4 = std::ranges::subrange(data.begin(), data.begin() + 4);
    std::cout << "First 4: ";
    for (int x : first4) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// After partial sort: 9 3 1 2 5 8 7 6 4
// Elements > 5 in subrange: 1
// First 4: 9 3 1 2
```

`subrange` satisfies `random_access_range` and `sized_range` when constructed from random-access iterators, so all algorithms that require those properties work on it directly.

### Q2: Show how subrange bridges legacy iterator-pair APIs and range-based algorithms

This is the most common real-world use case: you're calling a pre-C++20 function that returns a `std::pair<It, It>`, and you want to feed that result into a ranges pipeline. One `subrange` constructor call is all it takes.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

// Legacy API: returns a pair of iterators (old style)
template<typename It>
std::pair<It, It> find_run_of(It first, It last, int value) {
    auto start = std::find(first, last, value);
    auto end = start;
    while (end != last && *end == value) ++end;
    return {start, end};
}

int main() {
    std::vector<int> data = {1, 1, 3, 3, 3, 5, 5, 7};

    // Legacy API returns a pair of iterators
    auto [first, last] = find_run_of(data.begin(), data.end(), 3);

    // Bridge: wrap in subrange to use with ranges algorithms
    auto run = std::ranges::subrange(first, last);

    std::cout << "Run of 3s has " << std::ranges::distance(run)
              << " elements\n";

    // Now use range-based operations on the subrange
    auto transformed = run | std::views::transform([](int n) { return n * 10; });
    std::cout << "Transformed: ";
    for (int x : transformed) std::cout << x << ' ';
    std::cout << '\n';

    // ranges::equal_range also returns a subrange directly (C++20)
    auto eq = std::ranges::equal_range(data, 5);
    std::cout << "Equal range for 5: ";
    for (int x : eq) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Run of 3s has 3 elements
// Transformed: 30 30 30
// Equal range for 5: 5 5
```

Note that `ranges::equal_range` at the end already returns a `subrange` - you don't need to wrap it manually. Many ranges algorithms return subranges natively, so you'll often find yourself receiving one rather than constructing one.

### Q3: Use subrange with structured bindings to destructure `[begin, end)` from a function return

This pattern is the modern replacement for the erase-remove idiom. Instead of `v.erase(std::remove(...), v.end())` in one long expression, you get the removed-tail subrange, destructure it into its boundaries, and then erase. Each step is readable on its own.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // ranges::remove returns a subrange of the "removed" tail
    auto removed = std::ranges::remove(data, 5);

    // Destructure the subrange with structured bindings
    auto [tail_begin, tail_end] = removed;

    std::cout << "Valid elements: ";
    for (auto it = data.begin(); it != tail_begin; ++it)
        std::cout << *it << ' ';
    std::cout << '\n';

    // Actually erase the tail
    data.erase(tail_begin, tail_end);
    std::cout << "After erase: ";
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';

    // Another example: partition returns a subrange of the second partition
    std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    auto [second_begin, second_end] = std::ranges::partition(nums,
        [](int n) { return n % 2 == 0; });

    std::cout << "Odd partition: ";
    for (auto it = second_begin; it != second_end; ++it)
        std::cout << *it << ' ';
    std::cout << '\n';
}
// Expected output:
// Valid elements: 1 2 3 4 6 7 8 9 10
// After erase: 1 2 3 4 6 7 8 9 10
// Odd partition: (odd numbers in unspecified order)
```

The structured bindings `auto [tail_begin, tail_end] = removed` are directly unpacking the `subrange`'s begin and end. This replaces the old erase-remove idiom `v.erase(std::remove(v.begin(), v.end(), 5), v.end())` with code that names each piece. Many ranges algorithms - `partition`, `remove`, `unique`, `equal_range` - return subranges for exactly this reason.

---

## Notes

- `subrange` is a **borrowed range** - it doesn't own the data, so it's safe to return from functions when the underlying data outlives it.
- Use `subrange(it, sentinel, size_hint)` to provide a size when the sentinel type doesn't support `operator-`.
- `subrange::advance(n)` returns a new subrange with `begin()` advanced by `n` steps - useful for skipping elements.
- `std::ranges::subrange` satisfies `view` (it's cheap to copy - just two iterators/sentinels) and can be used anywhere a view is expected.
