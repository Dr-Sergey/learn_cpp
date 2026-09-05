# Use std::ranges::zip_transform (C++23) to transform multiple ranges

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/zip_transform_view>  

---

## Topic Overview

This file focuses on the **readability and range category** aspects of `zip_transform` compared to manual alternatives. While the sibling file covers element-wise operations, here we examine why `zip_transform` is superior to manual zipping and how range categories propagate.

### Why zip_transform is Better than Manual Alternatives

There are three common ways to combine elements from two ranges - and they are not equally good. The manual approaches either require awkward tuple unpacking or silently misbehave on mismatched sizes:

```cpp
// Manual: zip + transform with tuple unpacking
auto v1 = views::zip(a, b) | views::transform([](auto tup) {
    return std::get<0>(tup) + std::get<1>(tup);  // awkward
});

// Manual: index-based loop
for (size_t i = 0; i < a.size(); ++i)
    result.push_back(a[i] + b[i]);  // no bounds safety for mismatched sizes

// zip_transform: clean, safe, composable
auto v2 = views::zip_transform(std::plus{}, a, b);
```

The `zip_transform` version wins on every dimension: the function receives named parameters instead of a tuple, the size mismatch is handled automatically, and the whole thing stays lazy and composable.

### Range Category Rules

The range category of `zip_transform_view` is the **weakest** category among all input ranges. This is important because it determines which algorithms and operations are available on the result - for instance, whether you can use `[]` indexing or only forward iteration.

| Input Ranges | Result Category |
| --- | --- |
| All random access | Random access |
| Mix of random + bidirectional | Bidirectional |
| Any input-only | Input |
| Forward + bidirectional | Forward |

If the table feels abstract, the practical rule is: use `vector` inputs and you get full random access on the result. Mix in a `list` and you lose indexing. Mix in a `forward_list` and you lose backwards iteration too.

---

## Self-Assessment

### Q1: Compute the element-wise sum of two vectors using ranges::zip_transform

Let's start with a concrete business example - computing final prices after discounts - then show how `zip_transform` extends naturally to three ranges for invoice formatting.

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <functional>
#include <string>

int main() {
    std::vector<int> prices = {100, 200, 300, 400};
    std::vector<int> discounts = {10, 25, 50, 30};

    // === Element-wise subtraction: final prices ===
    auto final_prices = std::views::zip_transform(std::minus{}, prices, discounts);

    std::cout << "Final prices: ";
    for (int p : final_prices) std::cout << p << " ";
    std::cout << "\n";
    // 90 175 250 370

    // === String formatting from multiple ranges ===
    std::vector<std::string> items = {"Laptop", "Mouse", "Keyboard"};
    std::vector<double> costs = {999.99, 29.99, 79.99};
    std::vector<int> quantities = {1, 3, 2};

    auto invoice_lines = std::views::zip_transform(
        [](const std::string& item, double cost, int qty) {
            return item + " x" + std::to_string(qty)
                 + " = $" + std::to_string(cost * qty);
        },
        items, costs, quantities);

    std::cout << "\nInvoice:\n";
    for (const auto& line : invoice_lines)
        std::cout << "  " << line << "\n";

    // === vs manual approach (for comparison) ===
    // Manual zip + transform (more verbose, less clear):
    auto manual = std::views::zip(prices, discounts)
        | std::views::transform([](auto pair) {
            auto [p, d] = pair;
            return p - d;
          });

    std::cout << "\nManual zip+transform: ";
    for (int x : manual) std::cout << x << " ";
    std::cout << "\n";
    // Same result: 90 175 250 370

    return 0;
}
```

Notice that the three-range invoice example is still just one `zip_transform` call. The function receives `item`, `cost`, and `qty` directly as named parameters - no tuple index gymnastics required.

### Q2: Show that zip_transform is more readable than transform with a manually zipped input

This example puts all three approaches side by side so you can see exactly where the readability difference comes from. With four ranges the gap gets even wider.

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <functional>
#include <string>

int main() {
    std::vector<double> xs = {1.0, 2.0, 3.0, 4.0};
    std::vector<double> ys = {5.0, 6.0, 7.0, 8.0};

    // === Approach 1: Manual zip + transform (verbose) ===
    auto manual = std::views::zip(xs, ys)
        | std::views::transform([](auto t) {
            // Must destructure the tuple - visual noise
            auto [x, y] = t;
            return x * x + y * y;
          });

    // === Approach 2: zip_transform (clean) ===
    auto clean = std::views::zip_transform(
        [](double x, double y) { return x * x + y * y; },
        xs, ys);

    // Both produce the same result
    std::cout << "Manual: ";
    for (double v : manual) std::cout << v << " ";
    std::cout << "\n";

    std::cout << "Clean:  ";
    for (double v : clean) std::cout << v << " ";
    std::cout << "\n";
    // 26 40 58 80

    // === Approach 3 (pre-C++23): index loop (unsafe with mismatched sizes) ===
    std::cout << "Loop:   ";
    for (size_t i = 0; i < xs.size(); ++i)  // BUG if ys is smaller!
        std::cout << xs[i] * xs[i] + ys[i] * ys[i] << " ";
    std::cout << "\n";

    // === Readability comparison ===
    // zip_transform: function args are named (x, y) - self-documenting
    // zip + transform: must destructure tuple - extra cognitive load
    // index loop: no safety for size mismatch, no lazy evaluation

    // === Four-range example: only zip_transform stays readable ===
    std::vector<double> a = {1, 2}, b = {3, 4}, c = {5, 6}, d = {7, 8};
    auto expr = std::views::zip_transform(
        [](double a, double b, double c, double d) { return a*b + c*d; },
        a, b, c, d);
    std::cout << "\na*b + c*d: ";
    for (double v : expr) std::cout << v << " ";
    std::cout << "\n";
    // 38 56

    return 0;
}
```

The four-range case at the end is the stress test. With `zip + transform` you would be destructuring a tuple of four elements by index. With `zip_transform` you just write a lambda with four named parameters - the intent stays clear at a glance.

### Q3: Explain the range category of zip_transform based on its input range categories

The `static_assert` calls here are intentional - they are compile-time documentation of what category the view has, so you can see the "weakest wins" rule in action.

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <forward_list>
#include <ranges>
#include <functional>
#include <type_traits>

int main() {
    // === Range category = weakest of all input ranges ===

    // Both random_access -> result is random_access
    std::vector<int> v1 = {1, 2, 3};
    std::vector<int> v2 = {4, 5, 6};
    auto rr = std::views::zip_transform(std::plus{}, v1, v2);
    static_assert(std::ranges::random_access_range<decltype(rr)>);
    std::cout << "vector + vector = random_access_range: true\n";

    // vector (random) + list (bidirectional) -> bidirectional
    std::list<int> lst = {4, 5, 6};
    auto rb = std::views::zip_transform(std::plus{}, v1, lst);
    static_assert(std::ranges::bidirectional_range<decltype(rb)>);
    static_assert(!std::ranges::random_access_range<decltype(rb)>);
    std::cout << "vector + list = bidirectional_range: true\n";

    // vector (random) + forward_list (forward) -> forward
    std::forward_list<int> fwd = {4, 5, 6};
    auto rf = std::views::zip_transform(std::plus{}, v1, fwd);
    static_assert(std::ranges::forward_range<decltype(rf)>);
    static_assert(!std::ranges::bidirectional_range<decltype(rf)>);
    std::cout << "vector + forward_list = forward_range: true\n";

    // === Why this matters ===
    // Random access: supports [] indexing and .size()
    std::cout << "\nrr[1] = " << rr[1] << "\n";      // 7 (direct indexing!)
    std::cout << "rr size = " << std::ranges::distance(rr) << "\n";  // 3

    // Bidirectional: no [] but can iterate backwards
    // rb[1]  <- would NOT compile

    // Forward: can only iterate forward once direction
    // Cannot use with algorithms requiring bidirectional iterators

    // === Practical rule of thumb ===
    // If all inputs are vectors/arrays -> full random access
    // If any input is a list -> at best bidirectional
    // If any input is forward_list or a filter view -> at best forward

    return 0;
}
```

The reason this matters practically is that some algorithms - like `std::ranges::sort` - require random access iterators. If you pass a `zip_transform` result that involves a `list`, the compiler will refuse. The error message can be confusing if you don't know the "weakest wins" rule, so keep it in mind when mixing container types.

---

## Notes

- `zip_transform` function receives elements as **separate parameters**, not as a tuple - much cleaner than unpacking.
- The **range category** is determined at compile time based on input ranges. This affects which algorithms can use the result.
- `zip_transform` is **lazy** - the function is called only when elements are accessed.
- Prefer `zip_transform` over `zip | transform` for clarity and potential compile-time optimization.
- If you need to pipe additional views after `zip_transform`, the category of the result determines what views are available.
