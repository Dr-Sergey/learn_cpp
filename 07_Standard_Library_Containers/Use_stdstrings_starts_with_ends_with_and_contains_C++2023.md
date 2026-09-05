# Use std::string's starts_with, ends_with, and contains (C++20/23)

**Category:** Standard Library Containers  
**Standard:** C++20, C++23 (`starts_with`, `ends_with`)  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string/starts_with>  

---

## Topic Overview

C++20 added `starts_with()` and `ends_with()` to `std::string` and `std::string_view`, and C++23 followed up with `contains()`. These member functions replace a collection of awkward, error-prone idioms with clear, readable checks that communicate intent at a glance. If you've ever written `s.find(prefix) == 0` and squinted at it wondering what it means, these are the functions you were waiting for.

### Before vs After

If the table feels like a lot, the headline is simple: the old idioms are either inefficient, verbose, or easy to get subtly wrong. The new functions are none of those things.

| Check | Old way (pre-C++20) | New way |
| --- | --- | --- |
| Starts with prefix | `s.substr(0, prefix.size()) == prefix` | `s.starts_with(prefix)` |
| | `s.find(prefix) == 0` | |
| | `s.compare(0, prefix.size(), prefix) == 0` | |
| Ends with suffix | `s.size() >= suffix.size() && s.compare(s.size()-suffix.size(), suffix.size(), suffix) == 0` | `s.ends_with(suffix)` |
| Contains substring | `s.find(sub) != std::string::npos` | `s.contains(sub)` (C++23) |

### API Summary

Here's the full picture of what you can call and what each overload accepts:

```cpp
#include <string>
#include <string_view>

std::string s = "Hello, World!";

// === starts_with (C++20) ===
s.starts_with("Hello");    // true
s.starts_with("World");    // false
s.starts_with('H');        // true  (single char overload)
s.starts_with("");         // true  (empty prefix always matches)

// === ends_with (C++20) ===
s.ends_with("World!");     // true
s.ends_with("Hello");      // false
s.ends_with('!');          // true

// === contains (C++23) ===
s.contains("lo, W");       // true
s.contains("xyz");         // false
s.contains(',');           // true

// All three also work on std::string_view
std::string_view sv = s;
sv.starts_with("Hello");   // true
sv.contains("World");      // true
```

### Overloads

Each function comes in three flavors, so you can pass whatever you naturally have on hand:

1. `starts_with(string_view sv)` — check against a string/view
2. `starts_with(charT c)` — check against a single character
3. `starts_with(const charT* s)` — check against a C-string

---

## Self-Assessment

### Q1: Replace a substr(0, prefix.size()) == prefix check with str.starts_with(prefix)

The three "old way" functions below each represent a real pattern you'll find in existing code. Pay attention to the comments - each has a genuine downside that `starts_with` avoids.

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <vector>

// === Old way: multiple approaches, all error-prone ===
bool starts_with_old_1(const std::string& s, const std::string& prefix) {
    return s.substr(0, prefix.size()) == prefix;
    // Problem: creates a temporary string (allocation!)
}

bool starts_with_old_2(const std::string& s, const std::string& prefix) {
    return s.find(prefix) == 0;
    // Problem: scans ENTIRE string looking for prefix anywhere, then checks pos==0
    // O(n*m) when O(m) would suffice
}

bool starts_with_old_3(const std::string& s, const std::string& prefix) {
    return s.size() >= prefix.size() &&
           s.compare(0, prefix.size(), prefix) == 0;
    // Correct and efficient, but verbose and easy to get wrong
}

int main() {
    std::vector<std::string> paths = {
        "/usr/local/bin/gcc",
        "/usr/share/doc/readme.txt",
        "/home/user/.config/settings",
        "/tmp/scratch_001",
        "/usr/local/lib/libfoo.so"
    };

    std::string prefix = "/usr/local/";

    // === Old way ===
    std::cout << "Old way (substr):\n";
    for (const auto& p : paths) {
        if (p.substr(0, prefix.size()) == prefix)
            std::cout << "  " << p << "\n";
    }

    // === New way (C++20) ===
    std::cout << "\nNew way (starts_with):\n";
    for (const auto& p : paths) {
        if (p.starts_with(prefix))
            std::cout << "  " << p << "\n";
    }
    // Both output:
    //   /usr/local/bin/gcc
    //   /usr/local/lib/libfoo.so

    // === ends_with for file extensions ===
    std::cout << "\n.so files (ends_with):\n";
    for (const auto& p : paths) {
        if (p.ends_with(".so"))
            std::cout << "  " << p << "\n";
    }
    //   /usr/local/lib/libfoo.so

    // === Edge cases ===
    std::string s = "Hello";
    std::cout << std::boolalpha;
    std::cout << "\nEdge cases:\n";
    std::cout << "  starts_with(\"\"):     " << s.starts_with("") << "\n";       // true
    std::cout << "  starts_with(\"Hello\"): " << s.starts_with("Hello") << "\n"; // true (exact match)
    std::cout << "  starts_with(\"HelloX\"): " << s.starts_with("HelloX") << "\n"; // false (prefix longer)
    std::cout << "  ends_with(\"\"):       " << s.ends_with("") << "\n";         // true

    return 0;
}
```

Both the old and new loops produce identical results, but the new version tells a clearer story. Notice the edge cases too - an empty prefix or suffix always returns `true`, which is the mathematically correct behavior (every string trivially starts and ends with nothing).

- `starts_with(prefix)` compares the first `prefix.size()` characters and returns `false` immediately if the string is shorter than the prefix - no wasted work.
- Unlike `substr()`, it doesn't create a temporary string and incur a heap allocation.
- Unlike `find(prefix) == 0`, it doesn't scan the entire string looking for the pattern anywhere - it only looks at position zero.
- Complexity: O(min(size(), prefix.size())) - it stops as soon as it can.

### Q2: Use contains() for a simple substring search and show its readability advantage

The readability argument for `contains()` is real. The old idiom `find() != npos` requires you to remember a magic constant, mentally flip a double negation, and know that `npos` means "not found." `contains()` just says what you mean.

```cpp
#include <iostream>
#include <string>
#include <vector>

int main() {
    // === C++23 contains() ===
    std::vector<std::string> log_lines = {
        "[INFO] Server started on port 8080",
        "[ERROR] Connection refused: timeout",
        "[WARN] Disk usage at 85%",
        "[ERROR] Out of memory",
        "[INFO] Request processed in 42ms",
        "[DEBUG] Cache hit ratio: 0.95"
    };

    // === Old way: find() != npos ===
    std::cout << "Errors (old way):\n";
    for (const auto& line : log_lines) {
        if (line.find("[ERROR]") != std::string::npos)
            std::cout << "  " << line << "\n";
    }

    // === New way: contains() (C++23) ===
    std::cout << "\nErrors (C++23 contains):\n";
    for (const auto& line : log_lines) {
        if (line.contains("[ERROR]"))
            std::cout << "  " << line << "\n";
    }
    // Both output:
    //   [ERROR] Connection refused: timeout
    //   [ERROR] Out of memory

    // === Readability comparison ===
    std::string url = "https://example.com/api/v2/users?active=true";

    // Old: double negation, magic constant
    if (url.find("api/v2") != std::string::npos) { /* ... */ }
    if (url.find('?') != std::string::npos) { /* ... */ }

    // New: reads like English
    if (url.contains("api/v2")) { /* ... */ }
    if (url.contains('?')) { /* ... */ }

    // === Combining all three ===
    std::cout << "\nURL analysis:\n";
    std::cout << std::boolalpha;
    std::cout << "  HTTPS:      " << url.starts_with("https://") << "\n";  // true
    std::cout << "  Has query:  " << url.contains('?') << "\n";            // true
    std::cout << "  Is JSON:    " << url.ends_with(".json") << "\n";       // false
    std::cout << "  Has API v2: " << url.contains("api/v2") << "\n";      // true

    return 0;
}
```

When you use all three together in the URL analysis block, notice how the conditions read almost like a requirements checklist. That's the goal. Under the hood, `contains(x)` is equivalent to `find(x) != npos` - same semantics, just expressed clearly.

- `contains(x)` is semantically equivalent to `find(x) != npos` - no behavioral difference, only readability.
- Available for both `std::string` and `std::string_view`.
- `contains()` is C++23-only. If you're on C++20 or earlier, you'll need `find() != npos` or a small helper function.

### Q3: Show performance considerations: contains is O(n*m) — use find or Boyer-Moore for large text

For most use cases `contains()` is perfectly fast. The case where you need to think is when you're searching megabyte-sized strings with patterns longer than a few characters. Here's a benchmark that demonstrates the difference.

```cpp
#include <iostream>
#include <string>
#include <algorithm>
#include <functional>
#include <chrono>

int main() {
    // === Performance characteristics ===
    //
    // starts_with(prefix):  O(m)     - compare m chars at position 0
    // ends_with(suffix):    O(m)     - compare m chars at end
    // contains(pattern):    O(n*m)   - find() internally (naive search)
    //   where n = string length, m = pattern length
    //
    // For SHORT strings and patterns, this is perfectly fine.
    // For LARGE text with LONG patterns, consider alternatives.

    // === Generating test data ===
    std::string large_text(1'000'000, 'a');  // 1M chars
    large_text += "NEEDLE";                   // pattern at the very end

    std::string pattern = "NEEDLE";

    // === Method 1: contains / find (naive) ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        bool found = large_text.contains(pattern);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "contains():     found=" << found << "  time=" << us << " us\n";
    }

    // === Method 2: std::search with default (may use better algorithm) ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        auto it = std::search(large_text.begin(), large_text.end(),
                              pattern.begin(), pattern.end());
        bool found = (it != large_text.end());
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "std::search:    found=" << found << "  time=" << us << " us\n";
    }

    // === Method 3: Boyer-Moore (C++17) - O(n/m) best case ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        auto it = std::search(large_text.begin(), large_text.end(),
                              std::boyer_moore_searcher(pattern.begin(), pattern.end()));
        bool found = (it != large_text.end());
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "Boyer-Moore:    found=" << found << "  time=" << us << " us\n";
    }

    // === Method 4: Boyer-Moore-Horspool (C++17) - simpler, often faster ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        auto it = std::search(large_text.begin(), large_text.end(),
                              std::boyer_moore_horspool_searcher(pattern.begin(), pattern.end()));
        bool found = (it != large_text.end());
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "BM-Horspool:    found=" << found << "  time=" << us << " us\n";
    }

    // === When to use what ===
    std::cout << "\n=== Guidelines ===\n";
    std::cout << "Use starts_with/ends_with: Always. O(m), no alternatives needed.\n";
    std::cout << "Use contains/find:         For short strings (< few KB). Simple, fast enough.\n";
    std::cout << "Use Boyer-Moore:           For large text (MB+) with long patterns.\n";
    std::cout << "Use regex:                 Only when you need pattern matching, not substring search.\n";

    // === Complexity summary ===
    // | Algorithm            | Preprocessing | Search      | Best case   |
    // |---------------------|---------------|-------------|-------------|
    // | Naive (find/contains)| O(1)         | O(n*m)      | O(n)        |
    // | Boyer-Moore          | O(m + sigma) | O(n*m) worst| O(n/m)      |
    // | Boyer-Moore-Horspool | O(sigma)     | O(n*m) worst| O(n/m)      |
    // | KMP                  | O(m)         | O(n)        | O(n)        |
    //
    // sigma = alphabet size, n = text length, m = pattern length

    return 0;
}
```

The practical takeaway from this benchmark: `contains()` and `find()` use a naive search that is O(n×m) in the worst case, but for natural text and short patterns it typically runs in O(n) because mismatches happen quickly. Boyer-Moore (available since C++17 via `std::search`) preprocesses the pattern to skip large chunks of the text. The longer your pattern, the bigger the speedup - that's why its best case is O(n/m). Boyer-Moore-Horspool is a simplified variant with slightly less preprocessing overhead, and it often matches Boyer-Moore in practice.

- `starts_with` and `ends_with` are always O(m) regardless of string length - no performance concern there.
- Use `contains()` freely for strings in the kilobyte range. Only reach for `std::search` with a Boyer-Moore searcher when you're processing megabyte-scale text with meaningful pattern lengths.

---

## Notes

- **`starts_with` / `ends_with`** are C++20. **`contains`** is C++23.
- All three are `const noexcept` - no exceptions, no allocations, safe to use anywhere.
- Available on both `std::string` and `std::string_view` (and `std::basic_string<CharT>`).
- **Empty patterns:** `starts_with("")`, `ends_with("")`, and `contains("")` all return `true` - consistent with the mathematical definition of prefix/suffix/substring.
- **Case-insensitive:** None of these functions support case-insensitive comparison. For that, convert both strings to lowercase first, or use `std::ranges::equal` with a custom comparator.
- **`string_view` advantage:** These functions take `string_view`, so you can pass `const char*`, `string`, or `string_view` without triggering temporary allocations.
- **Compiler support:** `starts_with`/`ends_with` - GCC 11+, Clang 12+, MSVC 19.30+. `contains` - GCC 13+, Clang 17+, MSVC 19.34+.
