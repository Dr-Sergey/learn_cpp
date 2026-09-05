# Use std::to_chars and std::from_chars for allocation-free conversions

**Category:** Standard Library Utilities  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/charconv/to_chars>  

---

## Topic Overview

`std::to_chars` and `std::from_chars` (header `<charconv>`, C++17) convert numbers to/from character sequences **without heap allocation, exceptions, or locale overhead**. They are the fastest standard number conversion functions in C++.

The key insight is that most number-to-string conversions in performance-sensitive code don't need a full `std::string` - they just need some characters in a buffer. These functions give you exactly that, using a caller-provided buffer and returning a pointer-past-end so you know how many characters were written.

### Comparison with Alternatives

| Feature | `std::to_string` / `stoi` | `snprintf` / `sscanf` | `to_chars` / `from_chars` |
| --- | --- | --- | --- |
| Allocates | Yes (`std::string`) | No (user buffer) | No (user buffer) |
| Throws | Yes | No | No |
| Locale-dependent | Yes | Yes | No (always `"C"`) |
| Null-terminator | In result | Required | Not used |
| Thread-safe | Depends on locale | Depends on locale | Always |
| Round-trip guarantee | No | No | Yes (float/double) |

### API Summary

The API is intentionally minimal. You hand it pointers, it fills the buffer and tells you where it stopped.

```cpp
┌─────────────────────────────────────────────────────────┐
│  to_chars(first, last, value [, base/fmt/precision])    │
│  Returns: { ptr (past-end), ec (errc) }                │
│  On success: chars written to [first, ptr)              │
│  On failure: ec == errc::value_too_large                │
├─────────────────────────────────────────────────────────┤
│  from_chars(first, last, value [, base/fmt])            │
│  Returns: { ptr (past-parsed), ec (errc) }              │
│  On success: value is set, ptr past last parsed char    │
│  On failure: ec == errc::invalid_argument / out_of_range│
└─────────────────────────────────────────────────────────┘
```

### Core Examples

Here are the three most common patterns: integer to buffer, string_view to float, and hexadecimal output.

```cpp
#include <charconv>
#include <array>
#include <string_view>
#include <iostream>

int main() {
    // === to_chars: int -> stack buffer (ZERO allocation) ===
    std::array<char, 20> buf{};
    int num = 123456;
    auto [ptr, ec] = std::to_chars(buf.data(), buf.data() + buf.size(), num);
    // ptr points past the last written character
    std::string_view result(buf.data(), ptr - buf.data());
    std::cout << result << "\n"; // "123456"

    // === from_chars: string_view -> float (no std::string needed) ===
    std::string_view sv = "3.14159 extra";
    float f = 0.0f;
    auto [p2, e2] = std::from_chars(sv.data(), sv.data() + sv.size(), f);
    std::cout << f << "\n"; // 3.14159
    // p2 points to ' ' - stops at first non-numeric character

    // === Hexadecimal ===
    auto [p3, e3] = std::to_chars(buf.data(), buf.data() + buf.size(), 0xDEAD, 16);
    std::cout << std::string_view(buf.data(), p3) << "\n"; // "dead"
}
```

Notice that `from_chars` stops cleanly at the space in `"3.14159 extra"` and reports where it stopped via `p2`. This makes it very convenient for parsing tokens out of a larger buffer without copying.

---

## Self-Assessment

### Q1: Convert an int to chars using to_chars into a stack buffer and show no allocation occurs

**Answer:**

This example instruments `operator new` to prove that `to_chars` touches no heap memory at all.

```cpp
#include <charconv>
#include <array>
#include <string_view>
#include <iostream>
#include <cstdlib>
#include <cstddef>
#include <string>
#include <system_error>

// Custom global operator new to detect heap allocations
static int alloc_count = 0;
void* operator new(std::size_t size) {
    ++alloc_count;
    return std::malloc(size);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

int main() {
    alloc_count = 0;
    int before = alloc_count;

    // Stack buffer - NO heap allocation
    std::array<char, 20> buf{};
    int value = 987654321;

    auto [ptr, ec] = std::to_chars(buf.data(), buf.data() + buf.size(), value);

    int after = alloc_count;
    std::cout << "Allocations during to_chars: " << (after - before) << "\n";
    // Output: Allocations during to_chars: 0

    if (ec == std::errc{}) {
        std::string_view result(buf.data(), ptr - buf.data());
        std::cout << "Converted: " << result << "\n";
        // Output: Converted: 987654321
    }

    // Contrast with std::to_string which DOES allocate:
    before = alloc_count;
    std::string s = std::to_string(987654321); // heap alloc for std::string
    after = alloc_count;
    std::cout << "Allocations during to_string: " << (after - before) << "\n";
    // Output: Allocations during to_string: 1 (or more)

    // WHY allocation-free matters:
    // - Real-time systems: no unpredictable malloc latency
    // - Hot loops: converting millions of numbers without heap pressure
    // - Embedded: may not have a heap at all
}
```

`to_chars` writes directly into the caller-provided buffer (`std::array` on the stack). No `std::string` is constructed, no heap memory is touched. By instrumenting `operator new`, we can prove zero allocations occur during the conversion. `std::to_string`, by contrast, always allocates a `std::string`.

### Q2: Use from_chars to parse a float from a string_view without constructing a std::string

**Answer:**

The real power here is that `from_chars` operates on raw pointer ranges, so you can parse directly from `string_view` data - no null terminator and no copy needed.

```cpp
#include <charconv>
#include <string_view>
#include <iostream>
#include <vector>

// Simulates parsing a CSV line without ANY std::string construction
void parse_csv_line(std::string_view line) {
    std::vector<float> values;

    while (!line.empty()) {
        // Skip whitespace and commas
        auto pos = line.find_first_of("0123456789.-+");
        if (pos == std::string_view::npos) break;
        line.remove_prefix(pos);

        float val = 0.0f;
        auto [ptr, ec] = std::from_chars(line.data(), line.data() + line.size(), val);

        if (ec == std::errc{}) {
            values.push_back(val);
            line.remove_prefix(ptr - line.data());
        } else if (ec == std::errc::invalid_argument) {
            line.remove_prefix(1); // skip bad char
        } else { // errc::result_out_of_range
            std::cout << "Value out of range\n";
            break;
        }
    }

    // Print parsed values
    for (float v : values)
        std::cout << v << " ";
    std::cout << "\n";
}

int main() {
    // The string_view points to existing memory - no copy, no allocation
    std::string_view csv = "1.5, 2.7, 3.14, -0.001, 42.0";
    parse_csv_line(csv);
    // Output: 1.5 2.7 3.14 -0.001 42

    // Works on substrings too - no null-terminator needed!
    std::string_view partial = "temperature=98.6F";
    float temp = 0;
    auto start = partial.data() + partial.find('=') + 1;
    auto [ptr, ec] = std::from_chars(start, partial.data() + partial.size(), temp);
    if (ec == std::errc{})
        std::cout << "Temperature: " << temp << "\n";
    // Output: Temperature: 98.6
    // ptr points to 'F' - stopped at non-numeric character

    // KEY ADVANTAGE: stof("98.6") works but requires a null-terminated
    // std::string. from_chars works on any [first, last) char range.
}
```

`from_chars` takes raw `const char*` pointers - it works directly on `string_view` data without requiring a null-terminated `std::string`. This is critical for parsing protocols, file formats, and network data where creating intermediate strings wastes memory and time.

### Q3: Compare to_chars/from_chars with std::to_string and stoi for performance and correctness

**Answer:**

```cpp
#include <charconv>
#include <string>
#include <iostream>
#include <chrono>
#include <vector>
#include <array>
#include <cstdio>

int main() {
    constexpr int N = 1'000'000;

    // ===================== NUMBER -> STRING =====================

    // --- std::to_string (allocating) ---
    auto t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        std::string s = std::to_string(i); // heap alloc each time
        (void)s;
    }
    auto t2 = std::chrono::steady_clock::now();
    auto to_string_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1);

    // --- std::to_chars (allocation-free) ---
    std::array<char, 20> buf{};
    auto t3 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        std::to_chars(buf.data(), buf.data() + buf.size(), i);
    }
    auto t4 = std::chrono::steady_clock::now();
    auto to_chars_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t4 - t3);

    std::cout << "=== Number -> String ===\n";
    std::cout << "to_string: " << to_string_ms.count() << " ms\n";
    std::cout << "to_chars:  " << to_chars_ms.count() << " ms\n";
    // Typical: to_string ~80ms, to_chars ~15ms -> ~5x faster

    // ===================== STRING -> NUMBER =====================
    std::vector<std::string> inputs(N);
    for (int i = 0; i < N; ++i)
        inputs[i] = std::to_string(i * 3 + 7);

    // --- stoi (throwing, locale-dependent) ---
    int sum1 = 0;
    auto t5 = std::chrono::steady_clock::now();
    for (const auto& s : inputs) sum1 += std::stoi(s);
    auto t6 = std::chrono::steady_clock::now();

    // --- from_chars (error-code, locale-free) ---
    int sum2 = 0;
    auto t7 = std::chrono::steady_clock::now();
    for (const auto& s : inputs) {
        int val = 0;
        std::from_chars(s.data(), s.data() + s.size(), val);
        sum2 += val;
    }
    auto t8 = std::chrono::steady_clock::now();

    auto stoi_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t6 - t5);
    auto fc_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t8 - t7);

    std::cout << "\n=== String -> Number ===\n";
    std::cout << "stoi:       " << stoi_ms.count() << " ms (sum=" << sum1 << ")\n";
    std::cout << "from_chars: " << fc_ms.count() << " ms (sum=" << sum2 << ")\n";
    // Typical: stoi ~45ms, from_chars ~12ms -> ~3-4x faster

    // ===================== CORRECTNESS =====================
    // Locale issue: in de_DE locale, to_string(1234.5) -> "1234,500000"
    // to_chars always produces "1234.5" (dot decimal separator)
    // This matters for JSON, CSV, and wire protocols.

    // Round-trip guarantee:
    double d = 0.1 + 0.2; // 0.30000000000000004
    std::array<char, 64> dbuf{};
    auto [p, e] = std::to_chars(dbuf.data(), dbuf.data() + dbuf.size(), d);
    double recovered = 0.0;
    std::from_chars(dbuf.data(), p, recovered);
    std::cout << "\nRound-trip: " << std::boolalpha << (d == recovered) << "\n";
    // Output: Round-trip: true
    // to_string does NOT guarantee this!
}
```

- **Performance:** `to_chars`/`from_chars` are 3-5x faster due to no allocation, no locale, no exceptions.
- **Correctness:** `to_chars` is locale-independent and provides a round-trip guarantee for floating-point. `to_string` uses locale (comma vs dot) and `snprintf` format which may not round-trip.
- **Error handling:** `from_chars` uses `std::errc` error codes (zero-cost), while `stoi` throws exceptions (expensive on error paths).

---

## Notes

- **Floating-point support:** Full float/double support in `to_chars`/`from_chars` requires GCC 11+, Clang 14+, or MSVC 19.24+. Integer support was universal from C++17 day one.
- **No null terminator:** Neither function reads or writes `'\0'`. The range is always `[first, last)`.
- **Buffer sizing:** For integers, `std::numeric_limits<T>::digits10 + 2` chars suffice. For doubles, 24 chars is safe for most cases; 64 is conservative.
- **`chars_format`:** For floating-point: `scientific`, `fixed`, `hex`, `general` (default = shortest round-trip).
- **Use case overlap with item #364:** This topic focuses on allocation-free aspects and performance comparison; item #364 focuses on locale-independence.
- Compile with `-std=c++17 -Wall -Wextra`.
