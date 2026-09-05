# Use std::format (C++20) for type-safe string formatting

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/format>  

---

## Topic Overview

`std::format` gives you Python-style string formatting with **compile-time type checking**. If you've ever passed the wrong type to `printf` and gotten garbage output - or worse, a security vulnerability - you'll appreciate what this buys you. It replaces `printf` (unsafe) and `std::ostringstream` (slow and verbose) with a fast, type-safe, extensible alternative.

### Format Syntax

The general shape of a format placeholder is `{[arg_id][:format_spec]}`, where:

```text
format_spec = [[fill]align][sign][#][0][width][.precision][type]
```

That looks like a lot of options, but most of the time you'll only use a couple. Here's a quick reference for the ones you'll reach for most:

| Spec | Meaning | Example |
| --- | --- | --- |
| `{}` | Default formatting | `format("{}", 42)` -> `"42"` |
| `{:d}` | Decimal integer | `format("{:d}", 42)` -> `"42"` |
| `{:x}` | Hex (lowercase) | `format("{:x}", 255)` -> `"ff"` |
| `{:#x}` | Hex with prefix | `format("{:#x}", 255)` -> `"0xff"` |
| `{:08x}` | Zero-padded hex | `format("{:08x}", 255)` -> `"000000ff"` |
| `{:.3f}` | 3 decimal places | `format("{:.3f}", 3.14)` -> `"3.140"` |
| `{:>10}` | Right-aligned, width 10 | `format("{:>10}", "hi")` -> `"        hi"` |
| `{:*^10}` | Center-aligned, fill `*` | `format("{:*^10}", "hi")` -> `"****hi****"` |

### Core Syntax

Here's a tour of the most common patterns. Notice how positional arguments let you reuse a value multiple times in one format string:

```cpp
#include <format>
#include <string>
#include <iostream>

int main() {
    // Basic usage
    std::string s = std::format("Hello, {}! You are {} years old.", "Alice", 30);
    std::cout << s << "\n";
    // Output: Hello, Alice! You are 30 years old.

    // Positional arguments
    std::cout << std::format("{1} before {0}", "world", "hello") << "\n";
    // Output: hello before world

    // Numeric formatting
    std::cout << std::format("dec={0:d} hex={0:#x} oct={0:#o} bin={0:#b}", 42) << "\n";
    // Output: dec=42 hex=0x2a oct=0o52 bin=0b101010

    // Floating point
    std::cout << std::format("pi ≈ {:.6f}", 3.14159265) << "\n";
    // Output: pi ≈ 3.141593

    // Table formatting
    std::cout << std::format("{:<10} {:>8} {:>8}\n", "Name", "Score", "Grade");
    std::cout << std::format("{:<10} {:>8} {:>8}\n", "Alice", 95, "A+");
    std::cout << std::format("{:<10} {:>8} {:>8}\n", "Bob", 82, "B");
}
```

---

## Self-Assessment

### Q1: Replace a printf with std::format and show how type safety prevents format-string attacks

**Answer:**

The key difference here isn't just syntax - it's that `std::format` validates your format string at compile time, while `printf` trusts you blindly at runtime.

```cpp
#include <format>
#include <string>
#include <iostream>
#include <cstdio>

int main() {
    int count = 42;
    double price = 9.99;
    const char* item = "widget";

    // --- printf: type-unsafe ---
    printf("Sold %d %s for $%.2f\n", count, item, price);
    // Works, but:
    // printf("Sold %s %s for $%.2f\n", count, item, price);  // UB! %s with int
    // printf("Sold %d %s for $%.2f %n\n", count, item, price);  // %n = write to memory!

    // --- std::format: type-safe ---
    std::string msg = std::format("Sold {} {} for ${:.2f}", count, item, price);
    std::cout << msg << "\n";
    // Output: Sold 42 widget for $9.99

    // Type mismatches are compile-time errors:
    // std::format("{:d}", "hello");  // ERROR: 'd' not valid for string
    // std::format("{}", );           // ERROR: too few arguments

    // No %n equivalent — cannot write to memory through format strings
    // No format-string attack vector — the format string is validated at compile time

    // --- Security: user input in format string ---
    std::string user_input = "{}{}{}{}{}{}";
    // printf(user_input.c_str());     // DANGEROUS: format string attack!
    // std::format(user_input, ...);  // Would need matching args or runtime error
    // Better: std::format("{}", user_input);  // Treats input as DATA, not format
    std::cout << std::format("{}", user_input) << "\n";
    // Output: {}{}{}{}{}{}  (literal text, not interpreted as format)
}
```

When user input lands in a `printf` format string, it can read or write arbitrary memory - that's a real exploit class. With `std::format`, user input is always passed as an argument, never as the format string itself, so those attacks simply don't apply.

**Type safety guarantees:**

- Wrong type specifiers -> compile-time error (with C++20 `consteval` format string checking).
- Wrong number of arguments -> compile-time error.
- No `%n` equivalent -> no memory write attacks.
- User strings passed as arguments (not format strings) are always safe.

---

### Q2: Implement a custom std::formatter specialization for a user-defined type

**Answer:**

Once you provide a `std::formatter<T>` specialization, your type works everywhere `std::format` is used - including `std::print`, `std::format_to`, and any library built on top of `<format>`. Here's how to write one that also supports custom format specs:

```cpp
#include <format>
#include <string>
#include <iostream>

struct Point {
    double x, y;
};

// Custom formatter for Point
template<>
struct std::formatter<Point> {
    // Parse format spec: supports 'f' (fixed) and 'e' (scientific)
    char presentation = 'f';

    constexpr auto parse(std::format_parse_context& ctx) {
        auto it = ctx.begin();
        if (it != ctx.end() && (*it == 'f' || *it == 'e')) {
            presentation = *it++;
        }
        if (it != ctx.end() && *it != '}') {
            throw std::format_error("invalid format");
        }
        return it;
    }

    auto format(const Point& p, std::format_context& ctx) const {
        if (presentation == 'e') {
            return std::format_to(ctx.out(), "({:e}, {:e})", p.x, p.y);
        }
        return std::format_to(ctx.out(), "({:.2f}, {:.2f})", p.x, p.y);
    }
};

// Another example: Color with hex output
struct Color {
    uint8_t r, g, b;
};

template<>
struct std::formatter<Color> {
    constexpr auto parse(std::format_parse_context& ctx) {
        return ctx.begin();
    }

    auto format(const Color& c, std::format_context& ctx) const {
        return std::format_to(ctx.out(), "#{:02X}{:02X}{:02X}", c.r, c.g, c.b);
    }
};

int main() {
    Point p{3.14159, 2.71828};
    std::cout << std::format("Point: {}", p) << "\n";
    std::cout << std::format("Point: {:f}", p) << "\n";
    std::cout << std::format("Point: {:e}", p) << "\n";
    // Output:
    // Point: (3.14, 2.72)
    // Point: (3.14, 2.72)
    // Point: (3.141590e+00, 2.718280e+00)

    Color red{255, 0, 0};
    Color teal{0, 128, 128};
    std::cout << std::format("Red: {}, Teal: {}", red, teal) << "\n";
    // Output: Red: #FF0000, Teal: #008080
}
```

The `parse` method reads everything between `:` and `}` and stores whatever spec you choose. The `format` method does the actual writing into the output context. Between the two, you have full control over how your type appears in any format string.

---

### Q3: Demonstrate std::format_to for formatting into a preallocated buffer

**Answer:**

`std::format` always allocates and returns a `std::string`. If you already have a buffer, or need to avoid that allocation in a hot path, the `format_to` family of functions lets you write directly:

```cpp
#include <format>
#include <string>
#include <array>
#include <vector>
#include <iostream>
#include <iterator>

int main() {
    // --- format_to with a char array ---
    std::array<char, 64> buffer{};
    auto result = std::format_to(buffer.data(), "x={}, y={}", 10, 20);
    *result = '\0';  // null-terminate
    std::cout << buffer.data() << "\n";
    // Output: x=10, y=20

    // --- format_to with back_inserter (growing buffer) ---
    std::string str;
    std::format_to(std::back_inserter(str), "{} + {} = {}", 3, 4, 7);
    std::cout << str << "\n";
    // Output: 3 + 4 = 7

    // --- format_to_n: limit output length ---
    std::array<char, 10> small_buf{};
    auto [out, size] = std::format_to_n(
        small_buf.data(), small_buf.size() - 1,
        "Hello, {}!", "World"
    );
    *out = '\0';
    std::cout << "truncated: " << small_buf.data()
              << " (would need " << size << " chars)\n";
    // Output: truncated: Hello, Wo (would need 13 chars)

    // --- formatted_size: query output size without writing ---
    auto needed = std::formatted_size("The answer is {}", 42);
    std::cout << "needed: " << needed << " chars\n";
    // Output: needed: 17 chars

    // --- Building a log line efficiently ---
    std::vector<char> log_buf;
    log_buf.reserve(256);
    std::format_to(std::back_inserter(log_buf),
                   "[{}] {}: {}", "INFO", "main", "started");
    std::string log_line(log_buf.begin(), log_buf.end());
    std::cout << log_line << "\n";
    // Output: [INFO] main: started
}
```

The structured binding `auto [out, size]` from `format_to_n` is useful: `out` is where writing stopped (so you can null-terminate), and `size` tells you how many characters the full output would have needed even if some were truncated.

**When to use each:**

- `std::format` -> returns `std::string`, simplest API.
- `std::format_to` -> writes to any output iterator, avoids extra allocation if you have a buffer.
- `std::format_to_n` -> writes at most N characters, safe for fixed-size buffers.
- `std::formatted_size` -> returns the size without writing, for pre-allocation.

---

## Notes

- `std::format` validates format strings at **compile time** (C++20 `consteval`). Invalid format strings are compile errors.
- `std::print` and `std::println` (C++23) use the same format syntax but write directly to a stream.
- `std::format` is faster than `std::ostringstream` and often competitive with `snprintf`.
- To format chrono types: `std::format("{:%Y-%m-%d %H:%M:%S}", time_point)` (C++20).
- Custom formatters inherit the full format spec language - you can support width, alignment, fill, precision.
- The `<format>` header is available in GCC 13+, Clang 17+, MSVC 19.29+.
