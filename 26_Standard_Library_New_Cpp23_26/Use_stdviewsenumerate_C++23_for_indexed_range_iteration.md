# Use `std::views::enumerate` (C++23) for Indexed Range Iteration

**Category:** Standard Library New Cpp23 26  
**Standard:** C++23  
**Reference:** [cppreference — std::views::enumerate](https://en.cppreference.com/w/cpp/ranges/enumerate_view)  

---

## Topic Overview

`std::views::enumerate` produces a view of `(index, element)` pairs from any input range - analogous to Python's `enumerate()`. The index is a `range_difference_t` (typically `ptrdiff_t`), starting at zero. Combined with structured bindings, it replaces the manual counter pattern that has been a source of off-by-one errors and type mismatches for decades.

Before C++23, the common patterns for indexed iteration were fragile in different ways. Each approach has a real drawback:

| Pattern | Problems |
| ------- | -------- |
| `for (int i = 0; i < vec.size(); ++i)` | Signed/unsigned mismatch warning; manual indexing |
| `for (size_t i = 0; ...)` | Correct type but verbose; tempting to use `int` |
| Counter variable alongside range-for | Extra mutable variable; easy to forget increment |
| `std::distance(begin, it)` | O(n) for non-random-access; verbose |
| `views::enumerate` (C++23) | Zero overhead; correct type; structured bindings |

Here's the conceptual picture - `enumerate` wraps each element with its zero-based position, producing pairs you can destructure immediately:

```cpp
Source range:    [ "alpha", "beta", "gamma", "delta" ]
                      |        |        |        |
views::enumerate      v        v        v        v
                  (0,"alpha") (1,"beta") (2,"gamma") (3,"delta")
                      |
            auto [idx, val] = ...   <- structured binding
```

`enumerate` is lazy - it produces elements on demand and composes with other views in a pipeline. The returned elements are tuple-like with `.index()` and `.value()` accessors, or can be destructured via structured bindings.

---

## Self-Assessment

### Q1: How does `views::enumerate` replace manual index tracking

The difference in readability is immediate, but there's also a correctness benefit: the index type is `range_difference_t` (a signed type), which eliminates the classic signed/unsigned comparison warning you get with `size_t` or `int` loops. You also can't forget to increment the counter, because there is no counter:

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <print>

int main() {
    std::vector<std::string> names = {"Alice", "Bob", "Charlie", "Diana"};

    // OLD: manual counter
    // size_t idx = 0;
    // for (const auto& name : names) {
    //     std::println("{}: {}", idx, name);
    //     ++idx;
    // }

    // OLD: index-based loop
    // for (size_t i = 0; i < names.size(); ++i) {
    //     std::println("{}: {}", i, names[i]);
    // }

    // C++23: views::enumerate
    for (auto [idx, name] : std::views::enumerate(names)) {
        std::println("{}: {}", idx, name);
    }
    // 0: Alice
    // 1: Bob
    // 2: Charlie
    // 3: Diana

    // Pipe syntax
    for (auto [i, name] : names | std::views::enumerate) {
        // name is a reference to the original element - modifiable
        if (i == 2) name = "CHARLIE";
    }
    std::println("Modified: {}", names);
    // Modified: ["Alice", "Bob", "CHARLIE", "Diana"]

    // Index type is range_difference_t, not size_t
    for (auto [i, _] : names | std::views::enumerate) {
        // i is ptrdiff_t (signed) - no unsigned comparison issues
        static_assert(std::is_signed_v<decltype(i)>);
    }
}
```

The key insight: the element reference in `enumerate` is a true reference to the original range's element - modifications through it are reflected in the source container. You're not working with copies.

---

### Q2: How does `views::enumerate` compose with other range adaptors

Because `enumerate` is a lazy view, it slots naturally into range pipelines. You can filter on the index, transform the pairs, take a subset, and collect - all without allocating intermediate containers. Here are several practical composition patterns:

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <algorithm>
#include <print>

namespace rv = std::ranges::views;

int main() {
    std::vector<int> data = {10, 25, 3, 47, 8, 33, 12, 50};

    // Find indices of elements matching a predicate
    auto large_indices = data
        | rv::enumerate
        | rv::filter([](auto pair) {
            auto [idx, val] = pair;
            return val > 20;
        })
        | rv::transform([](auto pair) {
            return pair.index();   // .index() accessor
        })
        | std::ranges::to<std::vector>();

    std::println("Indices where val > 20: {}", large_indices);
    // [1, 3, 5, 7]

    // Enumerate + take: first N with indices
    std::vector<std::string> log_lines = {
        "Starting server...",
        "Listening on port 8080",
        "Connection from 10.0.0.1",
        "Request: GET /api/users",
        "Response: 200 OK",
    };

    std::println("First 3 lines:");
    for (auto [lineno, text] : log_lines | rv::enumerate | rv::take(3)) {
        std::println("  L{}: {}", lineno + 1, text);
    }

    // Enumerate reversed range
    std::vector<char> chars = {'a', 'b', 'c', 'd'};
    std::println("Reversed with original indices:");
    for (auto [i, ch] : chars | rv::reverse | rv::enumerate) {
        // Note: i is the enumerate index (0-based after reverse)
        std::println("  pos {}: '{}'", i, ch);
    }
    // pos 0: 'd'
    // pos 1: 'c'
    // pos 2: 'b'
    // pos 3: 'a'

    // Enumerate + zip for parallel iteration
    std::vector<std::string> keys = {"x", "y", "z"};
    std::vector<double> values = {1.1, 2.2, 3.3};

    for (auto [i, kv] : rv::zip(keys, values) | rv::enumerate) {
        auto [key, val] = kv;
        std::println("  [{}] {} = {:.1f}", i, key, val);
    }
}
```

Notice the `lineno + 1` in the log example - a small but common adjustment when you want 1-based line numbers in output. `enumerate` always starts at zero; adding 1 at the display site keeps the data model clean.

---

### Q3: What are the performance characteristics and edge cases of `views::enumerate`

Understanding what category of range `enumerate` produces matters for knowing what operations are available. If the source is random-access, the enumerate view is too - you get `operator[]` and `.size()`. If the source is only forward, so is the view. No overhead is added: `enumerate` is just a counter alongside the base iterator:

```cpp
#include <ranges>
#include <vector>
#include <list>
#include <print>
#include <forward_list>
#include <type_traits>

namespace rv = std::ranges::views;

// enumerate preserves the source range's category:
// random_access_range -> random_access enumerate view
// bidirectional_range -> bidirectional enumerate view
// forward_range       -> forward enumerate view
// input_range         -> input enumerate view

void range_category_examples() {
    std::vector<int> vec = {1, 2, 3};
    std::list<int> lst = {1, 2, 3};
    std::forward_list<int> fwd = {1, 2, 3};

    auto ev = vec | rv::enumerate;
    auto el = lst | rv::enumerate;
    auto ef = fwd | rv::enumerate;

    // Random access: supports [n] and size()
    static_assert(std::ranges::random_access_range<decltype(ev)>);
    auto [i, v] = ev[1];  // direct subscript
    std::println("ev[1] = ({}, {})", i, v);  // (1, 2)
    std::println("size = {}", ev.size());     // 3

    // Bidirectional: no subscript
    static_assert(std::ranges::bidirectional_range<decltype(el)>);
    // el[1];  // ERROR - not random access

    // Forward: forward only
    static_assert(std::ranges::forward_range<decltype(ef)>);
}

// Empty range
void empty_range() {
    std::vector<int> empty;
    for (auto [i, v] : empty | rv::enumerate) {
        // Body never executes - safe
        std::println("{}: {}", i, v);
    }
}

// Performance: zero overhead
// enumerate is a thin wrapper - just an incrementing counter
// alongside the base iterator. No extra allocation, no copying.
void performance_demo() {
    std::vector<int> data(1'000'000);
    std::ranges::iota(data, 0);

    long long sum = 0;
    for (auto [i, val] : data | rv::enumerate) {
        sum += i * val;  // compiler optimizes to single loop
    }
    std::println("Sum of i*val: {}", sum);
}

// Enumerate with mutable access
void mutable_access() {
    std::vector<int> data = {10, 20, 30, 40, 50};

    // Set each element to its index * 100
    for (auto [i, val] : data | rv::enumerate) {
        val = static_cast<int>(i) * 100;
    }
    std::println("After mutation: {}", data);
    // [0, 100, 200, 300, 400]
}

int main() {
    range_category_examples();
    empty_range();
    performance_demo();
    mutable_access();
}
```

The "zero overhead" claim is worth believing here: the resulting assembly for an enumerate loop over a vector is identical to a hand-written index loop. The abstraction is genuinely free.

---

## Notes

- `std::views::enumerate` is in `<ranges>` (C++23). Produces `(index, reference)` pairs.
- The index type is `range_difference_t<R>` - signed, typically `ptrdiff_t`. No unsigned comparison warnings.
- Elements are returned as a tuple-like type supporting `.index()`, `.value()`, and structured bindings `auto [i, v]`.
- The value in the pair is a **reference** to the original element - mutations propagate to the source range.
- The resulting view preserves the range category of the source (random-access, bidirectional, etc.).
- Zero overhead: just an incrementing counter alongside iteration. No extra allocation.
- Composes naturally with `filter`, `transform`, `take`, `reverse`, `zip`, and all other views.
- Feature-test macro: `__cpp_lib_ranges_enumerate >= 202302L`.
- Compilers: GCC 13+, Clang 17+, MSVC 19.36+ (VS 2022 17.6).
