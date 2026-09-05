# Use std::views pipeline as a functional transformation chain

**Category:** Functional Programming Patterns  
**Standard:** C++20, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges>  

---

## Topic Overview

`std::views` provides lazy, composable transformations over ranges - the C++ equivalent of functional map/filter/reduce pipelines. If you've written data processing code using loops and intermediate vectors, views let you express the same logic as a pipeline that reads naturally from left to right, with no intermediate allocations and no extra passes over the data.

The key word is **lazy**: views describe what to do, not when to do it. The actual computation happens element by element, only as you iterate. This makes pipelines naturally efficient even for large datasets.

### Basic Pipeline

Here's a straightforward pipeline that filters, transforms, and limits a range. Notice how you read it left to right exactly as you'd describe the operation in English - "take the data, keep only the even numbers, square them, take the first three":

```cpp
#include <ranges>
#include <vector>
#include <iostream>
#include <string>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Functional pipeline: filter -> transform -> take
    auto result = data
        | std::views::filter([](int x) { return x % 2 == 0; })  // evens
        | std::views::transform([](int x) { return x * x; })    // square
        | std::views::take(3);                                    // first 3

    for (int x : result)
        std::cout << x << " ";  // 4 16 36
    std::cout << "\n";

    // LAZY: nothing computed until iteration
    // No intermediate vectors created
}
```

The laziness is important here. When you iterate `result`, the pipeline processes elements one at a time from left to right. Once `take(3)` has seen three elements, iteration stops - the filter and transform never even look at elements 7, 8, 9, or 10. With a vector-based approach you'd filter everything first, transform everything, then take three: three full passes over potentially the entire dataset.

### Complex Pipeline with C++23

Views compose well with more complex data. Here's a realistic example using a struct, a projection onto a member, and C++23's `enumerate` view to get index-value pairs:

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <iostream>

struct Employee {
    std::string name;
    std::string department;
    double salary;
};

void report(const std::vector<Employee>& employees) {
    // C++23: chunk_by, zip_transform, enumerate
    auto engineering_salaries = employees
        | std::views::filter([](const auto& e) {
            return e.department == "Engineering";
          })
        | std::views::transform(&Employee::salary)
        | std::views::filter([](double s) { return s > 80000; });

    // C++23 enumerate:
    for (auto [i, salary] : engineering_salaries | std::views::enumerate) {
        std::cout << i << ": $" << salary << "\n";
    }
}
```

The `&Employee::salary` is a pointer-to-member used as a projection - it tells `transform` to extract the `salary` field from each employee. This is a clean alternative to a lambda when you just want one field.

### Composing Named Pipelines

Because views are just values, you can store them in variables and compose them. This lets you build reusable pipeline stages that you can mix and match:

```cpp
#include <ranges>

// Create reusable pipeline stages as variables
inline constexpr auto positive = std::views::filter([](auto x) { return x > 0; });
inline constexpr auto doubled = std::views::transform([](auto x) { return x * 2; });
inline constexpr auto first_five = std::views::take(5);

// Compose into a single pipeline
inline constexpr auto process = positive | doubled | first_five;

// Use:
// for (auto x : data | process) { ... }
```

Named pipelines like this read like domain vocabulary. Instead of inlining the lambda logic everywhere, you give the stages meaningful names and compose them. The pipeline `data | process` is self-documenting.

---

## Self-Assessment

### Q1: Why are views lazy and why does it matter

Views don't compute anything until iterated. `data | filter(f) | transform(g) | take(3)` processes only 3 elements through the entire pipeline - it doesn't filter all elements first, then transform all results, then take 3. This is O(1) extra memory (no intermediate containers) and may process far fewer elements depending on how early `take` or other short-circuiting views are satisfied. For large datasets or expensive predicates, this can be a dramatic speedup.

### Q2: What is the difference between views and actions

Views are lazy and non-owning - they describe a transformation but don't store data or modify the source range. Actions (available in the range-v3 library) are eager and mutating - `sort | unique` would sort in-place and remove duplicates. The C++ standard only has views; if you need eager in-place algorithms, use `std::ranges::sort`, `std::ranges::unique`, and similar algorithm calls directly.

### Q3: Show how to convert a view back to a container

Views are non-owning and lazy, so if you need to store the results, you have to materialize them into a container. C++23 makes this clean with `ranges::to`:

```cpp
#include <ranges>
#include <vector>

auto view = std::views::iota(1, 10)
    | std::views::transform([](int x) { return x * x; });

// C++23: ranges::to
auto vec = view | std::ranges::to<std::vector>();

// C++20 workaround:
std::vector<int> vec20(view.begin(), view.end());
```

The C++20 workaround using `begin`/`end` is verbose but equivalent. Once you have `ranges::to` in C++23, materialization is as clean as the pipeline itself.

---

## Notes

- Views compose with the `|` operator - read left to right, exactly like Unix pipes.
- `std::views::iota`, `std::views::repeat`, and `std::views::zip` are generators that produce values, not just filters that select from existing data.
- C++23 adds `ranges::to` for materializing views into containers, which was notably missing from C++20.
- Views don't own data - always make sure the source range outlives the view, or you'll have a dangling reference.
