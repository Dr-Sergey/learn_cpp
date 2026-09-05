# Use views::concat (C++26) to lazily concatenate multiple ranges

**Category:** Ranges Cpp20  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/concat_view>  

---

## Topic Overview

`views::concat` (C++26, P2542) lazily concatenates **two or more ranges** into a single range, iterating through each in sequence without copying any elements.

The motivation is simple: you often have several containers holding related data, and you want to treat them as one sequence for an algorithm or a loop - without the overhead of building a combined container.

### Before concat: The Problem

The old-fashioned approach allocates memory and copies every element:

```cpp
// Eager approach - copies all elements:
std::vector<int> combined;
combined.insert(combined.end(), a.begin(), a.end());
combined.insert(combined.end(), b.begin(), b.end());
combined.insert(combined.end(), c.begin(), c.end());
// Allocates memory, copies every element
```

### With views::concat

With `views::concat` you get a view that iterates the original ranges back-to-back, with zero allocation:

```cpp
// Lazy approach - zero copies, zero allocation:
auto combined = std::views::concat(a, b, c);
// Iterates a, then b, then c seamlessly
```

### Iterator Category Rules

The resulting iterator category is the **weakest** (most restrictive) among all input ranges. The reason this trips people up is that the concat view must be able to switch between ranges at each boundary - and it can only guarantee operations that every single input supports.

| Input ranges | concat iterator category |
| --- | --- |
| All random_access | random_access (if all sized + common) |
| All bidirectional | bidirectional |
| All forward | forward |
| Any input_only | input |
| Mixed (e.g., bidi + forward) | forward (the weakest) |

### Reference Type

The reference type of the concat view is the **common reference** of all input ranges' reference types. All ranges must have a compatible value type. In practice this usually means you're concatenating ranges of the same element type.

### Pre-C++26 Alternatives

| Alternative | Limitation |
| --- | --- |
| `views::join` on a range of ranges | Requires ranges stored in a container; heterogeneous types are harder |
| Boost.Range `join` | External dependency |
| Manual iterator adapter | Verbose, error-prone |

---

## Self-Assessment

### Q1: Concatenate three `std::vector<int>`s into a single lazy range using `views::concat`

Here is the basic usage - three separate vectors, one seamless view over all of them.

```cpp
#include <iostream>
#include <ranges>
#include <vector>
#include <functional>

int main() {
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20};
    std::vector<int> c = {100, 200, 300, 400};

    // Lazy concatenation - no copies
    auto all = std::views::concat(a, b, c);

    std::cout << "Concatenated: ";
    for (int x : all)
        std::cout << x << ' ';
    std::cout << '\n';

    // Works with range algorithms
    auto sum = std::ranges::fold_left(all, 0, std::plus{});
    std::cout << "Sum: " << sum << '\n';

    // Size is available if all inputs are sized
    std::cout << "Total elements: " << std::ranges::size(all) << '\n';
}
// Expected output:
// Concatenated: 1 2 3 10 20 100 200 300 400
// Sum: 836
// Total elements: 9
```

Because all three inputs are `vector<int>` (random-access and sized), the concat view is also sized - it just adds up the individual sizes. Any range algorithm that works on `int`s will work on `all` exactly as if it were one flat container.

- `views::concat(a, b, c)` creates a view that first yields all elements of `a`, then `b`, then `c`.
- No memory is allocated; the view stores references to the original containers.
- Since all three are `vector<int>` (random-access + sized), the concat view is also sized.
- `ranges::fold_left` (C++23) computes the sum across all concatenated elements.

### Q2: Show that `views::concat` avoids copying elements unlike `std::vector::insert`

This example makes the non-owning, zero-copy nature concrete. Writing through the view modifies the original container - the view is just a lens, not a copy.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    // Using strings to detect copies (moved-from strings become empty)
    std::vector<std::string> a = {"hello", "world"};
    std::vector<std::string> b = {"foo", "bar"};

    // Approach 1: vector::insert (COPIES)
    std::vector<std::string> combined;
    combined.insert(combined.end(), a.begin(), a.end());
    combined.insert(combined.end(), b.begin(), b.end());
    std::cout << "After insert, a[0] = \"" << a[0] << "\" (still valid - was copied)\n";

    // Approach 2: views::concat (NO COPIES)
    auto lazy = std::views::concat(a, b);
    std::cout << "concat view: ";
    for (const auto& s : lazy)
        std::cout << s << ' ';
    std::cout << '\n';

    // Modify through the view - changes the original!
    auto it = std::ranges::begin(lazy);
    *it = "HELLO";  // modifies a[0] directly
    std::cout << "After modifying via view, a[0] = \"" << a[0] << "\"\n";

    // Memory comparison
    std::cout << "combined occupies " << combined.capacity() * sizeof(std::string)
              << " bytes\n";
    std::cout << "concat view occupies ~" << sizeof(lazy) << " bytes (just iterators)\n";
}
// Expected output:
// After insert, a[0] = "hello" (still valid - was copied)
// concat view: hello world foo bar
// After modifying via view, a[0] = "HELLO"
// combined occupies <implementation-defined> bytes
// concat view occupies ~<small number> bytes (just iterators)
```

The write-through behaviour (`*it = "HELLO"` changing `a[0]`) is the clearest proof that no copy was made - you're directly touching the original element. If `a` had a million strings, the concat view would cost the same few bytes of iterator storage regardless.

- `vector::insert` copies every string into `combined` - if `a` had 1M strings, that's 1M copies.
- `views::concat` stores only references/iterators - O(1) memory regardless of range size.
- Writing through the concat view's iterator modifies the **original** container element.
- This proves the view is non-owning and zero-copy.

### Q3: Explain the iterator category of the concat view based on its input ranges

**The concat view's iterator category is the weakest common category among all input ranges.**

The reason this matters in practice is that algorithms like `std::ranges::sort` need random-access, and if you mix a `vector` with a `list` you'll lose that capability. Let's see it happen:

```cpp
#include <concepts>
#include <iostream>
#include <list>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> v = {1, 2, 3};       // random_access_range
    std::list<int>   l = {4, 5, 6};       // bidirectional_range

    // Case 1: All random-access
    auto c1 = std::views::concat(v, v);
    static_assert(std::ranges::random_access_range<decltype(c1)>);
    std::cout << "vector + vector = random_access\n";

    // Case 2: random-access + bidirectional = bidirectional
    auto c2 = std::views::concat(v, l);
    static_assert(std::ranges::bidirectional_range<decltype(c2)>);
    static_assert(!std::ranges::random_access_range<decltype(c2)>);
    std::cout << "vector + list = bidirectional\n";

    // Case 3: bidirectional + bidirectional = bidirectional
    auto c3 = std::views::concat(l, l);
    static_assert(std::ranges::bidirectional_range<decltype(c3)>);
    std::cout << "list + list = bidirectional\n";

    // Case 4: With filter_view (forward only)
    auto fv = v | std::views::filter([](int n) { return n > 1; });
    auto c4 = std::views::concat(v, fv);
    // filter_view is bidirectional when source is random_access,
    // but concat takes the weakest - still bidirectional here
    static_assert(std::ranges::bidirectional_range<decltype(c4)>);
    std::cout << "vector + filter_view = bidirectional\n";
}
// Expected output:
// vector + vector = random_access
// vector + list = bidirectional
// list + list = bidirectional
// vector + filter_view = bidirectional
```

Each case makes sense when you think about what the concat iterator has to do at a range boundary - it must advance or retreat from one range into the next, and it can only do so using operations that both ranges support.

**Category downgrade rules:**

| Rule | Why |
| --- | --- |
| Weakest input wins | concat must switch between ranges at boundaries - it can only guarantee operations supported by ALL inputs |
| Random-access requires all-sized + all-random-access | To compute `it + n`, concat needs to know each range's size to determine which sub-range to jump into |
| Common reference required | `*it` must return the same type regardless of which sub-range the iterator currently points to |

---

## Notes

- `views::concat` is a **C++26** feature (P2542R8). As of 2024, GCC 15 and MSVC have experimental support.
- Pre-C++26 workaround for same-type ranges: store them in a `vector<span<int>>` and use `views::join`.
- `concat` with a single range returns the range itself (degenerate case).
- Unlike `views::join`, `concat` takes its ranges as **individual arguments**, not as a range-of-ranges.
- `concat` is a **borrowed range** if all input ranges are borrowed ranges.
