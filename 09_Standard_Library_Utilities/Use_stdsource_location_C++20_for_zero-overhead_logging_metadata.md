# Use std::source_location (C++20) for zero-overhead logging metadata

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/source_location>  

---

## Topic Overview

`std::source_location` (header `<source_location>`, C++20) captures call-site information - file name, line number, column, and function name - as a first-class value at compile time. It replaces the `__FILE__`, `__LINE__`, `__func__` macros with a type-safe, non-macro mechanism. The overhead is literally zero: `current()` is `consteval`, meaning it resolves entirely at compile time, before your program even runs.

### Key Methods

| Method | Return Type | Description |
| --- | --- | --- |
| `file_name()` | `const char*` | Source file path (equivalent to `__FILE__`) |
| `line()` | `uint_least32_t` | Line number (equivalent to `__LINE__`) |
| `column()` | `uint_least32_t` | Column number (0 if unsupported) |
| `function_name()` | `const char*` | Function name (equivalent to `__func__`, but may be decorated) |
| `current()` | `source_location` | **Static method** - captures location at the point of call |

### The Key Trick: Default Parameter

This is the trick that makes the whole thing work, and it's worth slowing down on. `current()` captures the location where it is *evaluated*. If you call it inside a function body, it captures that function's location - useless for a logger. But if you put it as a **default parameter**, the default is evaluated at the *call site*, not inside the function. That's what gives you the caller's line number.

```cpp
// source_location::current() captures the call site when used as
// a DEFAULT PARAMETER — not the function definition site.

void log(const std::string& msg,
         std::source_location loc = std::source_location::current())
{
    // loc contains the CALLER's file/line/function,
    // NOT this function's location
}

// When called:
log("something happened");  // loc = this line, this file, this function
```

### Core Example

Here's a minimal logger built on this pattern. Notice that `process_data` calls `log` twice, and each call records a different line number from inside `process_data`:

```cpp
#include <source_location>
#include <iostream>
#include <string>

void log(const std::string& message,
         std::source_location loc = std::source_location::current())
{
    std::cout << loc.file_name() << ":" << loc.line()
              << " [" << loc.function_name() << "] "
              << message << "\n";
}

void process_data() {
    log("starting processing");  // logs this line in process_data
    // ... work ...
    log("processing complete");  // logs this line in process_data
}

int main() {
    log("program started");
    process_data();
    log("program ended");
}
// Output:
// main.cpp:16 [int main()] program started
// main.cpp:11 [void process_data()] starting processing
// main.cpp:13 [void process_data()] processing complete
// main.cpp:18 [int main()] program ended
```

Each log line tells you exactly which function called it and which line. No macros, no manual file/line passing.

### Log Levels with source_location

When you add log-level wrappers, the key is to forward the `source_location` through every wrapper layer as a default parameter. If any wrapper layer captures its own location instead, the whole chain breaks and you'll see the wrapper's file/line instead of the caller's.

```cpp
#include <source_location>
#include <iostream>
#include <string_view>

enum class LogLevel { Debug, Info, Warning, Error };

void log_impl(LogLevel level, std::string_view msg,
              std::source_location loc = std::source_location::current())
{
    const char* level_str[] = {"DEBUG", "INFO", "WARN", "ERROR"};
    std::cout << "[" << level_str[static_cast<int>(level)] << "] "
              << loc.file_name() << ":" << loc.line()
              << " (" << loc.function_name() << ") "
              << msg << "\n";
}

// Convenience wrappers — source_location propagated through default param
void log_debug(std::string_view msg,
               std::source_location loc = std::source_location::current()) {
    log_impl(LogLevel::Debug, msg, loc);
}
void log_error(std::string_view msg,
               std::source_location loc = std::source_location::current()) {
    log_impl(LogLevel::Error, msg, loc);
}

int main() {
    log_debug("initializing");
    log_error("disk full");
}
// [DEBUG] main.cpp:27 (int main()) initializing
// [ERROR] main.cpp:28 (int main()) disk full
```

---

## Self-Assessment

### Q1: Write a logging macro-free LOG function that records file, line, and function using source_location

**Answer:**

```cpp
#include <source_location>
#include <iostream>
#include <string_view>
#include <chrono>
#include <iomanip>

class Logger {
public:
    static void info(std::string_view msg,
                     std::source_location loc = std::source_location::current()) {
        print("INFO", msg, loc);
    }

    static void warn(std::string_view msg,
                     std::source_location loc = std::source_location::current()) {
        print("WARN", msg, loc);
    }

    static void error(std::string_view msg,
                      std::source_location loc = std::source_location::current()) {
        print("ERROR", msg, loc);
    }

private:
    static void print(const char* level, std::string_view msg,
                      const std::source_location& loc) {
        auto now = std::chrono::system_clock::now();
        auto time = std::chrono::system_clock::to_time_t(now);

        std::cout << std::put_time(std::localtime(&time), "%H:%M:%S")
                  << " [" << level << "] "
                  << loc.file_name() << ":" << loc.line()
                  << " " << loc.function_name()
                  << " - " << msg << "\n";
    }
};

void connect_database() {
    Logger::info("connecting to database");
    // simulate failure
    Logger::error("connection refused");
}

int main() {
    Logger::info("application starting");
    connect_database();
    Logger::warn("retrying in 5 seconds");
}
// Output:
// 14:30:00 [INFO] main.cpp:40 int main() - application starting
// 14:30:00 [INFO] main.cpp:35 void connect_database() - connecting to database
// 14:30:00 [ERROR] main.cpp:37 void connect_database() - connection refused
// 14:30:00 [WARN] main.cpp:42 int main() - retrying in 5 seconds
```

Each `Logger::info/warn/error` method takes `std::source_location` as a default parameter. When called, `current()` captures the caller's file, line, and function. The private `print` method receives the already-captured location - it does not re-capture. No macros needed, fully type-safe, zero runtime overhead.

### Q2: Show how source_location is captured at the call site, not inside the function

**Answer:**

This example puts the BAD and GOOD patterns side by side so the difference is clear. The bad version calls `current()` inside the function body; the good version uses it as a default parameter.

```cpp
#include <source_location>
#include <iostream>

// This function always reports its OWN location (BAD pattern)
void bad_log(const char* msg) {
    auto loc = std::source_location::current(); // captures HERE, not call site
    std::cout << "BAD: " << loc.file_name() << ":" << loc.line()
              << " " << loc.function_name() << " - " << msg << "\n";
}

// This function reports the CALLER's location (GOOD pattern)
void good_log(const char* msg,
              std::source_location loc = std::source_location::current()) {
    // loc was already captured at the call site (via default argument)
    std::cout << "GOOD: " << loc.file_name() << ":" << loc.line()
              << " " << loc.function_name() << " - " << msg << "\n";
}

void caller_function() {                   // line 19 (example)
    bad_log("hello from caller");          // line 20
    good_log("hello from caller");         // line 21
}

int main() {
    caller_function();
}
// Output:
// BAD: main.cpp:7 void bad_log(const char*) — hello from caller
//   ^ Reports bad_log's OWN line 7 — useless for debugging!
// GOOD: main.cpp:21 void caller_function() — hello from caller
//   ^ Reports the actual call site — line 21 in caller_function
```

`source_location::current()` captures the location where it's *evaluated*. Inside a function body, that's the function itself. As a default parameter, it's evaluated at the *call site*. This is why the default-parameter pattern is essential - it's the only way to capture caller information without macros.

### Q3: Compare source_location with `__FILE__`/`__LINE__` macros and explain portability benefits

| Aspect | `__FILE__` / `__LINE__` macros | `std::source_location` |
| --- | --- | --- |
| Mechanism | Preprocessor text substitution | Compiler built-in, type-safe value |
| Type | `const char*` / `int` (no structure) | `std::source_location` object |
| Function name | `__func__` (C99/C++11) | `function_name()` - often more decorated |
| Column | Not available | `column()` method |
| Passable as value | No (macros expand at use site) | Yes - first-class object, copyable |
| Default parameter trick | Impossible (macro expands in callee) | Natural - that's the intended pattern |
| Namespacing | Global macro pollution | `std::source_location` - scoped |
| Usable in templates | Awkward (must wrap in macro) | Direct - just a default parameter |
| consteval | No | `current()` is consteval |

```cpp
#include <source_location>
#include <iostream>

// OLD way: requires macros to capture call site
#define LOG_MACRO(msg) \
    log_with_macros(__FILE__, __LINE__, __func__, msg)

void log_with_macros(const char* file, int line, const char* func,
                     const char* msg) {
    std::cout << file << ":" << line << " " << func << " - " << msg << "\n";
}

// NEW way: no macros needed
void log_modern(const char* msg,
                std::source_location loc = std::source_location::current()) {
    std::cout << loc.file_name() << ":" << loc.line()
              << " " << loc.function_name() << " - " << msg << "\n";
}

int main() {
    LOG_MACRO("with macros");    // works, but requires macro
    log_modern("without macros"); // cleaner, no macro

    // KEY ADVANTAGE: source_location can be stored and passed around
    auto loc = std::source_location::current();
    // Can log 'loc' later, pass to error handler, store in exception, etc.
}
```

**Key portability benefits:**

1. **No macro hygiene issues** - no risk of name collisions, no parenthesization bugs
2. **Works in templates** - macros don't compose with templates; `source_location` does
3. **Storable** - can be stored in exceptions, error objects, log entries for deferred processing
4. **Consistent across compilers** - `__func__` format varies; `function_name()` is standardized (though decoration may differ)
5. **Column information** - unavailable via macros

---

## Notes

- **Zero overhead:** `source_location::current()` is `consteval` - it's resolved entirely at compile time. No runtime cost.
- **In exception types:** Store `source_location` in custom exception classes to track where the exception was thrown:

  ```cpp
  class my_error : public std::runtime_error {
      std::source_location loc_;
  public:
      my_error(const char* msg, std::source_location loc = std::source_location::current())
          : std::runtime_error(msg), loc_(loc) {}
  };
  ```

- **Gotcha with wrappers:** If you wrap a logging function, each wrapper must forward `source_location` as a default parameter, or the location will be the wrapper's, not the original caller's.
- **Compiler support:** GCC 11+, Clang 15+, MSVC 19.29+.
- **`function_name()` varies:** GCC returns the full signature; MSVC returns a decorated name. Use for debugging, not programmatic comparison.
- Compile with `-std=c++20 -Wall -Wextra`.
