# Use std::search with Boyer-Moore searcher for fast string pattern matching

**Category:** Standard Library Algorithms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/search>  

---

## Topic Overview

C++17 introduced **searcher objects** that can be passed to `std::search` for optimized pattern matching. The idea is to separate preprocessing (which depends only on the pattern) from searching (which depends on the text), so you can preprocess once and search many times efficiently.

| Searcher | Algorithm | Preprocessing | Average Case | Worst Case |
| --- | --- | --- | --- | --- |
| `default_searcher` | Naive | O(1) | O(n*m) | O(n*m) |
| `boyer_moore_searcher` | Boyer-Moore | O(m + s) | O(n/m) | O(n*m) |
| `boyer_moore_horspool_searcher` | Horspool | O(s) | O(n/m) | O(n*m) |

Where n = text length, m = pattern length, s = alphabet size.

### Syntax

Construct the searcher once from the pattern, then pass it to `std::search` as many times as you need:

```cpp
#include <algorithm>
#include <functional>

// Construct searcher (preprocesses the pattern)
auto searcher = std::boyer_moore_searcher(pat.begin(), pat.end());

// Use with std::search
auto it = std::search(text.begin(), text.end(), searcher);
```

### Why Boyer-Moore is Fast

Boyer-Moore skips characters by using two heuristics that let it jump past large chunks of text without examining every character:

1. **Bad character rule:** When a mismatch occurs, skip ahead based on where that character appears in the pattern.
2. **Good suffix rule:** Skip based on matching suffixes already seen.

This gives **O(n/m) average case** - often examining only a fraction of the text. With a 20-character pattern, the algorithm may skip 19 characters at a time through most of the text.

---

## Self-Assessment

### Q1: Use std::search with a Boyer-Moore searcher (C++17) for O(n/m) average-case string search

The first thing to notice is that you construct the `bm` searcher object outside the loop. All the pattern preprocessing happens at construction time, so reusing the same searcher object for multiple searches amortizes that cost.

```cpp
#include <iostream>
#include <string>
#include <algorithm>
#include <functional>

int main() {
    std::string text = "The quick brown fox jumps over the lazy dog. "
                       "The quick brown fox is clever.";
    std::string pattern = "brown fox";

    // === Boyer-Moore searcher ===
    auto bm = std::boyer_moore_searcher(pattern.begin(), pattern.end());

    auto it = std::search(text.begin(), text.end(), bm);

    if (it != text.end()) {
        auto pos = std::distance(text.begin(), it);
        std::cout << "Found '" << pattern << "' at position " << pos << "\n";
        // Found 'brown fox' at position 10
    }

    // === Find ALL occurrences ===
    std::cout << "\nAll occurrences of '" << pattern << "':\n";
    auto search_from = text.begin();
    while (search_from != text.end()) {
        auto found = std::search(search_from, text.end(), bm);
        if (found == text.end()) break;

        auto pos = std::distance(text.begin(), found);
        std::cout << "  Position " << pos << "\n";
        search_from = found + 1;  // move past match
    }
    // Position 10, Position 55

    // === Boyer-Moore-Horspool (simpler, slightly less optimal) ===
    auto bmh = std::boyer_moore_horspool_searcher(pattern.begin(), pattern.end());
    auto it2 = std::search(text.begin(), text.end(), bmh);
    if (it2 != text.end()) {
        std::cout << "\nHorspool found at "
                  << std::distance(text.begin(), it2) << "\n";
    }

    // === Not found ===
    std::string missing = "elephant";
    auto bm_miss = std::boyer_moore_searcher(missing.begin(), missing.end());
    auto found = std::search(text.begin(), text.end(), bm_miss);
    std::cout << "\n'" << missing << "' found: "
              << std::boolalpha << (found != text.end()) << "\n";  // false

    return 0;
}
```

The "find all occurrences" loop reuses the same `bm` object for every iteration - that's the pattern you want to use in real code. Each call to `std::search` returns an iterator to the next match, and you advance `search_from` past it to continue.

### Q2: Compare std::search with std::string::find for a pattern search over a large corpus

`string::find` is often heavily optimized by the standard library (SIMD instructions on modern x86), so it competes well for short patterns. Boyer-Moore tends to win for longer patterns or when you need to search the same pattern across many texts.

```cpp
#include <iostream>
#include <string>
#include <algorithm>
#include <functional>
#include <chrono>

int main() {
    // Build a large text ~1MB
    std::string base = "Lorem ipsum dolor sit amet, consectetur adipiscing elit. ";
    std::string text;
    text.reserve(1'000'000);
    while (text.size() < 1'000'000) text += base;
    text += "UNIQUE_PATTERN_HERE";  // add pattern at the end

    std::string pattern = "UNIQUE_PATTERN_HERE";

    // === Method 1: string::find ===
    auto t1 = std::chrono::high_resolution_clock::now();
    auto pos1 = text.find(pattern);
    auto t2 = std::chrono::high_resolution_clock::now();
    auto us_find = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    // === Method 2: std::search (default) ===
    t1 = std::chrono::high_resolution_clock::now();
    auto it2 = std::search(text.begin(), text.end(),
                           pattern.begin(), pattern.end());
    t2 = std::chrono::high_resolution_clock::now();
    auto us_search = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    // === Method 3: Boyer-Moore ===
    auto bm = std::boyer_moore_searcher(pattern.begin(), pattern.end());
    t1 = std::chrono::high_resolution_clock::now();
    auto it3 = std::search(text.begin(), text.end(), bm);
    t2 = std::chrono::high_resolution_clock::now();
    auto us_bm = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    std::cout << "Text size: " << text.size() << " chars\n";
    std::cout << "Pattern: \"" << pattern << "\" (" << pattern.size() << " chars)\n\n";

    std::cout << "string::find:     " << us_find << " us  pos=" << pos1 << "\n";
    std::cout << "search(default):  " << us_search << " us  pos="
              << std::distance(text.begin(), it2) << "\n";
    std::cout << "search(BM):       " << us_bm << " us  pos="
              << std::distance(text.begin(), it3) << "\n";

    // === When to use which ===
    // string::find: simple, good for occasional searches
    // Boyer-Moore: best when searching for the same pattern repeatedly
    //   (preprocessing cost amortized over multiple searches)
    // Boyer-Moore shines with long patterns and large alphabets

    return 0;
}
```

The pattern is placed at the very end to force the worst case for naive search: it has to scan the entire text before finding the match. Boyer-Moore can skip large sections using its heuristics even in this scenario, because the pattern characters do not appear in the Lorem ipsum filler.

### Q3: Show std::search_n finding N consecutive equal elements in a range

`search_n` is a separate algorithm - completely unrelated to Boyer-Moore - but it fits naturally in this topic because it uses the same `std::search`-family interface for a different kind of pattern: finding a run of N identical values.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    // === search_n: find N consecutive equal elements ===
    std::vector<int> data = {1, 2, 2, 2, 3, 4, 4, 4, 4, 5, 5};

    // Find 3 consecutive 2's
    auto it = std::search_n(data.begin(), data.end(), 3, 2);
    if (it != data.end()) {
        std::cout << "Found 3 consecutive 2's at index "
                  << std::distance(data.begin(), it) << "\n";  // index 1
    }

    // Find 4 consecutive 4's
    it = std::search_n(data.begin(), data.end(), 4, 4);
    if (it != data.end()) {
        std::cout << "Found 4 consecutive 4's at index "
                  << std::distance(data.begin(), it) << "\n";  // index 5
    }

    // Find 3 consecutive 5's - not enough
    it = std::search_n(data.begin(), data.end(), 3, 5);
    std::cout << "3 consecutive 5's found: " << std::boolalpha
              << (it != data.end()) << "\n";  // false

    // === search_n with predicate ===
    // Find 3 consecutive elements > 3
    it = std::search_n(data.begin(), data.end(), 3, 3,
        [](int elem, int value) { return elem > value; });
    if (it != data.end()) {
        std::cout << "\n3 consecutive elements > 3 at index "
                  << std::distance(data.begin(), it) << ": ";
        for (int i = 0; i < 3; ++i) std::cout << *(it + i) << " ";
        std::cout << "\n";  // 4 4 4 at index 5
    }

    // === Practical: detect repeated events ===
    std::string log = "..OK..OK..FAIL..FAIL..FAIL..OK..";
    // Find 3 consecutive '.' (a gap pattern)
    auto sit = std::search_n(log.begin(), log.end(), 3, '.');
    if (sit != log.end()) {
        std::cout << "\nFound 3 dots at position "
                  << std::distance(log.begin(), sit) << "\n";
    }

    return 0;
}
```

The predicate overload is the most flexible form: instead of checking for exact equality, you supply a binary predicate `(element, value)` that defines what "matches" means. This makes `search_n` useful for finding runs of elements that satisfy any condition.

---

## Notes

- **Boyer-Moore preprocessing** takes O(m + s) time and space. Construct the searcher **once** and reuse it for multiple searches over different texts.
- `std::search` with no searcher argument uses a naive O(n*m) algorithm.
- `string::find` is often highly optimized by the standard library (SSE4.2 on x86), so Boyer-Moore only wins for **long patterns** or **repeated searches**.
- **`search_n`** is unrelated to Boyer-Moore - it finds N consecutive copies of a value using a simple scan.
- All searchers are in `<functional>` (not `<algorithm>`).
