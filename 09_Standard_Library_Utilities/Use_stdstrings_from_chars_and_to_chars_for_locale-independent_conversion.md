# Use std::string's from_chars and to_chars for locale-independent conversion

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/from_chars>  

---

## Topic Overview

`std::from_chars` and `std::to_chars` (header `<charconv>`, C++17) provide the fastest, locale-independent number-to-string and string-to-number conversion in C++. They operate on raw `char` arrays with no allocation, no exceptions, no locale-dependent behavior (always uses `.` for decimal point), and no null-terminator requirement.

The reason people reach for these instead of `stoi` or `snprintf` comes down to two things: correctness and speed. If you're writing data to a file or network protocol, you cannot afford the decimal separator to silently change based on the user's locale. And if you're parsing millions of values in a hot loop, the overhead of exceptions and locale lookup adds up fast.

### Comparison

| Feature | `stoi`/`stod` | `sscanf`/`sprintf` | `from_chars`/`to_chars` |
| --- | --- | --- | --- |
| Allocation | Yes (`std::string`) | No | No |
| Exceptions | Yes (`invalid_argument`, `out_of_range`) | No | No (error codes) |
| Locale | Yes (locale-dependent) | Yes | No (always `"C"`) |
| Speed | Slow | Medium | Fastest |
| Null-terminated input | Required | Required | Not required (pointer+length) |
| Thread-safe | Depends on locale | Depends on locale | Always |
| Standard | C++11 | C89 | C++17 |

### from_chars - String to Number

The signature looks a little unusual if you're used to `stoi`, so let's walk through it. You pass two pointers that define the range to parse, plus a reference to the variable to fill. The result tells you where parsing stopped and whether it succeeded.

```cpp
#include <charconv>
#include <system_error>

// Signature:
std::from_chars_result from_chars(const char* first, const char* last,
                                  T& value, int base = 10);
// For floating-point:
std::from_chars_result from_chars(const char* first, const char* last,
                                  T& value, std::chars_format fmt = std::chars_format::general);

// Returns:
struct from_chars_result {
    const char* ptr;  // Points past the last parsed character
    std::errc ec;     // std::errc{} on success
};
```

Notice that the result struct tells you both the error code and where the parser stopped. The `ptr` field is useful when you're parsing multiple values out of a single buffer - you just advance your pointer and call again.

### to_chars - Number to String

The counterpart follows the same pointer-range pattern. You hand it a buffer and get back a pointer to one past the last written character.

```cpp
#include <charconv>

// Signature:
std::to_chars_result to_chars(char* first, char* last, T value);
std::to_chars_result to_chars(char* first, char* last, T value, int base);
// For floating-point:
std::to_chars_result to_chars(char* first, char* last, T value,
                              std::chars_format fmt, int precision);

// Returns:
struct to_chars_result {
    char* ptr;     // Points past the last written character
    std::errc ec;  // std::errc{} on success, value_too_large if buffer too small
};
```

### Core Examples

Here is everything in action - integer and double parsing, integer and hex formatting, and error handling without a try/catch in sight.

```cpp
#include <charconv>
#include <iostream>
#include <string_view>
#include <array>
#include <system_error>

int main() {
    // === from_chars: parse integer ===
    std::string_view input = "42 hello";
    int value = 0;
    auto [ptr, ec] = std::from_chars(input.data(), input.data() + input.size(), value);

    if (ec == std::errc{}) {
        std::cout << "Parsed: " << value << "\n"; // 42
        std::cout << "Remaining: " << std::string_view(ptr, input.end()) << "\n"; // " hello"
    }

    // === from_chars: parse double ===
    std::string_view dbl_input = "3.14159";
    double d = 0.0;
    auto [dp, dec] = std::from_chars(dbl_input.data(), dbl_input.data() + dbl_input.size(), d);
    std::cout << "Double: " << d << "\n"; // 3.14159

    // === to_chars: integer to string ===
    std::array<char, 32> buf{};
    auto [p, e] = std::to_chars(buf.data(), buf.data() + buf.size(), 12345);
    std::cout << "Formatted: " << std::string_view(buf.data(), p) << "\n"; // "12345"

    // === to_chars: hexadecimal ===
    auto [hp, he] = std::to_chars(buf.data(), buf.data() + buf.size(), 255, 16);
    std::cout << "Hex: " << std::string_view(buf.data(), hp) << "\n"; // "ff"

    // === Error handling (no exceptions!) ===
    std::string_view bad_input = "not_a_number";
    int bad_val = 0;
    auto [bp, bec] = std::from_chars(bad_input.data(),
                                      bad_input.data() + bad_input.size(), bad_val);
    if (bec == std::errc::invalid_argument)
        std::cout << "Parse failed: invalid argument\n";
    // Output: Parse failed: invalid argument
}
```

The parser stopped as soon as it hit a non-digit, stored the error code, and returned without throwing. You check `ec`, react, and move on.

---

## Self-Assessment

### Q1: Show that std::stoi throws std::invalid_argument on bad input while from_chars returns an error code

**Answer:**

This example shows the two approaches side by side so the cost difference is concrete.

```cpp
#include <charconv>
#include <string>
#include <iostream>
#include <stdexcept>

int main() {
    // === stoi: throws exception on bad input ===
    try {
        int val = std::stoi("not_a_number"); // throws!
        (void)val;
    } catch (const std::invalid_argument& e) {
        std::cout << "stoi threw: " << e.what() << "\n";
    }
    // Output: stoi threw: invalid_argument (or similar)

    try {
        int val = std::stoi("99999999999999999"); // out of range!
        (void)val;
    } catch (const std::out_of_range& e) {
        std::cout << "stoi threw: " << e.what() << "\n";
    }

    // === from_chars: returns error code, no exception ===
    const char bad[] = "not_a_number";
    int value = 0;
    auto [ptr, ec] = std::from_chars(bad, bad + sizeof(bad) - 1, value);

    if (ec == std::errc::invalid_argument) {
        std::cout << "from_chars: invalid argument (no exception thrown)\n";
    }
    // Output: from_chars: invalid argument (no exception thrown)

    // Out of range:
    const char huge[] = "99999999999999999";
    auto [ptr2, ec2] = std::from_chars(huge, huge + sizeof(huge) - 1, value);
    if (ec2 == std::errc::result_out_of_range) {
        std::cout << "from_chars: out of range (no exception thrown)\n";
    }

    // WHY from_chars is better for parsing untrusted input:
    // 1. No exception overhead - checking errc is a simple comparison
    // 2. No string construction needed - works on raw char*
    // 3. Predictable performance - no try/catch unwinding
    // 4. Can parse substrings without copying (no null terminator needed)
}
```

`stoi` throws `std::invalid_argument` and `std::out_of_range` - exceptions that involve heap allocation for the error message and unwinding cost. `from_chars` returns a trivial `from_chars_result` struct with an `std::errc` code - zero overhead error handling. For parsing millions of values (e.g., CSV files, network protocols), this difference is significant.

### Q2: Benchmark from_chars vs stoi for parsing millions of integers from a buffer

**Answer:**

```cpp
#include <charconv>
#include <string>
#include <vector>
#include <iostream>
#include <chrono>
#include <cstring>

int main() {
    // Prepare: 1 million integer strings
    constexpr int N = 1'000'000;
    std::vector<std::string> strings(N);
    for (int i = 0; i < N; ++i)
        strings[i] = std::to_string(i * 7 + 13); // "13", "20", "27", ...

    int total = 0;

    // === Benchmark stoi ===
    auto t1 = std::chrono::steady_clock::now();
    for (const auto& s : strings) {
        total += std::stoi(s);
    }
    auto t2 = std::chrono::steady_clock::now();
    auto stoi_time = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1);
    std::cout << "stoi:       " << stoi_time.count() << " ms (total=" << total << ")\n";

    total = 0;

    // === Benchmark from_chars ===
    auto t3 = std::chrono::steady_clock::now();
    for (const auto& s : strings) {
        int val = 0;
        std::from_chars(s.data(), s.data() + s.size(), val);
        total += val;
    }
    auto t4 = std::chrono::steady_clock::now();
    auto fc_time = std::chrono::duration_cast<std::chrono::milliseconds>(t4 - t3);
    std::cout << "from_chars: " << fc_time.count() << " ms (total=" << total << ")\n";

    // Typical results (GCC -O2):
    // stoi:       45 ms
    // from_chars: 12 ms
    // from_chars is ~3-4x faster

    // Why from_chars wins:
    // - No std::string temporary construction
    // - No exception setup overhead
    // - No locale lookup
    // - Simpler parsing loop (no heap)
    if (fc_time.count() > 0)
        std::cout << "Speedup: " << static_cast<double>(stoi_time.count()) / fc_time.count()
                  << "x\n";
}
```

`from_chars` avoids three major costs that `stoi` pays: exception infrastructure (try/catch setup), locale handling (checking current locale for digit characters), and `std::string` parameter construction. For bulk parsing workloads (CSV, JSON, log files), `from_chars` is the fastest standard option.

### Q3: Use to_chars to convert a double to a char buffer without locale dependency

**Answer:**

The default format (no explicit `chars_format`) produces the shortest decimal representation that round-trips exactly back to the same `double`. That is the round-trip guarantee - and it is something `printf`-style functions do not provide.

```cpp
#include <charconv>
#include <iostream>
#include <array>
#include <string_view>
#include <cstring>

int main() {
    double pi = 3.14159265358979;

    // === Basic conversion ===
    std::array<char, 64> buf{};
    auto [ptr, ec] = std::to_chars(buf.data(), buf.data() + buf.size(), pi);

    if (ec == std::errc{}) {
        std::string_view result(buf.data(), ptr - buf.data());
        std::cout << "Default: " << result << "\n";
        // Output: 3.14159265358979 (shortest round-trip representation)
    }

    // === Fixed notation with precision ===
    auto [p2, e2] = std::to_chars(buf.data(), buf.data() + buf.size(),
                                   pi, std::chars_format::fixed, 2);
    std::cout << "Fixed(2): " << std::string_view(buf.data(), p2) << "\n";
    // Output: 3.14

    // === Scientific notation ===
    auto [p3, e3] = std::to_chars(buf.data(), buf.data() + buf.size(),
                                   pi, std::chars_format::scientific, 6);
    std::cout << "Scientific(6): " << std::string_view(buf.data(), p3) << "\n";
    // Output: 3.141593e+00

    // === Locale independence demonstration ===
    // In German locale, printf would output "3,14159" (comma decimal)
    // to_chars ALWAYS uses '.' regardless of locale:
    double val = 1234.5678;
    auto [p4, e4] = std::to_chars(buf.data(), buf.data() + buf.size(), val);
    std::string_view formatted(buf.data(), p4);
    std::cout << "Locale-free: " << formatted << "\n";
    // Output: 1234.5678 - always uses '.', never ','

    // === Round-trip guarantee ===
    // Default to_chars produces the shortest representation that
    // round-trips back to the exact same double via from_chars
    double original = 0.1 + 0.2; // 0.30000000000000004
    auto [p5, e5] = std::to_chars(buf.data(), buf.data() + buf.size(), original);

    double recovered = 0.0;
    std::from_chars(buf.data(), p5, recovered);
    std::cout << "Round-trip match: " << std::boolalpha
              << (original == recovered) << "\n"; // true
}
```

`to_chars` with no format specifier produces the shortest decimal representation that round-trips perfectly through `from_chars`. It never consults the locale - the decimal separator is always `.`. This makes it safe for serialization formats (JSON, CSV, binary protocols) where locale-dependent formatting would corrupt data.

---

## Notes

- **Floating-point support:** Full `from_chars`/`to_chars` for `float`/`double` was slow to be implemented. GCC 11+, Clang 14+, and MSVC 19.24+ support it.
- **chars_format:** `std::chars_format::scientific`, `::fixed`, `::hex`, `::general` (default). `::general` uses the shortest of fixed/scientific.
- **Buffer too small:** `to_chars` sets `ec = std::errc::value_too_large` if the buffer can't hold the result. Always check `ec` or provide a conservatively large buffer.
- **No null terminator:** Neither `from_chars` nor `to_chars` reads or writes a null terminator. The range is `[first, last)`.
- **JSON/CSV serialization:** Use `to_chars`/`from_chars` for any portable data format. Using `snprintf` with `%f` in a French locale would write `"3,14"` instead of `"3.14"`, breaking parsers.
- **Performance tip:** For bulk conversion, reuse the same buffer across calls.
- Compile with `-std=c++17 -Wall -Wextra` (or `-std=c++20`).
