# Implement secure coding practices: input validation and output sanitization

**Category:** Safety and Security  
**Standard:** C++20  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>  

---

## Topic Overview

Secure coding in C++ requires validating all inputs at trust boundaries and sanitizing all outputs to prevent injection attacks. Since C++ has no built-in sandboxing or bounds checking (unlike managed languages), the programmer is responsible for every byte that enters and leaves the program.

### Trust Boundary Model

The mental model here is simple: draw a line around the part of your program where data is trusted. Everything crossing that line inbound needs validation; everything crossing it outbound needs sanitization. Skipping either direction is how injection vulnerabilities happen.

```cpp
+--------------------------------------------------+
|  Trusted Zone (your validated data)              |
|                                                  |
|  +---- Input Validation <----- User input        |
|  |     - Range checks         - Network data     |
|  |     - Type checks          - File contents    |
|  |     - Length limits        - Environment vars |
|  |     - Format validation    - CLI arguments    |
|  |                                               |
|  +---- Output Sanitization -----> SQL queries    |
|        - Escaping              - Shell commands  |
|        - Encoding              - HTML output     |
|        - Parameterized queries - Log messages    |
+--------------------------------------------------+
```

### Common Vulnerability Classes

| Vulnerability | Cause | Prevention |
| --- | --- | --- |
| Buffer overflow | Unchecked input length | Bounds checking, `std::string` |
| Integer overflow | Unchecked arithmetic | `std::cmp_less`, overflow builtins |
| SQL injection | String concatenation | Parameterized queries |
| Command injection | Unsanitized shell args | Avoid `system()`, use exec-family |
| Format string | User-controlled format | `std::format`, never `printf(user)` |
| Path traversal | Unsanitized file paths | Canonicalize, whitelist directories |

### Core Example

Here is a compact but complete demonstration of the two sides of the boundary: an integer parser that rejects everything that does not look exactly like a valid port number, a username validator that uses an allowlist, and a SQL escaper. Notice how each function has a single clear definition of what it accepts.

```cpp
#include <string>
#include <string_view>
#include <stdexcept>
#include <algorithm>
#include <cctype>
#include <iostream>
#include <charconv>

// === Input Validation: integer range ===
int parse_port(std::string_view input) {
    int port = 0;
    auto [ptr, ec] = std::from_chars(input.data(), input.data() + input.size(), port);
    if (ec != std::errc{} || ptr != input.data() + input.size())
        throw std::invalid_argument("Invalid port number");
    if (port < 1 || port > 65535)
        throw std::out_of_range("Port must be 1-65535");
    return port;
}

// === Input Validation: string allowlist ===
bool is_safe_username(std::string_view name) {
    if (name.empty() || name.size() > 64) return false;
    return std::all_of(name.begin(), name.end(), [](char c) {
        return std::isalnum(static_cast<unsigned char>(c)) || c == '_' || c == '-';
    });
}

// === Output Sanitization: SQL escaping ===
std::string escape_sql_string(std::string_view input) {
    std::string result;
    result.reserve(input.size() + 10);
    for (char c : input) {
        if (c == '\'') result += "''";      // escape single quote
        else if (c == '\\') result += "\\\\";
        else if (c == '\0') continue;       // strip null bytes
        else result += c;
    }
    return result;
}

int main() {
    std::cout << parse_port("8080") << "\n"; // 8080
    std::cout << std::boolalpha
              << is_safe_username("alice_01") << "\n"    // true
              << is_safe_username("'; DROP TABLE") << "\n"; // false
    std::cout << escape_sql_string("O'Brien") << "\n";  // O''Brien
}
```

The `std::from_chars` approach for integer parsing is worth internalizing. It does not skip leading spaces, it does not accept locale-dependent formats, and it tells you exactly how many characters it consumed - so you can reject trailing garbage. This is much harder to get wrong than `atoi` or `strtol`.

---

## Self-Assessment

### Q1: Validate all integer inputs before use: check ranges, check for truncation on conversion

**Answer:**

```cpp
#include <iostream>
#include <string_view>
#include <charconv>
#include <limits>
#include <cstdint>
#include <stdexcept>
#include <utility>

// Safe integer parser with full validation
template<typename T>
T safe_parse_int(std::string_view input) {
    // 1. Reject empty input
    if (input.empty())
        throw std::invalid_argument("Empty input");

    // 2. Reject leading/trailing whitespace (don't silently skip)
    if (std::isspace(static_cast<unsigned char>(input.front())) ||
        std::isspace(static_cast<unsigned char>(input.back())))
        throw std::invalid_argument("Leading/trailing whitespace");

    // 3. Parse into a wide type to detect truncation
    long long wide_val = 0;
    auto [ptr, ec] = std::from_chars(input.data(), input.data() + input.size(), wide_val);

    if (ec == std::errc::invalid_argument)
        throw std::invalid_argument("Not a valid integer");
    if (ec == std::errc::result_out_of_range)
        throw std::out_of_range("Value too large for long long");
    if (ptr != input.data() + input.size())
        throw std::invalid_argument("Trailing characters after number");

    // 4. Check for truncation when converting to target type
    if (wide_val < static_cast<long long>(std::numeric_limits<T>::min()) ||
        wide_val > static_cast<long long>(std::numeric_limits<T>::max()))
        throw std::out_of_range("Value out of range for target type");

    return static_cast<T>(wide_val);
}

// Safe narrowing conversion with explicit check
template<typename To, typename From>
To safe_narrow(From value) {
    auto result = static_cast<To>(value);
    if (static_cast<From>(result) != value) // round-trip check
        throw std::overflow_error("Narrowing conversion lost data");
    // Also check sign preservation
    if ((value < From{0}) != (result < To{0}))
        throw std::overflow_error("Sign change in narrowing conversion");
    return result;
}

int main() {
    // Valid inputs
    auto port = safe_parse_int<uint16_t>("8080");
    std::cout << "Port: " << port << "\n"; // 8080

    auto age = safe_parse_int<uint8_t>("25");
    std::cout << "Age: " << static_cast<int>(age) << "\n"; // 25

    // Truncation detection:
    try {
        safe_parse_int<uint8_t>("300"); // 300 > 255
    } catch (const std::out_of_range& e) {
        std::cout << "Caught: " << e.what() << "\n";
        // Output: Caught: Value out of range for target type
    }

    // Narrowing conversion safety:
    try {
        int big = 100000;
        auto small = safe_narrow<int16_t>(big); // 100000 > 32767
        (void)small;
    } catch (const std::overflow_error& e) {
        std::cout << "Narrow: " << e.what() << "\n";
        // Output: Narrow: Narrowing conversion lost data
    }

    // Sign change detection:
    try {
        int neg = -1;
        auto u = safe_narrow<unsigned>(neg); // -1 -> huge positive
        (void)u;
    } catch (const std::overflow_error& e) {
        std::cout << "Sign: " << e.what() << "\n";
        // Output: Sign: Sign change in narrowing conversion
    }
}
```

Every integer input needs three layers of validation: syntactic correctness (is it actually a number?), range (does it fit the target type?), and truncation (does narrowing lose data?). Using `from_chars` avoids locale and exception overhead during parsing. The round-trip check in `safe_narrow` catches any data loss during type conversion - if converting to a smaller type and back does not give you the same value, something was lost.

### Q2: Sanitize all strings used in SQL queries or shell commands to prevent injection attacks

**Answer:**

```cpp
#include <string>
#include <string_view>
#include <iostream>
#include <algorithm>
#include <array>
#include <cctype>

// === SQL Injection Prevention ===

// BAD: String concatenation is vulnerable
std::string build_query_UNSAFE(std::string_view username) {
    return "SELECT * FROM users WHERE name = '" + std::string(username) + "'";
    // Input: "'; DROP TABLE users; --"
    // Result: SELECT * FROM users WHERE name = ''; DROP TABLE users; --'
    // ^^^ SQL INJECTION!
}

// GOOD: Parameterized query (pseudo-code for illustration)
// In real code, use a DB library with prepared statements:
//   auto stmt = db.prepare("SELECT * FROM users WHERE name = ?");
//   stmt.bind(1, username);
//   auto result = stmt.execute();

// GOOD: Escape special characters if parameterized queries aren't available
std::string escape_sql(std::string_view input) {
    std::string result;
    result.reserve(input.size() * 2);
    for (char c : input) {
        switch (c) {
            case '\'': result += "''"; break;
            case '\\': result += "\\\\"; break;
            case '\0': break; // strip null bytes
            case '\n': result += "\\n"; break;
            case '\r': result += "\\r"; break;
            default: result += c;
        }
    }
    return result;
}

// === Command Injection Prevention ===

// BAD: system() with user input
void run_UNSAFE(std::string_view filename) {
    std::string cmd = "cat " + std::string(filename);
    // system(cmd.c_str()); // NEVER do this!
    // Input: "; rm -rf /" -> executes: cat ; rm -rf /
    std::cout << "[UNSAFE] Would run: " << cmd << "\n";
}

// GOOD: Validate against allowlist, never pass to shell
bool is_safe_filename(std::string_view name) {
    if (name.empty() || name.size() > 255) return false;
    // Only allow alphanumeric, dots, hyphens, underscores
    return std::all_of(name.begin(), name.end(), [](char c) {
        return std::isalnum(static_cast<unsigned char>(c))
               || c == '.' || c == '-' || c == '_';
    }) && name.find("..") == std::string_view::npos; // prevent path traversal
}

// === HTML Output Sanitization ===
std::string escape_html(std::string_view input) {
    std::string result;
    result.reserve(input.size() + 20);
    for (char c : input) {
        switch (c) {
            case '&':  result += "&amp;"; break;
            case '<':  result += "&lt;"; break;
            case '>':  result += "&gt;"; break;
            case '"':  result += "&quot;"; break;
            case '\'': result += "&#39;"; break;
            default:   result += c;
        }
    }
    return result;
}

int main() {
    // SQL escaping:
    std::string input = "O'Brien'; DROP TABLE users; --";
    std::cout << "Escaped SQL: " << escape_sql(input) << "\n";
    // Output: O''Brien''; DROP TABLE users; --

    // Filename validation:
    std::cout << std::boolalpha;
    std::cout << is_safe_filename("report.txt") << "\n";      // true
    std::cout << is_safe_filename("../../etc/passwd") << "\n"; // false
    std::cout << is_safe_filename("; rm -rf /") << "\n";       // false

    // HTML escaping:
    std::cout << escape_html("<script>alert('xss')</script>") << "\n";
    // Output: &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;
}
```

The golden rule is: never concatenate untrusted input into structured commands (SQL, shell, HTML). Use parameterized queries for SQL - the `?` placeholder approach means user data is never interpreted as SQL syntax, regardless of what characters it contains. Avoid `system()` for shell commands and use `exec`-family with explicit argument arrays instead. Escape all special characters before outputting to HTML. And notice that `is_safe_filename` explicitly blocks `..` path components to prevent directory traversal - an allowlist of valid characters alone is not enough if you do not also check for traversal patterns.

### Q3: Use a fuzzer to stress-test parsing code and find crashes in input validation logic

**Answer:**

```cpp
// === Fuzz target for libFuzzer ===
// Compile: clang++ -fsanitize=fuzzer,address -std=c++20 fuzz_parser.cpp

#include <cstdint>
#include <cstddef>
#include <string_view>
#include <charconv>
#include <stdexcept>

// The parsing function we want to fuzz-test
struct Config {
    int port;
    int timeout_ms;
    int max_connections;
};

Config parse_config_line(std::string_view line) {
    Config cfg{};

    // Parse "port:timeout:max_conn"
    auto pos1 = line.find(':');
    if (pos1 == std::string_view::npos)
        throw std::invalid_argument("Missing first delimiter");

    auto pos2 = line.find(':', pos1 + 1);
    if (pos2 == std::string_view::npos)
        throw std::invalid_argument("Missing second delimiter");

    auto field1 = line.substr(0, pos1);
    auto field2 = line.substr(pos1 + 1, pos2 - pos1 - 1);
    auto field3 = line.substr(pos2 + 1);

    auto parse_field = [](std::string_view sv, int& out, int min, int max) {
        auto [ptr, ec] = std::from_chars(sv.data(), sv.data() + sv.size(), out);
        if (ec != std::errc{} || ptr != sv.data() + sv.size())
            throw std::invalid_argument("Invalid integer field");
        if (out < min || out > max)
            throw std::out_of_range("Field out of range");
    };

    parse_field(field1, cfg.port, 1, 65535);
    parse_field(field2, cfg.timeout_ms, 0, 300000);
    parse_field(field3, cfg.max_connections, 1, 10000);
    return cfg;
}

// libFuzzer entry point
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Convert raw bytes to string_view
    std::string_view input(reinterpret_cast<const char*>(data), size);

    try {
        auto config = parse_config_line(input);
        // If parsing succeeds, do basic sanity checks
        if (config.port < 1 || config.port > 65535) __builtin_trap();
        if (config.timeout_ms < 0) __builtin_trap();
        if (config.max_connections < 1) __builtin_trap();
    } catch (const std::exception&) {
        // Expected - invalid input should throw, not crash
    }
    // If we get here without crash/trap, the parser handled the input safely
    return 0;
}

// To build and run:
// $ clang++ -fsanitize=fuzzer,address,undefined -std=c++20 -O1 fuzz_parser.cpp -o fuzz
// $ ./fuzz corpus/ -max_len=256 -runs=1000000
//
// The fuzzer will:
// 1. Generate random byte sequences
// 2. Mutate and combine inputs from the corpus
// 3. Detect: crashes, ASan heap errors, UBSan violations, assertion failures
// 4. Save crashing inputs to crash-<hash> files
//
// Key flags:
//   -max_len=256        Limit input size
//   -runs=1000000       Number of test cases
//   -dict=config.dict   Dictionary of interesting tokens (":", "65535", etc.)
//   -jobs=4             Parallel fuzzing jobs
```

Fuzz testing feeds random and mutated inputs to your parser to find crashes, undefined behavior, and logic bugs. libFuzzer (integrated with Clang) is coverage-guided, meaning it tracks which code paths each input exercises and preferentially mutates inputs that reach new paths. This makes it much more effective than pure random testing - it finds the edge cases your unit tests missed. Combined with AddressSanitizer (`-fsanitize=address`) and UBSan (`-fsanitize=undefined`), it catches memory errors and undefined behavior that might not crash immediately on their own. The `__builtin_trap()` calls after successful parsing are there to verify post-conditions: if the parser claims to have succeeded but the values are out of range, that is a logic bug worth finding.

---

## Notes

- **Defense in depth:** Layer multiple protections - input validation, output sanitization, memory-safe types, and runtime sanitizers.
- **Allowlist over blocklist:** It's safer to define what's allowed than to try to enumerate all dangerous characters.
- **Never trust `system()`:** Use `execvp()` or `posix_spawn()` with explicit argv arrays. The shell interprets metacharacters.
- **Fuzz early, fuzz often:** Add fuzz targets for all parsing code and run them in CI pipelines.
- **CWE-20 (Improper Input Validation)** is consistently in the CWE Top 25 most dangerous vulnerabilities.
- Compile with `-std=c++20 -Wall -Wextra -fsanitize=address,undefined`.
