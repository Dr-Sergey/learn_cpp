# Use std::ranges::find_last and find_last_if (C++23)

**Category:** Standard Library Algorithms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges/find_last>  

---

## Topic Overview

`std::ranges::find_last`, `find_last_if`, and `find_last_if_not` (C++23) search for the **last** matching element in a range. Before C++23, finding the last occurrence required using reverse iterators or manual backward scanning - both of which are workable but genuinely awkward.

### Signatures

All three variants return a `subrange` rather than a plain iterator:

```cpp
#include <algorithm>

// Find last element equal to value
auto [found, end] = std::ranges::find_last(range, value);

// Find last element satisfying predicate
auto [found, end] = std::ranges::find_last_if(range, pred);

// Find last element NOT satisfying predicate
auto [found, end] = std::ranges::find_last_if_not(range, pred);
```

### Return Type: `subrange`

This is the part that often surprises people coming from `std::find`. Instead of returning a single iterator, `find_last` returns a **`subrange`** from the found element to the end of the range. If nothing is found, the subrange is empty (begin == end).

```cpp
range:     [a, b, c, X, d, e, X, f, g]
find_last(X):         returned subrange: [X, f, g]
                                          ^ found
```

The subrange is useful because you often want to work with the tail of the sequence from the found element onward - for example, extracting a filename from a path, or a file extension.

### Pre-C++23 Alternatives Comparison

| Approach | Problem |
| --- | --- |
| `rbegin`/`rend` + `find` | Returns reverse_iterator, awkward to convert back |
| Manual loop from end | Verbose, error-prone |
| `find_last` (C++23) | Clean, returns subrange, works with projections |

The reverse iterator conversion issue is worth elaborating: `rit.base()` points one *past* the element you found, not to the element itself, so you have to subtract one. It works, but it's a trap waiting to catch you.

---

## Self-Assessment

### Q1: Use ranges::find_last to locate the last occurrence of a value in a range

The returned subrange starts at the found element and runs to the end of the original range. Use `.empty()` to check whether the search succeeded, and `.begin()` to get an iterator to the actual element:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    std::vector<int> data = {3, 7, 2, 7, 5, 7, 1};

    // Find the last occurrence of 7
    auto result = std::ranges::find_last(data, 7);

    if (!result.empty()) {
        auto pos = std::distance(data.begin(), result.begin());
        std::cout << "Last 7 found at index " << pos << "\n";  // index 5
        std::cout << "Value: " << *result.begin() << "\n";      // 7
    }

    // result is a subrange [found_element, end_of_range)
    std::cout << "Elements from last 7 to end: ";
    for (int x : result) std::cout << x << " ";  // 7 1
    std::cout << "\n";

    // Not found case
    auto r2 = std::ranges::find_last(data, 99);
    std::cout << "99 found: " << std::boolalpha << !r2.empty() << "\n";  // false

    // Works with strings
    std::string path = "/usr/local/bin/gcc";
    auto slash = std::ranges::find_last(path, '/');
    if (!slash.empty()) {
        // Everything after the last '/'
        std::string filename(slash.begin() + 1, slash.end());
        std::cout << "Filename: " << filename << "\n";  // gcc
    }

    return 0;
}
```

The string path example is a great illustration of why `subrange` is the right return type here: you naturally want to work with the portion after the last slash, and the subrange gives you exactly that.

### Q2: Show that find_last is more efficient than reversing the range and using find

This example puts the pre-C++23 approach side by side with the C++23 approach so you can see exactly how the reverse iterator conversion was done - and why it's easy to get wrong:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <ranges>

int main() {
    std::vector<int> data = {1, 2, 3, 2, 5, 2, 7, 8, 9};

    // === Method 1: Pre-C++23 - reverse iterators (awkward) ===
    {
        auto rit = std::find(data.rbegin(), data.rend(), 2);
        if (rit != data.rend()) {
            // Convert reverse_iterator to forward index
            auto idx = std::distance(data.begin(), rit.base()) - 1;
            std::cout << "Reverse find: index " << idx << "\n";  // 5
        }
        // Problem: reverse_iterator -> forward conversion is confusing
        // rit.base() points one PAST the found element
    }

    // === Method 2: C++23 find_last (clean) ===
    {
        auto result = std::ranges::find_last(data, 2);
        auto idx = std::distance(data.begin(), result.begin());
        std::cout << "find_last:    index " << idx << "\n";  // 5
        // Clean, no iterator conversion needed
    }

    // === find_last_if: last element satisfying a predicate ===
    {
        auto result = std::ranges::find_last_if(data, [](int n) {
            return n % 2 == 0;  // last even number
        });
        if (!result.empty()) {
            std::cout << "Last even: " << *result.begin()
                      << " at index " << std::distance(data.begin(), result.begin())
                      << "\n";  // 8 at index 7
        }
    }

    // === Efficiency ===
    // For bidirectional ranges: find_last iterates backward - O(n) worst case
    // For forward-only ranges: find_last must iterate forward tracking last match - O(n)
    // Both are O(n), but find_last avoids the confusing reverse_iterator dance

    // === find_last_if_not ===
    {
        auto result = std::ranges::find_last_if_not(data, [](int n) {
            return n < 5;  // last element >= 5
        });
        std::cout << "Last >= 5: " << *result.begin() << "\n";  // 9
    }

    return 0;
}
```

Both approaches are O(n), so there's no performance difference. The win is entirely in clarity and correctness - you no longer have to remember the off-by-one in `rit.base()`.

### Q3: Implement a split_back function using find_last_if to split on the last delimiter

This demonstrates a practical real-world use: splitting a string at the last occurrence of a delimiter, which is exactly what you need for path manipulation and extension extraction:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <algorithm>
#include <vector>
#include <utility>

// Split a string at the LAST occurrence of a delimiter character
auto split_back(std::string_view sv, char delim)
    -> std::pair<std::string_view, std::string_view>
{
    auto result = std::ranges::find_last(sv, delim);
    if (result.empty()) {
        return {sv, {}};  // no delimiter found: entire string is "before"
    }
    auto pos = std::distance(sv.begin(), result.begin());
    return {sv.substr(0, pos), sv.substr(pos + 1)};
}

// Split on last predicate match
auto split_back_if(std::string_view sv, auto pred)
    -> std::pair<std::string_view, std::string_view>
{
    auto result = std::ranges::find_last_if(sv, pred);
    if (result.empty()) {
        return {sv, {}};
    }
    auto pos = std::distance(sv.begin(), result.begin());
    return {sv.substr(0, pos), sv.substr(pos + 1)};
}

int main() {
    // === File path splitting ===
    auto [dir, file] = split_back("/usr/local/bin/gcc", '/');
    std::cout << "Dir:  '" << dir << "'\n";   // /usr/local/bin
    std::cout << "File: '" << file << "'\n";  // gcc

    // === Extension extraction ===
    auto [name, ext] = split_back("archive.tar.gz", '.');
    std::cout << "\nName: '" << name << "'\n";  // archive.tar
    std::cout << "Ext:  '" << ext << "'\n";     // gz

    // === Split on last whitespace ===
    auto [prefix, last_word] = split_back_if("the quick brown fox", [](char c) {
        return c == ' ';
    });
    std::cout << "\nPrefix:    '" << prefix << "'\n";     // the quick brown
    std::cout << "Last word: '" << last_word << "'\n";    // fox

    // === No delimiter ===
    auto [all, none] = split_back("nodots", '.');
    std::cout << "\nAll:  '" << all << "'\n";   // nodots
    std::cout << "None: '" << none << "'\n";    // (empty)

    return 0;
}
```

The extension extraction example `"archive.tar.gz"` is a good test case: there are two dots, and you want everything after the *last* one. `find_last` gets you there naturally.

---

## Notes

- **C++23 required.** Compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- `find_last` returns a **`subrange`**, not an iterator. Use `.begin()` to get the iterator to the found element, or `.empty()` to check if anything was found.
- Supports **projections**: `find_last(people, "Alice", &Person::name)`.
- For **bidirectional** ranges, the implementation can traverse backward efficiently. For **forward-only** ranges, it must scan forward while remembering the last match.
- If you need just the iterator (like classic `find`), use `result.begin()`.
