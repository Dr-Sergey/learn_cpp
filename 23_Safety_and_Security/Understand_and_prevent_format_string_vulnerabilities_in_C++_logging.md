# Understand and prevent format string vulnerabilities in C++ logging

**Category:** Safety and Security  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/format>  

---

## Topic Overview

Format string vulnerabilities occur when user-controlled input is passed as the format string argument to `printf`-family or logging functions. The attacker embeds format specifiers (`%x`, `%s`, `%n`) to read or write arbitrary memory. This topic focuses specifically on how these vulnerabilities manifest in logging frameworks and how C++20's `std::format`/`std::print` eliminate the attack surface entirely.

Logging is a particularly dangerous place for this vulnerability because developers tend to trust their own logging calls. The instinct is "it's just a log message" - but if user input flows into that message as the format string rather than as a data argument, the attacker is in the driver's seat.

### Attack Surface in Logging

Here's the flow that matters. The difference between the vulnerable path and the safe path is just where user input enters the pipeline:

```cpp
Logging call flow:
  Application code -> Logger API -> Format engine -> Output sink

VULNERABLE path (user controls format string):
  log(user_input)           ->  printf(user_input)
  spdlog::info(user_input)  ->  fmt::format(user_input)  [runtime_format]

SAFE path (user is only DATA):
  log("{}", user_input)             ->  printf("%s", user_input)
  spdlog::info("{}", user_input)    ->  fmt::format("{}", user_input)
```

The rule is the same in both cases: user input must always appear after a fixed, developer-controlled format string. Never as the format string itself.

### Why Logging Is Especially Dangerous

| Factor | Risk |
| --- | --- |
| Log messages often include user input | Attacker-controlled data reaches the format engine |
| Logging is everywhere in the codebase | Large attack surface |
| Developers assume logging is "safe" | Less scrutiny in code review |
| Log output may go to terminals, files, databases | Secondary injection targets (log injection) |

The "less scrutiny" point is worth dwelling on. In security-critical code like authentication or input validation, developers are naturally careful. In logging code, they often aren't - which is exactly why attackers target it.

### Core Example

Here's the simplest possible illustration: the same user input handled unsafely and safely, and then using the modern C++23 approach:

```cpp
#include <cstdio>
#include <format>
#include <print>
#include <string>

void unsafe_log(const char* message) {
    std::printf(message); // user_input IS the format string - VULNERABLE!
}

void safe_log_printf(const char* message) {
    std::printf("%s", message); // user_input is DATA, not format - safe
}

void safe_log_modern(const std::string& message) {
    std::println("{}", message); // C++23: format string is literal, user is argument
}

int main() {
    const char* user_input = "%x %x %x %n"; // attacker payload

    // unsafe_log(user_input);    // reads stack values, %n writes to memory!
    safe_log_printf(user_input);  // prints literal "%x %x %x %n"
    safe_log_modern(user_input);  // prints literal "%x %x %x %n"
}
```

The `unsafe_log` line is commented out because on a real system it would leak stack data and potentially write to memory. The two safe versions just print the `%x %x %x %n` string as literal text - those characters have no special meaning when the user is the argument rather than the format string.

---

## Self-Assessment

### Q1: Show why printf(user_input) is a format string vulnerability while printf("%s", user_input) is not

**Answer:**

This example actually runs the vulnerable call (commented out) and the safe one, so you can see exactly what output each produces and understand why the outputs differ:

```cpp
#include <cstdio>
#include <cstring>
#include <iostream>

// Compile: g++ -std=c++20 -Wall -Wformat -Wformat-security fmt_vuln.cpp

int main() {
    // The Vulnerability

    int secret = 0xDEADBEEF;
    int write_target = 0;

    // Simulated "user input" containing format specifiers
    const char* user_input = "%08x.%08x.%08x.%08x";

    // VULNERABLE: user_input is the FORMAT STRING
    std::printf("Vulnerable output: ");
    std::printf(user_input);
    //          ^^^^^^^^^^
    // printf interprets user_input as format string.
    // Each %x reads the next value from the stack.
    // Attacker sees stack contents: local variables, return addresses, canaries.
    //
    // Worse: %n WRITES to memory!
    //   printf("%100x%n") -> writes 100 to address read from stack
    //   This enables arbitrary write -> code execution

    std::printf("\n");

    // The Fix

    // SAFE: user_input is DATA (argument to %s)
    std::printf("Safe output: ");
    std::printf("%s", user_input);
    //          ^^^^  ^^^^^^^^^^
    // Format string is the literal "%s" - controlled by developer.
    // user_input is passed as the string argument.
    // printf outputs it verbatim: "%08x.%08x.%08x.%08x"
    // No format specifiers are interpreted!

    std::printf("\n");

    // Compiler Warning
    // With -Wformat-security, the compiler warns:
    //   warning: format string is not a string literal (potentially insecure)
    //
    // This catches: printf(variable)  but NOT: printf(variable) hidden behind
    // a function pointer or macro.

    // Output:
    // Vulnerable output: deadbeef.00000000.7ffd1234.004005a0  (stack leak!)
    // Safe output: %08x.%08x.%08x.%08x                       (literal text)
}
```

When `printf(user_input)` is called, `printf` scans `user_input` for `%` specifiers and reads arguments from the caller's stack frame. Since no extra arguments were passed, it reads whatever is on the stack - leaking secrets. With `printf("%s", user_input)`, the format string is the developer-controlled literal `"%s"`, so `user_input` is treated purely as a string value. The `%x` characters in `user_input` are printed literally.

### Q2: Demonstrate that std::format / std::print eliminates format string vulnerabilities by design

**Answer:**

This example shows not just that `std::format` is safer, but exactly why - and what happens if you try each of the attacks that work against `printf`:

```cpp
#include <format>
#include <print>
#include <string>
#include <iostream>

// std::format and std::print are IMMUNE to format string attacks because:
// 1. Format string must be a compile-time constant (consteval checked)
// 2. Arguments are type-safe (no varargs, no stack reading)
// 3. There is no %n equivalent (no write primitive)

int main() {
    std::string user_input = "%x %x %x %n";  // attacker payload

    // std::format: Format string is compile-time
    auto result = std::format("{}", user_input);
    std::cout << result << "\n";
    // Output: %x %x %x %n
    // The "{}" is a compile-time constant. user_input is the argument.
    // The %x characters are just data - std::format doesn't interpret them.

    // std::print: Same safety
    std::print("User said: {}\n", user_input);
    // Output: User said: %x %x %x %n

    // Why attack CANNOT work

    // Attempt 1: Pass malicious string as format
    // std::format(user_input);  // COMPILE ERROR!
    // Error: format string must be a compile-time constant
    //        (basic_format_string requires consteval construction)

    // Attempt 2: Use std::vformat with runtime string
    auto runtime_result = std::vformat(user_input, std::make_format_args());
    // This COMPILES but throws std::format_error at RUNTIME
    // because "%x" is not a valid std::format specifier.
    // Even if it were, there are no arguments to read - no stack leak.

    // Attempt 3: There is no {n} or equivalent write specifier
    // std::format has NO mechanism to write to memory via format specifiers.
    // The worst case is a runtime exception, never memory corruption.

    // Type safety comparison
    int value = 42;
    // printf: type mismatch is silent UB
    // printf("%s", value);  // UB: reads 42 as pointer, crashes or leaks

    // std::format: type mismatch is compile error
    // std::format("{:s}", value);  // Error: 's' not valid for int
    auto safe = std::format("{}", value);  // OK: "42"
    std::println("Value: {}", safe);

    // Logging library integration
    // spdlog (uses fmt::format internally):
    //   spdlog::info("{}", user_input);      // SAFE: user is data
    //   spdlog::info(user_input);            // DANGEROUS with fmt < 10
    //   spdlog::info(fmt::runtime(user_input)); // Explicit opt-in to runtime format

    std::println("All outputs are safe - attacker payload was treated as data.");
}
```

`std::format` requires the format string to be a compile-time constant (`consteval` validation via `basic_format_string`). This means user input literally cannot be used as the format string without explicit `std::vformat` - and even then, there is no `%n` equivalent and arguments are type-safe (no varargs stack reading). The vulnerability class is eliminated by design, not mitigated.

### Q3: Audit a logging framework for user-controlled format strings and fix them

**Answer:**

A real-world audit has four steps: find the vulnerable patterns, redesign the API to make them impossible, fix all call sites, and add CI checks to prevent regressions. Here's all four steps in code:

```cpp
#include <format>
#include <print>
#include <string>
#include <string_view>
#include <source_location>
#include <iostream>
#include <chrono>
#include <utility>
#include <cstdarg>

// Step 1: Identify vulnerable patterns

// Simulated logging framework (simplified spdlog-like API)
namespace old_logger {
    // VULNERABLE: accepts runtime format string
    void log(const char* fmt, ...) {
        // Uses va_list internally -> classic printf vulnerability
        va_list args;
        va_start(args, fmt);
        std::vprintf(fmt, args);
        va_end(args);
        std::printf("\n");
    }
}

// Vulnerable call sites to audit:
void audit_examples(const std::string& username, const std::string& action) {
    // PATTERN 1: User input directly as format string
    // old_logger::log(username.c_str());  // VULNERABLE!

    // PATTERN 2: Concatenated string as format string
    // auto msg = "User " + username + " did " + action;
    // old_logger::log(msg.c_str());  // VULNERABLE if username contains %x

    // PATTERN 3: Error messages from external systems
    // old_logger::log(db_error.c_str());  // VULNERABLE if DB returns %s

    // PATTERN 4: Correct usage (user input as argument)
    old_logger::log("User %s performed %s", username.c_str(), action.c_str());
    // SAFE: format string is literal, user data is argument
}

// Step 2: Build a safe logging API

namespace safe_logger {
    enum class Level { DEBUG, INFO, WARN, ERROR };

    constexpr std::string_view level_str(Level lv) {
        switch (lv) {
            case Level::DEBUG: return "DEBUG";
            case Level::INFO:  return "INFO";
            case Level::WARN:  return "WARN";
            case Level::ERROR: return "ERROR";
        }
        return "UNKNOWN";
    }

    // Safe: format string is compile-time, arguments are type-safe
    template <typename... Args>
    void log(Level level,
             std::format_string<Args...> fmt,  // compile-time validated!
             Args&&... args) {
        auto now = std::chrono::system_clock::now();
        auto time_str = std::format("{:%Y-%m-%d %H:%M:%S}", now);
        auto message = std::format(fmt, std::forward<Args>(args)...);
        std::println("[{}] [{}] {}", time_str, level_str(level), message);
    }

    template <typename... Args>
    void info(std::format_string<Args...> fmt, Args&&... args) {
        log(Level::INFO, fmt, std::forward<Args>(args)...);
    }

    template <typename... Args>
    void error(std::format_string<Args...> fmt, Args&&... args) {
        log(Level::ERROR, fmt, std::forward<Args>(args)...);
    }
}

// Step 3: Fix all call sites

void fixed_examples(const std::string& username, const std::string& action) {
    // ALL patterns: user input is always an ARGUMENT, never the format string

    safe_logger::info("User logged in: {}", username);
    safe_logger::info("User {} performed {}", username, action);

    // If you NEED dynamic format strings (localization, config), use vformat:
    std::string localized_pattern = "Benutzer: {}, Aktion: {}"; // from translation file
    auto msg = std::vformat(localized_pattern, std::make_format_args(username, action));
    safe_logger::info("Localized: {}", msg);

    // The localized_pattern is from a TRUSTED source (translation file).
    // If it came from user input, vformat would throw on invalid format specifiers.
}

// Step 4: Audit checklist

// clang-tidy checks:
//   -checks='cert-err33-c,bugprone-format*,modernize-use-std-print'
//
// Grep patterns to find vulnerable code:
//   grep -rn 'printf\s*(' --include='*.cpp' | grep -v '"%'
//   grep -rn 'log\s*(' --include='*.cpp' | grep -v '"'
//   grep -rn 'spdlog::\w\+(' --include='*.cpp' | grep -v '"'
//
// Manual review checklist:
//   [ ] Every printf/log call has a STRING LITERAL as first argument
//   [ ] No string concatenation before passing to format function
//   [ ] External error messages are passed as %s/{} arguments
//   [ ] Localized strings come from trusted sources (resource files)
//   [ ] CI enforces -Wformat -Wformat-security -Werror=format-security

int main() {
    std::string user = "alice%x%x%n"; // attacker trying format injection
    std::string act = "login";

    fixed_examples(user, act);
    // Output (safe):
    // [2024-01-15 10:30:45] [INFO] User logged in: alice%x%x%n
    // [2024-01-15 10:30:45] [INFO] User alice%x%x%n performed login
    // [2024-01-15 10:30:45] [INFO] Localized: Benutzer: alice%x%x%n, Aktion: login
    // The %x%n is printed literally - no stack leak, no memory write.
}
```

The audit process: (1) Find all logging calls where the first argument is a variable, not a string literal. (2) Replace the logging API to require `std::format_string<Args...>` which enforces compile-time format validation. (3) Fix every call site so user data is always a `{}` argument. (4) Enforce `-Wformat-security` and clang-tidy checks in CI to prevent regressions.

---

## Notes

- **CWE-134 (Use of Externally-Controlled Format String)** is in the Top 25.
- **`-Wformat-security`** (GCC/Clang) warns when format string is not a literal - enable with `-Werror=format-security`.
- **spdlog >= 1.13 / fmt >= 10:** By default require compile-time format strings; runtime strings need explicit `fmt::runtime()`.
- **Log injection:** Even with safe formatting, check for newline injection (`\n`) in log messages - attackers can forge log entries.
- **`std::vformat`** is the explicit opt-in for runtime format strings. It cannot write to memory (no `%n` equivalent) and throws on invalid specifiers.
- Compile with `-std=c++23 -Wformat -Wformat-security -Werror=format-security`.
