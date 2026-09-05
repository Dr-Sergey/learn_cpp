# Write Compile-Time String Parsing Using `consteval` and `std::string_view`

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/consteval>  

---

## Topic Overview

### Compile-Time Date Parsing

`consteval` + `std::string_view` enables parsing and validating structured strings at compile time. Invalid formats cause compile errors instead of runtime failures - you get a compiler diagnostic pointing at the bad literal in your source code, not a crash or bad-data bug at runtime.

The date string `"YYYY-MM-DD"` is a good example to learn from because it is short, has a well-known format, and the validation logic is non-trivial (leap years, month lengths). If you can write this parser, you can write parsers for any fixed-format string your project needs.

### Key Techniques

| Technique | Description |
| --- | --- |
| `consteval` function | Guarantees compile-time execution |
| `std::string_view` | Non-owning, `constexpr`-friendly string access |
| `throw` in `consteval` | Becomes a compile error with a message |
| `static_assert` | Tests parser output at compile time |

### Pattern

The general structure is always the same. Write a `consteval` function that takes a `std::string_view`, validates it character by character, and throws a string literal on any error. Callers store the result in `constexpr` variables and the compiler does the work.

```cpp
struct Date { int year, month, day; };

consteval Date parse_date(std::string_view sv) {
    // Parse "YYYY-MM-DD" - throw on invalid format
    // ...
}

constexpr auto xmas = parse_date("2024-12-25");  // OK
// constexpr auto bad = parse_date("2024-13-01"); // Compile error: month > 12
```

---

## Self-Assessment

### Q1: Write a `consteval` parser that validates a date string `"YYYY-MM-DD"` format at compile time

The parser below breaks down into a few helper functions: one to read a fixed-width digit sequence, one to check leap years, and one to compute the number of days in a given month. Splitting it up this way keeps each piece testable on its own and makes the main `parse_date` function readable - which is especially valuable when a future developer needs to understand the format rules.

```cpp
#include <iostream>
#include <string_view>
#include <cstdint>

struct Date {
    int year;
    int month;
    int day;
};

// === Helper: parse N digits at position ===
consteval int parse_digits(std::string_view sv, std::size_t pos, std::size_t count) {
    int result = 0;
    for (std::size_t i = 0; i < count; ++i) {
        char c = sv[pos + i];
        if (c < '0' || c > '9')
            throw "Non-digit character in date string";
        result = result * 10 + (c - '0');
    }
    return result;
}

// === Check if year is a leap year ===
consteval bool is_leap_year(int year) {
    return (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0);
}

// === Days in month ===
consteval int days_in_month(int year, int month) {
    constexpr int days[] = {0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
    if (month == 2 && is_leap_year(year)) return 29;
    return days[month];
}

// === Main parser ===
consteval Date parse_date(std::string_view sv) {
    if (sv.size() != 10)
        throw "Date must be exactly 10 characters (YYYY-MM-DD)";
    if (sv[4] != '-' || sv[7] != '-')
        throw "Date must use '-' separators (YYYY-MM-DD)";

    int year = parse_digits(sv, 0, 4);
    int month = parse_digits(sv, 5, 2);
    int day = parse_digits(sv, 8, 2);

    if (year < 1 || year > 9999)
        throw "Year must be 1-9999";
    if (month < 1 || month > 12)
        throw "Month must be 1-12";
    if (day < 1 || day > days_in_month(year, month))
        throw "Day is out of range for the given month/year";

    return {year, month, day};
}

// === Compile-time verified dates ===
constexpr auto new_year    = parse_date("2024-01-01");
constexpr auto leap_day    = parse_date("2024-02-29");
constexpr auto christmas   = parse_date("2024-12-25");
constexpr auto end_of_year = parse_date("2024-12-31");

// Compile-time assertions
static_assert(new_year.year == 2024 && new_year.month == 1 && new_year.day == 1);
static_assert(leap_day.month == 2 && leap_day.day == 29);
static_assert(christmas.month == 12 && christmas.day == 25);

int main() {
    std::cout << "=== Compile-Time Date Parser ===\n";
    auto print = [](const char* label, const Date& d) {
        std::cout << label << ": " << d.year << "-"
                  << (d.month < 10 ? "0" : "") << d.month << "-"
                  << (d.day < 10 ? "0" : "") << d.day << "\n";
    };

    print("New Year   ", new_year);
    print("Leap Day   ", leap_day);
    print("Christmas  ", christmas);
    print("End of Year", end_of_year);

    std::cout << "\nAll dates validated at compile time.\n";
    return 0;
}
```

Notice that `parse_date("2024-02-29")` succeeds because 2024 is a leap year. If you changed it to `"2023-02-29"`, the code would not compile - the leap year check inside `days_in_month` would cause the parser to throw "Day is out of range for the given month/year". The compiler catches the bug before the program ever exists.

**Expected output:**

```text
=== Compile-Time Date Parser ===
New Year   : 2024-01-01
Leap Day   : 2024-02-29
Christmas  : 2024-12-25
End of Year: 2024-12-31

All dates validated at compile time.
```

### Q2: Use `static_assert` with the parser to reject invalid date literals in source code

You can write a thin `is_valid_date` wrapper that calls `parse_date` and returns `true`. When you combine this with `static_assert`, you get a self-documenting way to express "this date string must be valid" as a compile-time requirement. Anyone reading the code can see the assertion and understand the intent immediately.

The year 1900 case below is a classic gotcha worth knowing about: 1900 is divisible by 4 but also by 100, and it is not divisible by 400, so it is not a leap year. The parser correctly rejects February 29, 1900. This is exactly the kind of subtle rule that is easy to get wrong in a runtime validator - but the `static_assert` makes the correct behavior part of the compilation contract.

```cpp
#include <iostream>
#include <string_view>

struct Date { int year, month, day; };

consteval int parse_digits(std::string_view sv, std::size_t pos, std::size_t count) {
    int result = 0;
    for (std::size_t i = 0; i < count; ++i) {
        char c = sv[pos + i];
        if (c < '0' || c > '9') throw "Non-digit character";
        result = result * 10 + (c - '0');
    }
    return result;
}

consteval bool is_leap(int y) { return (y%4==0 && y%100!=0) || y%400==0; }

consteval int dim(int y, int m) {
    constexpr int d[] = {0,31,28,31,30,31,30,31,31,30,31,30,31};
    return m == 2 && is_leap(y) ? 29 : d[m];
}

consteval Date parse_date(std::string_view sv) {
    if (sv.size() != 10) throw "Must be YYYY-MM-DD (10 chars)";
    if (sv[4] != '-' || sv[7] != '-') throw "Must use '-' separators";
    int y = parse_digits(sv, 0, 4);
    int m = parse_digits(sv, 5, 2);
    int d = parse_digits(sv, 8, 2);
    if (m < 1 || m > 12) throw "Month out of range (1-12)";
    if (d < 1 || d > dim(y, m)) throw "Day out of range for month";
    return {y, m, d};
}

// === Helper: consteval bool so static_assert works ===
consteval bool is_valid_date(std::string_view sv) {
    parse_date(sv); // throws (= compile error) if invalid
    return true;
}

// === VALID dates - these all compile ===
static_assert(is_valid_date("2024-01-01"));
static_assert(is_valid_date("2024-02-29"));  // 2024 is a leap year
static_assert(is_valid_date("2000-02-29"));  // 2000 is a leap year (divisible by 400)
static_assert(is_valid_date("2024-12-31"));

// === INVALID dates - UNCOMMENT to see compile errors ===
// static_assert(is_valid_date("2024-13-01"));  // Month 13 -> "Month out of range"
// static_assert(is_valid_date("2024-02-30"));  // Feb 30 -> "Day out of range"
// static_assert(is_valid_date("2023-02-29"));  // 2023 not leap -> "Day out of range"
// static_assert(is_valid_date("2024/01/01"));  // Wrong separator -> "Must use '-'"
// static_assert(is_valid_date("24-01-01"));    // Too short -> "Must be YYYY-MM-DD"
// static_assert(is_valid_date("abcd-ef-gh"));  // Non-digit -> "Non-digit character"
// static_assert(is_valid_date("1900-02-29"));  // 1900 not leap -> "Day out of range"

int main() {
    std::cout << "=== static_assert Date Validation ===\n\n";

    std::cout << "Valid dates (compiled successfully):\n";
    std::cout << "  2024-01-01\n";
    std::cout << "  2024-02-29 (leap year)\n";
    std::cout << "  2000-02-29 (leap year, divisible by 400)\n";
    std::cout << "  2024-12-31\n";

    std::cout << "\nInvalid dates (would cause compile errors):\n";
    std::cout << "  2024-13-01 -> \"Month out of range (1-12)\"\n";
    std::cout << "  2024-02-30 -> \"Day out of range for month\"\n";
    std::cout << "  2023-02-29 -> \"Day out of range for month\" (not a leap year)\n";
    std::cout << "  2024/01/01 -> \"Must use '-' separators\"\n";
    std::cout << "  1900-02-29 -> \"Day out of range for month\" (not a leap year)\n";

    std::cout << "\nThe throw message appears in the compiler error output.\n";

    return 0;
}
```

**Expected output:**

```text
=== static_assert Date Validation ===

Valid dates (compiled successfully):
  2024-01-01
  2024-02-29 (leap year)
  2000-02-29 (leap year, divisible by 400)
  2024-12-31

Invalid dates (would cause compile errors):
  2024-13-01 -> "Month out of range (1-12)"
  2024-02-30 -> "Day out of range for month"
  2023-02-29 -> "Day out of range for month" (not a leap year)
  2024/01/01 -> "Must use '-' separators"
  1900-02-29 -> "Day out of range for month" (not a leap year)

The throw message appears in the compiler error output.
```

### Q3: Show how compile-time parsing avoids runtime validation overhead for fixed-format constants

When you have a set of date constants that are part of your program's configuration - release schedules, deadline dates, calendar epochs - compile-time parsing eliminates both the validation cost and the parsing cost at startup. The "What Happens in the Binary" section is worth understanding: the `config_dates` array is stored as 5 plain `Date` structs (12 bytes each) in the read-only data segment. The string literals do not need to appear in the binary at all. The parser code is never emitted.

```cpp
#include <iostream>
#include <string_view>
#include <chrono>
#include <cstring>
#include <cstddef>

struct Date { int year, month, day; };

// === Compile-time parser (from Q1) ===
consteval int ct_digits(std::string_view sv, std::size_t pos, std::size_t count) {
    int r = 0;
    for (std::size_t i = 0; i < count; ++i) {
        if (sv[pos+i] < '0' || sv[pos+i] > '9') throw "bad digit";
        r = r * 10 + (sv[pos+i] - '0');
    }
    return r;
}

consteval Date ct_parse(std::string_view sv) {
    if (sv.size() != 10 || sv[4] != '-' || sv[7] != '-') throw "bad format";
    return {ct_digits(sv,0,4), ct_digits(sv,5,2), ct_digits(sv,8,2)};
}

// === Runtime parser ===
Date rt_parse(const char* s) {
    Date d{};
    // Manual parsing with validation
    if (std::strlen(s) != 10 || s[4] != '-' || s[7] != '-')
        return {-1, -1, -1}; // error

    d.year = (s[0]-'0')*1000 + (s[1]-'0')*100 + (s[2]-'0')*10 + (s[3]-'0');
    d.month = (s[5]-'0')*10 + (s[6]-'0');
    d.day = (s[8]-'0')*10 + (s[9]-'0');
    return d;
}

// === Compile-time config: parsed at compile time, zero cost ===
constexpr Date config_dates[] = {
    ct_parse("2024-01-01"),
    ct_parse("2024-03-15"),
    ct_parse("2024-06-21"),
    ct_parse("2024-09-22"),
    ct_parse("2024-12-25"),
};

int main() {
    const char* date_strings[] = {
        "2024-01-01", "2024-03-15", "2024-06-21", "2024-09-22", "2024-12-25"
    };

    constexpr int ITERATIONS = 1000000;

    // === Benchmark: runtime parsing ===
    auto start = std::chrono::high_resolution_clock::now();
    volatile Date sink;
    for (int iter = 0; iter < ITERATIONS; ++iter) {
        for (const auto& s : date_strings) {
            sink = rt_parse(s);
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();

    // === Benchmark: compile-time table lookup ===
    auto start2 = std::chrono::high_resolution_clock::now();
    for (int iter = 0; iter < ITERATIONS; ++iter) {
        for (const auto& d : config_dates) {
            sink = d;
        }
    }
    auto end2 = std::chrono::high_resolution_clock::now();
    auto us2 = std::chrono::duration_cast<std::chrono::microseconds>(end2 - start2).count();

    std::cout << "=== Compile-Time vs Runtime Parsing ===\n";
    std::cout << "5 dates x " << ITERATIONS << " iterations\n\n";
    std::cout << "Runtime parsing:      " << us << " us\n";
    std::cout << "Compile-time (lookup): " << us2 << " us\n";
    std::cout << "Speedup: " << static_cast<double>(us) / (us2 > 0 ? us2 : 1) << "x\n";

    std::cout << "\n=== What Happens in the Binary ===\n";
    std::cout << "Compile-time: config_dates[] is a 5-element array in .rodata\n";
    std::cout << "  Each Date is 12 bytes (3 ints). Total: 60 bytes.\n";
    std::cout << "  No parsing code emitted. No string storage needed.\n\n";
    std::cout << "Runtime: date_strings[] + parsing function in .text\n";
    std::cout << "  String literals in .rodata (55 bytes) + parse code (~100 bytes).\n";
    std::cout << "  Parsing runs every time at startup.\n";

    std::cout << "\n=== Compile-Time Parsed Dates ===\n";
    for (const auto& d : config_dates) {
        std::cout << "  " << d.year << "-"
                  << (d.month < 10 ? "0" : "") << d.month << "-"
                  << (d.day < 10 ? "0" : "") << d.day << "\n";
    }

    return 0;
}
```

**Expected output (timing varies):**

```text
=== Compile-Time vs Runtime Parsing ===
5 dates x 1000000 iterations

Runtime parsing:      28000 us
Compile-time (lookup): 8000 us
Speedup: 3.5x

=== What Happens in the Binary ===
Compile-time: config_dates[] is a 5-element array in .rodata
  Each Date is 12 bytes (3 ints). Total: 60 bytes.
  No parsing code emitted. No string storage needed.

Runtime: date_strings[] + parsing function in .text
  String literals in .rodata (55 bytes) + parse code (~100 bytes).
  Parsing runs every time at startup.

=== Compile-Time Parsed Dates ===
  2024-01-01
  2024-03-15
  2024-06-21
  2024-09-22
  2024-12-25
```

---

## Notes

- `consteval` + `throw` gives compile-time error messages - the throw string appears in compiler output.
- `std::string_view` is the ideal parameter type: `constexpr`, non-owning, full string API.
- Parsed results in `constexpr` arrays go to `.rodata` - zero startup cost, zero runtime validation.
- This pattern works for: dates, IP addresses, URLs, color codes, regex patterns, version strings.
- Leap year check: divisible by 4, not by 100, unless by 400 - the parser can enforce this at compile time.
- For runtime user input, you still need a runtime parser - `consteval` is for source-code literals only.
