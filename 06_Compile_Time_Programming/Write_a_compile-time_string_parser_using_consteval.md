# Write a Compile-Time String Parser Using `consteval`

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/consteval>  

---

## Topic Overview

### Compile-Time String Parsing

`consteval` functions **must** execute at compile time. There is no runtime fallback - if the compiler cannot evaluate the call during compilation, the code is rejected. This makes `consteval` the ideal foundation for parsers that validate string literals: any invalid input produces a compile error with a message instead of a runtime exception or silent bad data.

Combined with `std::string_view` (which is `constexpr`-friendly and avoids any allocation), you can write parsers for fixed-format strings - IP addresses, date strings, version numbers, color codes - that run entirely at compile time for constants defined in source code. The key insight is that you are not losing anything: these are literals that never change, so there is no reason to parse them at runtime.

### Key Pattern

The general shape is always the same: a `consteval` function that takes a `std::string_view`, does character-by-character validation, builds the result struct, and throws a string literal on error. The `throw` in a `consteval` context becomes a compile error.

```cpp
struct IPv4 { uint8_t octets[4]; };

consteval IPv4 parse_ip(std::string_view sv) {
    // Parse and validate - any error is a compile error
    // ...
}

constexpr auto addr = parse_ip("192.168.1.1");  // OK: compile-time parsed
// constexpr auto bad = parse_ip("999.0.0.1");  // ERROR: compile error!
```

### Why Compile-Time Parsing

| Benefit | Description |
| --- | --- |
| Zero runtime cost | Parsed result is a constant in the binary |
| Error at compile time | Invalid literals fail compilation, not at runtime |
| No startup parsing | No `inet_pton()` calls during initialization |
| Self-documenting | Parser code serves as format specification |

---

## Self-Assessment

### Q1: Implement a `consteval` IP address parser that converts `"192.168.1.1"` to four `uint8_t` values

The parser walks through the string one character at a time. Each digit accumulates into `current_value`. When it hits a dot (or the end of string), it validates the accumulated value and stores it as the next octet. Any condition that violates the IPv4 format throws a string literal, which becomes a compile error with that string in the diagnostic output.

```cpp
#include <iostream>
#include <cstdint>
#include <string_view>
#include <stdexcept>
#include <array>

struct IPv4 {
    uint8_t octets[4]{};

    friend std::ostream& operator<<(std::ostream& os, const IPv4& ip) {
        return os << static_cast<int>(ip.octets[0]) << "."
                  << static_cast<int>(ip.octets[1]) << "."
                  << static_cast<int>(ip.octets[2]) << "."
                  << static_cast<int>(ip.octets[3]);
    }
};

// === consteval IP parser ===
consteval IPv4 parse_ip(std::string_view sv) {
    IPv4 result{};
    int octet_idx = 0;
    int current_value = 0;
    int digit_count = 0;

    for (std::size_t i = 0; i <= sv.size(); ++i) {
        if (i == sv.size() || sv[i] == '.') {
            // Validate
            if (digit_count == 0)
                throw "Empty octet in IP address";
            if (current_value > 255)
                throw "Octet value > 255";
            if (octet_idx > 3)
                throw "Too many octets";

            result.octets[octet_idx] = static_cast<uint8_t>(current_value);
            ++octet_idx;
            current_value = 0;
            digit_count = 0;
        } else if (sv[i] >= '0' && sv[i] <= '9') {
            current_value = current_value * 10 + (sv[i] - '0');
            ++digit_count;
            if (digit_count > 3)
                throw "Octet has too many digits";
        } else {
            throw "Invalid character in IP address";
        }
    }

    if (octet_idx != 4)
        throw "IP address must have exactly 4 octets";

    return result;
}

// === Compile-time verified IP addresses ===
constexpr auto localhost  = parse_ip("127.0.0.1");
constexpr auto gateway    = parse_ip("192.168.1.1");
constexpr auto broadcast  = parse_ip("255.255.255.255");
constexpr auto google_dns = parse_ip("8.8.8.8");

// Compile-time assertions
static_assert(localhost.octets[0] == 127);
static_assert(localhost.octets[3] == 1);
static_assert(gateway.octets[2] == 1);
static_assert(broadcast.octets[0] == 255);

int main() {
    std::cout << "=== Compile-Time IP Parser ===\n";
    std::cout << "localhost:  " << localhost << "\n";
    std::cout << "gateway:    " << gateway << "\n";
    std::cout << "broadcast:  " << broadcast << "\n";
    std::cout << "google_dns: " << google_dns << "\n";

    std::cout << "\nAll addresses parsed and validated at compile time.\n";
    std::cout << "Invalid literals would cause a compile error.\n";

    return 0;
}
```

At runtime, the function body never runs. The compiled binary contains only four `IPv4` structs as read-only data. The string literals `"127.0.0.1"` etc. do not even need to appear in the binary.

**Expected output:**

```text
=== Compile-Time IP Parser ===
localhost:  127.0.0.1
gateway:    192.168.1.1
broadcast:  255.255.255.255
google_dns: 8.8.8.8

All addresses parsed and validated at compile time.
Invalid literals would cause a compile error.
```

### Q2: Show the compile error when an invalid IP address literal is used

The key insight is that in a `consteval` context, a `throw` expression is not an exception - it is a compile error. The string you throw is not an exception message; it is a compiler diagnostic. This turns format validation into a compile-time guarantee: you physically cannot produce a binary that contains an invalid IP address constant.

The reason this works is that the C++ standard says a constant expression may not contain a `throw`. So when the `consteval` evaluator hits a `throw`, the entire constant evaluation fails, and the compiler reports an error at the call site - with the throw message appearing in the compiler's diagnostic output.

```cpp
#include <iostream>
#include <cstdint>
#include <string_view>

struct IPv4 { uint8_t octets[4]{}; };

consteval IPv4 parse_ip(std::string_view sv) {
    IPv4 result{};
    int octet_idx = 0;
    int current_value = 0;
    int digit_count = 0;

    for (std::size_t i = 0; i <= sv.size(); ++i) {
        if (i == sv.size() || sv[i] == '.') {
            if (digit_count == 0)
                throw "Empty octet";
            if (current_value > 255)
                throw "Octet value exceeds 255";
            if (octet_idx > 3)
                throw "Too many octets (expected 4)";
            result.octets[octet_idx++] = static_cast<uint8_t>(current_value);
            current_value = 0;
            digit_count = 0;
        } else if (sv[i] >= '0' && sv[i] <= '9') {
            current_value = current_value * 10 + (sv[i] - '0');
            ++digit_count;
        } else {
            throw "Invalid character in IP address";
        }
    }
    if (octet_idx != 4) throw "Expected exactly 4 octets";
    return result;
}

int main() {
    // === Valid addresses compile fine ===
    constexpr auto ok1 = parse_ip("10.0.0.1");
    constexpr auto ok2 = parse_ip("192.168.0.1");

    std::cout << "Valid addresses compiled successfully.\n";

    // === UNCOMMENT ANY LINE BELOW TO SEE A COMPILE ERROR ===

    // constexpr auto err1 = parse_ip("999.0.0.1");
    // ERROR: "Octet value exceeds 255"
    // The throw in consteval is a compile error, not a runtime exception.

    // constexpr auto err2 = parse_ip("1.2.3.4.5");
    // ERROR: "Too many octets (expected 4)"

    // constexpr auto err3 = parse_ip("1.2.3");
    // ERROR: "Expected exactly 4 octets"

    // constexpr auto err4 = parse_ip("1.2.abc.4");
    // ERROR: "Invalid character in IP address"

    // constexpr auto err5 = parse_ip("1..2.3");
    // ERROR: "Empty octet"

    std::cout << "\n=== How It Works ===\n";
    std::cout << "consteval functions MUST run at compile time.\n";
    std::cout << "A throw in consteval = compile error (not runtime exception).\n";
    std::cout << "The error message appears in the compiler output.\n";
    std::cout << "\nExample compiler error for parse_ip(\"999.0.0.1\"):\n";
    std::cout << "  error: expression '<throw-expression>' is not a constant expression\n";
    std::cout << "  note: subexpression 'throw \"Octet value exceeds 255\"'\n";

    return 0;
}
```

Try uncommenting one of the error lines to see what the compiler says. The throw message will appear in the diagnostic, giving you a human-readable explanation of exactly what went wrong - right at the line in your source where you wrote the bad literal.

**Expected output (when all invalid lines are commented):**

```text
Valid addresses compiled successfully.

=== How It Works ===
consteval functions MUST run at compile time.
A throw in consteval = compile error (not runtime exception).
The error message appears in the compiler output.

Example compiler error for parse_ip("999.0.0.1"):
  error: expression '<throw-expression>' is not a constant expression
  note: subexpression 'throw "Octet value exceeds 255"'
```

### Q3: Compare compile-time parsing vs runtime parsing in terms of binary size and startup cost

This benchmark makes the performance benefit concrete by parsing the same set of addresses both at compile time and at runtime, then comparing how long the runtime parsing takes. Note the important limitation in the comparison table's last row: compile-time parsing only applies to string literals known at compile time. If the user types an IP address at runtime, you still need a runtime parser.

```cpp
#include <iostream>
#include <cstdint>
#include <string_view>
#include <chrono>
#include <cstring>  // for runtime parsing comparison
#include <cstddef>

struct IPv4 { uint8_t octets[4]{}; };

// === Compile-time parser ===
consteval IPv4 parse_ip_ct(std::string_view sv) {
    IPv4 result{};
    int idx = 0, val = 0, digits = 0;
    for (std::size_t i = 0; i <= sv.size(); ++i) {
        if (i == sv.size() || sv[i] == '.') {
            if (val > 255 || digits == 0 || idx > 3) throw "Invalid IP";
            result.octets[idx++] = static_cast<uint8_t>(val);
            val = 0; digits = 0;
        } else if (sv[i] >= '0' && sv[i] <= '9') {
            val = val * 10 + (sv[i] - '0'); ++digits;
        } else throw "Bad char";
    }
    if (idx != 4) throw "Need 4 octets";
    return result;
}

// === Runtime parser (same algorithm) ===
IPv4 parse_ip_rt(const char* s) {
    IPv4 result{};
    int idx = 0, val = 0;
    while (*s) {
        if (*s == '.') {
            result.octets[idx++] = static_cast<uint8_t>(val);
            val = 0;
        } else {
            val = val * 10 + (*s - '0');
        }
        ++s;
    }
    result.octets[idx] = static_cast<uint8_t>(val);
    return result;
}

// === Compile-time addresses: zero cost at startup ===
constexpr IPv4 addresses[] = {
    parse_ip_ct("10.0.0.1"),
    parse_ip_ct("192.168.1.1"),
    parse_ip_ct("172.16.0.1"),
    parse_ip_ct("8.8.8.8"),
    parse_ip_ct("8.8.4.4"),
    parse_ip_ct("1.1.1.1"),
    parse_ip_ct("255.255.255.0"),
    parse_ip_ct("127.0.0.1"),
};

int main() {
    // === Compare startup cost ===
    std::cout << "=== Compile-Time vs Runtime Parsing ===\n\n";

    const char* ip_strings[] = {
        "10.0.0.1", "192.168.1.1", "172.16.0.1", "8.8.8.8",
        "8.8.4.4", "1.1.1.1", "255.255.255.0", "127.0.0.1"
    };

    auto start = std::chrono::high_resolution_clock::now();
    volatile IPv4 rt_results[8];  // volatile to prevent optimization
    for (int iter = 0; iter < 10000; ++iter) {
        for (int i = 0; i < 8; ++i) {
            rt_results[i] = parse_ip_rt(ip_strings[i]);
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();

    std::cout << "Runtime: " << us << " us to parse 8 IPs x 10000 iterations\n";
    std::cout << "Compile-time: 0 us (already in binary)\n";

    std::cout << "\n=== Comparison Table ===\n";
    std::cout << "+-------------------+-------------------+--------------------+\n";
    std::cout << "| Aspect            | Runtime Parsing   | Compile-Time       |\n";
    std::cout << "+-------------------+-------------------+--------------------+\n";
    std::cout << "| Startup cost      | O(n) per address  | 0                  |\n";
    std::cout << "| Validation        | At runtime        | At compile time    |\n";
    std::cout << "| Binary size (data)| String literals   | Parsed struct      |\n";
    std::cout << "| Error handling    | Exceptions/errno  | Compile error      |\n";
    std::cout << "| Dynamic input     | Supported         | Literals only      |\n";
    std::cout << "+-------------------+-------------------+--------------------+\n";

    std::cout << "\nCompile-time parsed addresses:\n";
    for (const auto& a : addresses) {
        std::cout << "  " << static_cast<int>(a.octets[0]) << "."
                  << static_cast<int>(a.octets[1]) << "."
                  << static_cast<int>(a.octets[2]) << "."
                  << static_cast<int>(a.octets[3]) << "\n";
    }

    return 0;
}
```

Compile-time parsing is for constants, not for user input. For anything the user types or reads from a file, you still need a runtime parser - `consteval` simply does not apply there. But for your configuration constants, compile-time parsing gives you free validation and zero startup cost.

**Expected output (timing varies):**

```text
=== Compile-Time vs Runtime Parsing ===

Runtime: 385 us to parse 8 IPs x 10000 iterations
Compile-time: 0 us (already in binary)

=== Comparison Table ===
+-------------------+-------------------+--------------------+
| Aspect            | Runtime Parsing   | Compile-Time       |
+-------------------+-------------------+--------------------+
| Startup cost      | O(n) per address  | 0                  |
| Validation        | At runtime        | At compile time    |
| Binary size (data)| String literals   | Parsed struct      |
| Error handling    | Exceptions/errno  | Compile error      |
| Dynamic input     | Supported         | Literals only      |
+-------------------+-------------------+--------------------+

Compile-time parsed addresses:
  10.0.0.1
  192.168.1.1
  172.16.0.1
  8.8.8.8
  8.8.4.4
  1.1.1.1
  255.255.255.0
  127.0.0.1
```

---

## Notes

- `consteval` functions that `throw` produce compile errors with the throw message - a clean error reporting mechanism.
- Use `std::string_view` (not `std::string`) in `consteval` - it's always available at compile time.
- Parsed results live in `.rodata` - no runtime allocation, no startup cost.
- Compile-time parsing is ideal for configuration constants: IP addresses, URLs, date formats, regex patterns.
- Limitation: only works for literals known at compile time. User input still needs runtime parsing.
