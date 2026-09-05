# Know the difference between std::string::substr returning a copy vs string_view::substr returning a view

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string_view/substr>  

---

## Topic Overview

### The Key Difference

The two `substr` methods share the same name but behave completely differently. One copies data onto the heap; the other just adjusts a pointer.

| Method | Returns | Allocates? | Cost |
| --- | --- | --- | --- |
| `std::string::substr(pos, len)` | A new `std::string` (copy) | Yes - always heap allocates (unless SSO) | O(n) where n = substring length |
| `std::string_view::substr(pos, len)` | A new `std::string_view` (pointer + length) | No - just adjusts pointer/length | O(1) constant time |

Here's the core idea in code - same call site, very different runtime behavior:

```cpp
std::string s = "Hello, World!";
std::string_view sv = s;

auto sub1 = s.substr(7, 5);    // Returns std::string "World" - ALLOCATES
auto sub2 = sv.substr(7, 5);   // Returns std::string_view pointing into s - ZERO COST

// sub1 owns its own memory (independent copy)
// sub2 is just a pointer into s's buffer (dependent - s must stay alive!)
```

`sub2` is essentially two numbers: a pointer and a length. It's incredibly cheap, but it's also borrowed from `s`.

### When to Use Each

If the table feels like a lot, it boils down to one question: do you need to own the data, or just read it?

| Scenario | Use |
| --- | --- |
| Need a persistent independent copy | `std::string::substr` |
| Parsing, tokenizing, searching (read-only) | `std::string_view::substr` |
| Returning from a function where source may die | `std::string::substr` |
| Hot loop processing substrings | `std::string_view::substr` |

### The Dangling View Danger

`string_view` does NOT own its data. If the underlying string is destroyed, the view dangles. This is the most common mistake people make with `string_view`:

```cpp
std::string_view get_name() {
    std::string temp = "Alice";
    return temp;  // DANGLING! temp is destroyed, view points to freed memory
}
```

The compiler may not warn you. The program may even appear to work sometimes. Don't rely on it.

---

## Self-Assessment

### Q1: Show that `std::string::substr` always allocates while `std::string_view::substr` is zero-cost

This benchmark makes the performance gap concrete. Watch how the same loop runs orders of magnitude faster when you switch from `string` to `string_view`:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <chrono>
#include <algorithm>
#include <type_traits>

int main() {
    std::string original(10000, 'x');  // 10KB string
    std::string_view sv = original;

    // Measure string::substr (allocating)
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 1'000'000; ++i) {
        auto sub = original.substr(100, 500);  // Allocates 500-char string each time
        (void)sub;
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    // Measure string_view::substr (zero-copy)
    auto t3 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 1'000'000; ++i) {
        auto sub = sv.substr(100, 500);  // Just pointer arithmetic - no allocation
        (void)sub;
    }
    auto t4 = std::chrono::high_resolution_clock::now();

    auto ms1 = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();
    auto ms2 = std::chrono::duration_cast<std::chrono::microseconds>(t4 - t3).count();

    std::cout << "string::substr 1M times:      " << ms1 << " us\n";
    std::cout << "string_view::substr 1M times:  " << ms2 << " us\n";
    std::cout << "Speedup: ~" << ms1 / std::max(ms2, 1L) << "x\n";

    // Typical output:
    // string::substr 1M times:      45000 us
    // string_view::substr 1M times: 150 us
    // Speedup: ~300x

    // Proof that types differ:
    auto copy = original.substr(0, 5);  // std::string
    auto view = sv.substr(0, 5);        // std::string_view

    static_assert(std::is_same_v<decltype(copy), std::string>);
    static_assert(std::is_same_v<decltype(view), std::string_view>);

    // string owns its data - modify original, copy is unaffected:
    original[0] = 'A';
    std::cout << "copy[0]: " << copy[0] << "\n";  // 'x' (independent copy)
    std::cout << "view[0]: " << view[0] << "\n";  // 'A' (reflects original's change!)

    return 0;
}
```

Notice the last two lines - they nail down the ownership difference. The copy doesn't see the change; the view does, because it was never its own data.

- `string::substr` calls `new` internally (or uses SSO for small strings), copies characters, and returns an independent `std::string`. This is O(n) with heap allocation.
- `string_view::substr` simply constructs a new `string_view{data() + pos, len}` - two pointer operations, O(1), zero allocation.
- The view reflects changes to the original (it's not a copy), while the string copy is independent.

### Q2: Demonstrate a dangling view bug from calling `string_view::substr` on a temporary string

These are real bugs. The compiler usually won't stop you from writing them:

```cpp
#include <iostream>
#include <string>
#include <string_view>

// BUG 1: Returning a view of a local string
std::string_view get_extension_BUGGY(const std::string& filename) {
    std::string lower = filename;  // Local copy
    // ... transform to lowercase ...
    auto dot = lower.rfind('.');
    return std::string_view(lower).substr(dot);  // DANGLING! lower destroyed at return
}

// BUG 2: View of a temporary
std::string_view trim_buggy() {
    return std::string_view(std::string("  hello  ")).substr(2, 5);
    // The temporary std::string is destroyed at the semicolon
    // The returned string_view points to freed memory!
}

// BUG 3: View of a concatenation result
void process_buggy() {
    std::string a = "hello";
    std::string b = " world";
    std::string_view sv = a + b;  // a + b creates a temporary std::string
    // Temporary dies at end of this statement
    std::cout << sv << "\n";  // UB! Dangling view
}

// CORRECT versions:
std::string get_extension_SAFE(const std::string& filename) {
    auto dot = filename.rfind('.');
    return filename.substr(dot);  // Returns std::string (owning copy)
}

std::string_view get_extension_view_SAFE(std::string_view filename) {
    // Safe IF caller ensures filename outlives the return value
    auto dot = filename.rfind('.');
    return filename.substr(dot);  // View into the caller's data
}

void process_safe() {
    std::string a = "hello";
    std::string b = " world";
    std::string result = a + b;         // Store the concatenation
    std::string_view sv = result;       // View into stored result
    std::cout << sv << "\n";            // OK - result is alive
}

int main() {
    // Safe usage:
    std::string file = "document.pdf";
    auto ext = get_extension_SAFE(file);
    std::cout << "Extension: " << ext << "\n";  // ".pdf"

    auto ext_view = get_extension_view_SAFE(file);
    std::cout << "Extension view: " << ext_view << "\n";  // ".pdf" (valid - file is alive)

    process_safe();

    return 0;
}
```

The safe versions follow a simple mental model: if you're going to return a view, the caller must own the underlying data. If the callee owns the data, return a `std::string`.

Key rules to avoid dangling views:

1. Never return `string_view` pointing to a local variable.
2. Never create `string_view` from a temporary `std::string` (dies at end of statement).
3. Ensure the source string outlives all views into it.
4. When in doubt, return `std::string` (safe but slower).

### Q3: Use `string_view::substr` in a parser hot path

Here's where `string_view::substr` really pays off. A CSV parser that does zero allocations during the parse itself:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <vector>
#include <charconv>

// CSV parser using string_view - zero allocations during parsing!
struct CSVParser {
    std::string_view data;

    std::vector<std::vector<std::string_view>> parse() {
        std::vector<std::vector<std::string_view>> rows;
        std::size_t pos = 0;

        while (pos < data.size()) {
            std::vector<std::string_view> row;
            std::size_t line_end = data.find('\n', pos);
            if (line_end == std::string_view::npos) line_end = data.size();

            std::string_view line = data.substr(pos, line_end - pos);

            // Split line by commas - all string_view::substr, zero allocation
            std::size_t col_start = 0;
            while (col_start < line.size()) {
                std::size_t comma = line.find(',', col_start);
                if (comma == std::string_view::npos) comma = line.size();
                row.push_back(line.substr(col_start, comma - col_start));
                col_start = comma + 1;
            }

            rows.push_back(std::move(row));
            pos = line_end + 1;
        }
        return rows;
    }
};

// Simple key=value config parser
std::pair<std::string_view, std::string_view> parse_kv(std::string_view line) {
    auto eq = line.find('=');
    if (eq == std::string_view::npos) return {line, {}};
    return {line.substr(0, eq), line.substr(eq + 1)};  // ZERO allocation
}

int main() {
    // CSV parsing - the CSV data string is kept alive throughout
    std::string csv_data = "name,age,city\nAlice,30,NYC\nBob,25,LA\nCharlie,35,Chicago\n";
    CSVParser parser{csv_data};
    auto table = parser.parse();

    for (const auto& row : table) {
        for (const auto& cell : row) {
            std::cout << "[" << cell << "] ";
        }
        std::cout << "\n";
    }
    // [name] [age] [city]
    // [Alice] [30] [NYC]
    // [Bob] [25] [LA]
    // [Charlie] [35] [Chicago]

    // Key-value parsing
    std::string config = "timeout=30";
    auto [key, value] = parse_kv(config);
    std::cout << "Key: " << key << ", Value: " << value << "\n";
    // Key: timeout, Value: 30

    return 0;
}
```

Notice the key discipline: `csv_data` stays alive for the entire lifetime of `table`. All the views inside `table` point into that one string. The moment `csv_data` is gone, all those views dangle.

Allocation comparison:

- With `std::string::substr`: N rows x M columns x average cell length bytes allocated. For a 1MB CSV: thousands of allocations.
- With `std::string_view::substr`: Zero allocations for substring operations. Only the `vector` grows. The entire parse operates on views into the original data.

---

## Additional Examples

### C++23: `std::string::substr` Rvalue Overload

C++23 adds an rvalue-qualified `substr` that moves instead of copying - you get the efficiency of a view but still own the data:

```cpp
// C++23
std::string s = "Hello World";
auto sub = std::move(s).substr(6);  // Moves the buffer, no allocation!
// sub == "World", s is in moved-from state
```

---

## Notes

- `string::substr` = safe (owns data) but expensive (allocates).
- `string_view::substr` = fast (O(1)) but dangerous (dangling if source dies).
- In parser hot paths, `string_view::substr` can be 100-300x faster.
- Always ensure the source string outlives all `string_view` references.
- C++23 adds rvalue `string::substr` that moves the buffer - best of both worlds.
