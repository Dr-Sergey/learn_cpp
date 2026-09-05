# Prevent format string vulnerabilities with type-safe formatting

**Category:** Safety and Security  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/format>  

---

## Topic Overview

This topic focuses on how `std::format`'s compile-time format string validation immunizes code against format string attacks, the vararg audit check in clang-tidy, and automated migration from `printf` to `std::format`/`std::print`. It complements #561 (attack demonstration and printf audit).

The key insight here is architectural: `printf` and `std::format` have fundamentally different designs, and the design of `std::format` makes an entire class of vulnerabilities structurally impossible.

### Why printf Is Fundamentally Broken for Security

Here's a side-by-side view of the two architectures. Notice that `printf` has no type information and no argument count - it's entirely driven by whatever the format string says:

```cpp
printf architecture:           std::format architecture:

  format string  -+            format string --+ (compile-time constant)
  va_list args   -+-> output   typed args -----+-> output
                  |             argument count -+  (checked at compile time)
  NO type info ---+
  NO arg count
  -> Attacker controls          -> CANNOT be controlled
    stack reading/writing         by user input
```

The `printf` side has no way to know how many arguments were passed or what their types are - it reads whatever the format string asks for, even if that means reading off the end of the stack.

### Compile-Time Safety of std::format

Here's the concrete comparison of what each function allows versus rejects:

| Property | `printf` | `std::format` |
| --- | --- | --- |
| Format string from variable | Allowed (dangerous) | Rejected (compile error) |
| Type mismatch | Silent UB | Compile error |
| Argument count mismatch | Silent UB | Compile error |
| Memory write (`%n`) | Possible | No equivalent |
| Stack reading (`%x`) | Possible | No equivalent |

The bottom two rows are the critical ones. There is simply no mechanism in `std::format` to read past the arguments you provided, or to write to memory via a format specifier. Those attack vectors don't exist in the API.

### Core Example

Let's see the practical difference: user-supplied `%x%x%x%n` is handled safely by `std::format` because it ends up as data, not as instructions:

```cpp
#include <format>
#include <iostream>
#include <string>

int main() {
    // printf: format string attacks possible
    // const char* user = "%x%x%x%n";
    // printf(user); // DANGEROUS: reads/writes stack memory

    // std::format: format string MUST be compile-time literal
    auto result = std::format("Hello, {}! You are {} years old.", "Alice", 30);
    std::cout << result << "\n";

    // Compile-time error examples:
    // std::format("{} {}", 1);       // ERROR: too few arguments
    // std::format("{}", 1, 2);       // ERROR: too many arguments
    // std::string fmt = "{}";
    // std::format(fmt, 42);          // ERROR: non-consteval format string
}
```

The commented-out error examples are important - they're things the compiler flat-out refuses to build. You get a compile error, not a runtime vulnerability.

---

## Self-Assessment

### Q1: Show a format string attack via printf(user_input) and how std::format prevents it

**Answer:**

This example shows both the vulnerable logger and the safe one, so you can see the exact line where the security boundary lies:

```cpp
#include <cstdio>
#include <format>
#include <iostream>
#include <string>

void vulnerable_logger(const char* msg) {
    // VULNERABLE: msg becomes the format string
    // printf(msg); // If msg = "%x%x%x%n", reads/writes stack!
    printf("[VULN] Would interpret: ");
    printf("%s", msg); // Safe demonstration
    printf("\n");
}

void safe_logger(const char* msg) {
    // SAFE: std::format only accepts compile-time format strings
    auto output = std::format("[SAFE] {}", msg);
    std::cout << output << "\n";
    // Even if msg contains "%x%x%n", it's just a string argument
}

int main() {
    const char* attack = "%08x.%08x.%08x.%n";

    vulnerable_logger(attack);
    // If we had used printf(attack):
    //   %08x  -> reads 4 bytes from stack (x3)
    //   %n    -> writes count to stack address -> arbitrary write!

    safe_logger(attack);
    // Output: [SAFE] %08x.%08x.%08x.%n
    // The format specifiers in the user input are treated as literal text

    // Anatomy of the safety guarantee
    // std::format("{}", value):
    //   1. "{}" is evaluated at compile time (consteval)
    //   2. Compile verifies: 1 placeholder, 1 argument, types match
    //   3. At runtime, value is formatted according to its TYPE
    //   4. User input NEVER reaches the format parser as a format string

    // This is immune because:
    // - The user can ONLY provide arguments, never the format string
    // - Arguments are formatted based on their C++ type, not format specifiers
    // - There's no mechanism equivalent to %n (memory write)
    // - Argument count is fixed at compile time (no stack overflow)
}
```

`printf(user_input)` treats the user string as the format string, allowing `%x` to read stack memory and `%n` to write to it. `std::format` requires the format string to be a compile-time constant - user input can only be passed as an argument through `{}` placeholders, which formats it according to its C++ type with no stack access.

### Q2: Explain why std::format is immune: the format string is checked at compile time

**Answer:**

The reason this is so secure is not a convention or a guideline - it's enforced by the C++ type system. Here's the deeper explanation of how `std::format_string` works and what the compiler actually validates:

```cpp
#include <format>
#include <iostream>
#include <string>
#include <source_location>

// Deep dive: WHY std::format is immune

// The format function signature (simplified):
// template<typename... Args>
// std::string format(std::format_string<Args...> fmt, Args&&... args);
//
// std::format_string is a consteval wrapper:
// - Its constructor only accepts compile-time string literals
// - The constructor VALIDATES the format string at compile time:
//   a) Counts {} placeholders
//   b) Matches each to the corresponding argument type
//   c) Validates format specifiers ({:d}, {:s}, etc.) against types
//
// If ANY check fails -> compile error, not runtime error

// What the compiler sees:
// std::format("{} is {} years old", name, age);
//
// compile-time check:
//   placeholder 0: {} -> matches arg 0 (name: const char*)  OK
//   placeholder 1: {} -> matches arg 1 (age: int)           OK
//   arg count: 2 placeholders, 2 args                      OK
//   -> PASS

// What the compiler rejects:
// std::format("{:d}", "hello");   // :d requires integer, got string -> ERROR
// std::format("{} {}", 1);        // 2 placeholders, 1 arg -> ERROR
// std::string fmt = "{}";
// std::format(fmt, 42);           // fmt is not consteval -> ERROR

int main() {
    // Proof: user input cannot be a format string
    std::string user_input = "%x %x %n"; // "malicious"

    // This is the ONLY way to use user input - as an argument:
    auto safe1 = std::format("Message: {}", user_input);
    std::cout << safe1 << "\n";
    // Output: Message: %x %x %n (literal characters)

    // You CANNOT do this (compile error):
    // auto bad = std::format(user_input, 42); // NOT consteval!

    // C++26 runtime_format: still safe
    // For dynamic format strings (from config files, etc.):
    // auto result = std::vformat(runtime_fmt, std::make_format_args(args...));
    // Even runtime_format:
    //   - Validates placeholder count matches argument count at runtime
    //   - Throws std::format_error on mismatch (not UB)
    //   - No %n equivalent (cannot write to memory)
    //   - No stack reading (args are in a type-erased arg store, not va_list)

    // Comparison of failure modes
    // printf("%d", "hello"):
    //   -> Silent undefined behavior (reads "hello" pointer as int)
    //   -> Compiler may warn but doesn't prevent it
    //
    // std::format("{:d}", "hello"):
    //   -> COMPILE ERROR: format specifier 'd' requires arithmetic type
    //   -> Code cannot be built until fixed
}
```

`std::format_string<Args...>` is a `consteval` type whose constructor validates the format string against the argument types at compile time. This is enforced by the language - there's no way to bypass it. Even with C++26's `runtime_format`, the type-erased argument store (`make_format_args`) prevents stack reading, and there's no `%n` equivalent.

The reason this trips people up is that it feels like `std::format` is just a safer `printf`. It's actually something more fundamental: the whole idea of "format string as user data" is not representable in the type system. The compile error is the proof.

### Q3: Audit a codebase for printf with non-literal format strings using clang-tidy's cppcoreguidelines-pro-type-vararg

**Answer:**

Here's a comprehensive audit strategy. The goal is zero `printf`-family calls in new code, and a clear migration path for old code:

```cpp
// Comprehensive audit strategy

// 1. clang-tidy: cppcoreguidelines-pro-type-vararg
//    Flags ALL uses of C-style variadic functions (printf, scanf, etc.)
//    This is the nuclear option - forces complete migration away from varargs.
//
//    Command:
//    clang-tidy -checks='cppcoreguidelines-pro-type-vararg' \
//               -warnings-as-errors='*' src/*.cpp
//
//    Findings:
//    src/logger.cpp:42: warning: do not call c-style variadic functions
//        printf("User: %s\n", name);
//                              ^

// 2. Clang/GCC -Wformat-nonliteral (stricter than -Wformat-security)
//    Warns when format string is not a string literal,
//    EVEN IF format arguments are provided.
//
//    void log(const char* fmt, ...) {
//        va_list args;
//        va_start(args, fmt);
//        vprintf(fmt, args);  // WARNING: format is not a literal
//        va_end(args);
//    }

// 3. modernize-use-std-print (clang-tidy 17+)
//    AUTOMATICALLY rewrites printf -> std::print
//
//    Before:  printf("Hello %s, you are %d\n", name, age);
//    After:   std::print("Hello {}, you are {}\n", name, age);
//
//    Command:
//    clang-tidy -checks='modernize-use-std-print' -fix src/*.cpp

// 4. Semi-automated grep audit script:
//
// #!/bin/bash
// echo "=== Non-literal printf format strings ==="
// grep -rn 'printf\s*(' src/ | grep -v 'printf\s*("' | grep -v '//'
//
// echo "=== All sprintf (prefer std::format) ==="
// grep -rn 'sprintf\s*(' src/ | grep -v '//'
//
// echo "=== All snprintf (prefer std::format) ==="
// grep -rn 'snprintf\s*(' src/ | grep -v '//'
//
// echo "=== syslog with non-literal format ==="
// grep -rn 'syslog\s*(' src/ | grep -v 'syslog\s*([^,]*,"'

// 5. .clang-tidy configuration for CI:
// ---
// Checks: >
//   -*,
//   cppcoreguidelines-pro-type-vararg,
//   modernize-use-std-print,
//   cert-err33-c
// WarningsAsErrors: >
//   cppcoreguidelines-pro-type-vararg

// Real-world migration example
#include <cstdio>
#include <format>
#include <iostream>
#include <string>

// BEFORE (vulnerable to format string attacks if log_msg comes from user):
void old_log(const char* log_msg) {
    // printf(log_msg); // FLAGGED by all checks above
    printf("[LOG] ");
    printf("%s", log_msg);  // safe but still uses varargs
    printf("\n");
}

// AFTER (completely safe, flagged by modernize-use-std-print):
void new_log(std::string_view log_msg) {
    auto output = std::format("[LOG] {}\n", log_msg);
    std::cout << output;  // or std::print("[LOG] {}\n", log_msg);
}

int main() {
    old_log("test message %x%x%n"); // safe because we use %s
    new_log("test message %x%x%n"); // safe by design
}
```

A layered audit combines: (1) `cppcoreguidelines-pro-type-vararg` to flag all variadic function usage, (2) `-Wformat-nonliteral` to catch non-literal format strings even in wrapper functions, (3) `modernize-use-std-print` for automated printf-to-print migration, and (4) grep-based searches for patterns static analysis might miss. The goal is zero `printf`-family calls in new code.

---

## Notes

- **`std::format_string`** uses `consteval` - the format is checked at compile time, not runtime. This is a language-level guarantee, not just a library convention.
- **`modernize-use-std-print`** (clang-tidy 17+) can automatically rewrite `printf` calls to `std::print` calls.
- **Wrapper functions** that forward format strings to `vprintf` are especially dangerous - use `__attribute__((format(printf, N, M)))` to enable compiler checks, or better yet, switch to `std::format`.
- **`_FORTIFY_SOURCE`** (glibc) disables `%n` at runtime as a defense-in-depth measure, but don't rely on it.
- Compile with `-std=c++20 -Wall -Wextra -Wformat -Wformat-security -Wformat-nonliteral`.
