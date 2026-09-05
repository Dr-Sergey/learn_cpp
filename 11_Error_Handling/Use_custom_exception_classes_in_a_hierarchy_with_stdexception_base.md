# Use custom exception classes in a hierarchy with std::exception base

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/error/exception>  

---

## Topic Overview

Custom exception hierarchies let you define **domain-specific error types** that integrate seamlessly with the standard `try/catch` mechanism. By deriving from `std::exception` (or its subclasses), you get polymorphic catching, meaningful `what()` messages, and the ability to carry structured error data - like HTTP status codes, database error codes, or network timeouts - right on the exception object.

### Design Principles

The most important decision is how granular to make your hierarchy. Too flat and you cannot distinguish between different kinds of errors; too deep and catching becomes awkward. Here is the contrast:

```cpp
// Good hierarchy:                      Bad hierarchy:
// std::exception                       std::exception
//   └── std::runtime_error               └── MyError (catches EVERYTHING)
//         └── AppError
//               ├── NetworkError       Flat, no granularity - can't
//               │     ├── HttpError    distinguish network from DB errors
//               │     └── DnsError
//               └── DatabaseError
//                     ├── QueryError
//                     └── ConnectionError
```

A three-level hierarchy (app root, domain layer, specific error) is usually enough for any application. Going deeper than that rarely pays off.

### Rules for Custom Exceptions

| Rule | Why |
| --- | --- |
| Derive from `std::exception` or a subclass | Compatibility with existing `catch (const std::exception&)` handlers |
| Override `what()` via constructor string | Provides human-readable error message |
| Catch by `const` reference | Prevents slicing of derived exception data |
| Catch most-derived first | C++ matches first matching `catch` clause in order |
| Keep exceptions lightweight | They're copied during stack unwinding |
| Make `what()` `noexcept` | It's declared `noexcept` in the base class |

### Core Pattern

The simplest custom exception just adds a name to an existing standard class. The base class handles string lifetime for you, so this pattern is nearly zero boilerplate:

```cpp
#include <stdexcept>
#include <string>

class AppError : public std::runtime_error {
public:
    // Pass message to runtime_error - it manages the string lifetime
    explicit AppError(const std::string& msg)
        : std::runtime_error(msg) {}
};

class NetworkError : public AppError {
    int status_code_;
public:
    NetworkError(const std::string& msg, int code)
        : AppError(msg), status_code_(code) {}

    int status_code() const noexcept { return status_code_; }
};
```

The pattern scales up naturally: each level adds domain-specific fields, and the `what()` message is built by passing a formatted string to the base constructor.

### Important Notes

- `std::exception::what()` is `virtual` and `noexcept` - never throw from it.
- `std::runtime_error` and `std::logic_error` store the message string internally - safe to construct from temporary strings.
- Avoid slicing: `throw NetworkError(...)` is caught correctly by `catch (const AppError&)` - but only by reference, not by value.

---

## Self-Assessment

### Q1: Design a three-level exception hierarchy: `AppException` -> `NetworkException` -> `TimeoutException`

Each level in this hierarchy adds fields that are specific to that layer. `AppException` gets an error ID. `NetworkException` adds host and port. `TimeoutException` adds timeout limits and elapsed time. The `what()` message is built in the constructor and passed up the chain:

**Solution - Complete Three-Level Hierarchy:**

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <chrono>

// Level 1: Application root exception
class AppException : public std::runtime_error {
    int error_id_;
public:
    AppException(const std::string& msg, int id = 0)
        : std::runtime_error(msg), error_id_(id) {}

    int error_id() const noexcept { return error_id_; }
};

// Level 2: Network-specific error
class NetworkException : public AppException {
    std::string host_;
    int port_;
public:
    NetworkException(const std::string& msg,
                     const std::string& host, int port, int id = 1000)
        : AppException(msg, id), host_(host), port_(port) {}

    const std::string& host() const noexcept { return host_; }
    int port() const noexcept { return port_; }
};

// Level 3: Timeout-specific error
class TimeoutException : public NetworkException {
    std::chrono::milliseconds timeout_;
    std::chrono::milliseconds elapsed_;
public:
    TimeoutException(const std::string& host, int port,
                     std::chrono::milliseconds timeout,
                     std::chrono::milliseconds elapsed)
        : NetworkException(
              "Connection to " + host + ":" + std::to_string(port) +
              " timed out after " + std::to_string(elapsed.count()) + "ms" +
              " (limit: " + std::to_string(timeout.count()) + "ms)",
              host, port, 1001)
        , timeout_(timeout)
        , elapsed_(elapsed) {}

    std::chrono::milliseconds timeout() const noexcept { return timeout_; }
    std::chrono::milliseconds elapsed() const noexcept { return elapsed_; }
};

// Usage
void connect_to_server(const std::string& host, int port) {
    using namespace std::chrono_literals;
    // Simulate a timeout
    throw TimeoutException(host, port, 3000ms, 3500ms);
}

int main() {
    try {
        connect_to_server("db.example.com", 5432);
    }
    // MOST DERIVED FIRST - order matters!
    catch (const TimeoutException& e) {
        std::cout << "[TIMEOUT] " << e.what() << "\n";
        std::cout << "  Host: " << e.host() << ":" << e.port() << "\n";
        std::cout << "  Elapsed: " << e.elapsed().count()
                  << "ms / Limit: " << e.timeout().count() << "ms\n";
    }
    catch (const NetworkException& e) {
        std::cout << "[NETWORK] " << e.what() << "\n";
        std::cout << "  Host: " << e.host() << ":" << e.port() << "\n";
    }
    catch (const AppException& e) {
        std::cout << "[APP] " << e.what()
                  << " (error_id=" << e.error_id() << ")\n";
    }
    catch (const std::exception& e) {
        std::cout << "[STD] " << e.what() << "\n";
    }
}
// Expected output:
//   [TIMEOUT] Connection to db.example.com:5432 timed out after 3500ms (limit: 3000ms)
//     Host: db.example.com:5432
//     Elapsed: 3500ms / Limit: 3000ms
```

---

### Q2: Show catch ordering: most derived first to avoid catching too broadly

This is one of those rules that is easy to state but surprisingly easy to get wrong. C++ matches the first `catch` clause whose type is compatible - and a base class is always compatible with a derived class. Put the base class first and you will never reach the derived handler:

**Solution - Correct vs Incorrect Ordering:**

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

class AppError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class DbError : public AppError {
public:
    using AppError::AppError;
};

class QueryError : public DbError {
    std::string query_;
public:
    QueryError(const std::string& msg, const std::string& query)
        : DbError(msg), query_(query) {}
    const std::string& query() const noexcept { return query_; }
};

void run_query() {
    throw QueryError("syntax error near 'SELCT'", "SELCT * FROM users");
}

// BAD: Base class catches FIRST - derived handlers never execute
void bad_ordering() {
    try {
        run_query();
    }
    catch (const std::exception& e) {  // catches EVERYTHING - too broad
        std::cout << "std::exception: " << e.what() << "\n";
        // QueryError caught here - we lose access to .query()
    }
    catch (const AppError& e) {  // NEVER REACHED
        std::cout << "AppError: " << e.what() << "\n";
    }
    catch (const QueryError& e) {  // NEVER REACHED
        std::cout << "QueryError: " << e.what()
                  << " [query=" << e.query() << "]\n";
    }
}
// Compiler warning: "exception handler will never be executed"

// GOOD: Most derived first
void good_ordering() {
    try {
        run_query();
    }
    catch (const QueryError& e) {  // most derived - matched first
        std::cout << "QueryError: " << e.what()
                  << "\n  Query: " << e.query() << "\n";
    }
    catch (const DbError& e) {  // catches DbError but NOT QueryError
        std::cout << "DbError: " << e.what() << "\n";
    }
    catch (const AppError& e) {  // catches AppError but not Db/Query
        std::cout << "AppError: " << e.what() << "\n";
    }
    catch (const std::exception& e) {  // catch-all for anything else
        std::cout << "Unknown: " << e.what() << "\n";
    }
}

int main() {
    std::cout << "--- correct ordering ---\n";
    good_ordering();
}
// Expected output:
//   --- correct ordering ---
//   QueryError: syntax error near 'SELCT'
//     Query: SELCT * FROM users
```

The simple rule to remember is: `catch` clauses are checked top-to-bottom, and the first match wins. Base classes match derived objects. So always order from most specific to least specific:

```cpp
// catch clauses are checked TOP-TO-BOTTOM.
// First match wins. Base classes match derived objects.
//
// ORDER: derived -> base -> std::exception -> catch(...)
//        most specific -> least specific
```

---

### Q3: Explain why catching by value vs by reference matters for polymorphic exception objects

Catching by value causes "slicing" - the derived part of the exception object is literally cut off during the copy, and you end up with a base-class object. You lose all the structured fields you carefully added. This is why `catch (const SomeError&)` is always the right form.

**The Slicing Problem:**

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

class DetailedError : public std::runtime_error {
    int code_;
public:
    DetailedError(const std::string& msg, int code)
        : std::runtime_error(msg), code_(code) {}
    int code() const noexcept { return code_; }
};

void throw_detailed() {
    throw DetailedError("database timeout", 504);
}

int main() {
    // BAD: Catch by VALUE -> SLICING
    try {
        throw_detailed();
    }
    catch (std::runtime_error e) {  // catches BY VALUE
        std::cout << "Type: " << typeid(e).name() << "\n";
        // Type is std::runtime_error, NOT DetailedError!
        // The DetailedError part (code_) is SLICED OFF.
        // e.what() still works, but all derived data is lost.

        // This would fail:
        // auto& d = dynamic_cast<DetailedError&>(e);  // throws bad_cast!
    }

    // GOOD: Catch by CONST REFERENCE -> full polymorphism preserved
    try {
        throw_detailed();
    }
    catch (const std::runtime_error& e) {  // catches BY REFERENCE
        std::cout << "Type: " << typeid(e).name() << "\n";
        // Type is DetailedError - full polymorphic object preserved!

        // Can safely downcast:
        if (auto* d = dynamic_cast<const DetailedError*>(&e)) {
            std::cout << "Code: " << d->code() << "\n";  // 504
        }
    }
}
// Expected output (MSVC):
//   Type: class std::runtime_error   (sliced!)
//   Type: class DetailedError        (preserved!)
//   Code: 504
```

Here is exactly what happens to the object during slicing:

```cpp
// throw DetailedError("db timeout", 504):
//
// Original object in exception storage:
//   runtime_error part:
//     what_ = "db timeout"
//   DetailedError part:
//     code_ = 504
//
// catch (std::runtime_error e):      <- copy by VALUE
//   runtime_error part:
//     what_ = "db timeout"           <- only this is copied
//     (DetailedError part GONE)
//
// catch (const std::runtime_error& e):  <- reference to ORIGINAL
//   -> Points to the full DetailedError object
//   -> code_ still accessible via dynamic_cast
```

**The Rules:**

| Catch Style | Polymorphism | Performance | Recommended? |
| --- | --- | --- | --- |
| `catch (Exception e)` | Sliced | Bad (copy) | Never |
| `catch (Exception& e)` | Preserved | Good (reference) | OK |
| `catch (const Exception& e)` | Preserved | Good (reference) | **Best** |
| `catch (...)` | N/A | N/A | Last resort only |

**Always catch by `const` reference** - it preserves the full polymorphic object, avoids unnecessary copies, and prevents accidental modification.

---

## Notes

- **Virtual inheritance:** If your hierarchy has diamond patterns, use `virtual` inheritance from `std::exception` to avoid ambiguous base classes.
- **`noexcept` on `what()`:** The base class `what()` is `noexcept`. Custom overrides must also be `noexcept`.
- **Don't inherit from `std::exception` directly** unless you have a reason - prefer `std::runtime_error` or `std::logic_error` for their string management.
- **`throw;` vs `throw e;`:** Inside a catch block, `throw;` rethrows the original (no slicing). `throw e;` throws a **copy** (may slice!).
- **Keep constructors simple** - exception constructors run during error handling; if they throw, you get `std::terminate()`.
- **`std::exception_ptr`** stores any exception polymorphically - use with `std::current_exception()` and `std::rethrow_exception()` for cross-thread propagation.
