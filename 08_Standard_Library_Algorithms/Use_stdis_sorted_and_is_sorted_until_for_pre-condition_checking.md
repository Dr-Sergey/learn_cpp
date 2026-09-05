# Use std::is_sorted and is_sorted_until for pre-condition checking

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/is_sorted>  

---

## Topic Overview

### What They Do

Here are the four signatures you will use most:

```cpp
#include <algorithm>

bool std::is_sorted(first, last);             // true if sorted in ascending order
bool std::is_sorted(first, last, comp);       // true if sorted according to comp

auto it = std::is_sorted_until(first, last);       // iterator to first out-of-order element
auto it = std::is_sorted_until(first, last, comp); // with custom comparator
```

| Function | Returns | Useful for |
| --- | --- | --- |
| `is_sorted` | `bool` | Assertions, precondition checks |
| `is_sorted_until` | Iterator | Diagnostics - shows WHERE sorting breaks |

Both are O(n) - a single pass comparing adjacent elements.

### Why Use Them

Binary search algorithms (`lower_bound`, `upper_bound`, `binary_search`, `equal_range`) require sorted input. Calling them on unsorted data is **undefined behavior**. Using `is_sorted` as a precondition check catches bugs early - you get a clear assertion failure at the call site instead of mysterious wrong answers or crashes deep inside the algorithm.

---

## Self-Assessment

### Q1: Use is_sorted as an assertion before calling binary search algorithms

The pattern here is straightforward: assert that the range is sorted before you call any algorithm that depends on it. In debug builds this fires immediately if the data is wrong; in release builds the assert compiles away.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cassert>
#include <string>

int main() {
    std::vector<int> data = {1, 3, 5, 7, 9, 11, 13};

    // === Assert sorted before binary search ===
    assert(std::is_sorted(data.begin(), data.end()));

    auto it = std::lower_bound(data.begin(), data.end(), 7);
    if (it != data.end() && *it == 7) {
        std::cout << "Found 7 at index " << (it - data.begin()) << "\n";  // index 3
    }

    // === What happens with unsorted data ===
    std::vector<int> bad = {5, 2, 8, 1, 9};

    // This assertion would FIRE and stop the program:
    // assert(std::is_sorted(bad.begin(), bad.end()));

    if (!std::is_sorted(bad.begin(), bad.end())) {
        std::cout << "ERROR: data is not sorted — binary search is unsafe!\n";
    }

    // === With custom comparator ===
    std::vector<std::string> names = {"Alice", "Bob", "Charlie", "Dave"};
    assert(std::is_sorted(names.begin(), names.end()));  // lexicographic order

    // Descending order check
    std::vector<int> desc = {9, 7, 5, 3, 1};
    assert(std::is_sorted(desc.begin(), desc.end(), std::greater<>{}));

    // Binary search on descending with matching comparator
    bool found = std::binary_search(desc.begin(), desc.end(), 5, std::greater<>{});
    std::cout << "Found 5 in descending: " << std::boolalpha << found << "\n";  // true

    return 0;
}
```

The key thing to remember is that whenever you pass a custom comparator to a binary search algorithm, you must use the same comparator in your `is_sorted` check - the sortedness criterion and the search criterion must match.

### Q2: Use is_sorted_until to find the first out-of-order element for diagnostic purposes

`is_sorted_until` returns an iterator to the first element that is out of order. This is more useful than a plain `is_sorted` check when you want to understand *where* the problem is, not just *whether* there is a problem.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

void diagnose_sort_order(const std::vector<int>& v, const std::string& label) {
    auto it = std::is_sorted_until(v.begin(), v.end());
    if (it == v.end()) {
        std::cout << label << ": fully sorted\n";
    } else {
        auto idx = it - v.begin();
        std::cout << label << ": sort breaks at index " << idx
                  << " (value " << *it << " < previous " << *(it - 1) << ")\n";
        std::cout << "  Sorted prefix length: " << idx << " of " << v.size() << "\n";
    }
}

int main() {
    std::vector<int> sorted    = {1, 2, 3, 4, 5};
    std::vector<int> almost    = {1, 2, 3, 2, 5};    // breaks at index 3
    std::vector<int> end_break = {1, 2, 3, 4, 1};    // breaks at last element
    std::vector<int> reversed  = {5, 4, 3, 2, 1};    // breaks at index 1

    diagnose_sort_order(sorted, "sorted");
    diagnose_sort_order(almost, "almost");
    diagnose_sort_order(end_break, "end_break");
    diagnose_sort_order(reversed, "reversed");
    // sorted:    fully sorted
    // almost:    sort breaks at index 3 (value 2 < previous 3)
    // end_break: sort breaks at index 4 (value 1 < previous 4)
    // reversed:  sort breaks at index 1 (value 4 < previous 5)

    // === Find the longest sorted prefix ===
    std::vector<int> data = {1, 3, 5, 7, 4, 6, 8, 10};
    auto until = std::is_sorted_until(data.begin(), data.end());
    size_t sorted_len = std::distance(data.begin(), until);
    std::cout << "\nLongest sorted prefix: " << sorted_len << " elements\n";
    // 4 elements: {1, 3, 5, 7}

    // === Useful for debugging sort-related bugs ===
    // After a custom sort operation, check if the result is actually sorted:
    // assert(std::is_sorted(v.begin(), v.end()) &&
    //        "Custom sort produced unsorted output!");

    return 0;
}
```

The "longest sorted prefix" trick at the end - computing `std::distance` from `begin` to the result of `is_sorted_until` - gives you a quick way to see how much of a partially sorted range is already in order.

### Q3: Write a debug-mode wrapper for binary_search that asserts the is_sorted precondition

This is the production-ready version of the pattern: wrap the binary search functions in thin templates that assert sortedness in debug builds and vanish completely in release builds. The `#ifdef NDEBUG` split is the standard way to achieve zero-overhead checking that still catches bugs during development.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cassert>
#include <string>

// === Debug wrapper for binary search algorithms ===

#ifdef NDEBUG
// Release: no overhead
template <typename Iter, typename T>
auto safe_lower_bound(Iter first, Iter last, const T& value) {
    return std::lower_bound(first, last, value);
}

template <typename Iter, typename T>
bool safe_binary_search(Iter first, Iter last, const T& value) {
    return std::binary_search(first, last, value);
}
#else
// Debug: assert precondition
template <typename Iter, typename T>
auto safe_lower_bound(Iter first, Iter last, const T& value) {
    assert(std::is_sorted(first, last) && "safe_lower_bound: range is not sorted!");
    return std::lower_bound(first, last, value);
}

template <typename Iter, typename T>
bool safe_binary_search(Iter first, Iter last, const T& value) {
    assert(std::is_sorted(first, last) && "safe_binary_search: range is not sorted!");
    return std::binary_search(first, last, value);
}
#endif

// === With custom comparator ===
template <typename Iter, typename T, typename Comp>
auto safe_lower_bound(Iter first, Iter last, const T& value, Comp comp) {
    assert(std::is_sorted(first, last, comp) && "range not sorted with given comparator!");
    return std::lower_bound(first, last, value, comp);
}

int main() {
    // === Works fine on sorted data ===
    std::vector<int> sorted = {1, 3, 5, 7, 9, 11};

    auto it = safe_lower_bound(sorted.begin(), sorted.end(), 7);
    std::cout << "lower_bound(7): " << *it << " at index "
              << (it - sorted.begin()) << "\n";  // 7 at index 3

    bool found = safe_binary_search(sorted.begin(), sorted.end(), 5);
    std::cout << "binary_search(5): " << std::boolalpha << found << "\n";  // true

    // === Descending with comparator ===
    std::vector<int> desc = {11, 9, 7, 5, 3, 1};
    auto it2 = safe_lower_bound(desc.begin(), desc.end(), 7, std::greater<>{});
    std::cout << "lower_bound(7, desc): " << *it2 << "\n";  // 7

    // === This would ASSERT in debug mode ===
    // std::vector<int> unsorted = {5, 2, 8, 1};
    // safe_binary_search(unsorted.begin(), unsorted.end(), 2);  // ASSERTION FAILURE!

    std::cout << "All checks passed.\n";

    return 0;
}
// Compile debug:   g++ -std=c++17 -g -O0 file.cpp
// Compile release: g++ -std=c++17 -O2 -DNDEBUG file.cpp
```

In **debug mode** (default), `assert(is_sorted(...))` fires if the input is unsorted, catching the bug immediately with a clear message. In **release mode** (`-DNDEBUG`), the assert is compiled away - zero overhead. This pattern catches a very common class of bugs: calling binary search on data that accidentally became unsorted (for example, after inserting without maintaining order).

---

## Notes

- **Complexity:** `is_sorted` and `is_sorted_until` are both O(n), performing exactly `n-1` comparisons.
- **Empty/single-element ranges:** Always considered sorted.
- **`is_sorted`** is equivalent to `is_sorted_until(first, last) == last`.
- **C++20 ranges:** `std::ranges::is_sorted(v)` - no begin/end needed. Supports projections.
- **Performance tip:** In hot paths, the O(n) `is_sorted` check before an O(log n) binary search may be too expensive for production. Use it only in debug builds or tests.
- **For containers with built-in ordering** (`std::set`, `std::map`): No need to check - they're always sorted by definition.
