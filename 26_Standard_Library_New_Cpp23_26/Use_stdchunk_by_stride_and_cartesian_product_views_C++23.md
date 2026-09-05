# Use `chunk_by`, `stride`, and `cartesian_product` Views (C++23)

**Category:** Standard Library New Cpp23 26  
**Standard:** C++23  
**Reference:** [cppreference — range adaptors](https://en.cppreference.com/w/cpp/ranges#Range_adaptors)  

---

## Topic Overview

C++23 significantly expanded the ranges library with new view adaptors that eliminate common hand-written loop patterns. This file is a quick reference covering seven key additions: `chunk_by`, `stride`, `cartesian_product`, `zip`, `zip_transform`, `adjacent`, and `pairwise`. Each solves a specific iteration pattern that previously required verbose, error-prone manual code.

| View | Purpose | Input | Output element |
| --- | --- | --- | --- |
| `chunk_by(pred)` | Split range where predicate is false between adjacent elements | Range | Subranges (groups) |
| `stride(n)` | Take every n-th element | Range | Elements |
| `cartesian_product(R1,R2,...)` | All combinations from multiple ranges | N ranges | Tuples |
| `zip(R1,R2,...)` | Parallel iteration over multiple ranges | N ranges | Tuples of references |
| `zip_transform(f,R1,R2,...)` | Zip then apply function | N ranges + function | Transformed values |
| `adjacent<N>` | Sliding window of N adjacent elements | Range | Tuples (N refs) |
| `pairwise` | `adjacent<2>` - consecutive pairs | Range | Pairs |

To make the relationships concrete, here is what each view produces given the same input sequence:

```cpp
Source: [1, 1, 2, 2, 2, 3, 3]

 chunk_by(equal):     [1,1] [2,2,2] [3,3]    <- groups of equal elements
 stride(3):           [1, 2, 3]               <- every 3rd element
 adjacent<3>:         (1,1,2)(1,2,2)(2,2,2)(2,2,3)(2,3,3)  <- sliding triple
 pairwise:            (1,1)(1,2)(2,2)(2,2)(2,3)(3,3)       <- consecutive pairs
```

All views are lazy and compose with pipe syntax. They work with any range that satisfies their minimum iterator requirements.

---

## Self-Assessment

### Q1: How do `chunk_by` and `stride` work, and what patterns do they replace

`chunk_by` is like Python's `itertools.groupby` - it walks through the range and starts a new group whenever the predicate returns false for a pair of adjacent elements. `stride` is the "take every Nth element" pattern you might have written with a manual index before. Here is both in action:

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <print>
#include <algorithm>
#include <cctype>

namespace rv = std::ranges::views;

int main() {
    // chunk_by: split where predicate fails

    // Group consecutive equal elements (like Python's itertools.groupby)
    std::vector<int> data = {1, 1, 2, 2, 2, 3, 1, 1};
    std::println("chunk_by (equal groups):");
    for (auto group : data | rv::chunk_by(std::equal_to{})) {
        std::println("  {}", group | std::ranges::to<std::vector>());
    }
    // [1, 1]  [2, 2, 2]  [3]  [1, 1]

    // Group by ascending runs
    std::vector<int> vals = {1, 3, 5, 2, 4, 1, 6, 7};
    std::println("chunk_by (ascending runs):");
    for (auto run : vals | rv::chunk_by(std::less{})) {
        std::println("  {}", run | std::ranges::to<std::vector>());
    }
    // [1, 3, 5]  [2, 4]  [1, 6, 7]

    // Split text into word/non-word groups
    std::string text = "hello  world   foo";
    std::println("chunk_by (word boundaries):");
    for (auto group : text | rv::chunk_by([](char a, char b) {
        return std::isalpha(a) == std::isalpha(b);
    })) {
        auto s = group | std::ranges::to<std::string>();
        std::println("  '{}'", s);
    }
    // 'hello'  '  '  'world'  '   '  'foo'

    // stride: take every N-th element

    auto numbers = rv::iota(0, 20);

    // Every 3rd element
    std::println("stride(3): {}",
        numbers | rv::stride(3) | std::ranges::to<std::vector>());
    // [0, 3, 6, 9, 12, 15, 18]

    // Downsample: take every 4th sample from a signal
    std::vector<double> signal = {0.0, 0.1, 0.3, 0.5, 0.8, 1.0,
                                   0.9, 0.7, 0.4, 0.2, 0.0, -0.1};
    auto downsampled = signal | rv::stride(4)
                              | std::ranges::to<std::vector>();
    std::println("Downsampled: {}", downsampled);
    // [0.0, 0.8, 0.4]

    // Combine: chunk of stride - matrix column selection
    std::vector<int> flat_matrix = {
        1, 2, 3, 4,   // row 0
        5, 6, 7, 8,   // row 1
        9, 10, 11, 12  // row 2
    };
    // Extract column 1 (0-indexed) from a 4-column flat matrix
    auto col1 = flat_matrix | rv::drop(1) | rv::stride(4)
                             | std::ranges::to<std::vector>();
    std::println("Column 1: {}", col1);  // [2, 6, 10]
}
```

The column extraction example at the end is a nice demonstration of how views compose: `drop(1)` skips to the first column-1 element, then `stride(4)` hops row by row. No index arithmetic, no off-by-one errors.

---

### Q2: How do `zip`, `zip_transform`, and `cartesian_product` work

`zip` lets you walk multiple ranges in parallel, which replaces the classic `for (int i = 0; i < n; ++i)` index trick. `zip_transform` fuses the zip with a function, avoiding intermediate tuple allocations. `cartesian_product` gives you every combination - think nested loops without the nesting:

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <print>
#include <cmath>
#include <numeric>
#include <functional>

namespace rv = std::ranges::views;

int main() {
    // zip: parallel iteration
    std::vector<std::string> names = {"Alice", "Bob", "Charlie"};
    std::vector<int> scores = {95, 87, 92};
    std::vector<char> grades = {'A', 'B', 'A'};

    // Binary zip
    std::println("zip (name, score):");
    for (auto [name, score] : rv::zip(names, scores)) {
        std::println("  {} scored {}", name, score);
    }

    // N-ary zip - stops at shortest
    std::println("zip (name, score, grade):");
    for (auto [name, score, grade] : rv::zip(names, scores, grades)) {
        std::println("  {} scored {} (grade {})", name, score, grade);
    }

    // zip_transform: zip + apply function
    std::vector<double> xs = {1.0, 2.0, 3.0, 4.0};
    std::vector<double> ys = {4.0, 3.0, 2.0, 1.0};

    // Compute Euclidean distances from origin for (x,y) pairs
    auto distances = rv::zip_transform(
        [](double x, double y) { return std::hypot(x, y); },
        xs, ys
    ) | std::ranges::to<std::vector>();
    std::println("Distances: {}", distances);
    // [4.123, 3.606, 3.606, 4.123]

    // Dot product via zip_transform + accumulate
    auto products = rv::zip_transform(std::multiplies{}, xs, ys);
    double dot = std::ranges::fold_left(products, 0.0, std::plus{});
    std::println("Dot product: {}", dot);  // 20.0

    // cartesian_product: all combinations
    std::vector<char> suits = {'H', 'D', 'C', 'S'};
    std::vector<std::string> ranks = {"A", "K", "Q", "J"};

    std::println("First 8 cards (cartesian_product):");
    int count = 0;
    for (auto [rank, suit] : rv::cartesian_product(ranks, suits)) {
        std::println("  {}{}", rank, suit);
        if (++count >= 8) break;
    }
    // AH AD AC AS KH KD KC KS

    // Cartesian product for grid coordinates
    std::println("3x3 grid:");
    for (auto [r, c] : rv::cartesian_product(rv::iota(0, 3), rv::iota(0, 3))) {
        std::print("({},{}) ", r, c);
    }
    std::println("");
    // (0,0) (0,1) (0,2) (1,0) (1,1) (1,2) (2,0) (2,1) (2,2)
}
```

Note that `cartesian_product` varies the rightmost range fastest - the output is in lexicographic order. That matches the natural reading order for grid coordinates and nested loop ordering.

---

### Q3: How do `adjacent` and `pairwise` work for sliding window patterns

`pairwise` (which is just `adjacent<2>`) and `adjacent<N>` give you overlapping windows over a sequence. These replace a lot of "compare element i with element i-1" loops that are easy to get wrong at the boundaries:

```cpp
#include <ranges>
#include <vector>
#include <print>
#include <cmath>
#include <string>
#include <algorithm>

namespace rv = std::ranges::views;

int main() {
    // pairwise (= adjacent<2>): consecutive pairs
    std::vector<int> data = {1, 4, 2, 8, 5, 7};

    // Compute differences between consecutive elements
    std::println("Pairwise differences:");
    for (auto [a, b] : data | rv::pairwise) {
        std::print("{:+d} ", b - a);
    }
    std::println("");  // +3 -2 +6 -3 +2

    // Check if sorted (all pairs satisfy a <= b)
    bool sorted = std::ranges::all_of(
        data | rv::pairwise,
        [](auto pair) { auto [a, b] = pair; return a <= b; }
    );
    std::println("Sorted: {}", sorted);  // false

    // adjacent<N>: sliding window of size N
    std::vector<double> prices = {100.0, 102.5, 101.0, 105.0, 103.5, 107.0};

    // 3-element moving average
    std::println("3-period moving average:");
    for (auto [a, b, c] : prices | rv::adjacent<3>) {
        std::println("  {:.2f}", (a + b + c) / 3.0);
    }
    // 101.17, 102.83, 103.17, 105.17

    // Detect local minima (sliding triple where middle < both neighbors)
    std::println("Local minima:");
    for (auto [i, triple] : prices | rv::adjacent<3> | rv::enumerate) {
        auto [prev, curr, next] = triple;
        if (curr < prev && curr < next) {
            std::println("  Index {}: {:.1f}", i + 1, curr);
        }
    }

    // pairwise_transform (= adjacent_transform<2>)
    // Compute derivatives (finite differences)
    std::vector<double> signal = {0.0, 1.0, 4.0, 9.0, 16.0, 25.0};
    auto derivative = signal
        | rv::pairwise_transform([](double a, double b) { return b - a; })
        | std::ranges::to<std::vector>();
    std::println("Derivative: {}", derivative);
    // [1.0, 3.0, 5.0, 7.0, 9.0]

    // Second derivative
    auto second_deriv = derivative
        | rv::pairwise_transform([](double a, double b) { return b - a; })
        | std::ranges::to<std::vector>();
    std::println("2nd derivative: {}", second_deriv);
    // [2.0, 2.0, 2.0, 2.0]  - constant (quadratic input)

    // adjacent_transform<3>: apply to triples
    auto smoothed = signal
        | rv::adjacent_transform<3>(
            [](double a, double b, double c) { return (a + b + c) / 3.0; })
        | std::ranges::to<std::vector>();
    std::println("Smoothed (window=3): {}", smoothed);
    // [1.667, 4.667, 9.667, 16.667]
}
```

The second derivative example is a nice illustration of lazy composition: you pipe the derivative result through another `pairwise_transform` to get the second derivative. Because views are lazy, none of the intermediate work materializes until you call `ranges::to`.

---

## Notes

- All views are in `<ranges>` (C++23) and compose with pipe `|` syntax.
- **`chunk_by(pred)`**: groups consecutive elements where `pred(a, b)` is true between adjacent elements. The predicate takes two arguments (current, next).
- **`stride(n)`**: takes every n-th element. O(1) advance for random-access ranges.
- **`cartesian_product`**: yields all combinations. The rightmost range varies fastest (lexicographic order). Size is the product of input sizes.
- **`zip`**: parallel iteration. Stops at the shortest range. Elements are tuples of references.
- **`zip_transform(f, ranges...)`**: fused zip+transform. More efficient than `zip | transform` as it avoids constructing intermediate tuples.
- **`adjacent<N>`**: sliding window of exactly N elements. Produces N - 1 fewer elements than the input. `pairwise` = `adjacent<2>`.
- **`pairwise_transform(f)`** and **`adjacent_transform<N>(f)`**: fused sliding window + function application.
- These views are lazy - they compute on iteration, not on construction.
- Feature-test macros: `__cpp_lib_ranges_chunk_by >= 202202L`, `__cpp_lib_ranges_stride >= 202207L`, `__cpp_lib_ranges_cartesian_product >= 202207L`, `__cpp_lib_ranges_zip >= 202110L`.
- Compilers: GCC 13+, Clang 17+, MSVC 19.34+ (varies by view).
