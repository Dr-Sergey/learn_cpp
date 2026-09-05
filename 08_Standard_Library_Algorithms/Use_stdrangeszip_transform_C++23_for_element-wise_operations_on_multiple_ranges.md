# Use std::ranges::zip_transform (C++23) for element-wise operations on multiple ranges

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/zip_transform_view>  

---

## Topic Overview

`std::views::zip_transform` (C++23) is the clean way to apply a function element-wise across multiple ranges at once. Think of it as a generalized "map" over several input sequences combined into one step - it zips the ranges together and applies your transform in a single, lazy operation.

If you have ever written `a[i] + b[i]` in a loop, this is that pattern, but composable and bounds-safe:

```text
a:               [1,    2,    3   ]
b:               [10,   20,   30  ]
zip_transform(+): [11,   22,   33  ]
```

### Syntax

The call puts the function first, followed by as many input ranges as you need:

```cpp
#include <ranges>

// Apply fn to elements from r1, r2, ... in parallel
auto view = std::views::zip_transform(fn, r1, r2, ...);
```

### zip + transform vs zip_transform

Before C++23 you had to zip first and then manually unpack the tuple inside a transform. With `zip_transform` that ceremony disappears:

```cpp
// Two steps: zip then transform
auto v1 = std::views::zip(a, b) | std::views::transform([](auto pair) {
    auto [x, y] = pair;
    return x + y;
});

// Single step: zip_transform (cleaner)
auto v2 = std::views::zip_transform(std::plus{}, a, b);
```

The second version is shorter, easier to read, and passes the elements directly as separate function arguments instead of bundling them into a tuple.

---

## Self-Assessment

### Q1: Add two vectors element-wise using zip_transform and a plus functor

Here we try `zip_transform` with the standard `std::plus` functor, then with `std::multiplies`, then with a custom lambda - and finally materialize the result and compute a dot product on top of it.

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <functional>

int main() {
    std::vector<int> a = {1, 2, 3, 4, 5};
    std::vector<int> b = {10, 20, 30, 40, 50};

    // === Element-wise addition ===
    auto sums = std::views::zip_transform(std::plus{}, a, b);

    std::cout << "a + b: ";
    for (int x : sums) std::cout << x << " ";
    std::cout << "\n";
    // 11 22 33 44 55

    // === Element-wise multiplication ===
    auto products = std::views::zip_transform(std::multiplies{}, a, b);
    std::cout << "a * b: ";
    for (int x : products) std::cout << x << " ";
    std::cout << "\n";
    // 10 40 90 160 250

    // === Custom lambda ===
    auto weighted = std::views::zip_transform(
        [](int x, int y) { return x * 3 + y; }, a, b);
    std::cout << "3a+b:  ";
    for (int x : weighted) std::cout << x << " ";
    std::cout << "\n";
    // 13 26 39 52 65

    // === Materialize with ranges::to ===
    auto result = std::views::zip_transform(std::plus{}, a, b)
                | std::ranges::to<std::vector>();
    // result is std::vector<int>{11, 22, 33, 44, 55}

    // === Dot product using zip_transform + fold_left ===
    auto dot = std::ranges::fold_left(
        std::views::zip_transform(std::multiplies{}, a, b),
        0, std::plus{});
    std::cout << "Dot product: " << dot << "\n";  // 550

    return 0;
}
```

The view is lazy - nothing is computed until you iterate. Piping into `ranges::to` forces evaluation and stores the results in a `vector`. The dot product example shows how neatly `zip_transform` composes with `fold_left` for a one-expression map-reduce.

### Q2: Show that zip_transform stops at the shortest input range

One of the nicest safety properties of `zip_transform` is that it never reads past the end of any input. It automatically limits itself to the shortest range, so mismatched sizes are handled gracefully instead of causing undefined behavior.

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <functional>
#include <string>

int main() {
    std::vector<int> short_vec = {1, 2};
    std::vector<int> long_vec = {10, 20, 30, 40, 50};

    // === Stops at shortest ===
    auto result = std::views::zip_transform(std::plus{}, short_vec, long_vec);

    std::cout << "Mismatched sizes: ";
    for (int x : result) std::cout << x << " ";
    std::cout << "\n";
    // 11 22  - only 2 elements (min of 2, 5)

    std::cout << "Result size: " << std::ranges::distance(result) << "\n";  // 2

    // === Three ranges of different sizes ===
    std::vector<int> r1 = {1, 2, 3, 4};
    std::vector<int> r2 = {10, 20};
    std::vector<int> r3 = {100, 200, 300};

    auto three_sum = std::views::zip_transform(
        [](int a, int b, int c) { return a + b + c; },
        r1, r2, r3);

    std::cout << "Three ranges: ";
    for (int x : three_sum) std::cout << x << " ";
    std::cout << "\n";
    // 111 222  - stops at r2 (size 2, the shortest)

    // === Practical: safe column-wise operations ===
    std::vector<std::string> headers = {"Name", "Age", "City"};
    std::vector<std::string> values = {"Alice", "30"};  // missing City!

    auto pairs = std::views::zip_transform(
        [](const std::string& h, const std::string& v) {
            return h + "=" + v;
        }, headers, values);

    for (auto& s : pairs) std::cout << s << " ";
    std::cout << "\n";
    // Name=Alice Age=30  - safely stops, no crash on missing City

    return 0;
}
```

The "missing City" example makes this practical: if your data has ragged rows you get partial results instead of a crash, with no special-case code on your part.

### Q3: Use zip_transform with three ranges to compute a linear combination: a*x + b*y + c

Three-range `zip_transform` is where the advantage over manual indexing really shows. A loop over three indices gives you no safety and no composability; this gives you both.

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <cmath>
#include <functional>
#include <iterator>

int main() {
    // === Linear combination: a*x + b ===
    std::vector<double> a = {2.0, 3.0, 1.0, 4.0};
    std::vector<double> x = {1.0, 2.0, 3.0, 4.0};
    std::vector<double> b = {0.5, 1.0, 1.5, 2.0};

    // Compute a[i]*x[i] + b[i] for each i
    auto linear = std::views::zip_transform(
        [](double ai, double xi, double bi) { return ai * xi + bi; },
        a, x, b);

    std::cout << "a*x + b: ";
    for (double v : linear) std::cout << v << " ";
    std::cout << "\n";
    // 2.5, 7.0, 4.5, 18.0

    // === RGB color blending ===
    std::vector<int> r1 = {255, 0, 128};
    std::vector<int> g1 = {0, 255, 128};
    std::vector<int> b1 = {0, 0, 128};

    std::vector<int> r2 = {0, 128, 64};
    std::vector<int> g2 = {128, 0, 64};
    std::vector<int> b2 = {255, 128, 64};

    double alpha = 0.5;

    // Blend each channel: lerp(c1, c2, alpha)
    auto blend = [alpha](int c1, int c2) -> int {
        return static_cast<int>(c1 * (1.0 - alpha) + c2 * alpha);
    };

    auto blended_r = std::views::zip_transform(blend, r1, r2);
    auto blended_g = std::views::zip_transform(blend, g1, g2);
    auto blended_b = std::views::zip_transform(blend, b1, b2);

    std::cout << "\nBlended RGB:\n";
    for (size_t i = 0; i < 3; ++i) {
        auto it_r = std::ranges::begin(blended_r);
        auto it_g = std::ranges::begin(blended_g);
        auto it_b = std::ranges::begin(blended_b);
        std::advance(it_r, i);
        std::advance(it_g, i);
        std::advance(it_b, i);
        std::cout << "  Pixel " << i << ": (" << *it_r << ", " << *it_g << ", " << *it_b << ")\n";
    }

    // === Euclidean distance between two point vectors ===
    std::vector<double> p1 = {1.0, 2.0, 3.0};
    std::vector<double> p2 = {4.0, 6.0, 3.0};

    auto sq_diff = std::views::zip_transform(
        [](double a, double b) { return (a - b) * (a - b); },
        p1, p2);

    double dist = std::sqrt(std::ranges::fold_left(sq_diff, 0.0, std::plus{}));
    std::cout << "\nDistance: " << dist << "\n";  // 5.0

    return 0;
}
```

Notice how the Euclidean distance example reduces to a single readable expression: `zip_transform` the squared differences, then `fold_left` to sum them, then `sqrt`. That chain is far more self-documenting than the equivalent index loop.

---

## Notes

- **C++23 required.** Compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- `zip_transform` is equivalent to `zip(...) | transform(apply(fn))` but more concise and potentially more efficient.
- **Lazy:** No computation happens until the view is iterated.
- Always stops at the **shortest** input range - safe by design.
- The function receives one element from each range as separate arguments (not as a tuple).
- Combines well with `ranges::to` and `fold_left` for materializing or reducing results.
