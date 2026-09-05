# Understand Compile-Time Format Checking with `std::basic_format_string`

**Category:** Standard Library New Cpp23 26  
**Standard:** C++20, C++23 (core)  
**Reference:** [cppreference — std::basic_format_string](https://en.cppreference.com/w/cpp/utility/format/basic_format_string)  

---

## Topic Overview

`std::basic_format_string<CharT, Args...>` (aliased as `std::format_string<Args...>` for `char`) is the mechanism that makes `std::format` and `std::print` validate format strings at compile time. When you write `std::format("{} + {} = {}", a, b, c)`, the string literal is not just a `const char*` - it is wrapped in a `std::format_string<decltype(a), decltype(b), decltype(c)>` whose constructor is `consteval` and checks the format specifiers against the argument types during compilation.

This means mismatched format strings - wrong number of arguments, invalid specifiers for the type, or malformed syntax - are compile-time errors, not runtime crashes. That is a huge deal if you have ever spent time tracking down a `printf` bug that only showed up on a specific input.

| Error type | `printf` | `std::format` (C++20+) |
| --- | --- | --- |
| Wrong arg count | UB at runtime | Compile error |
| Wrong type specifier | UB at runtime | Compile error |
| Malformed format string | UB at runtime | Compile error |
| Dynamic format string | N/A | `std::vformat` (runtime check) |

Here is how the two-phase design works under the hood. The `consteval` constructor runs entirely at compile time, and only then does the runtime formatter execute:

```cpp
std::format("{:.2f}", 3.14)
     |                  |
     v                  v
+---------------------------------------+
| std::format_string<double> ctor       |
| +-------------------------------+     |
| | consteval parse("{:.2f}")     |     |
| | verify: double supports 'f'  |     | <- compile-time
| | verify: arg count matches    |     |
| +-------------------------------+     |
+---------------------------------------+
         |
         v
+---------------------------------------+
| std::vformat(fmt.get(), args...)      | <- runtime formatting
+---------------------------------------+
```

The key thing to internalize: the `consteval` constructor means the format string must be a compile-time constant - either a string literal or a `constexpr` variable. For format strings that come from user input or config files, you must use `std::vformat` or `std::runtime_format` (C++26), which defer the validation to runtime and can throw `std::format_error` instead of giving you a compile error.

---

## Self-Assessment

### Q1: How does compile-time format checking catch errors, and what errors does it prevent

The example below shows valid formats that all compile, alongside commented-out calls that would each produce a compiler error with a meaningful message. Notice especially the false intuition in the `{:f}` on `int` case - the standard actually allows it.

```cpp
#include <format>
#include <string>
#include <print>

void demonstrate_compile_time_checks() {
    int x = 42;
    double y = 3.14;
    std::string s = "hello";

    // VALID: all compile successfully
    auto r1 = std::format("{}", x);           // "42"
    auto r2 = std::format("{:08x}", x);       // "0000002a"
    auto r3 = std::format("{:.2f}", y);       // "3.14"
    auto r4 = std::format("{:>10}", s);       // "     hello"
    auto r5 = std::format("{0} {0} {1}", x, y);  // positional: "42 42 3.14"

    // COMPILE ERRORS: caught at compile time

    // Error: too few arguments
    // auto e1 = std::format("{} {}", x);

    // Error: too many arguments
    // auto e2 = std::format("{}", x, y);

    // Error: 'd' (integer) specifier on a double
    // auto e3 = std::format("{:d}", y);

    // Error: 'f' (float) specifier on an int
    // auto e4 = std::format("{:f}", x);  // Actually OK - int is valid with f

    // Error: 'p' (pointer) specifier on an int
    // auto e5 = std::format("{:p}", x);

    // Error: closing brace without opening
    // auto e6 = std::format("value }");

    // Error: unmatched opening brace
    // auto e7 = std::format("value {");

    std::println("All valid formats passed compile-time checks");
}

int main() {
    demonstrate_compile_time_checks();
}
```

Every one of those commented-out errors is caught before your program even runs. The compiler effectively parses and type-checks your format string as part of overload resolution. If the code compiles, the format string is valid for the given argument types - that is the guarantee.

---

### Q2: How do you write your own API with compile-time format checking

This is where `std::format_string` really shines. By accepting it as a parameter type, your own functions inherit the same compile-time safety that `std::format` has - no extra machinery needed.

```cpp
#include <format>
#include <print>
#include <string>
#include <iostream>
#include <source_location>
#include <chrono>
#include <string_view>
#include <utility>

// Custom log function with compile-time format checking
enum class LogLevel { Debug, Info, Warn, Error };

template <typename... Args>
void log(LogLevel level,
         std::format_string<Args...> fmt,   // <- compile-time checked!
         Args&&... args) {
    const char* prefix;
    switch (level) {
        case LogLevel::Debug: prefix = "[DEBUG]"; break;
        case LogLevel::Info:  prefix = "[INFO] "; break;
        case LogLevel::Warn:  prefix = "[WARN] "; break;
        case LogLevel::Error: prefix = "[ERROR]"; break;
    }

    auto msg = std::format(fmt, std::forward<Args>(args)...);
    std::println("{} {}", prefix, msg);
}

// Format-checked assertion with source location
template <typename... Args>
void check(bool condition,
           std::format_string<Args...> fmt,
           Args&&... args,
           std::source_location loc = std::source_location::current()) {
    if (!condition) {
        auto msg = std::format(fmt, std::forward<Args>(args)...);
        std::println(stderr, "ASSERTION FAILED at {}:{}: {}",
                     loc.file_name(), loc.line(), msg);
        std::abort();
    }
}

// Typed metric reporting
template <typename... Args>
void report_metric(std::string_view name,
                   double value,
                   std::format_string<Args...> detail_fmt,
                   Args&&... args) {
    auto detail = std::format(detail_fmt, std::forward<Args>(args)...);
    std::println("METRIC [{}] = {:.4f} ({})", name, value, detail);
}

int main() {
    // All format strings are validated at compile time
    log(LogLevel::Info, "User {} logged in from {}", "Alice", "192.168.1.1");
    log(LogLevel::Error, "Failed after {} retries: error {:#x}", 3, 0xDEAD);

    // These would NOT compile:
    // log(LogLevel::Info, "User {} logged in from {}");       // missing args
    // log(LogLevel::Info, "User {:d}", "Alice");              // d on string

    int x = 42;
    check(x > 0, "Expected positive, got {}", x);

    report_metric("latency_ms", 12.345,
                  "p99 over last {} requests", 1000);
}
```

The trick is in the function signature: `std::format_string<Args...>` is deduced from the call site using template argument deduction. The `consteval` constructor of `format_string` fires during that deduction and rejects the call if the string is invalid. Your callers get the error message at their call site, not deep inside your logging library.

---

### Q3: How do you handle runtime format strings and what is `std::runtime_format` (C++26)

Sometimes the format string is not known until runtime - it comes from a user, a config file, a database. In that case you cannot use `std::format_string` because there is no compile-time constant to check. The solution is `std::vformat`, which checks the string at runtime and throws `std::format_error` on failure.

```cpp
#include <format>
#include <print>
#include <string>
#include <iostream>
#include <stdexcept>
#include <string_view>
#include <utility>

// Runtime format strings with std::vformat
// When the format string comes from user input, config files, etc.
void log_dynamic(std::string_view fmt_str, int value, double rate) {
    try {
        // std::vformat checks the format string at RUNTIME
        auto args = std::make_format_args(value, rate);
        std::string result = std::vformat(fmt_str, args);
        std::println("{}", result);
    } catch (const std::format_error& e) {
        std::println(stderr, "Format error: {}", e.what());
    }
}

// C++26: std::runtime_format - explicit opt-in to runtime checking
// void log_runtime_cpp26(std::string runtime_fmt, int x) {
//     // Makes it clear this is intentionally a runtime format string
//     std::print(std::runtime_format(runtime_fmt), x);
// }

// Compile-time vs runtime decision pattern
template <typename... Args>
struct LogMessage {
    std::string_view text;

    // Compile-time checked
    static LogMessage from_literal(std::format_string<Args...> fmt,
                                   Args&&... args) {
        return { std::format(fmt, std::forward<Args>(args)...) };
    }

    // Runtime checked (for config-driven formats)
    static LogMessage from_dynamic(std::string_view fmt,
                                   Args&&... args) {
        auto store = std::make_format_args(args...);
        return { std::vformat(fmt, store) };
    }
};

// Safely wrapping vformat with validation
class FormatValidator {
public:
    // Pre-validate a format string for a known set of arg types
    template <typename... Args>
    static bool is_valid(std::string_view fmt) noexcept {
        try {
            // Try to format with dummy values
            auto args = std::make_format_args(Args{}...);
            std::vformat(fmt, args);
            return true;
        } catch (const std::format_error&) {
            return false;
        }
    }
};

int main() {
    // Runtime format string - checked at runtime
    std::string user_fmt = "Value: {}, Rate: {:.2f}";
    log_dynamic(user_fmt, 42, 3.14159);  // "Value: 42, Rate: 3.14"

    // Invalid runtime format - caught as exception
    log_dynamic("Bad: {:d}", 42, 3.14);  // d on double -> format_error

    // Validate before use
    std::println("Valid: {}",
        FormatValidator::is_valid<int, double>("Val={}, Rate={:.2f}"));
    std::println("Invalid: {}",
        FormatValidator::is_valid<int>("Val={:p}"));  // p on int
}
```

The reason `std::runtime_format` (C++26) is useful is that it gives you a way to explicitly pass a runtime string to `std::format` itself, rather than having to drop down to `std::vformat` with the more verbose `make_format_args` dance. It signals your intent clearly: yes, this is intentionally a runtime string, and yes, you know the trade-off.

---

## Notes

- `std::format_string<Args...>` (alias for `basic_format_string<char, type_identity_t<Args>...>`) uses a `consteval` constructor - the format string must be a constant expression.
- Compile-time checking covers: argument count, type compatibility with specifiers, brace matching, and positional argument validity.
- For runtime format strings (user input, config), use `std::vformat` which throws `std::format_error` on invalid formats.
- C++26 adds `std::runtime_format(str)` as an explicit wrapper to pass runtime strings to `std::format`/`std::print` without using `vformat` directly.
- To create custom APIs with compile-time format checking, accept `std::format_string<Args...>` and forward to `std::format`.
- The `consteval` constructor interacts with template argument deduction - the format string literal's character array decays into `format_string` during overload resolution.
- Do not try to store a `format_string` object - it is designed for immediate consumption, not persistence.
- Feature-test macro: `__cpp_lib_format >= 202106L` (C++20 core), `__cpp_lib_format >= 202207L` (C++23 extensions).
