# Use range adaptors: filter, transform, take, drop, and their composition

**Category:** Ranges Cpp20  
**Standard:** C++20, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges#Range_adaptors>  

---

## Topic Overview

Range adaptors are lightweight, **lazy** wrappers around a source range. They transform, filter, or truncate elements on demand - no intermediate containers are created. The key mental model is that an adaptor is just a description of what to do, not a thing that has already done it.

### Adaptor Quick-Reference

| Adaptor | Header | What it does |
| --- | --- | --- |
| `views::filter(pred)` | `<ranges>` | Keep only elements satisfying `pred` |
| `views::transform(fn)` | `<ranges>` | Apply `fn` to every element |
| `views::take(n)` | `<ranges>` | Yield at most the first `n` elements |
| `views::drop(n)` | `<ranges>` | Skip first `n`, yield the rest |
| `views::take_while(pred)` | `<ranges>` | Yield elements while `pred` is true |
| `views::drop_while(pred)` | `<ranges>` | Skip elements while `pred` is true |
| `views::reverse` | `<ranges>` | Reverse (requires bidirectional) |
| `views::keys` / `views::values` | `<ranges>` | Project 1st / 2nd element of pair-like |

### Composition with `|`

Adaptors compose via the **pipe operator** `|`, reading left-to-right:

```cpp
source | filter(pred) | transform(fn) | take(n)
```

This is equivalent to `take(n, transform(fn, filter(pred, source)))` but far more readable. The left-to-right reading order is intentional: you see the data flow in the same direction you think about it.

### Lazy Evaluation Model

```cpp
Source:  1  2  3  4  5  6  7  8  9  10  ...  (infinite iota)
           |  filter(even)  -> 2  4  6  8  10  ...
              |  transform(square)  -> 4  16  36  64  100  ...
                 |  take(3)  -> 4  16  36  STOP
```

No element beyond the 6 (source) is ever touched. Each element is "pulled" from the pipeline one at a time. This is what makes composing views with an infinite source (`views::iota`) safe and practical.

### Materializing with `ranges::to<>` (C++23)

Views are **non-owning**. When you need an owning container, materialize:

```cpp
auto vec = source | views::filter(pred) | std::ranges::to<std::vector>();
```

Pre-C++23 alternative: construct from the view's iterator pair.

---

## Self-Assessment

### Q1: Write a pipeline using `|` that filters even numbers, squares them, and takes the first 5

This example uses an infinite source (`views::iota(1)`), which is completely fine because `take(5)` stops the pipeline early. Only the first five even numbers (2 through 10) are ever examined from the infinite sequence.

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    namespace rv = std::ranges::views;

    auto source = rv::iota(1);  // infinite: 1, 2, 3, ...

    auto pipeline = source
        | rv::filter([](int n) { return n % 2 == 0; })   // 2, 4, 6, 8, 10, ...
        | rv::transform([](int n) { return n * n; })       // 4, 16, 36, 64, 100, ...
        | rv::take(5);                                      // first 5 only

    for (int x : pipeline)
        std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output: 4 16 36 64 100
```

Notice that `pipeline` is just a view object - no numbers have been computed yet. The computation happens inside the `for` loop, one element at a time. No vector or allocation is needed anywhere in this process.

### Q2: Explain lazy evaluation: no computation occurs until the pipeline is iterated

**Lazy evaluation** means that calling `source | filter(pred) | transform(fn)` constructs a **view object** (a small struct holding iterators and function objects) but performs **zero computation**. Work happens only when you iterate:

| Action | What happens |
| --- | --- |
| `auto v = src \| filter(f) \| transform(g);` | Constructs view descriptors - `f` and `g` are **never called** |
| `auto it = v.begin();` | Advances source until `f` returns `true` for the first element, then applies `g` |
| `++it;` | Advances source to the **next** element passing `f`, applies `g` |
| Never iterate `v` | `f` and `g` are **never invoked** at all |

Here's a proof using side-effect counters to make the lazy evaluation visible:

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    int filter_calls = 0;
    int transform_calls = 0;

    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    auto pipeline = data
        | std::views::filter([&](int n) { ++filter_calls; return n % 2 == 0; })
        | std::views::transform([&](int n) { ++transform_calls; return n * n; })
        | std::views::take(2);

    std::cout << "Before iteration: filter=" << filter_calls
              << " transform=" << transform_calls << '\n';

    for (int x : pipeline)
        std::cout << x << ' ';
    std::cout << '\n';

    std::cout << "After iteration: filter=" << filter_calls
              << " transform=" << transform_calls << '\n';
}
// Expected output:
// Before iteration: filter=0 transform=0
// 4 16
// After iteration: filter=4 transform=2
```

Before the loop, both counters are zero - the pipeline exists but hasn't done anything. During iteration, `filter` is called 4 times (for elements 1, 2, 3, 4) and `transform` only 2 times (for elements 2 and 4), because `take(2)` stopped the pipeline after finding two matches. Elements 5 through 10 were never examined.

### Q3: Show a case where materializing a view into a vector with `ranges::to` (C++23) is necessary

The reason this matters is lifetimes. If the source range is a temporary, any view over it becomes a dangling reference the moment the temporary is destroyed. Materializing with `ranges::to` creates an independent, owning copy of the filtered results before the temporary disappears.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

std::vector<std::string> get_words() {
    return {"hello", "world", "foo", "bar", "baz", "quux"};
}

int main() {
    // The source is a temporary - the view would dangle!
    // auto bad = get_words() | std::views::filter(  // dangling ref to destroyed vector
    //     [](const std::string& s) { return s.size() > 3; });

    // Materializing immediately captures the filtered results:
    auto long_words = get_words()
        | std::views::filter([](const std::string& s) { return s.size() > 3; })
        | std::ranges::to<std::vector>();

    // Now long_words is an owning vector<string> that's safe to use anywhere
    std::ranges::sort(long_words);
    for (const auto& w : long_words)
        std::cout << w << ' ';
    std::cout << '\n';
}
// Expected output: hello quux world
```

Materialization is also necessary when you need random-access on a filtered view (filter only gives you bidirectional), or when you need to iterate the results multiple times and the source is a single-pass range.

**When materialization is necessary:**

| Scenario | Why you must materialize |
| --- | --- |
| Source is a temporary (rvalue) | View would hold a dangling reference |
| You need random-access (e.g., sort) on a filter view | filter_view is only bidirectional |
| You need to iterate the result multiple times | Some views (e.g., over input ranges) are single-pass |
| You need to pass ownership to another function | Views are non-owning by design |

---

## Notes

- **`drop(n)` + `take(m)`** is a common idiom for "skip `n`, then take `m`" - equivalent to a slice.
- Adaptors can be **pre-composed** into reusable objects: `auto even_squares = rv::filter(is_even) | rv::transform(square);` - then `data | even_squares`.
- `views::filter` caches `begin()` on first call (O(N) once), so repeating `begin()` is O(1) afterwards.
- Pre-C++23 materialization: `std::vector<int> v(pipeline.begin(), pipeline.end());` or use `ranges::copy` + `back_inserter`.
