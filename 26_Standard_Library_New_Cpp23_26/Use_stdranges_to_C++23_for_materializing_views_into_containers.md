# Use `std::ranges::to` (C++23) for Materializing Views into Containers

**Category:** Standard Library New Cpp23 26  
**Standard:** C++23  
**Reference:** [cppreference — std::ranges::to](https://en.cppreference.com/w/cpp/ranges/to)  

---

## Topic Overview

`std::ranges::to<Container>()` converts any range or view pipeline into a concrete container. Before C++23, materializing a view required verbose construction: `std::vector<int>(view.begin(), view.end())`. This was particularly painful with nested containers or complex pipelines. `ranges::to` makes the conversion a single, composable call that works with pipe syntax.

The function supports two forms: `ranges::to<Container>(range)` (function call) and `range | ranges::to<Container>()` (pipe). It recursively converts nested ranges, respects allocators, and can deduce element types via CTAD.

| Pattern | Pre-C++23 | C++23 with `ranges::to` |
| --- | --- | --- |
| View -> vector | `vector<int>(v.begin(), v.end())` | `v \| ranges::to<vector>()` |
| Nested view -> vector of vector | Manual loop | `v \| ranges::to<vector<vector<int>>>()` |
| View -> set | `set<int>(v.begin(), v.end())` | `v \| ranges::to<set>()` |
| View -> string | `string(v.begin(), v.end())` | `v \| ranges::to<string>()` |
| View -> map | Construct from pair range | `v \| ranges::to<map>()` |

The pipeline picture is worth keeping in mind - every step before `ranges::to` is lazy, and `ranges::to` is the single point where all the lazy computation runs and the result is collected into memory:

```cpp
  ┌─────────────┐    filter    ┌──────────┐  transform  ┌──────────┐  ranges::to  ┌──────────────┐
  │ source range │────────────>│   view    │────────────>│   view   │─────────────>│  std::vector  │
  └─────────────┘             └──────────┘             └──────────┘             └──────────────┘
                              lazy                     lazy                     materialized
```

`ranges::to` also accepts additional arguments forwarded to the container's constructor (e.g., an allocator or a comparator for `std::set`).

---

## Self-Assessment

### Q1: How do you use `ranges::to` with pipe syntax and different container types

The most natural way to use `ranges::to` is at the end of a pipe chain. The type you specify in angle brackets tells it what container to build, and CTAD figures out the element type automatically when you use unqualified names like `std::vector` rather than `std::vector<int>`:

```cpp
#include <ranges>
#include <vector>
#include <set>
#include <string>
#include <map>
#include <iostream>
#include <algorithm>

namespace rv = std::ranges::views;

int main() {
    std::vector<int> data = {5, 3, 1, 4, 1, 5, 9, 2, 6, 5, 3};

    // View -> vector (with filter + transform)
    auto evens_squared = data
        | rv::filter([](int x) { return x % 2 == 0; })
        | rv::transform([](int x) { return x * x; })
        | std::ranges::to<std::vector>();
    // evens_squared: {16, 4, 36}

    // View -> set (automatic deduplication + sorting)
    auto unique_sorted = data
        | std::ranges::to<std::set>();
    // unique_sorted: {1, 2, 3, 4, 5, 6, 9}

    // View -> string
    auto greeting = rv::iota('A', 'A' + 26)
        | rv::take(5)
        | std::ranges::to<std::string>();
    // greeting: "ABCDE"

    // View -> map from pairs
    auto index_map = data
        | rv::enumerate
        | rv::take(5)
        | std::ranges::to<std::map<std::size_t, int>>();
    // index_map: {0:5, 1:3, 2:1, 3:4, 4:1}

    // Function-call syntax (equivalent)
    auto vec2 = std::ranges::to<std::vector>(
        data | rv::filter([](int x) { return x > 4; })
    );

    for (auto x : evens_squared) std::cout << x << ' ';
    std::cout << '\n';  // 16 4 36
    std::cout << greeting << '\n';  // ABCDE
}
```

Notice that converting to `std::set` automatically deduplicates and sorts - the container's own behavior takes care of that. You get deduplication for free just by choosing the right target container.

---

### Q2: How does `ranges::to` handle nested containers and recursive conversion

`ranges::to` is clever about nesting: if the target container's `value_type` is itself a container, and the source is a range of ranges, it converts the inner ranges automatically. This is what makes the "chunk a flat range into a matrix" pattern work without any manual loop:

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <iostream>

namespace rv = std::ranges::views;

int main() {
    // Nested range: chunk a flat range into groups
    auto matrix = rv::iota(1, 13)       // [1..12]
        | rv::chunk(4)                    // [[1,2,3,4],[5,6,7,8],[9,10,11,12]]
        | std::ranges::to<std::vector<std::vector<int>>>();

    // ranges::to recursively converts inner views to vector<int>
    for (const auto& row : matrix) {
        for (int val : row) std::cout << val << ' ';
        std::cout << '\n';
    }
    // Output:
    // 1 2 3 4
    // 5 6 7 8
    // 9 10 11 12

    // Split string into vector<string>
    std::string csv = "alpha,beta,gamma,delta";
    auto tokens = csv
        | rv::split(',')
        | rv::transform([](auto&& sub) {
            return sub | std::ranges::to<std::string>();
        })
        | std::ranges::to<std::vector<std::string>>();

    for (const auto& t : tokens) std::cout << '[' << t << "] ";
    std::cout << '\n';  // [alpha] [beta] [gamma] [delta]

    // Nested: vector<vector<string>> from split-then-chunk
    std::string data = "a,b,c,d,e,f";
    auto grouped = data
        | rv::split(',')
        | rv::transform([](auto&& s) {
            return s | std::ranges::to<std::string>();
        })
        | rv::chunk(2)
        | std::ranges::to<std::vector<std::vector<std::string>>>();

    // grouped: [["a","b"], ["c","d"], ["e","f"]]
}
```

The key insight here: `ranges::to` applies itself recursively - if the target container's `value_type` is itself a container and the source range is a range of ranges, the inner ranges are converted too. You don't need to manually loop over the inner elements.

---

### Q3: How do you pass allocators and custom comparators with `ranges::to`

Any extra arguments you pass to `ranges::to<C>(args...)` are forwarded directly to the container's constructor. This is how you inject an allocator or a custom comparator without abandoning the pipe syntax:

```cpp
#include <ranges>
#include <vector>
#include <set>
#include <memory>
#include <iostream>
#include <memory_resource>
#include <functional>

namespace rv = std::ranges::views;

int main() {
    auto nums = rv::iota(1, 11);  // [1..10]

    // Pass allocator to vector via additional argument
    std::pmr::monotonic_buffer_resource pool;
    std::pmr::polymorphic_allocator<int> alloc{&pool};

    auto pmr_vec = nums
        | rv::filter([](int x) { return x % 2 == 0; })
        | std::ranges::to<std::pmr::vector<int>>(alloc);
    // pmr_vec uses the pmr allocator

    // Pass comparator to set
    auto desc_set = nums
        | std::ranges::to<std::set<int, std::greater<>>>();
    // desc_set: {10, 9, 8, 7, 6, 5, 4, 3, 2, 1}

    for (int x : desc_set) std::cout << x << ' ';
    std::cout << '\n';

    // Reserve hint via ranges::to with vector
    // ranges::to calls reserve() if the source range has a known size
    auto large = rv::iota(0, 10000)
        | rv::transform([](int x) { return x * x; })
        | std::ranges::to<std::vector>();
    // The vector is allocated in one shot - no reallocation

    // Conversion chain: view -> deque -> reversed vector
    #include <deque>
    auto deq = rv::iota(1, 6)
        | std::ranges::to<std::deque>();
    auto rev_vec = deq
        | rv::reverse
        | std::ranges::to<std::vector>();
    // rev_vec: {5, 4, 3, 2, 1}

    std::cout << "PMR vec size: " << pmr_vec.size() << '\n';
    std::cout << "Large vec size: " << large.size() << '\n';
}
```

The `reserve()` optimization is worth knowing about: when the source range models `sized_range` (meaning it knows its size up front), `ranges::to` calls `.reserve()` on the target before filling it. That means a `transform` of a `vector` into another `vector` won't trigger any reallocations.

---

## Notes

- `std::ranges::to` is in `<ranges>`. Two syntaxes: `ranges::to<C>(rng, args...)` and `rng | ranges::to<C>(args...)`.
- Type deduction via CTAD: `ranges::to<vector>()` deduces the element type - no need to specify `vector<int>`.
- Additional arguments after the range are forwarded to the container constructor (allocators, comparators, bucket counts).
- If the source range models `sized_range`, `ranges::to` calls `.reserve()` on the target container, avoiding reallocation.
- Recursive conversion: if the target's `value_type` is a container and the source is a range-of-ranges, inner ranges are converted automatically.
- `ranges::to` works with any type constructible from an iterator pair or that has `push_back`/`insert`.
- Feature-test macro: `__cpp_lib_ranges_to_container >= 202202L`.
- Compilers: GCC 13+, Clang 17+, MSVC 19.34+ (VS 2022 17.4).
