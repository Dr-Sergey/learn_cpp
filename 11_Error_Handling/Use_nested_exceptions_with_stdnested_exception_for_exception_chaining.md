# Use nested exceptions with std::nested_exception for exception chaining

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/error/nested_exception>  

---

## Topic Overview

`std::nested_exception` (C++11) is the **base class** that enables exception chaining. When you catch an exception and want to re-throw it wrapped inside a new, higher-level exception, `std::throw_with_nested` creates an object that inherits from both your exception type and `std::nested_exception`, storing the currently active exception.

> **Note:** This topic complements the `throw_with_nested` file. Here we focus on the `std::nested_exception` class itself, its mechanics, and direct usage patterns.

### `std::nested_exception` Class

Here is the relevant part of the class interface - the two members you will use most are `rethrow_nested()` to get back to the inner exception, and `nested_ptr()` if you want to inspect it without throwing:

```cpp
class nested_exception {
public:
    nested_exception() noexcept;                    // captures current_exception()
    nested_exception(const nested_exception&) = default;
    virtual ~nested_exception() = default;

    [[noreturn]] void rethrow_nested() const;       // rethrows stored exception
    std::exception_ptr nested_ptr() const noexcept; // access stored ptr
};
```

### How the Mechanism Works

The reason this trips people up is that `std::throw_with_nested` does not literally wrap your exception inside another object at runtime - it synthesises a new type on the fly that inherits from both your exception class and `std::nested_exception`. Here is what the compiler-generated chain looks like:

```cpp
Inside a catch block:

  catch (const LowLevelError& e) {
      std::throw_with_nested(HighLevelError("context"));
  }

  What happens:

  1. throw_with_nested creates a NEW type:

     class _Generated : public HighLevelError,
                        public std::nested_exception { ... };

  2. nested_exception() constructor captures std::current_exception()

     -> stores a reference to the LowLevelError

  3. The _Generated object is thrown

  Result:
  +----------------------------------+
  | _Generated object                |
  |   +-- HighLevelError data        |  <- your outer exception
  |   +-- nested_exception           |
  |         +-- exception_ptr -------+--> LowLevelError (preserved)
  +----------------------------------+
```

The outer exception is what the caller catches. The inner (original) exception is reachable via `rethrow_nested()` or `nested_ptr()`.

### Key Functions

| Function | Purpose |
| --- | --- |
| `std::throw_with_nested(e)` | Throw `e` with current exception nested inside |
| `std::rethrow_if_nested(e)` | If `e` has a nested exception, rethrow it; otherwise do nothing |
| `nested_exception::rethrow_nested()` | Rethrow the stored inner exception |
| `nested_exception::nested_ptr()` | Get the stored `exception_ptr` |

### Important Notes

- `std::throw_with_nested` **must** be called inside a `catch` block - it uses `std::current_exception()`.
- If the exception type is `final`, `throw_with_nested` cannot derive from it - it throws without nesting.
- `nested_exception` uses `exception_ptr` internally - reference-counted, thread-safe to copy.
- Available since **C++11** (not C++20 as the template suggested).

---

## Self-Assessment

### Q1: Chain exceptions using `std::throw_with_nested` and unwrap them with `std::rethrow_if_nested`

**Solution - Three-Layer Exception Chain:**

This example builds a realistic multi-layer scenario: a file-read failure at the bottom propagates through a config-parser and then a server startup function. Watch how each layer wraps the exception from below, so the caller gets the full story:

```cpp
#include <iostream>
#include <stdexcept>
#include <exception>
#include <string>
#include <fstream>

// Layer 1: File I/O
std::string read_config_file(const std::string& path) {
    std::ifstream f(path);
    if (!f.is_open())
        throw std::runtime_error("cannot open '" + path + "'");
    return "contents";
}

// Layer 2: Configuration parsing
int parse_port(const std::string& config_path) {
    try {
        auto content = read_config_file(config_path);
        throw std::invalid_argument("missing 'port' key");  // simulate parse error
    } catch (...) {
        std::throw_with_nested(
            std::runtime_error("failed to parse config '" + config_path + "'"));
    }
}

// Layer 3: Server startup
void start_server(const std::string& config) {
    try {
        int port = parse_port(config);
    } catch (...) {
        std::throw_with_nested(
            std::runtime_error("server startup failed"));
    }
}

// Unwinder using rethrow_if_nested
void print_chain(const std::exception& e, int depth = 0) {
    std::cerr << std::string(depth * 2, ' ') << e.what() << "\n";
    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        print_chain(nested, depth + 1);
    } catch (...) {
        std::cerr << std::string((depth + 1) * 2, ' ') << "(unknown)\n";
    }
}

int main() {
    try {
        start_server("app.conf");
    } catch (const std::exception& e) {
        std::cerr << "FATAL:\n";
        print_chain(e);
    }
}
// Expected output:
//   FATAL:
//   server startup failed
//     failed to parse config 'app.conf'
//       cannot open 'app.conf'
```

`print_chain` is a recursive unwinder: each call prints the current message, then calls `rethrow_if_nested` to reach the next layer. When there is nothing more to unwrap, `rethrow_if_nested` returns normally and the recursion ends.

**Step-by-step unwinding:**

1. Catch `std::exception& e` - it's `runtime_error("server startup failed")` with a nested exception.
2. `rethrow_if_nested(e)` - rethrows the nested `runtime_error("failed to parse config")`.
3. Catch again, `rethrow_if_nested` - rethrows `runtime_error("cannot open 'app.conf'")`.
4. Catch again, `rethrow_if_nested` - no nesting - returns normally - recursion ends.

---

### Q2: Write a recursive exception printer that traverses the nested chain

**Solution - Production-Quality Chain Formatter:**

Sometimes you want more than a simple indented dump. This example shows three different presentation formats for the same chain: a tree, an arrow chain, and root-cause extraction. Each is useful in different contexts (log files, error banners, monitoring dashboards):

```cpp
#include <iostream>
#include <exception>
#include <stdexcept>
#include <sstream>
#include <string>
#include <vector>

// Collect all messages into a vector
std::vector<std::string> collect_chain(const std::exception& e) {
    std::vector<std::string> messages;
    messages.push_back(e.what());

    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        auto inner = collect_chain(nested);
        messages.insert(messages.end(), inner.begin(), inner.end());
    } catch (...) {
        messages.push_back("(unknown non-std exception)");
    }

    return messages;
}

// Format as indented tree
std::string format_tree(const std::exception& e, int depth = 0) {
    std::ostringstream oss;
    oss << std::string(depth * 2, ' ');
    if (depth == 0) oss << "Error: ";
    else oss << "Caused by: ";
    oss << e.what() << "\n";

    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        oss << format_tree(nested, depth + 1);
    } catch (...) {
        oss << std::string((depth + 1) * 2, ' ') << "Caused by: (unknown)\n";
    }
    return oss.str();
}

// Format as single-line arrow chain
std::string format_arrow(const std::exception& e) {
    auto msgs = collect_chain(e);
    std::ostringstream oss;
    for (size_t i = 0; i < msgs.size(); ++i) {
        if (i > 0) oss << " -> ";
        oss << msgs[i];
    }
    return oss.str();
}

// Get root cause (deepest nested exception)
std::string root_cause(const std::exception& e) {
    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& nested) {
        return root_cause(nested);
    } catch (...) {
        return "(unknown)";
    }
    return e.what();  // no nesting - this IS the root
}

// Demo
void layer_c() { throw std::runtime_error("disk full"); }
void layer_b() {
    try { layer_c(); }
    catch (...) { std::throw_with_nested(std::runtime_error("write failed")); }
}
void layer_a() {
    try { layer_b(); }
    catch (...) { std::throw_with_nested(std::runtime_error("save document failed")); }
}

int main() {
    try {
        layer_a();
    } catch (const std::exception& e) {
        std::cout << "=== Tree format ===\n";
        std::cout << format_tree(e);

        std::cout << "\n=== Arrow format ===\n";
        std::cout << format_arrow(e) << "\n";

        std::cout << "\n=== Root cause ===\n";
        std::cout << root_cause(e) << "\n";

        std::cout << "\n=== Chain messages ===\n";
        auto msgs = collect_chain(e);
        for (size_t i = 0; i < msgs.size(); ++i)
            std::cout << "[" << i << "] " << msgs[i] << "\n";
    }
}
// Expected output:
//   === Tree format ===
//   Error: save document failed
//     Caused by: write failed
//       Caused by: disk full
//
//   === Arrow format ===
//   save document failed -> write failed -> disk full
//
//   === Root cause ===
//   disk full
//
//   === Chain messages ===
//   [0] save document failed
//   [1] write failed
//   [2] disk full
```

Notice that `root_cause` just recurses until `rethrow_if_nested` returns normally, at which point the current exception has no inner layer and is therefore the root.

---

### Q3: Explain when nested exceptions are more informative than replacing the original exception

**Replacing vs Nesting - Comparison:**

The key intuition here is that when you catch and rethrow a completely new exception, you are discarding everything the original exception knew about itself. With nesting, you preserve it:

```cpp
// BAD: Original exception information is LOST
try {
    database_query("SELECT ...");
} catch (const db::connection_error& e) {
    throw app::SaveError("save failed");
    // Lost: which connection? what error code? timeout?
}

// GOOD: Original exception is PRESERVED
try {
    database_query("SELECT ...");
} catch (...) {
    std::throw_with_nested(app::SaveError("save failed"));
    // Preserved: the full db::connection_error with all its data
}
```

**When Nesting Is More Informative:**

| Scenario | Replacing | Nesting |
| --- | --- | --- |
| **Multi-layer architecture** | "save failed" | "save failed -> db error -> connection refused to 10.0.0.5:5432" |
| **Third-party libraries** | Must stringify upstream error | Preserves upstream exception type + data |
| **Log analysis** | Single message - hard to find root cause | Full chain - root cause obvious |
| **Programmatic handling** | Can only inspect outer type | Can `dynamic_cast` inner exception to get structured data |
| **Test diagnostics** | "test failed" | "test failed -> setup failed -> fixture file missing" |

**When Nesting Is NOT Needed:**

1. **You're re-throwing unchanged:** Use `throw;` - no wrapping needed.
2. **No additional context:** If the inner exception already says everything, don't wrap.
3. **Error codes / `std::expected`:** These don't use exceptions - use error enums or structs instead.
4. **Same abstraction level:** Wrapping `IoError` in another `IoError` adds noise, not value.

**Accessing Nested Data Programmatically:**

One of the more powerful aspects of nesting is that the inner exception is still a full typed object. You can `dynamic_cast` into it after rethrowing and read its structured fields, rather than parsing a string:

```cpp
try {
    operation();
} catch (const AppError& e) {
    // Access outer exception normally
    std::cout << "App error: " << e.what() << "\n";

    // Access inner exception via nested_exception
    try {
        std::rethrow_if_nested(e);
    } catch (const DbError& db_err) {
        // Can access structured fields from the original exception!
        std::cout << "Root DB error code: " << db_err.error_code() << "\n";
        std::cout << "Query: " << db_err.query() << "\n";
    }
}
```

---

## Notes

- `std::nested_exception` is available since **C++11**, not C++20.
- `nested_exception::nested_ptr()` returns `exception_ptr` - can be null if constructed outside a catch block.
- `rethrow_nested()` is `[[noreturn]]` - it always throws. Calling it when `nested_ptr()` is null calls `std::terminate()`.
- Maximum practical nesting depth: 3-5 levels. Deeper chains become noise rather than information.
- **Performance:** Nesting adds one `exception_ptr` copy (reference count increment) per level - negligible on error paths.
- **Thread safety:** `exception_ptr` is reference-counted and safe to copy across threads.
