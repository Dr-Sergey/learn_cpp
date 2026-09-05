# Understand the logic_error vs runtime_error hierarchy

**Category:** Error Handling  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/error/exception>  

---

## Topic Overview

The C++ standard library exception hierarchy splits errors into two fundamental categories: **`std::logic_error`** (programming bugs that could be prevented) and **`std::runtime_error`** (environmental failures that cannot be prevented by code alone). Getting this split right matters because it affects how you respond: bugs need to be fixed, environmental failures need to be handled gracefully.

### The Standard Exception Hierarchy

The full hierarchy is wider than most people realize. Here is the complete tree from `std::exception` downward:

```cpp
std::exception
├── std::logic_error
│   ├── std::invalid_argument     // bad parameter value
│   ├── std::domain_error         // math domain violation
│   ├── std::length_error         // exceeds max_size()
│   ├── std::out_of_range         // index/key out of range
│   └── std::future_error         // future/promise misuse (logic)
│
├── std::runtime_error
│   ├── std::overflow_error       // arithmetic overflow
│   ├── std::underflow_error      // arithmetic underflow
│   ├── std::range_error          // result out of representable range
│   ├── std::system_error         // OS/system call failure
│   │   └── std::filesystem::filesystem_error
│   └── std::format_error (C++20) // std::format/std::vformat
│
├── std::bad_alloc                // operator new failure
│   └── std::bad_array_new_length
├── std::bad_cast                 // dynamic_cast failure
├── std::bad_typeid               // typeid on null pointer
├── std::bad_optional_access      // .value() on empty optional
├── std::bad_expected_access (C++23)
├── std::bad_variant_access
├── std::bad_function_call
└── std::bad_weak_ptr
```

Notice that `std::bad_alloc` and the `bad_*` family are direct children of `std::exception` - they do not fit neatly into the logic/runtime split.

### The Key Distinction

If the table feels like a lot, it boils down to one question: **could a programmer have prevented this by writing correct code?**

| Aspect | `std::logic_error` | `std::runtime_error` |
| --- | --- | --- |
| **Nature** | Bug in the program's logic | External/environmental failure |
| **Preventable?** | Yes - fix the code | No - can't control the environment |
| **Example** | Passing -1 to `sqrt()` | Disk full, network down |
| **Correct response** | Fix the caller's code | Catch and handle gracefully |
| **In production** | Should never be thrown (indicates a bug) | Expected to occur occasionally |
| **Alternative** | `assert()` / contracts | Exception or `std::expected` |

### Which Standard Functions Throw Which

Here are some concrete examples from the standard library that illustrate the split:

| Function | Exception | Category |
| --- | --- | --- |
| `std::stoi("abc")` | `std::invalid_argument` | logic_error |
| `std::stoi("99999999999")` | `std::out_of_range` | logic_error |
| `std::vector::at(999)` | `std::out_of_range` | logic_error |
| `std::string::substr(999)` | `std::out_of_range` | logic_error |
| `new int[1'000'000'000]` | `std::bad_alloc` | (direct from exception) |
| `std::filesystem::copy(...)` | `std::filesystem_error` | runtime_error |
| `std::make_error_code(...)` | `std::system_error` | runtime_error |

All the "bad index" and "bad conversion" errors are `logic_error` because a well-written caller would have validated the input first. File system and OS errors are `runtime_error` because they can happen even with perfect code.

### Important Notes

- `std::exception::what()` returns a `const char*` - always override it in custom exceptions.
- Catch by `const` reference: `catch (const std::exception& e)`.
- The distinction is **semantic** - the compiler doesn't enforce it. You must choose correctly.

---

## Self-Assessment

### Q1: Explain the intent: `logic_error` is a bug, `runtime_error` is environment failure

The key thing to internalize is that the *correct* response to each kind of error is different. For `logic_error`, the right answer is to fix the calling code - not to write better error handling. For `runtime_error`, the right answer is to catch it and deal with it gracefully at runtime.

**Detailed Explanation:**

```cpp
#include <iostream>
#include <stdexcept>
#include <vector>
#include <fstream>
#include <string>
#include <system_error>

// ========== logic_error examples - these are PROGRAMMER BUGS ==========

void demonstrate_logic_errors() {
    // 1. invalid_argument: caller passed bad data
    try {
        int val = std::stoi("not_a_number");
    } catch (const std::invalid_argument& e) {
        std::cout << "logic_error: " << e.what() << "\n";
        // FIX: validate input before calling stoi()
    }

    // 2. out_of_range: caller used wrong index
    try {
        std::vector<int> v = {1, 2, 3};
        int x = v.at(10);  // index 10 doesn't exist
    } catch (const std::out_of_range& e) {
        std::cout << "logic_error: " << e.what() << "\n";
        // FIX: check index < v.size() before access
    }

    // 3. domain_error: mathematical precondition violated
    // (not thrown by stdlib, but available for user code)
    try {
        throw std::domain_error("sqrt of negative number");
    } catch (const std::domain_error& e) {
        std::cout << "logic_error: " << e.what() << "\n";
        // FIX: validate input >= 0 before calling sqrt
    }
}

// ========== runtime_error examples - these are ENVIRONMENT FAILURES ==========

void demonstrate_runtime_errors() {
    // 1. filesystem_error: file doesn't exist (not a bug - just reality)
    try {
        std::ifstream f("nonexistent_file.txt");
        if (!f) throw std::runtime_error("file not found");
    } catch (const std::runtime_error& e) {
        std::cout << "runtime_error: " << e.what() << "\n";
        // HANDLE: show error message, try alternative path, etc.
    }

    // 2. system_error: OS resource unavailable
    try {
        throw std::system_error(std::make_error_code(std::errc::no_such_file_or_directory));
    } catch (const std::system_error& e) {
        std::cout << "runtime_error: " << e.what() << "\n";
        // HANDLE: retry, fallback, notify user
    }
}

int main() {
    demonstrate_logic_errors();
    demonstrate_runtime_errors();
}
// Expected output:
//   logic_error: stoi
//   logic_error: invalid vector subscript (or similar)
//   logic_error: sqrt of negative number
//   runtime_error: file not found
//   runtime_error: No such file or directory
```

**The Mental Model:**

- **`logic_error`:** "The programmer made a mistake. Fix the code."
- **`runtime_error`:** "The world is imperfect. Handle it gracefully."

---

### Q2: Show that catching `logic_error` in production code often indicates a design flaw

The reason this is a code smell is subtle: if you catch an `out_of_range` and continue, you have silently accepted a bug. The code has a wrong loop bound or an unchecked index, and you are just papering over it. The right fix is to eliminate the out-of-bounds access entirely, not to catch the exception it throws.

**Solution - Why Catching `logic_error` Is a Code Smell:**

```cpp
#include <iostream>
#include <stdexcept>
#include <vector>
#include <string>
#include <cassert>

// BAD: Catching logic_error to "handle" a bug
void bad_design(const std::vector<int>& data) {
    try {
        // Using .at() and catching out_of_range means the programmer
        // KNOWS the index might be wrong but "handles" it with a catch.
        // This hides the bug instead of fixing it.
        for (size_t i = 0; i <= data.size(); ++i) {  // off-by-one bug!
            process(data.at(i));
        }
    } catch (const std::out_of_range&) {
        // "Handling" the bug - this is a code smell!
        // The real fix is: i < data.size()
        std::cout << "Caught out_of_range - 'handled' it\n";
    }
}

// GOOD: Fix the logic, don't catch the bug
void good_design(const std::vector<int>& data) {
    for (size_t i = 0; i < data.size(); ++i) {  // correct loop bound
        process(data[i]);  // operator[] - no exception, just correct code
    }
}

// GOOD: Use assertions for internal contracts
int safe_element(const std::vector<int>& v, size_t idx) {
    assert(idx < v.size() && "Bug: index out of bounds");
    return v[idx];
}

// GOOD: Validate USER INPUT at boundaries, then use contracts internally
int parse_and_access(const std::string& user_input, const std::vector<int>& data) {
    // Validate external input -> return error (not assert)
    int idx;
    try {
        idx = std::stoi(user_input);
    } catch (const std::invalid_argument&) {
        std::cerr << "Invalid index: " << user_input << "\n";
        return -1;
    }

    // After validation, use contracts for internal logic
    if (idx < 0 || static_cast<size_t>(idx) >= data.size()) {
        std::cerr << "Index out of range: " << idx << "\n";
        return -1;
    }

    return data[idx];  // known to be safe - no .at(), no try/catch
}

int main() {
    std::vector<int> v = {10, 20, 30};
    std::cout << parse_and_access("1", v) << "\n";   // 20
    std::cout << parse_and_access("abc", v) << "\n";  // -1
    std::cout << parse_and_access("99", v) << "\n";   // -1
}
```

The `parse_and_access` function shows the right pattern: use exceptions to validate user-supplied input (because that is a genuine runtime uncertainty), but once you have validated the input, write code that is provably correct and does not need to throw at all.

**The Rule:**

| Exception Type | Caught in Production? | Why? |
| --- | --- | --- |
| `std::runtime_error` | Yes - handle gracefully | Environment failures are expected |
| `std::logic_error` | Rarely - indicates a bug | Fix the code instead of catching |
| `std::bad_alloc` | Sometimes | Resource exhaustion - may be recoverable |
| `std::exception` (catch-all) | At top level only | Log and exit gracefully |

---

### Q3: Write a custom exception class inheriting `runtime_error` with structured context fields

When you derive from `std::runtime_error`, you get string management for free through the base class constructor. The `what()` message is handled automatically. Your job is to add structured fields - status codes, URLs, timestamps, whatever makes the error actionable:

**Solution - Rich Exception with Context:**

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <chrono>
#include <sstream>

// Custom exception with structured fields for diagnostics
class HttpError : public std::runtime_error {
    int status_code_;
    std::string url_;
    std::string method_;
    std::chrono::milliseconds latency_;

    static std::string build_message(int code, const std::string& method,
                                      const std::string& url,
                                      std::chrono::milliseconds latency) {
        std::ostringstream oss;
        oss << "HTTP " << code << " " << method << " " << url
            << " (" << latency.count() << "ms)";
        return oss.str();
    }

public:
    HttpError(int status_code, const std::string& method,
              const std::string& url, std::chrono::milliseconds latency)
        : std::runtime_error(build_message(status_code, method, url, latency))
        , status_code_(status_code)
        , url_(url)
        , method_(method)
        , latency_(latency) {}

    int status_code() const noexcept { return status_code_; }
    const std::string& url() const noexcept { return url_; }
    const std::string& method() const noexcept { return method_; }
    std::chrono::milliseconds latency() const noexcept { return latency_; }

    bool is_client_error() const noexcept { return status_code_ >= 400 && status_code_ < 500; }
    bool is_server_error() const noexcept { return status_code_ >= 500; }
    bool is_retryable() const noexcept {
        return status_code_ == 429 || status_code_ == 503 || status_code_ == 502;
    }
};

// More specific derived exception
class TimeoutError : public HttpError {
public:
    TimeoutError(const std::string& method, const std::string& url,
                 std::chrono::milliseconds latency)
        : HttpError(408, method, url, latency) {}
};

// Simulated HTTP client
void http_get(const std::string& url) {
    using namespace std::chrono_literals;
    // Simulate a timeout
    throw TimeoutError("GET", url, 5000ms);
}

int main() {
    try {
        http_get("https://api.example.com/data");
    }
    // Catch most derived first!
    catch (const TimeoutError& e) {
        std::cout << "Timeout: " << e.what() << "\n";
        std::cout << "  Retryable: " << (e.is_retryable() ? "yes" : "no") << "\n";
    }
    catch (const HttpError& e) {
        std::cout << "HTTP error: " << e.what() << "\n";
        std::cout << "  Status: " << e.status_code() << "\n";
        std::cout << "  URL: " << e.url() << "\n";
        if (e.is_retryable())
            std::cout << "  -> Will retry\n";
    }
    catch (const std::runtime_error& e) {
        std::cout << "Runtime error: " << e.what() << "\n";
    }
    catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << "\n";
    }
}
// Expected output:
//   Timeout: HTTP 408 GET https://api.example.com/data (5000ms)
//     Retryable: yes
```

The `build_message` static helper constructs the `what()` string and passes it to the `runtime_error` base constructor. This is the standard pattern - you get all the string lifetime management for free.

The hierarchy you should build toward for a real application looks like this:

```cpp
// std::exception
//   └── std::runtime_error          (environment failures)
//         └── AppException          your root
//               ├── NetworkError
//               │     ├── HttpError structured fields
//               │     ├── DnsError
//               │     └── TimeoutError
//               ├── DatabaseError
//               └── ConfigError
```

**Design Rules for Custom Exception Hierarchies:**

1. **Always derive from `std::exception`** (or a standard derived class).
2. **Override `what()` via the `runtime_error(string)` constructor** - don't override `what()` directly.
3. **Add structured fields** (status codes, URLs, etc.) accessible via `const` member functions.
4. **Catch most-derived first** - C++ matches the first `catch` clause in order.
5. **Catch by `const` reference** - avoids slicing.

---

## Notes

- **Slicing danger:** `catch (std::exception e)` (by value) slices off derived data. Always catch by `const` reference.
- **`std::exception::what()` returns `const char*`** - the string must remain valid for the exception's lifetime. `std::runtime_error` manages this via an internal `std::string`.
- **Don't throw `std::logic_error` in library code** for input validation - use `std::expected` or `std::invalid_argument` only when the API documents that the caller must validate.
- **`std::bad_alloc` doesn't derive from `logic_error` or `runtime_error`** - it derives directly from `std::exception`. This is because memory exhaustion is neither a logic bug nor purely an environment issue.
- **Keep exception classes lightweight** - they're copied during stack unwinding. Avoid large members or heap allocations in the constructor.
