# Use std::ranges::views::zip and views::enumerate (C++23)

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/zip_view>  

---

## Topic Overview

`std::views::zip` and `std::views::enumerate` (C++23) solve two extremely common iteration patterns:

- **`zip`** - iterate over multiple ranges in lockstep, producing tuples
- **`enumerate`** - iterate with `(index, value)` pairs (replaces index-based for loops)

### Quick Reference

```cpp
#include <ranges>

// Zip: pair elements from multiple ranges
for (auto [a, b] : std::views::zip(vec1, vec2)) { ... }

// Enumerate: get (index, value) pairs
for (auto [i, val] : std::views::enumerate(vec)) { ... }
```

### Before vs After

Here's what these views replace in practice:

```cpp
// Pre-C++23: index-based loop
for (size_t i = 0; i < v.size(); ++i)
    std::cout << i << ": " << v[i] << "\n";

// C++23: enumerate
for (auto [i, val] : v | std::views::enumerate)
    std::cout << i << ": " << val << "\n";

// Pre-C++23: parallel iteration
for (size_t i = 0; i < std::min(a.size(), b.size()); ++i)
    result.push_back(a[i] + b[i]);

// C++23: zip
for (auto [x, y] : std::views::zip(a, b))
    result.push_back(x + y);
```

---

## Self-Assessment

### Q1: Use views::zip to iterate over two ranges simultaneously with structured bindings

`zip` produces reference tuples, so you can modify elements in-place through them. Notice that three-way zip works just as naturally as two-way:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <ranges>

int main() {
    std::vector<std::string> names = {"Alice", "Bob", "Carol", "Dave"};
    std::vector<int> scores = {95, 87, 92, 78};

    // === Zip two ranges ===
    std::cout << "Scores:\n";
    for (auto [name, score] : std::views::zip(names, scores)) {
        std::cout << "  " << name << ": " << score << "\n";
    }
    // Alice: 95, Bob: 87, Carol: 92, Dave: 78

    // === Zip three ranges ===
    std::vector<char> grades = {'A', 'B', 'A', 'C'};
    std::cout << "\nFull report:\n";
    for (auto [name, score, grade] : std::views::zip(names, scores, grades)) {
        std::cout << "  " << name << " scored " << score << " (grade " << grade << ")\n";
    }

    // === Zip allows modification through references ===
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20, 30};
    for (auto [x, y] : std::views::zip(a, b)) {
        x += y;  // modifies 'a' in-place
    }
    std::cout << "\nAfter zip-modify, a: ";
    for (int x : a) std::cout << x << " ";  // 11 22 33
    std::cout << "\n";

    // === Dot product with zip ===
    std::vector<double> v1 = {1.0, 2.0, 3.0};
    std::vector<double> v2 = {4.0, 5.0, 6.0};
    double dot = 0;
    for (auto [x, y] : std::views::zip(v1, v2))
        dot += x * y;
    std::cout << "Dot product: " << dot << "\n";  // 32

    return 0;
}
```

The in-place modification through `zip` is genuinely useful. Since `x` in the loop body is a reference into `a`, writing `x += y` updates the original vector directly - no indexing required.

### Q2: Rewrite a classic index-based for loop using views::enumerate

`enumerate` eliminates the common pattern where you maintain a separate counter variable just to track the position of the current element. The index and value arrive together as a pair:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <ranges>

int main() {
    std::vector<std::string> fruits = {"apple", "banana", "cherry", "date"};

    // === Old way: index-based ===
    std::cout << "Index-based:\n";
    for (size_t i = 0; i < fruits.size(); ++i) {
        std::cout << "  " << i << ". " << fruits[i] << "\n";
    }

    // === C++23 way: enumerate ===
    std::cout << "\nEnumerate:\n";
    for (auto [i, fruit] : fruits | std::views::enumerate) {
        std::cout << "  " << i << ". " << fruit << "\n";
    }
    // 0. apple, 1. banana, 2. cherry, 3. date

    // === enumerate gives a reference - we can modify ===
    for (auto [i, fruit] : fruits | std::views::enumerate) {
        fruit = "[" + std::to_string(i) + "] " + fruit;
    }
    std::cout << "\nModified:\n";
    for (const auto& f : fruits) std::cout << "  " << f << "\n";
    // [0] apple, [1] banana, [2] cherry, [3] date

    // === Enumerate with pipeline ===
    std::vector<int> data = {10, 20, 30, 40, 50};
    std::cout << "\nEven-indexed elements:\n";
    for (auto [i, val] : data | std::views::enumerate
                               | std::views::filter([](auto p) {
                                   return std::get<0>(p) % 2 == 0;
                                 }))
    {
        std::cout << "  [" << i << "] = " << val << "\n";
    }
    // [0] = 10, [2] = 30, [4] = 50

    return 0;
}
```

The modification example shows that `enumerate` produces real references to the original elements, not copies. Writing to `fruit` in the loop body updates `fruits[i]` directly.

### Q3: Show that zip with ranges of different sizes stops at the shortest range

This is the safety guarantee that makes `zip` usable without worrying about bounds: it always stops at the end of whichever range runs out first. No undefined behavior, no out-of-bounds access:

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <string>
#include <list>

int main() {
    std::vector<int> short_range = {1, 2, 3};
    std::vector<int> long_range = {10, 20, 30, 40, 50, 60};

    // === Zip stops at the shorter range ===
    std::cout << "Zip different sizes:\n";
    for (auto [a, b] : std::views::zip(short_range, long_range)) {
        std::cout << "  (" << a << ", " << b << ")\n";
    }
    // (1, 10), (2, 20), (3, 30)  - only 3 pairs, not 6

    // === This is intentional and safe ===
    // No out-of-bounds access
    auto zipped = std::views::zip(short_range, long_range);
    std::cout << "Zipped size: " << std::ranges::distance(zipped) << "\n";  // 3

    // === Practical: safe parallel iteration with heterogeneous containers ===
    std::list<std::string> labels = {"x", "y"};  // only 2
    std::vector<double> values = {3.14, 2.71, 1.41, 1.73};  // 4

    for (auto [label, val] : std::views::zip(labels, values)) {
        std::cout << label << " = " << val << "\n";
    }
    // x = 3.14, y = 2.71  - stops at 2 (shortest)

    // === zip_view is sized if all inputs are sized ===
    // Size is min of all input sizes

    // === Empty range makes zip produce nothing ===
    std::vector<int> empty;
    auto zipped_empty = std::views::zip(short_range, empty);
    std::cout << "\nWith empty: " << std::ranges::distance(zipped_empty)
              << " elements\n";  // 0

    return 0;
}
```

The heterogeneous container example (`std::list` and `std::vector`) is a good reminder that `zip` works across different container types, not just matching pairs of vectors.

---

## Notes

- **C++23 required.** Compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- **`enumerate`** index type is `std::ranges::range_difference_t` (typically `ptrdiff_t`), not `size_t`.
- Both `zip` and `enumerate` produce **reference tuples** - you can modify the original range through them.
- `zip` with ranges of different sizes stops at the **shortest** range - no undefined behavior.
- **`views::zip_transform(fn, r1, r2)`** is the combined zip+transform - use it when you want to apply a function to zipped elements without a separate transform step.
