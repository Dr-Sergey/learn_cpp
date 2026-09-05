# Use std::ranges::chunk_by (C++23) to group consecutive elements by predicate

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/chunk_by_view>  

---

## Topic Overview

`std::views::chunk_by` (C++23) splits a range into subranges where consecutive elements satisfy a binary predicate. A new "chunk" starts whenever the predicate returns `false` for two adjacent elements. Think of it as scanning through the sequence and cutting it wherever neighboring elements no longer "belong together" by your definition.

Here's the simplest case - grouping runs of equal integers:

```cpp
Input:    [1, 1, 2, 2, 2, 3, 1, 1]
chunk_by(equal_to{}):
  Group 1: [1, 1]
  Group 2: [2, 2, 2]
  Group 3: [3]
  Group 4: [1, 1]
```

### Key Properties

| Property | Detail |
| --- | --- |
| Header | `<ranges>` |
| Standard | C++23 |
| Lazy | Yes - groups are computed on demand |
| Groups are | `subrange` views into the original range |
| Predicate called on | Adjacent pairs: `pred(elem[i], elem[i+1])` |
| New chunk starts when | `pred` returns `false` |

### Comparison with Other Grouping

It helps to keep `chunk_by` distinct from the other chunking and windowing tools:

| Approach | What it does |
| --- | --- |
| `chunk_by(pred)` | Groups consecutive elements where pred(a,b) is true |
| `chunk(n)` | Splits into fixed-size chunks of n |
| `adjacent<N>` | Sliding window of N elements |
| Manual loop with `adjacent_find` | Pre-C++23 equivalent |

---

## Self-Assessment

### Q1: Group consecutive equal integers using chunk_by(std::ranges::equal_to{})

This is the canonical use case. Notice that the four groups in the output include two separate groups of `1`s - `chunk_by` groups *consecutive* runs, not all elements with the same value globally:

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <functional>

int main() {
    std::vector<int> data = {1, 1, 2, 2, 2, 3, 3, 1, 1, 1, 4};

    std::cout << "Groups of consecutive equal elements:\n";
    for (auto group : data | std::views::chunk_by(std::ranges::equal_to{})) {
        std::cout << "  [";
        bool first = true;
        for (int x : group) {
            if (!first) std::cout << ", ";
            std::cout << x;
            first = false;
        }
        std::cout << "] (size " << std::ranges::distance(group) << ")\n";
    }
    // [1, 1] (size 2)
    // [2, 2, 2] (size 3)
    // [3, 3] (size 2)
    // [1, 1, 1] (size 3)
    // [4] (size 1)

    // === Run-Length Encoding ===
    std::cout << "\nRun-Length Encoding:\n";
    for (auto group : data | std::views::chunk_by(std::ranges::equal_to{})) {
        auto first_elem = *std::ranges::begin(group);
        auto count = std::ranges::distance(group);
        std::cout << "  " << first_elem << " x " << count << "\n";
    }
    // 1 x 2, 2 x 3, 3 x 2, 1 x 3, 4 x 1

    return 0;
}
```

The run-length encoding pattern at the end is a natural fit for `chunk_by` - you get each run as a subrange, so you just need the first element and the count.

### Q2: Group words by their first letter using a predicate on first characters

The predicate does not have to be `equal_to`. Any binary function that tests whether two adjacent elements "belong in the same group" works. Here we group by first letter, and then extend the idea to grouping log entries by severity level:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <algorithm>

int main() {
    // Words sorted by first letter
    std::vector<std::string> words = {
        "apple", "avocado", "banana", "blueberry", "cherry",
        "cranberry", "date", "dragonfruit"
    };

    // chunk_by: group words where consecutive pairs share the same first letter
    auto same_first_letter = [](const std::string& a, const std::string& b) {
        return a[0] == b[0];
    };

    std::cout << "Words grouped by first letter:\n";
    for (auto group : words | std::views::chunk_by(same_first_letter)) {
        char letter = (*std::ranges::begin(group))[0];
        std::cout << "  '" << letter << "': ";
        for (const auto& w : group) std::cout << w << " ";
        std::cout << "\n";
    }
    // 'a': apple avocado
    // 'b': banana blueberry
    // 'c': cherry cranberry
    // 'd': date dragonfruit

    // === Grouping log entries by severity ===
    struct LogEntry {
        std::string level;
        std::string message;
    };

    std::vector<LogEntry> logs = {
        {"INFO", "Starting"},  {"INFO", "Loading config"},
        {"WARN", "Slow query"}, {"WARN", "High memory"},
        {"ERROR", "Crash"},
        {"INFO", "Recovered"}, {"INFO", "Running"}
    };

    auto same_level = [](const LogEntry& a, const LogEntry& b) {
        return a.level == b.level;
    };

    std::cout << "\nLog groups:\n";
    for (auto group : logs | std::views::chunk_by(same_level)) {
        auto level = std::ranges::begin(group)->level;
        auto count = std::ranges::distance(group);
        std::cout << "  " << level << " (" << count << " entries)\n";
    }
    // INFO (2 entries), WARN (2 entries), ERROR (1 entries), INFO (2 entries)

    return 0;
}
```

Notice the log output ends with a second INFO group - that's the same "consecutive runs only" behavior. If you want all INFO entries together regardless of position, you'd sort by level first.

### Q3: Show that chunk_by is lazy and the groups are subranges of the original

This is an important property to understand. `chunk_by` does not copy anything - it produces a view object, and the actual scanning only happens when you iterate. The groups themselves are also views into the original data, not copies:

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <functional>

int main() {
    std::vector<int> data = {1, 1, 2, 2, 3};

    // === Laziness demonstration ===
    // chunk_by constructs a view - no iteration happens here
    auto chunked = data | std::views::chunk_by(std::ranges::equal_to{});
    // chunked is a view object; nothing has been computed yet

    std::cout << "View created (no computation yet)\n";

    // Only when we iterate do elements get examined
    auto it = std::ranges::begin(chunked);
    // Now the first group boundary is found

    // === Groups are subranges (views into original data) ===
    for (auto group : chunked) {
        // group is a subrange - a view, not a copy
        auto* ptr = &*std::ranges::begin(group);
        std::cout << "Group at address " << ptr << ": ";
        for (int x : group) std::cout << x << " ";
        std::cout << "\n";
    }

    // Prove they refer to original data: modify through the subrange
    auto first_group = *std::ranges::begin(chunked);
    *std::ranges::begin(first_group) = 99;  // modifies data[0]

    std::cout << "\nAfter modifying through view, data[0] = " << data[0] << "\n";
    // data[0] = 99 - confirms it's a view, not a copy

    // === Composable with other views ===
    std::vector<int> nums = {1, 1, 2, 2, 2, 3, 3, 4, 4, 4, 4};

    // Get only groups of size >= 3
    for (auto group : nums
            | std::views::chunk_by(std::ranges::equal_to{})
            | std::views::filter([](auto g) {
                return std::ranges::distance(g) >= 3;
              }))
    {
        std::cout << "Large group: ";
        for (int x : group) std::cout << x << " ";
        std::cout << " (size " << std::ranges::distance(group) << ")\n";
    }
    // Large group: 2 2 2 (size 3)
    // Large group: 4 4 4 4 (size 4)

    return 0;
}
```

The mutation test confirms that a group is a direct view into the original vector - writing through it changes `data[0]`. The composability example is equally useful: you can pipe `chunk_by` into `filter`, `transform`, or any other view adaptor just like any other range.

---

## Notes

- **`chunk_by` requires C++23.** Check compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- The predicate is called on **adjacent pairs** `(elem[i], elem[i+1])` - a new chunk starts when the predicate returns `false`.
- **Not the same as SQL GROUP BY:** `chunk_by` only groups *consecutive* matching elements. If you need all equal elements grouped regardless of position, sort first.
- Pre-C++23 equivalent: use `std::adjacent_find` in a loop to find group boundaries, then create subranges manually.
- **`views::chunk(n)`** is a different view: it splits into fixed-size chunks of `n` elements each.
