# Nest exceptions with std::throw_with_nested for diagnostic chains

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/error/throw_with_nested>  

---

## Topic Overview

`std::throw_with_nested` (C++11) allows you to **chain exceptions** - catching a low-level exception and re-throwing it wrapped inside a higher-level exception. This creates a diagnostic chain showing the full error path from the root cause up to the user-facing layer.

### The Problem: Lost Context

Without exception nesting, you lose the original cause the moment you throw a higher-level error. With nesting, the original exception is preserved inside the new one and can be recovered at any point by traversing the chain.

```cpp
Without nesting:
  catch (db_error& e) {
      throw app_error("save failed");  // <- original db_error is LOST
  }

With nesting:
  catch (db_error& e) {
      std::throw_with_nested(app_error("save failed"));
      // <- app_error contains a nested db_error inside
      // Full chain: app_error -> db_error (original cause preserved)
  }
```

### How It Works Internally

The reason this works is that `throw_with_nested` creates an anonymous type that multiply-inherits from both your exception class and `std::nested_exception`. The `nested_exception` base stores a copy of the current exception via `std::current_exception()`. That's why it must be called inside a `catch` block - there must be a "current exception" to capture.

```cpp
std::throw_with_nested(outer_exception) creates:

┌──────────────────────────────────────────────┐
│  class _Nested : public outer_exception,     │
│                  public std::nested_exception │
│  {                                           │
│      // nested_exception stores              │
│      // std::current_exception() capture     │
│      exception_ptr nested_;                  │
│  };                                          │
└──────────────────────────────────────────────┘

The thrown object inherits from BOTH:

  - Your outer exception (e.g., app_error)
  - std::nested_exception (holds the inner exception_ptr)
```

### Core API

Here's the complete pattern: throwing with nesting, and the recursive traversal that unpacks the chain.

```cpp
#include <exception>
#include <stdexcept>

// Throw a new exception that nests the current exception
// MUST be called inside a catch block (uses std::current_exception())
try {
    low_level_op();
} catch (...) {
    std::throw_with_nested(std::runtime_error("high-level context"));
}

// Check and unwind nested exceptions
void print_chain(const std::exception& e, int depth = 0) {
    std::cerr << std::string(depth * 2, ' ') << e.what() << "\n";
    try {
        std::rethrow_if_nested(e);  // rethrows nested exception if present
    } catch (const std::exception& nested) {
        print_chain(nested, depth + 1);  // recurse
    } catch (...) {
        std::cerr << std::string((depth + 1) * 2, ' ') << "(unknown nested exception)\n";
    }
}
```

`std::rethrow_if_nested` does a `dynamic_cast` to check whether the exception carries a `nested_exception` base. If it does, it re-throws the nested exception. If not, it returns without doing anything. That's what makes the recursive traversal safe - it terminates naturally when it reaches a plain exception with no nested component.

### Important Notes

- `std::throw_with_nested` **must** be called inside a `catch` block - it captures `std::current_exception()`.
- `std::rethrow_if_nested` does nothing if the exception does not derive from `std::nested_exception`.
- Nesting adds a small overhead (extra inheritance + `exception_ptr` copy), but it's negligible on error paths.
- Works with any exception type that can be derived from (must be non-final class).

---

## Self-Assessment

### Q1: Wrap a low-level I/O exception in a higher-level exception using `throw_with_nested`

**Solution - Layered Exception Wrapping:**

This example builds a three-level chain: I/O at the bottom, config parsing in the middle, application startup at the top. Each layer catches whatever went wrong below it and wraps it in its own exception type, adding context without discarding the original cause.

```cpp
#include <iostream>
#include <fstream>
#include <stdexcept>
#include <exception>
#include <string>

// ---- Layer 1: Low-level I/O ----
class IoError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

std::string read_raw(const std::string& path) {
    std::ifstream f(path);
    if (!f.is_open())
        throw IoError("cannot open '" + path + "'");
    std::string content((std::istreambuf_iterator<char>(f)),
                         std::istreambuf_iterator<char>());
    if (f.bad())
        throw IoError("read error on '" + path + "'");
    return content;
}

// ---- Layer 2: Config parsing ----
class ConfigError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

int load_port(const std::string& config_path) {
    try {
        auto content = read_raw(config_path);
        // Simulate parse failure
        throw std::invalid_argument("expected integer for 'port'");
    } catch (...) {
        // Wrap whatever went wrong in a ConfigError
        std::throw_with_nested(ConfigError("failed to load config '" + config_path + "'"));
    }
}

// ---- Layer 3: Application startup ----
class AppError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

void start_server() {
    try {
        int port = load_port("server.conf");
    } catch (...) {
        std::throw_with_nested(AppError("server startup failed"));
    }
}

// ---- Recursive unwinder ----
void print_exception_chain(const std::exception& e, int depth = 0) {
    std::cerr << std::string(depth * 2, ' ') << "-> " << e.what() << "\n";
    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        print_exception_chain(nested, depth + 1);
    } catch (...) {
        std::cerr << std::string((depth + 1) * 2, ' ') << "-> (unknown exception)\n";
    }
}

int main() {
    try {
        start_server();
    } catch (const std::exception& e) {
        std::cerr << "FATAL ERROR - exception chain:\n";
        print_exception_chain(e);
    }
}
// Expected output:
//   FATAL ERROR - exception chain:
//   -> server startup failed
//     -> failed to load config 'server.conf'
//       -> cannot open 'server.conf'
```

**The chain preserves every level of context:**

- `AppError` - what the user sees
- `ConfigError` - what subsystem failed
- `IoError` - the root cause

A developer reading the log immediately knows to look at the file system rather than hunting through the application logic.

### Q2: Print the full exception chain using a recursive `rethrow_if_nested` traversal

**Solution - Generic Exception Chain Printer:**

This takes the traversal idea further by extracting it into reusable utilities: one that formats the whole chain as a string, one that measures the depth, and one that extracts just the root cause. These are the functions you'd put in a logging utility class.

```cpp
#include <iostream>
#include <exception>
#include <stdexcept>
#include <string>
#include <sstream>

// Utility: format the entire chain into a string
std::string format_chain(const std::exception& e, int depth = 0) {
    std::ostringstream oss;
    oss << std::string(depth * 2, ' ');

    if (depth == 0)
        oss << "Error: ";
    else
        oss << "Caused by: ";

    oss << e.what() << "\n";

    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        oss << format_chain(nested, depth + 1);
    } catch (...) {
        oss << std::string((depth + 1) * 2, ' ') << "Caused by: (non-std::exception)\n";
    }

    return oss.str();
}

// Utility: get chain depth
int chain_depth(const std::exception& e) {
    int depth = 1;
    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        depth += chain_depth(nested);
    } catch (...) {
        depth += 1;
    }
    return depth;
}

// Utility: get root cause
std::string root_cause(const std::exception& e) {
    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        return root_cause(nested);
    } catch (...) {
        return "(unknown root cause)";
    }
    return e.what();  // no nesting — this IS the root cause
}

// Demo: build a 4-level chain
void level1() { throw std::runtime_error("disk I/O failure"); }

void level2() {
    try { level1(); }
    catch (...) { std::throw_with_nested(std::runtime_error("database write failed")); }
}

void level3() {
    try { level2(); }
    catch (...) { std::throw_with_nested(std::runtime_error("order save failed")); }
}

void level4() {
    try { level3(); }
    catch (...) { std::throw_with_nested(std::runtime_error("checkout failed")); }
}

int main() {
    try {
        level4();
    } catch (const std::exception& e) {
        std::cout << format_chain(e);
        std::cout << "Chain depth: " << chain_depth(e) << "\n";
        std::cout << "Root cause: " << root_cause(e) << "\n";
    }
}
// Expected output:
//   Error: checkout failed
//     Caused by: order save failed
//       Caused by: database write failed
//         Caused by: disk I/O failure
//   Chain depth: 4
//   Root cause: disk I/O failure
```

**How `rethrow_if_nested` Works:**

1. Checks if the exception object derives from `std::nested_exception` (via `dynamic_cast`).
2. If yes, calls `nested_exception::rethrow_nested()` which re-throws the stored `exception_ptr`.
3. If no, does nothing (returns without throwing).
4. The recursive pattern: `catch` the re-thrown nested exception, process it, recurse.

### Q3: Explain when exception nesting provides more diagnostic value than a single message

**When Nesting Is Superior to a Single Message:**

| Scenario | Single Message | Nested Chain |
| --- | --- | --- |
| **Multi-layer architecture** | "save failed" - which layer? why? | "save failed -> db error -> connection refused" |
| **Library wrapping** | "config error" - was it parse? I/O? permission? | "config error -> parse error -> invalid JSON at line 42" |
| **Async/cross-thread** | "task failed" | "task failed -> worker exception -> out of memory" |
| **Third-party code** | Must stringify and lose type information | Preserves original exception type and `what()` |

**When Nesting IS Appropriate:**

1. **Translating between abstraction layers** - each layer adds its own context without losing the original.
2. **Library boundaries** - wrap internal exceptions in your public exception type.
3. **Logging/diagnostics** - the full chain helps developers pinpoint the root cause.
4. **When you catch and re-throw** - always use `throw_with_nested` instead of plain `throw new_exception`.

**When Nesting Is NOT Needed:**

1. **Single-layer error** - no need to wrap if there's no additional context to add.
2. **Performance-critical paths** - nesting adds allocation and `exception_ptr` overhead.
3. **Error codes / `std::expected`** - these don't use exceptions at all.
4. **Simple re-throw** - if you just want to propagate unchanged, use `throw;` (not `throw e;`).

```cpp
// BAD: wraps but adds no value
catch (const IoError& e) {
    std::throw_with_nested(IoError(e.what()));  // same type, same message — pointless
}

// GOOD: adds higher-level context
catch (const IoError& e) {
    std::throw_with_nested(ConfigError("failed to load 'app.conf'"));
    // Now the chain tells you: config loading failed BECAUSE of an I/O error
}

// GOOD: just re-throw if no context to add
catch (...) {
    cleanup();
    throw;  // propagate unchanged — no nesting needed
}
```

The rule of thumb: wrap when you can add context that the level above genuinely needs. Don't wrap just for the sake of wrapping.

---

## Notes

- `std::throw_with_nested` works with any copyable exception type that can serve as a base class.
- If the exception type is `final`, `throw_with_nested` cannot derive from it - it falls back to throwing the exception without nesting.
- `std::nested_exception` stores an `exception_ptr` - this extends the inner exception's lifetime.
- **Alternative in C++23:** `std::stacktrace` can provide automatic stack traces, reducing the need for manual nesting in some cases.
- **Thread safety:** `exception_ptr` is reference-counted and safe to copy across threads.
- **Don't over-nest:** 2-3 levels of nesting are informative; 10+ levels are noise. Nest at abstraction boundaries, not at every function.
