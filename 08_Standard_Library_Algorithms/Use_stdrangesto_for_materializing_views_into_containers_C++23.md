# Use std::ranges::to for materializing views into containers (C++23)

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/to>  

---

## Topic Overview

`std::ranges::to` (C++23) converts a range or view into a concrete container. Before C++23, materializing a view pipeline into a `vector`, `set`, or `map` required awkward manual construction - you had to either pass iterators to the container constructor, or use `std::ranges::copy` with a `back_inserter`. Neither approach is terrible, but neither reads cleanly either.

### Before vs After

Here's the contrast that motivates `ranges::to`:

```cpp
// Pre-C++23 - verbose
auto view = v | views::filter(pred) | views::transform(fn);
std::vector<int> result(view.begin(), view.end());  // or
std::vector<int> result;
std::ranges::copy(view, std::back_inserter(result));

// C++23 - elegant
auto result = v | views::filter(pred)
                | views::transform(fn)
                | std::ranges::to<std::vector>();
```

### Syntax

Both pipe syntax and direct function call syntax work:

```cpp
#include <ranges>

// Pipe syntax (most common)
auto vec = view | std::ranges::to<std::vector>();
auto set = view | std::ranges::to<std::set>();

// Function call syntax
auto vec = std::ranges::to<std::vector>(view);

// With nested conversion (recursive)
auto vec_of_vec = nested_view | std::ranges::to<std::vector<std::vector<int>>>();
```

### How It Works

1. If the container has a range constructor, use it directly
2. Otherwise, if the container has `push_back`/`insert`, iterate and insert
3. Recursively applies `to` for nested containers

---

## Self-Assessment

### Q1: Convert a filtered view to a std::vector using ranges::to\<std::vector\>()

The pipeline stays lazy until `ranges::to` forces evaluation. At that point the whole chain is materialized into a concrete, ownable container:

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <set>
#include <string>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // === Filter + transform -> vector ===
    auto evens_squared = data
        | std::views::filter([](int x) { return x % 2 == 0; })
        | std::views::transform([](int x) { return x * x; })
        | std::ranges::to<std::vector>();

    std::cout << "Even squares: ";
    for (int x : evens_squared) std::cout << x << " ";
    std::cout << "\n";
    // 4 16 36 64 100

    // Type is std::vector<int> - fully materialized
    static_assert(std::same_as<decltype(evens_squared), std::vector<int>>);

    // === Materialize to set (deduplicates) ===
    std::vector<int> dupes = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5};
    auto unique_set = dupes | std::ranges::to<std::set>();
    std::cout << "Unique sorted: ";
    for (int x : unique_set) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5 6 9

    // === Materialize to string ===
    auto upper = std::string("hello world")
        | std::views::transform([](char c) { return static_cast<char>(std::toupper(c)); })
        | std::ranges::to<std::string>();
    std::cout << "Upper: " << upper << "\n";  // HELLO WORLD

    return 0;
}
```

Notice how `| std::ranges::to<std::set>()` simultaneously materializes and deduplicates - you get a sorted, unique container from a range with duplicates in a single expression.

### Q2: Show that ranges::to performs a single-pass collection without intermediate allocations

This example instruments the pipeline so you can see the element-by-element interleaving. The entire filter-transform-take chain is processed in a single pass - there are no intermediate vectors created between stages:

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <list>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // === Single-pass: the entire pipeline is lazy until ranges::to ===
    // No intermediate vectors/lists are created for each stage
    auto result = data
        | std::views::filter([](int x) {
            std::cout << "  filter(" << x << ")\n";
            return x % 2 == 0;
          })
        | std::views::transform([](int x) {
            std::cout << "  transform(" << x << ")\n";
            return x * 10;
          })
        | std::views::take(3)
        | std::ranges::to<std::vector>();

    // Output shows interleaved filter/transform - single pass through data:
    //   filter(1)       <- rejected
    //   filter(2)       <- accepted
    //   transform(2)    <- transformed immediately
    //   filter(3)       <- rejected
    //   filter(4)       <- accepted
    //   transform(4)    <- etc.
    //   filter(5)       <- rejected
    //   filter(6)       <- accepted
    //   transform(6)
    // Stopped at take(3) - never touches 7, 8, 9, 10

    std::cout << "\nResult: ";
    for (int x : result) std::cout << x << " ";
    std::cout << "\n";
    // 20 40 60

    // === ranges::to can use size hints for efficient allocation ===
    // If the source range is sized, ranges::to can reserve() before inserting
    auto sized_result = data | std::ranges::to<std::vector>();
    // vector likely allocated once with capacity 10

    // === Works with non-random-access containers too ===
    auto as_list = data
        | std::views::filter([](int x) { return x > 5; })
        | std::ranges::to<std::list>();
    // std::list<int> with elements {6, 7, 8, 9, 10}

    std::cout << "List: ";
    for (int x : as_list) std::cout << x << " ";
    std::cout << "\n";

    return 0;
}
```

The `take(3)` early-exit is a great illustration: because the pipeline is lazy, elements 7 through 10 are never even examined. If you had built this with intermediate `std::vector` materializations at each stage, you'd have done unnecessary work.

### Q3: Use ranges::to to build a map from a transformed range of pairs

`ranges::to` understands map-like containers too - as long as the range produces pair-like values, it can insert them directly into a `map` or `unordered_map`:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <map>
#include <unordered_map>
#include <utility>

int main() {
    // === Build map from index-value pairs ===
    std::vector<std::string> words = {"zero", "one", "two", "three", "four"};

    auto index_map = words
        | std::views::enumerate                 // C++23: (index, value) pairs
        | std::views::transform([](auto pair) {
            auto [idx, word] = pair;
            return std::pair{static_cast<int>(idx), std::string(word)};
          })
        | std::ranges::to<std::map>();

    std::cout << "Index map:\n";
    for (auto& [k, v] : index_map)
        std::cout << "  " << k << " -> " << v << "\n";
    // 0 -> zero, 1 -> one, 2 -> two, ...

    // === Build unordered_map: word -> length ===
    auto length_map = words
        | std::views::transform([](const std::string& w) {
            return std::pair{w, w.size()};
          })
        | std::ranges::to<std::unordered_map>();

    std::cout << "\nLength map:\n";
    for (auto& [word, len] : length_map)
        std::cout << "  " << word << " -> " << len << "\n";

    // === Frequency map from a collection ===
    std::vector<int> data = {1, 2, 2, 3, 3, 3, 4, 4, 4, 4};
    std::map<int, int> freq;
    for (int x : data) freq[x]++;

    // Display
    std::cout << "\nFrequency:\n";
    for (auto& [val, count] : freq)
        std::cout << "  " << val << " appears " << count << " times\n";

    // === Nested views: vector<vector<int>> ===
    std::vector<int> flat = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    auto grid = flat
        | std::views::chunk(3)
        | std::views::transform([](auto chunk) {
            return chunk | std::ranges::to<std::vector>();
          })
        | std::ranges::to<std::vector>();

    std::cout << "\n3x3 Grid:\n";
    for (auto& row : grid) {
        std::cout << "  ";
        for (int x : row) std::cout << x << " ";
        std::cout << "\n";
    }
    // 1 2 3 / 4 5 6 / 7 8 9

    return 0;
}
```

The nested grid example is worth studying: each inner chunk is materialized with its own `ranges::to<std::vector>()` call inside the `transform` lambda, and then the outer `ranges::to<std::vector>()` collects those vectors. The recursive materialization composes naturally.

---

## Notes

- **C++23 required.** Compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- `ranges::to` is the **missing piece** that makes view pipelines fully practical - without it, converting views back to containers was verbose.
- **Size optimization:** When the source range models `sized_range`, `ranges::to` can call `reserve()` on the target container to avoid reallocations.
- **Recursive conversion:** `ranges::to<vector<vector<int>>>()` recursively materializes nested views.
- Pipe syntax `| ranges::to<Container>()` is the idiomatic usage - the parentheses are required.
- Works with any container constructible from a range, or that supports `push_back`/`insert`/`emplace`.
