# Prevent format string vulnerabilities and audit for unsafe printf-family calls

**Category:** Safety and Security  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/format>  

---

## Topic Overview

A format string vulnerability occurs when user-controlled input is passed as the format string to `printf`-family functions. This allows attackers to read or write arbitrary memory. This topic covers the attack mechanism, how `std::format`/`std::print` (C++20/23) eliminate the vulnerability by design, and how to audit codebases for unsafe patterns.

The core rule is simple: the format string must always be a hard-coded literal that you control, never a variable that comes from outside your program.

### The Vulnerability

Here's the shortest way to see what "vulnerable" versus "safe" looks like at a glance:

```cpp
VULNERABLE:   printf(user_input);           // user controls format specifiers
SAFE:         printf("%s", user_input);      // format string is a literal
SAFE:         std::print("{}", user_input);  // type-safe, no format interpretation
```

The difference is whether `user_input` is treated as a format string or as data. When it's the format string, the attacker gets to drive.

### Attack Surface

The same bug shows up across the whole family of C formatting functions:

| Function | Vulnerable pattern | Safe pattern |
| --- | --- | --- |
| `printf` | `printf(buf)` | `printf("%s", buf)` |
| `sprintf` | `sprintf(dst, buf)` | `snprintf(dst, sz, "%s", buf)` |
| `fprintf` | `fprintf(fp, buf)` | `fprintf(fp, "%s", buf)` |
| `syslog` | `syslog(LOG_INFO, buf)` | `syslog(LOG_INFO, "%s", buf)` |

### What an Attacker Can Do

The reason this vulnerability is so dangerous is the range of things an attacker can accomplish just by crafting the right input string:

```cpp
Input: "%x %x %x %x"       -> Leak stack values (information disclosure)
Input: "%s"                  -> Read memory at stack-pointed address (crash or leak)
Input: "%n"                  -> WRITE to memory (arbitrary write primitive!)
Input: "%100000d"            -> Buffer overflow (extremely long output)
```

Notice that `%n` moves from reading to writing. That single specifier turns an information leak into full arbitrary memory write capability - which is how attackers achieve code execution.

### Core Example

Let's look at all three approaches side by side so the contrast is obvious:

```cpp
#include <cstdio>
#include <format>
#include <iostream>
#include <string>

int main() {
    // VULNERABLE
    const char* user_input = "%x %x %x %x"; // attacker-controlled

    // printf(user_input);  // DANGEROUS: interprets %x as format specifiers
                            // Leaks stack values!

    // SAFE alternatives
    printf("%s\n", user_input);                   // C-style safe: treats as string
    std::cout << user_input << "\n";              // iostream: no format interpretation
    std::string result = std::format("{}", user_input); // C++20: type-safe
    std::cout << result << "\n";                  // Output: "%x %x %x %x" (literal)
}
```

All three safe versions print the literal text `%x %x %x %x` rather than leaking stack values. The format specifiers in the input are just characters, not instructions.

---

## Self-Assessment

### Q1: Show that printf(user_input) is a format string vulnerability while printf("%s", user_input) is not

**Answer:**

This example shows exactly how the stack leaks happen and why the fix works. Read the comments carefully - the mechanism is subtle but important:

```cpp
#include <cstdio>
#include <cstring>

// This demonstrates the vulnerability. DO NOT use in production.
void demonstrate_format_string_vuln() {
    int secret = 0xDEADBEEF;
    char buffer[256];

    // Simulated attacker input
    const char* malicious = "%08x.%08x.%08x.%08x";

    // VULNERABLE: user input IS the format string
    // printf(malicious);
    // This would print hex values from the stack, potentially including
    // the secret variable or return addresses.
    // Output might be: cff6f7e0.00000001.deadbeef.0804a000
    //                                    ^^^^^^^^ leaked secret!

    // Even worse with %n:
    // const char* write_attack = "AAAA%n";
    // printf(write_attack);
    // %n writes the number of bytes printed so far to an address on the stack
    // This gives arbitrary memory WRITE capability!

    // SAFE: format string is a literal, user input is an argument
    printf("User said: %s\n", malicious);
    // Output: User said: %08x.%08x.%08x.%08x
    // The %08x is treated as literal text, not format specifiers.

    // WHY printf(user) is dangerous
    // printf("format", args...)  - first arg is the FORMAT STRING
    // printf(user)               - user controls the format string
    //   %x  -> reads next value from stack (leak)
    //   %s  -> reads string from stack address (crash/leak)
    //   %n  -> writes to stack address (arbitrary write)
    //   %99999d -> writes huge output (buffer overflow)

    // Compiler warning (GCC/Clang):
    // -Wformat-security warns about printf(variable)
    // -Werror=format-security makes it a compile error
    (void)secret;
    (void)buffer;
}

int main() {
    demonstrate_format_string_vuln();
    // Output: User said: %08x.%08x.%08x.%08x
}
```

When `printf(user_input)` is called, `printf` scans the user-provided string for `%` specifiers and reads additional arguments from the stack for each one. Since no arguments were passed, it reads arbitrary stack memory. The `%n` specifier is especially dangerous because it writes the count of characters printed so far to a memory address, enabling arbitrary memory writes. `printf("%s", user_input)` is safe because the format string is the hardcoded `"%s"` literal - the user input is just a string argument, treated as data, not as instructions.

### Q2: Explain how std::format and std::print eliminate format string attacks by design

**Answer:**

The reason `std::format` is immune is not just a convention - it's enforced by the language itself at compile time. Pay attention to the comments explaining why certain lines are compile errors:

```cpp
#include <format>
#include <iostream>
#include <string>
#include <string_view>

int main() {
    std::string user_input = "%x %x %x %n"; // "malicious" input

    // std::format: compile-time format string check

    // 1. Format string MUST be a compile-time constant:
    std::string safe = std::format("User said: {}", user_input);
    std::cout << safe << "\n";
    // Output: User said: %x %x %x %n
    // The "%x" and "%n" in user_input are just characters, not specifiers!

    // 2. You CANNOT pass a runtime string as the format:
    // std::string fmt = "{}";
    // std::format(fmt, 42);        // COMPILE ERROR: fmt is not consteval
    // This is the key security property!

    // 3. Argument count is checked at compile time:
    // std::format("{} {}", 1);     // COMPILE ERROR: too few arguments
    // std::format("{}", 1, 2);     // COMPILE ERROR: too many arguments

    // 4. Type safety - no stack reading:
    // std::format uses variadic templates, not va_args
    // Each argument's type is known at compile time
    // No mechanism to "read past" the provided arguments

    // Why printf is fundamentally vulnerable
    // printf uses C variadic arguments (va_list)
    // - No type information about arguments
    // - No count of arguments
    // - Format string controls how many stack values to read
    // - %n writes to memory through va_arg
    //
    // std::format uses C++ variadic templates
    // - Full type information for every argument
    // - Exact argument count known at compile time
    // - Format string validated at compile time
    // - No mechanism equivalent to %n

    // std::print (C++23): same safety + direct output
    // std::print("User: {}\n", user_input); // safe, compile-time checked

    // Runtime format strings (when needed)
    // C++26 adds std::runtime_format for dynamic format strings
    // Even then, the argument count and types are checked:
    // std::vformat(runtime_fmt, std::make_format_args(arg1, arg2));
    // This can throw std::format_error at runtime, but CANNOT
    // read arbitrary stack memory or write to memory.
}
```

`std::format` requires the format string to be a compile-time constant (checked via `consteval`). This makes it impossible for user input to be interpreted as a format string. Additionally, `std::format` uses C++ variadic templates (not C `va_args`), so the number and types of arguments are known at compile time - there's no mechanism to read extra values from the stack or write to memory via `%n`. The vulnerability class is eliminated, not just mitigated.

### Q3: Audit a codebase for unsafe printf-family calls using clang-tidy's cert-err33-c check

**Answer:**

A good audit uses three layers: compiler flags, static analysis, and grep. Each one catches things the others might miss:

```cpp
// Auditing with compiler warnings and clang-tidy

// Step 1: Enable compiler format warnings (GCC/Clang)
// g++ -Wformat -Wformat-security -Werror=format-security code.cpp
//
// -Wformat:          Warn about format string/argument type mismatches
// -Wformat-security: Warn when format string is not a string literal
//                    and has no format arguments
// -Werror=format-security: Make it a hard error

// Step 2: Use clang-tidy checks
// clang-tidy -checks='cert-*,cppcoreguidelines-pro-type-vararg' code.cpp
//
// cert-err33-c:                     Check return values of C library functions
// cppcoreguidelines-pro-type-vararg: Flag all uses of C-style varargs
//
// The vararg check flags ALL printf-family calls, which is aggressive
// but forces migration to std::format/std::print.

// Step 3: Automated grep audit
// Find all printf-family calls where format is not a literal:
// grep -rn 'printf\s*(' src/ | grep -v 'printf\s*("'   # non-literal format
// grep -rn 'sprintf\s*(' src/                            # all sprintf (unsafe)
// grep -rn 'system\s*(' src/                             # shell injection risk

// Example of what each tool catches

#include <cstdio>
#include <string>

void example_audit() {
    const char* user = "alice";
    std::string msg = "hello";

    // Caught by -Wformat-security:
    // printf(user);                    // WARNING: format not a literal
    // printf(msg.c_str());             // WARNING: format not a literal

    // OK - format is a literal:
    printf("Name: %s\n", user);         // OK
    printf("Message: %s\n", msg.c_str()); // OK

    // Caught by cppcoreguidelines-pro-type-vararg:
    // printf("Name: %s\n", user);      // WARNING: do not use C varargs
    // This check flags ALL printf, pushing you to std::format

    // Caught by -Wformat:
    // printf("%d\n", "not an int");    // WARNING: format %d expects int
    // printf("%s\n");                  // WARNING: missing argument for %s

    // The BEST fix: replace printf with std::format/std::print entirely
    // auto result = std::format("Name: {}", user);
    // std::print("Message: {}\n", msg);
}

int main() {
    example_audit();
}

// .clang-tidy configuration
// Create .clang-tidy in project root:
//
// Checks: '-*,
//   cert-err33-c,
//   cert-err34-c,
//   cppcoreguidelines-pro-type-vararg,
//   bugprone-format-security'
// WarningsAsErrors: 'cppcoreguidelines-pro-type-vararg'
//
// CI/CD integration
// Add to your build pipeline:
// clang-tidy --warnings-as-errors='*' -p build/ src/*.cpp
```

A three-layer audit approach works best: (1) Compiler warnings (`-Wformat-security`) catch the most dangerous cases (non-literal format strings) at zero cost. (2) clang-tidy's `cppcoreguidelines-pro-type-vararg` flags ALL C-style variadic function calls, enforcing migration to type-safe alternatives. (3) Grep-based search finds patterns that static analysis might miss. The ultimate fix is replacing all `printf`-family calls with `std::format`/`std::print`.

---

## Notes

- **`-Wformat-security`** should be enabled in every C++ project - it's the single most effective flag against format string bugs.
- **`%n` is disabled** by default on many modern platforms (glibc `_FORTIFY_SOURCE`, Windows MSVC) but don't rely on this as your only defense.
- **`std::format`** format strings are compile-time constants - the key property that eliminates the entire vulnerability class.
- **Migration path:** Replace `printf` -> `std::print`, `sprintf` -> `std::format`, `snprintf` -> `std::format` with size check. Use clang-tidy's `modernize-use-std-print` check for automated migration.
- **CWE-134 (Use of Externally-Controlled Format String)** remains in the CWE Top 25.
- Compile with `-std=c++20 -Wall -Wextra -Wformat-security`.
