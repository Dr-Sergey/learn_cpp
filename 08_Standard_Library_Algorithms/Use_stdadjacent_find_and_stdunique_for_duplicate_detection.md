# Use std::adjacent_find and std::unique for duplicate detection

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/unique>  

---

## Topic Overview

### std::adjacent_find

`std::adjacent_find` finds the **first pair of adjacent equal elements** (or elements satisfying a predicate). It returns an iterator to the first element of the pair, or `end()` if no pair is found.

```cpp
auto it = std::adjacent_find(first, last);         // equality
auto it = std::adjacent_find(first, last, pred);   // custom predicate
```

Use cases: detecting consecutive duplicates, finding the first adjacent pair with a specific relationship (out of order, difference above a threshold, etc.).

### std::unique

`std::unique` removes **consecutive duplicate** elements by moving unique elements to the front. It returns an iterator to the new logical end. Like `std::remove`, it does **not** actually erase elements from the container.

```cpp
auto new_end = std::unique(first, last);         // equality
auto new_end = std::unique(first, last, pred);   // custom predicate
```

The elements after `new_end` are in an unspecified (moved-from) state.

### Key Insight

The most important thing to understand about both of these algorithms is that they only operate on **consecutive** duplicates. To catch non-adjacent duplicates scattered throughout the range, you must sort first. The reason this trips people up is that the function is named `unique` but it does not produce a globally unique range unless the data is already sorted.

| Need | Algorithm | Prerequisite |
| --- | --- | --- |
| Find first duplicate pair | `adjacent_find` | Sort first for global duplicates |
| Remove all duplicates | `sort` + `unique` + `erase` | Sort first |
| Check if any duplicates exist | `adjacent_find` after sort | Sort first |
| C++20 | `std::ranges::unique` | Supports projections |

---

## Self-Assessment

### Q1: Use adjacent_find to detect the first pair of consecutive equal elements

`adjacent_find` is useful any time you want to know whether two neighboring elements have a particular relationship - not just equality but any predicate.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
#include <cmath>

int main() {
    // === Basic: find first consecutive duplicate ===
    std::vector<int> v = {1, 2, 3, 3, 4, 5, 5, 6};

    auto it = std::adjacent_find(v.begin(), v.end());
    if (it != v.end()) {
        std::cout << "First duplicate pair: " << *it << " at index "
                  << (it - v.begin()) << "\n";
    }
    // Output: First duplicate pair: 3 at index 2

    // === Find ALL adjacent duplicate pairs ===
    auto pos = v.begin();
    while ((pos = std::adjacent_find(pos, v.end())) != v.end()) {
        std::cout << "Duplicate pair at index " << (pos - v.begin())
                  << ": " << *pos << "\n";
        ++pos;  // advance past this pair
    }
    // Duplicate pair at index 2: 3
    // Duplicate pair at index 5: 5

    // === Custom predicate: find first adjacent pair that's out of order ===
    std::vector<int> data = {1, 3, 5, 4, 7, 8};
    auto unsorted = std::adjacent_find(data.begin(), data.end(),
                                       std::greater<>{});
    if (unsorted != data.end()) {
        std::cout << "Sort break: " << *unsorted << " > " << *(unsorted + 1)
                  << " at index " << (unsorted - data.begin()) << "\n";
    }
    // Output: Sort break: 5 > 4 at index 2

    // === Practical: detect value jumps > threshold ===
    std::vector<double> temps = {20.1, 20.3, 20.2, 25.8, 25.9};
    auto jump = std::adjacent_find(temps.begin(), temps.end(),
        [](double a, double b) { return std::abs(a - b) > 3.0; });
    if (jump != temps.end()) {
        std::cout << "Temperature jump: " << *jump << " -> " << *(jump + 1) << "\n";
    }
    // Output: Temperature jump: 20.2 -> 25.8

    // === Check if sorted (adjacent_find with greater) ===
    std::vector<int> sorted = {1, 2, 3, 4, 5};
    bool is_sorted = std::adjacent_find(sorted.begin(), sorted.end(),
                                        std::greater<>{}) == sorted.end();
    std::cout << "Is sorted: " << std::boolalpha << is_sorted << "\n";  // true

    return 0;
}
```

The `is_sorted` check at the end using `adjacent_find` is actually equivalent to `std::is_sorted` - both check whether any adjacent pair violates the ordering. It is a handy idiom to recognize.

### Q2: Apply unique + erase to remove consecutive duplicates from a sorted vector

The erase step after `std::unique` is mandatory, just as it is with `std::remove_if`. Without it, the vector still has its original size with garbage in the tail.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    // === Remove consecutive duplicates from sorted data ===
    std::vector<int> v = {1, 1, 2, 3, 3, 3, 4, 4, 5};

    // unique moves unique elements to front, returns new logical end
    auto new_end = std::unique(v.begin(), v.end());

    std::cout << "After unique (before erase):\n";
    std::cout << "  Unique part: ";
    for (auto it = v.begin(); it != new_end; ++it)
        std::cout << *it << " ";
    std::cout << "\n";
    // 1 2 3 4 5

    std::cout << "  Full vector (size " << v.size() << "): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5 3 4 4 5  <- tail is garbage (moved-from)

    // Erase the tail
    v.erase(new_end, v.end());
    std::cout << "After erase (size " << v.size() << "): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5

    // === Full pattern: sort + unique + erase (remove all duplicates) ===
    std::vector<int> unsorted = {5, 3, 1, 3, 5, 2, 1, 4, 3};
    std::sort(unsorted.begin(), unsorted.end());
    unsorted.erase(std::unique(unsorted.begin(), unsorted.end()), unsorted.end());

    std::cout << "\nAfter sort+unique+erase: ";
    for (int x : unsorted) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5

    // === Custom predicate: remove "close enough" duplicates ===
    std::vector<double> measurements = {1.0, 1.001, 1.002, 2.0, 2.001, 3.0};
    // Treat values within 0.01 of each other as duplicates
    measurements.erase(
        std::unique(measurements.begin(), measurements.end(),
                    [](double a, double b) { return std::abs(a - b) < 0.01; }),
        measurements.end()
    );

    std::cout << "Deduplicated measurements: ";
    for (double x : measurements) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3

    // === C++20 alternative: std::erase + ranges ===
    // std::vector<int> v2 = {1,1,2,2,3};
    // std::ranges::sort(v2);
    // auto [new_end2, _] = std::ranges::unique(v2);
    // v2.erase(new_end2, v2.end());

    return 0;
}
```

The `sort + unique + erase` triple is the canonical pattern for deduplicating a vector. You will see this combination so often that it is worth memorizing as a unit. The custom predicate variant is handy for floating-point data where you want to treat nearly-equal values as the same.

### Q3: Show a bug where unique is applied to an unsorted range and not all duplicates are removed

This is the most common mistake with `std::unique` and it is easy to miss because the code compiles and runs without errors - it just silently produces wrong output.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <set>

int main() {
    // === BUG: unique on unsorted data ===
    std::vector<int> v = {3, 1, 3, 2, 1, 3};

    std::cout << "Original: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Original: 3 1 3 2 1 3

    // unique only removes CONSECUTIVE duplicates!
    v.erase(std::unique(v.begin(), v.end()), v.end());

    std::cout << "After unique+erase (unsorted): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // After unique+erase (unsorted): 3 1 3 2 1 3
    // NOTHING was removed! No consecutive duplicates exist.

    // === Another example where SOME but not all are removed ===
    std::vector<int> v2 = {1, 1, 2, 3, 1, 1, 2};
    v2.erase(std::unique(v2.begin(), v2.end()), v2.end());

    std::cout << "Partial removal: ";
    for (int x : v2) std::cout << x << " ";
    std::cout << "\n";
    // Partial removal: 1 2 3 1 2
    // The consecutive 1,1 at positions 0-1 was reduced to one 1.
    // The consecutive 1,1 at positions 4-5 was reduced to one 1.
    // But 1 still appears twice (non-consecutively).

    // === FIX: sort first, then unique ===
    std::vector<int> v3 = {3, 1, 3, 2, 1, 3};
    std::sort(v3.begin(), v3.end());           // {1, 1, 2, 3, 3, 3}
    v3.erase(std::unique(v3.begin(), v3.end()), v3.end());

    std::cout << "Fixed (sorted first): ";
    for (int x : v3) std::cout << x << " ";
    std::cout << "\n";
    // Fixed (sorted first): 1 2 3  <- all duplicates removed

    // === Alternative: use std::set for O(n log n) dedup without modifying order ===
    std::vector<int> v4 = {3, 1, 3, 2, 1, 3};
    std::vector<int> result;
    std::set<int> seen;
    for (int x : v4) {
        if (seen.insert(x).second) {
            result.push_back(x);
        }
    }
    std::cout << "Order-preserving dedup: ";
    for (int x : result) std::cout << x << " ";
    std::cout << "\n";
    // Order-preserving dedup: 3 1 2

    return 0;
}
```

`std::unique` only compares adjacent elements. Non-adjacent duplicates are completely invisible to it. On unsorted data, duplicates may be scattered throughout the range, so `unique` may miss them entirely - as the first example shows, nothing at all was removed even though the value 3 appears three times. The fix is always: **sort first, then apply unique + erase**. If you need to preserve the original order while deduplicating, use a `std::set` or `std::unordered_set` as a seen-tracker instead.

---

## Notes

- **`adjacent_find` complexity:** O(n), exactly n-1 comparisons.
- **`unique` complexity:** O(n), exactly n-1 comparisons/moves.
- **`unique` does NOT erase** - like `remove`, it returns the new logical end. Always call `.erase()`.
- **C++20:** `std::ranges::unique` returns a subrange of the "garbage" tail, making the erase step cleaner.
- **For `std::list`:** Use the member function `lst.unique()` instead - it actually erases in O(1) per removal.
- **`adjacent_find` as `is_sorted` check:** `adjacent_find(first, last, greater<>{}) == last` is equivalent to `is_sorted(first, last)`.
