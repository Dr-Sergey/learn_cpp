# Use std::error_code and std::error_category for system error handling

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/error/error_code>  

---

## Topic Overview

`std::error_code` is a lightweight, non-throwing error type that carries both an **integer error value** and a pointer to an **error category** that gives it meaning. This system allows different libraries and subsystems to define their own error codes that interoperate through a common interface.

### Architecture

The mental model that helps here is: the integer value alone means nothing - a `2` from the OS is completely different from a `2` in your custom HTTP category. The category is what gives the number its identity:

```cpp
+-------------------------------------------------------------+
| std::error_code                                             |
|  +----------+  +----------------------------------+        |
|  | int value |  | const error_category* category   |        |
|  |    = 2    |  |    -> system_category()           |        |
|  +----------+  +----------------------------------+        |
|                                                             |
|  .value()     -> 2                                          |
|  .category()  -> system_category (errno on POSIX, Win32 on Windows) |
|  .message()   -> "No such file or directory"                |
|  operator bool -> true (non-zero = error)                   |
+-------------------------------------------------------------+

Standard categories:
  std::system_category()   -> OS-level errors (errno / GetLastError)
  std::generic_category()  -> portable POSIX-like errors (std::errc)
  Custom categories         -> your library's error codes
```

### `error_code` vs `error_condition`

This distinction trips people up at first. Think of it this way: `error_code` is the raw platform value - the exact thing the OS reported. `error_condition` is the portable abstraction you use when you want to ask "is this a permission error?" in a way that works on every platform:

| Type | Purpose | Comparison |
| --- | --- | --- |
| `std::error_code` | **Platform-specific** error (what actually went wrong) | Exact match |
| `std::error_condition` | **Portable** error category (abstract equivalence) | Cross-platform match |

```cpp
// error_code: platform-specific
std::error_code ec = std::make_error_code(std::errc::no_such_file_or_directory);
// On Linux: value=2 (ENOENT), On Windows: value=2 (ERROR_FILE_NOT_FOUND)

// error_condition: portable check
if (ec == std::errc::no_such_file_or_directory) {
    // Works on ALL platforms - maps through category::equivalent()
}
```

### Core API

Here is the minimal set of operations you will use every day - create a code, check whether it represents an error (`bool` conversion), and compare portably:

```cpp
#include <system_error>
#include <iostream>

std::error_code ec;  // default: value=0, category=system_category -> "no error"

// Create from errc enum (portable)
ec = std::make_error_code(std::errc::permission_denied);
std::cout << ec.value() << ": " << ec.message() << "\n";  // 13: Permission denied

// Check for error
if (ec) { /* has error */ }
if (!ec) { /* success */ }

// Compare portably
if (ec == std::errc::permission_denied) { /* portable check */ }
```

### Important Notes

- `std::error_code` is a lightweight value type (~8-16 bytes) - no heap allocation, no exception overhead.
- The integer value alone is meaningless without its category - `ENOENT (2)` in system_category is different from `2` in your custom category.
- `std::system_error` is an exception that wraps an `error_code`.

---

## Self-Assessment

### Q1: Define a custom error category with a `message()` override and register custom error codes

**Solution - Custom HTTP Error Category:**

There are five steps to wiring up a custom error category. Read through them in order - each step is small, but they all need to be present for `make_error_code` and the `operator==` comparisons to work correctly:

```cpp
#include <iostream>
#include <system_error>
#include <string>

// Step 1: Define error codes as an enum
enum class HttpError {
    ok = 0,
    bad_request = 400,
    unauthorized = 401,
    forbidden = 403,
    not_found = 404,
    internal_server_error = 500,
    service_unavailable = 503
};

// Step 2: Define a custom error category
class HttpErrorCategory : public std::error_category {
public:
    const char* name() const noexcept override {
        return "http";
    }

    std::string message(int ev) const override {
        switch (static_cast<HttpError>(ev)) {
            case HttpError::ok:                     return "OK";
            case HttpError::bad_request:            return "Bad Request";
            case HttpError::unauthorized:           return "Unauthorized";
            case HttpError::forbidden:              return "Forbidden";
            case HttpError::not_found:              return "Not Found";
            case HttpError::internal_server_error:  return "Internal Server Error";
            case HttpError::service_unavailable:    return "Service Unavailable";
            default:                                return "Unknown HTTP error " + std::to_string(ev);
        }
    }

    // Optional: map to portable std::errc conditions
    std::error_condition default_error_condition(int ev) const noexcept override {
        switch (static_cast<HttpError>(ev)) {
            case HttpError::not_found:
                return std::errc::no_such_file_or_directory;
            case HttpError::forbidden:
                return std::errc::permission_denied;
            default:
                return std::error_condition(ev, *this);
        }
    }
};

// Step 3: Singleton accessor
const HttpErrorCategory& http_category() {
    static HttpErrorCategory instance;
    return instance;
}

// Step 4: Enable make_error_code for our enum
namespace std {
    template <>
    struct is_error_code_enum<HttpError> : true_type {};
}

std::error_code make_error_code(HttpError e) {
    return {static_cast<int>(e), http_category()};
}

// Step 5: Use it
std::error_code fetch_resource(const std::string& url) {
    if (url.empty())
        return HttpError::bad_request;
    if (url == "/secret")
        return HttpError::forbidden;
    if (url == "/missing")
        return HttpError::not_found;
    return {};  // success
}

int main() {
    auto ec = fetch_resource("/missing");

    std::cout << "Category: " << ec.category().name() << "\n";
    std::cout << "Value: " << ec.value() << "\n";
    std::cout << "Message: " << ec.message() << "\n";

    // Portable comparison works through default_error_condition:
    if (ec == std::errc::no_such_file_or_directory)
        std::cout << "-> maps to POSIX 'no such file'\n";

    // Check for any error:
    if (ec)
        std::cout << "-> operation failed\n";
}
// Expected output:
//   Category: http
//   Value: 404
//   Message: Not Found
//   -> maps to POSIX 'no such file'
//   -> operation failed
```

The `default_error_condition` override is what makes `ec == std::errc::no_such_file_or_directory` work even though `ec` is in the http category. The comparison machinery calls through the category to ask "are these equivalent?"

---

### Q2: Show how `std::filesystem` functions use `error_code` overloads to avoid exception overhead

**Solution - Filesystem Dual API:**

The standard filesystem library is a good example of the dual-API pattern: every operation has a throwing version and a non-throwing version that takes an `error_code` by reference. Use the throwing version for one-off calls; use the `error_code` version when you are iterating over hundreds of files and cannot afford an exception per missing entry:

```cpp
#include <iostream>
#include <filesystem>
#include <system_error>

namespace fs = std::filesystem;

int main() {
    // ========== Version 1: Throwing (default) ==========
    try {
        auto size = fs::file_size("/nonexistent/file.txt");
        std::cout << "Size: " << size << "\n";
    } catch (const fs::filesystem_error& e) {
        std::cout << "Exception: " << e.what() << "\n";
        std::cout << "  Code: " << e.code().value()
                  << " (" << e.code().message() << ")\n";
        std::cout << "  Path: " << e.path1() << "\n";
    }

    // ========== Version 2: Non-throwing (error_code overload) ==========
    std::error_code ec;
    auto size = fs::file_size("/nonexistent/file.txt", ec);
    if (ec) {
        std::cout << "Error code: " << ec.value()
                  << " (" << ec.message() << ")\n";
        // No exception thrown, no stack unwinding - deterministic performance
    } else {
        std::cout << "Size: " << size << "\n";
    }

    // ========== Practical use: batch operations ==========
    // When checking MANY files, the error_code overload avoids
    // the overhead of throwing/catching for each missing file
    std::vector<std::string> paths = {
        "file1.txt", "file2.txt", "missing.txt", "file3.txt"
    };

    for (const auto& p : paths) {
        std::error_code ec2;
        bool exists = fs::exists(p, ec2);
        if (ec2) {
            std::cout << p << ": error checking (" << ec2.message() << ")\n";
        } else if (!exists) {
            std::cout << p << ": not found\n";  // expected condition, not an error
        } else {
            auto sz = fs::file_size(p, ec2);
            if (!ec2)
                std::cout << p << ": " << sz << " bytes\n";
        }
    }

    // Common filesystem functions with error_code overloads:
    // fs::exists(path, ec)
    // fs::file_size(path, ec)
    // fs::copy(from, to, ec)
    // fs::remove(path, ec)
    // fs::rename(from, to, ec)
    // fs::create_directories(path, ec)
    // fs::last_write_time(path, ec)
}
```

Notice that in the batch loop, a missing file is not an error we want to throw for - it is a normal expected condition. The `error_code` overload lets you treat it as a simple boolean check rather than a caught exception.

**Why Use the `error_code` Overload:**

| Scenario | Throwing Version | `error_code` Overload |
| --- | --- | --- |
| Single file operation | Fine - simple try/catch | Unnecessary complexity |
| Batch operations (1000+ files) | Slow - throwing for each missing file | Fast - no unwinding |
| Expected failures (file may not exist) | Misuse of exceptions for flow control | Correct - check and continue |
| Performance-critical code | ~1-10 us per throw | ~10 ns per check |

---

### Q3: Compare `std::error_code` with `errno` and explain why `error_code` is more composable

**Head-to-Head Comparison:**

| Aspect | `errno` | `std::error_code` |
| --- | --- | --- |
| **Type** | Global `int` (thread-local) | Value type with category |
| **Namespace** | Single flat integer space | Multiple categories (system, generic, custom) |
| **Meaning of value `2`** | Always `ENOENT` | Depends on category (could be HTTP 200, your custom code, etc.) |
| **Thread safety** | Thread-local (C11/C++11) | Value type - inherently safe |
| **Composability** | Can't mix with library errors | Different libraries define own categories |
| **Message lookup** | `strerror(errno)` | `ec.message()` - category provides the message |
| **Portable comparison** | Platform-specific values | `error_condition` enables cross-platform checks |
| **Type safety** | Just an `int` - easy to misuse | Strongly typed with category |

The code below puts both approaches side-by-side so you can see exactly what goes wrong with `errno` and what `error_code` does instead:

```cpp
#include <iostream>
#include <system_error>
#include <cerrno>
#include <cstring>
#include <fstream>

void errno_approach() {
    // BAD: Problems with errno:
    errno = 0;
    FILE* f = fopen("nonexistent.txt", "r");
    if (!f) {
        // 1. errno is a GLOBAL - any function call between here could change it
        int saved = errno;
        std::cout << "errno = " << saved << ": " << strerror(saved) << "\n";
        // 2. errno values are PLATFORM-SPECIFIC - ENOENT = 2 on POSIX, different on Windows
        // 3. Can't distinguish YOUR error codes from SYSTEM error codes
        // 4. No category - all errors are just integers
    }
}

void error_code_approach() {
    // GOOD: std::error_code is composable:
    std::error_code sys_err = std::make_error_code(std::errc::no_such_file_or_directory);
    // Custom library error with SAME value 2 - but DIFFERENT category:
    // std::error_code http_err = HttpError::bad_request;  // value=400, http_category

    std::cout << "System error: " << sys_err.value() << " ["
              << sys_err.category().name() << "] " << sys_err.message() << "\n";

    // Portable comparison:
    if (sys_err == std::errc::no_such_file_or_directory)
        std::cout << "-> portable 'file not found' check works!\n";

    // Convert errno to error_code:
    errno = EACCES;
    std::error_code from_errno(errno, std::system_category());
    std::cout << "From errno: " << from_errno.message() << "\n";
}

int main() {
    errno_approach();
    error_code_approach();
}
// Expected output (Linux):
//   errno = 2: No such file or directory
//   System error: 2 [generic] No such file or directory
//   -> portable 'file not found' check works!
//   From errno: Permission denied
```

**Composability in Practice:**

The key insight is that `errno` is a flat global integer namespace - every library has to share the same pool of numbers, so they collide. `error_code` solves this by pairing the number with its category, so the same integer value can mean completely different things in different contexts:

```cpp
errno world:                           error_code world:
+-------------+                       +-------------------------+
| errno = 2   | <- ENOENT             | {2, system_category()}  | <- OS error
| errno = 13  | <- EACCES             | {2, http_category()}    | <- HTTP 200 OK
| errno = 22  | <- EINVAL             | {2, grpc_category()}    | <- gRPC UNKNOWN
| ... just ints, all ONE namespace     | Same value, DIFFERENT meaning - safe! |
+-------------+                       +-------------------------+

Multiple libraries can define their own categories.
Values never collide because the category distinguishes them.
```

---

## Notes

- **`std::errc`** is a portable enum of POSIX error codes - use it for cross-platform error generation.
- **`std::system_error`** is an exception that wraps `std::error_code` - throw it when you want exception-based error handling with system error context.
- **`std::error_code` is cheap** - 8-16 bytes, no allocation, trivially copyable.
- **Specializing `is_error_code_enum`** enables implicit conversion from your enum to `error_code`.
- **`error_condition`** is for portable comparison; `error_code` is for platform-specific reporting. Use `error_condition` in `if` comparisons.
- **Boost.System** was the original implementation - `std::error_code` was standardized from Boost in C++11.
