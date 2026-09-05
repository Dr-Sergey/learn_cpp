# Use std::print and std::println (C++23) for formatted output

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/io/print>  

---

## Topic Overview

C++23 adds `std::print` and `std::println` (header `<print>`) as direct replacements for the `std::cout << std::format(...)` pattern. They combine the type-safety of `std::format` with the convenience of a single function call, and they write directly to the output stream without allocating an intermediate string. If you already know `std::format`, you already know the format syntax - it's identical.

### Comparison

| Feature | `printf` | `cout << format(...)` | `std::print` / `std::println` |
| --- | --- | --- | --- |
| Type-safe | No | Yes | Yes |
| Format string syntax | `%d %s` | `{} {:x}` | `{} {:x}` (same as format) |
| Creates temp string | No | Yes (`format` returns `string`) | No (writes directly) |
| Newline | Manual `\n` | Manual `<< "\n"` | `println` adds newline automatically |
| Flushing | No (buffered) | No (unless `std::endl`) | No (unless writing to `stderr` or terminal) |
| Unicode | Platform-dependent | Poor | Handles UTF-8 correctly |
| Custom types | No | Yes (via `formatter`) | Yes (same `formatter`) |

### Signatures

```cpp
#include <print>

// Print to stdout:
std::print(fmt, args...);        // no trailing newline
std::println(fmt, args...);      // with trailing newline
std::println();                  // just a newline (C++26)

// Print to a specific stream:
std::print(stream, fmt, args...);
std::println(stream, fmt, args...);

// Print to stderr:
std::print(stderr, "Error: {}\n", msg);
```

### Core Examples

Notice that `std::println()` with no arguments gives you a bare newline - handy after a loop of `std::print` calls:

```cpp
#include <print>
#include <string>
#include <vector>

int main() {
    // Basic usage
    std::println("Hello, {}!", "world");      // Hello, world!
    std::print("No newline here. ");
    std::println("But here.");                 // No newline here. But here.

    // Formatted values
    int x = 42;
    double pi = 3.14159;
    std::println("x = {}, pi = {:.2f}", x, pi);  // x = 42, pi = 3.14

    // Alignment and fill
    std::println("|{:<10}|{:>10}|{:^10}|", "left", "right", "center");
    // |left      |     right|  center  |

    // Integer formats
    std::println("dec={0} hex={0:#x} oct={0:#o} bin={0:#b}", 255);
    // dec=255 hex=0xff oct=0377 bin=0b11111111

    // Print to stderr
    std::println(stderr, "Warning: value {} out of range", x);

    // Iterating (replaces cout loop)
    std::vector<int> nums = {1, 2, 3, 4, 5};
    for (int n : nums)
        std::print("{} ", n);
    std::println();  // trailing newline after: 1 2 3 4 5
}
```

### Custom Formatter with println

Any type that has a `std::formatter` specialization works with `std::println` automatically - you write the formatter once and it works everywhere:

```cpp
#include <print>
#include <format>
#include <string>
#include <vector>

struct Point {
    double x, y;
};

template <>
struct std::formatter<Point> : std::formatter<std::string> {
    auto format(const Point& p, auto& ctx) const {
        return std::formatter<std::string>::format(
            std::format("({:.1f}, {:.1f})", p.x, p.y), ctx);
    }
};

int main() {
    Point p{3.14, 2.71};
    std::println("Point: {}", p);  // Point: (3.1, 2.7)

    std::vector<Point> points = {{0, 0}, {1, 1}, {2, 4}};
    for (const auto& pt : points)
        std::println("  {}", pt);
    // (0.0, 0.0)
    // (1.0, 1.0)
    // (2.0, 4.0)
}
```

### Flushing Behavior

`std::print` does not flush after every write - it buffers for performance. This matters whenever you need output to appear immediately (progress bars, prompts, crash diagnostics):

```cpp
#include <print>
#include <thread>
#include <chrono>

int main() {
    // std::print does NOT flush — output is buffered for performance
    for (int i = 0; i < 100; ++i)
        std::print(".");  // dots accumulate in buffer

    // When does flushing happen?
    // 1. When buffer is full (implementation-defined, typically 4-8 KB)
    // 2. When the program ends normally
    // 3. When printing to a terminal (some implementations auto-flush)
    // 4. When you explicitly flush:
    std::fflush(stdout);

    // For progress indicators, explicit flush may be needed:
    for (int i = 0; i <= 100; i += 10) {
        std::print("\rProgress: {}%", i);
        std::fflush(stdout);  // ensure immediate display
    }
    std::println();  // final newline
}
```

---

## Self-Assessment

### Q1: Replace std::cout << std::format(...) with std::println for cleaner output code

**Answer:**

The output is identical - the difference is purely ergonomic. `std::println` removes the `std::cout <<`, the `<< "\n"`, and the intermediate string allocation all at once:

```cpp
#include <format>
#include <iostream>
#include <print>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    int age;
    double salary;
};

int main() {
    Employee emp{"Alice", 30, 75000.50};
    std::vector<int> scores = {95, 87, 92, 78, 100};

    // BEFORE (C++20): verbose, creates temp string
    std::cout << std::format("Name: {}, Age: {}, Salary: ${:.2f}\n",
                             emp.name, emp.age, emp.salary);
    std::cout << std::format("Scores: ");
    for (int s : scores)
        std::cout << std::format("{} ", s);
    std::cout << "\n";

    // AFTER (C++23): cleaner, no temp string, auto newline
    std::println("Name: {}, Age: {}, Salary: ${:.2f}",
                 emp.name, emp.age, emp.salary);
    std::print("Scores: ");
    for (int s : scores)
        std::print("{} ", s);
    std::println();

    // Output (identical for both):
    // Name: Alice, Age: 30, Salary: $75000.50
    // Scores: 95 87 92 78 100

    // Table formatting — much cleaner with println
    std::println("{:<15} {:>5} {:>12}", "Name", "Age", "Salary");
    std::println("{:-<15} {:->5} {:->12}", "", "", "");
    std::println("{:<15} {:>5} {:>12.2f}", emp.name, emp.age, emp.salary);
    // Name              Age       Salary
    // --------------- ----- ------------
    // Alice              30     75000.50
}
```

**Explanation:** `std::println` replaces the `std::cout << std::format(...) << "\n"` pattern. It uses the same format string syntax but writes directly to the stream (no intermediate `std::string` allocation) and appends a newline automatically. `std::print` is the same without the newline.

### Q2: Show that std::print does not flush by default and when that matters for performance

**Answer:**

The performance win from buffering is real - flushing on every write means a system call on every write, which can be 10-100x slower for many small outputs. The downside is that output may not appear immediately, and buffered output can be lost if the program crashes before the buffer flushes:

```cpp
#include <print>
#include <chrono>
#include <iostream>

int main() {
    // === Demonstration: print is buffered ===
    // This loop writes 10,000 dots to stdout.
    // They may not appear until the buffer flushes.
    auto start = std::chrono::steady_clock::now();

    for (int i = 0; i < 10'000; ++i)
        std::print(".");  // buffered — fast

    auto buffered_time = std::chrono::steady_clock::now() - start;

    std::println();  // newline + may trigger flush
    std::println("Buffered: {} us",
        std::chrono::duration_cast<std::chrono::microseconds>(buffered_time).count());

    // === When buffering MATTERS ===

    // 1. Progress bars — need immediate display
    for (int i = 0; i <= 100; i += 25) {
        std::print("\rProgress: {:3d}%", i);
        std::fflush(stdout);  // force display NOW
    }
    std::println();

    // 2. Interleaved stdout/stderr — order may be surprising
    std::print("stdout message ");          // buffered
    std::println(stderr, "stderr message");  // stderr is often unbuffered
    // stderr message may appear BEFORE stdout message!

    // 3. Crash debugging — buffered output may be lost
    std::print("About to do risky operation...");
    // If program crashes here, this message may never appear!
    // Fix: std::fflush(stdout); or use std::println(stderr, ...)

    // === Performance comparison concept ===
    // std::print: write to buffer -> kernel writes buffer to device (few syscalls)
    // std::cout with endl: write + flush EACH TIME (many syscalls)
    // The buffered approach can be 10-100x faster for many small writes

    std::println("\nDone.");
}
```

**Explanation:** `std::print` writes to a buffer (typically 4-8 KB). The OS flushes the buffer when it's full, when the program exits, or when explicitly requested. This is faster than flushing after every write (`std::endl`), but means output isn't immediately visible. For interactive output (progress bars, prompts), call `std::fflush(stdout)` after printing.

### Q3: Write a custom std::formatter and use it with std::println

**Answer:**

The `parse` method reads your custom format spec (the text between `:` and `}`), and the `format` method writes the output. Once the formatter is defined, the type works with `std::print`, `std::println`, `std::format`, and `std::format_to` - all using the same code path:

```cpp
#include <print>
#include <format>
#include <string>

struct Color {
    uint8_t r, g, b;
};

// Custom formatter for Color
// Supports {:hex} for hex format, default for "rgb(r, g, b)"
template <>
struct std::formatter<Color> {
    bool hex_mode = false;

    // Parse format spec: {} for default, {:hex} for hex
    constexpr auto parse(std::format_parse_context& ctx) {
        auto it = ctx.begin();
        if (it != ctx.end() && *it != '}') {
            // Check for "hex" spec
            if (*it == 'h') {
                hex_mode = true;
                it += 3;  // skip "hex"
            }
        }
        return it;
    }

    auto format(const Color& c, auto& ctx) const {
        if (hex_mode) {
            return std::format_to(ctx.out(), "#{:02x}{:02x}{:02x}",
                                  c.r, c.g, c.b);
        }
        return std::format_to(ctx.out(), "rgb({}, {}, {})",
                              c.r, c.g, c.b);
    }
};

int main() {
    Color red{255, 0, 0};
    Color sky{135, 206, 235};

    // Default format
    std::println("Red:  {}", red);    // Red:  rgb(255, 0, 0)
    std::println("Sky:  {}", sky);    // Sky:  rgb(135, 206, 235)

    // Hex format
    std::println("Red:  {:hex}", red);  // Red:  #ff0000
    std::println("Sky:  {:hex}", sky);  // Sky:  #87ceeb

    // Works in all format contexts
    auto s = std::format("Color is {:hex}", sky);
    std::println("{}", s);  // Color is #87ceeb

    // In tables
    Color palette[] = {{255,0,0}, {0,255,0}, {0,0,255}, {255,255,0}};
    std::println("{:<20} {}", "Default", "Hex");
    for (const auto& c : palette)
        std::println("{:<20} {:hex}", c, c);
    // rgb(255, 0, 0)       #ff0000
    // rgb(0, 255, 0)       #00ff00
    // rgb(0, 0, 255)       #0000ff
    // rgb(255, 255, 0)     #ffff00
}
```

**Explanation:** A custom `std::formatter<T>` specialization has two methods: `parse()` reads the format spec (text between `:` and `}`) and `format()` writes the output. Once defined, the type works with `std::print`, `std::println`, `std::format`, and `std::format_to` - all using the same formatter. The `parse` method enables custom format specifiers (`:hex` here).

---

## Notes

- **Availability:** `std::print` / `std::println` are C++23. Supported in GCC 14+, Clang 18+, MSVC 17.7+.
- **No `std::endl` equivalent:** `std::println` does NOT flush. Use `std::fflush(stdout)` if you need immediate output.
- **Unicode:** `std::print` handles UTF-8 correctly on Windows (sets console mode), unlike `printf` which may corrupt non-ASCII characters.
- **Binary size:** `std::print` may increase binary size compared to `printf` due to template instantiation, similar to `std::format`.
- **`std::println()` with no args:** C++26 allows `std::println()` as a simple newline. In C++23, use `std::println("")` or `std::print("\n")`.
- **Thread safety:** `std::print` to the same stream from multiple threads requires synchronization, same as `printf` or `cout`.
- Compile with `-std=c++23 -Wall -Wextra`.
