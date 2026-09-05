# Use std::ranges::views::adjacent and adjacent_transform (C++23)

**Category:** Lambda and Functional  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/adjacent_view>  

---

## Topic Overview

C++23 introduces `views::adjacent<N>` and `views::adjacent_transform<N>` for sliding-window operations over ranges. These views replace the manual index manipulation, `zip + drop(1)` tricks, and raw iterator loops that used to be the only way to look at consecutive elements together. The idea is simple: instead of seeing one element at a time, you see a window of N consecutive elements as a tuple.

```cpp
namespace rv = std::ranges::views;
std::vector<int> v = {10, 20, 30, 40, 50};

// adjacent<2> - pairs of consecutive elements
for (auto [a, b] : v | rv::adjacent<2>)
    std::cout << "(" << a << "," << b << ") ";
// (10,20) (20,30) (30,40) (40,50)

// adjacent_transform<2> - apply function to consecutive pairs
for (auto diff : v | rv::adjacent_transform<2>(std::minus{}))
    std::cout << diff << " ";
// -10 -10 -10 -10
```

Notice that a range of 5 elements gives you 4 pairs - the window slides one position at a time and stops when it can't fit.

### Aliases

For the very common case of pairs, the standard provides shorter names:

| View | Alias for N=2 |
| --- | --- |
| `views::adjacent<2>` | `views::pairwise` |
| `views::adjacent_transform<2>` | `views::pairwise_transform` |

---

## Self-Assessment

### Q1: Use `views::adjacent<2>` to iterate pairs and compute differences

This example shows the three window sizes - pairs, pairs-with-transform, and triples. The structured binding `auto [prev, curr]` makes the code feel like natural pseudocode once you read it aloud.

```cpp
#include <iostream>
#include <vector>
#include <ranges>

namespace rv = std::ranges::views;

int main() {
    std::vector<int> prices = {100, 105, 102, 110, 108, 115};

    // adjacent<2>: sliding window of size 2
    std::cout << "Consecutive pairs:\n";
    for (auto [prev, curr] : prices | rv::adjacent<2>) {
        int change = curr - prev;
        std::cout << "  " << prev << " -> " << curr
                  << " (" << (change >= 0 ? "+" : "") << change << ")\n";
    }

    // pairwise (alias for adjacent<2>)
    std::cout << "\nDifferences via pairwise_transform:\n";
    for (auto diff : prices | rv::pairwise_transform(std::minus{})) {
        std::cout << "  " << diff << "\n";
    }

    // adjacent<3>: sliding window of size 3
    std::cout << "\nTriples:\n";
    for (auto [a, b, c] : prices | rv::adjacent<3>) {
        std::cout << "  (" << a << ", " << b << ", " << c << ")\n";
    }
}
// Expected output:
//   Consecutive pairs:
//     100 -> 105 (+5)
//     105 -> 102 (-3)
//     102 -> 110 (+8)
//     110 -> 108 (-2)
//     108 -> 115 (+7)
//
//   Differences via pairwise_transform:
//     -5
//     3
//     -8
//     2
//     -7
//
//   Triples:
//     (100, 105, 102)
//     (105, 102, 110)
//     (102, 110, 108)
//     (110, 108, 115)
```

---

### Q2: Apply `adjacent_transform<3>` to compute a running three-element average

The transform lambda receives exactly N arguments matching the window size - here three `double` values for a 3-element moving average. There's no index bookkeeping, no bounds checking, no off-by-one error to worry about.

```cpp
#include <iostream>
#include <vector>
#include <ranges>

namespace rv = std::ranges::views;

int main() {
    std::vector<double> temps = {20.0, 22.5, 19.0, 23.0, 21.5, 24.0, 20.5};

    std::cout << "Temperatures: ";
    for (double t : temps) std::cout << t << " ";
    std::cout << "\n";

    // adjacent_transform<3>: apply function to each sliding 3-element window
    auto moving_avg = temps | rv::adjacent_transform<3>(
        [](double a, double b, double c) {
            return (a + b + c) / 3.0;
        });

    std::cout << "3-element moving average:\n";
    int i = 0;
    for (double avg : moving_avg) {
        std::cout << "  window [" << i << "," << i+2 << "]: " << avg << "\n";
        ++i;
    }

    // N elements -> N-2 averages for window size 3
    // (20.0+22.5+19.0)/3 = 20.5
    // (22.5+19.0+23.0)/3 = 21.5
    // (19.0+23.0+21.5)/3 = 21.167
    // (23.0+21.5+24.0)/3 = 22.833
    // (21.5+24.0+20.5)/3 = 22.0

    // adjacent_transform<2> for derivatives
    std::vector<double> signal = {0, 1, 4, 9, 16, 25}; // x^2
    std::cout << "\nDiscrete derivative (x^2): ";
    for (auto d : signal | rv::adjacent_transform<2>(std::minus{})) {
        std::cout << d << " ";
    }
    std::cout << "\n";
    // -1 -3 -5 -7 -9  (negated differences: 0-1, 1-4, 4-9, ...)
}
```

---

### Q3: Explain how adjacent views relate to the classic `zip(range, range | drop(1))` pattern

Before C++23, the standard way to process consecutive pairs was to zip the range with a shifted copy of itself. That idiom works, but it has serious problems that `adjacent<N>` avoids entirely - especially for single-pass ranges where iterating the range twice is either wrong or impossible.

```cpp
#include <iostream>
#include <vector>
#include <ranges>

namespace rv = std::ranges::views;

int main() {
    std::vector<int> v = {10, 20, 30, 40, 50};

    // C++20: zip + drop(1) to get consecutive pairs
    std::cout << "zip + drop(1):\n";
    for (auto [a, b] : rv::zip(v, v | rv::drop(1))) {
        std::cout << "  (" << a << ", " << b << ")\n";
    }

    // C++23: adjacent<2> does the same thing!
    std::cout << "adjacent<2>:\n";
    for (auto [a, b] : v | rv::adjacent<2>) {
        std::cout << "  (" << a << ", " << b << ")\n";
    }

    // Both produce: (10,20) (20,30) (30,40) (40,50)
}
```

The equivalence is exact for random-access ranges, but `adjacent<N>` is strictly better in general:

```cpp
adjacent<2>(r)  =  zip(r, r | drop(1))
adjacent<3>(r)  =  zip(r, r | drop(1), r | drop(2))
adjacent<N>(r)  =  zip(r, r | drop(1), ..., r | drop(N-1))
```

**Why `adjacent` is better:**

| Aspect | `zip + drop` | `adjacent<N>` |
| --- | --- | --- |
| Readability | Verbose | **Concise** |
| N > 2 | Very verbose | Just change N |
| Single-pass ranges | **Broken** (iterates twice) | **Works** (single traversal) |
| Iterator invalidation | Two independent iterators | **One coordinated window** |
| Intent | Unclear | **Obvious** |

---

## Notes

- **`adjacent<1>`** is equivalent to `views::transform` wrapping each element in a 1-tuple.
- **`adjacent<0>`** produces a range of empty tuples (one per original element + 1).
- **Element type:** `adjacent<N>` yields `tuple<T&, T&, ..., T&>` (N references to elements in the original range).
- **`adjacent_transform<N>(f)`** is equivalent to `adjacent<N> | transform(apply(f))` but more efficient because the tuple is never materialized.
- **Requires C++23:** As of 2024, GCC 14+ and MSVC 17.8+ support these views.
