# Use try/catch at the right granularity: not too fine, not too coarse

**Category:** Error Handling  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Re-catch>  

---

## Topic Overview

Exception handling is most effective when **try/catch boundaries** align with **logical transaction boundaries** - the level at which you can meaningfully respond to or report an error. Too fine-grained catching adds noise and hides intent. Too coarse-grained catching swallows diagnostic details and prevents targeted recovery.

### The Granularity Spectrum

Think of it in terms of what you can actually do with the exception at each level. If the only thing you can do is log and return, wrapping every individual call in its own try/catch is pure noise. Group the calls that belong together, and catch once at the boundary of the logical operation:

```cpp
Too fine (anti-pattern)             Right level                  Too coarse (anti-pattern)
----------------------              ----------                   ----------------------
try { a(); } catch (...) {}         try {                        try {
try { b(); } catch (...) {}             a();                         entire_main();
try { c(); } catch (...) {}             b();                     } catch (...) {
                                        c();                         // what failed? which
// BAD: Can't tell which failed     } catch (const DbErr&) {}        // step? no context.
// BAD: Hides control flow          // One transaction boundary   }
// BAD: Swallows errors silently                                  // BAD: Swallows ALL errors
```

### Guidelines for Choosing Granularity

| Guideline | Explanation |
| --- | --- |
| Wrap a **logical transaction** | Group operations that should succeed or fail together |
| Catch where you **can act** | Only catch if you can retry, log, translate, or clean up |
| Don't catch what you can't handle | Let it propagate to a caller who can |
| Catch by **const reference** | Avoids slicing and unnecessary copying |
| One catch per **error category** | Don't catch `...` unless you're at the top level |

### The "Catch By Const Reference" Rule

```cpp
// BAD: catch by value - SLICES derived exceptions
try { throw DerivedError("details"); }
catch (BaseError e) {    // copies + slices - DerivedError -> BaseError
    e.what();            // loses DerivedError::what() override
}

// BAD: catch by non-const reference (unless you modify and rethrow)
try { throw SomeError(); }
catch (SomeError& e) { }  // works but non-const suggests mutation

// GOOD: catch by const reference
try { throw DerivedError("details"); }
catch (const BaseError& e) {  // no copy, no slicing, preserves vtable
    e.what();                  // calls DerivedError::what()
}
```

---

## Self-Assessment

### Q1: Show a try block around a single expression vs an entire function and explain the readability tradeoff

**Solution - Fine vs Coarse Granularity:**

Read through the three versions below and notice how the "too fine" version makes the happy path almost impossible to follow - the error handling code outnumbers the actual logic:

```cpp
#include <iostream>
#include <fstream>
#include <string>
#include <stdexcept>
#include <vector>

struct Config {
    std::string host;
    int port;
    std::string db_name;
};

Config parse_config(const std::string& text);        // may throw parse_error
void connect_db(const std::string& host, int port);  // may throw connection_error
std::vector<std::string> query(const std::string& q); // may throw query_error

// BAD: TOO FINE - wrapping every expression individually
void too_fine(const std::string& path) {
    std::string text;
    try {
        std::ifstream f(path);
        text.assign(std::istreambuf_iterator<char>(f), {});
    } catch (const std::exception& e) {
        std::cerr << "Read error\n";
        return;  // early return - disrupts flow
    }

    Config cfg;
    try {
        cfg = parse_config(text);
    } catch (const std::exception& e) {
        std::cerr << "Parse error\n";
        return;
    }

    try {
        connect_db(cfg.host, cfg.port);
    } catch (const std::exception& e) {
        std::cerr << "Connect error\n";
        return;
    }

    // Problems:
    //   - 3 try/catch blocks for 1 logical operation (initialize)
    //   - Each catch does the same thing - log and return
    //   - Hard to read the "happy path"
    //   - Can't distinguish errors in a unified handler
}

// GOOD: one try for one logical transaction
void right_granularity(const std::string& path) {
    try {
        std::ifstream f(path);
        if (!f) throw std::runtime_error("Cannot open " + path);
        std::string text(std::istreambuf_iterator<char>(f), {});

        auto cfg = parse_config(text);
        connect_db(cfg.host, cfg.port);
        auto results = query("SELECT * FROM users");

        for (const auto& row : results)
            std::cout << row << "\n";

    } catch (const std::runtime_error& e) {
        // One catch for the whole "load config and connect" transaction
        std::cerr << "Initialization failed: " << e.what() << "\n";
    }
    // Clean happy path visible in the try block
    // One error handling point for one logical operation
}

// BAD: TOO COARSE - wrapping main()
int too_coarse_main() {
    try {
        // 500 lines of application logic...
        // Which of the 50 possible throw sites failed?
        // Can we retry? Which part? We don't know.
    } catch (const std::exception& e) {
        std::cerr << "Something failed: " << e.what() << "\n";
    }
    return 0;
}

int main() {
    right_granularity("config.txt");
}
```

**Readability comparison:**

```cpp
Too Fine:                    Right Level:                 Too Coarse:
3 try/catch, 3 returns       1 try/catch                  1 try/catch
Happy path fragmented         Happy path clear             Happy path buried
9 lines of error handling     3 lines of error handling    3 lines, no context
Easy to miss an error         All errors caught            All errors caught
                              at the right level           but can't act on them
```

---

### Q2: Demonstrate how catching at the wrong level swallows context needed for diagnosis

**Solution - Context Loss Through Wrong-Level Catching:**

The reason this matters is that rich exception types carry structured data - not just a string message, but typed fields like error codes and query text. When you catch too early and re-throw as a base type, all of that structured data is gone forever:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>

// Exception hierarchy with context
struct DatabaseError : std::runtime_error {
    std::string query;
    int error_code;
    DatabaseError(const std::string& msg, const std::string& q, int code)
        : std::runtime_error(msg), query(q), error_code(code) {}
};

struct ConnectionError : std::runtime_error {
    std::string host;
    int port;
    ConnectionError(const std::string& msg, const std::string& h, int p)
        : std::runtime_error(msg), host(h), port(p) {}
};

// Low-level function with rich error info
void execute_query(const std::string& q) {
    throw DatabaseError("Syntax error near 'FORM'", q, 1064);
}

// BAD: catching TOO EARLY swallows context
void bad_repository_layer(const std::string& query) {
    try {
        execute_query(query);
    } catch (const std::exception& e) {
        // PROBLEM: catches as base exception, re-throws as generic
        throw std::runtime_error("Database operation failed");
        // Lost: query string, error code 1064, original type
    }
}

// GOOD: let specific exceptions propagate, or re-throw with context
void good_repository_layer(const std::string& query) {
    // Option A: Don't catch - let it propagate with full context
    execute_query(query);

    // Option B: Catch, add context, re-throw
    // try {
    //     execute_query(query);
    // } catch (DatabaseError& e) {
    //     std::throw_with_nested(
    //         std::runtime_error("Repository: user lookup failed"));
    // }
}

int main() {
    // Bad path: context lost
    std::cout << "=== Bad catching level ===\n";
    try {
        bad_repository_layer("SELECT * FORM users");
    } catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << "\n";
        // Output: "Database operation failed"
        // We lost: the query, the error code, the specific error type
        // Can't tell if it's a syntax error, connection issue, or permission problem
    }

    // Good path: context preserved
    std::cout << "\n=== Right catching level ===\n";
    try {
        good_repository_layer("SELECT * FORM users");
    } catch (const DatabaseError& e) {
        std::cout << "Error: " << e.what() << "\n";
        std::cout << "Query: " << e.query << "\n";
        std::cout << "Code:  " << e.error_code << "\n";
        // Output preserves ALL diagnostic info
    }
}
// Expected output:
//   === Bad catching level ===
//   Error: Database operation failed
//
//   === Right catching level ===
//   Error: Syntax error near 'FORM'
//   Query: SELECT * FORM users
//   Code:  1064
```

**Rule of thumb:**

```cpp
Do NOT catch if you:            DO catch if you:
---------------------           ------------------
- Can't recover                 - Can retry the operation
- Would just re-throw           - Need to translate the error for a higher layer
- Would lose context            - Are at a natural boundary (HTTP handler, main loop)
- Are in a library function     - Need to log AND propagate (throw_with_nested)
```

---

### Q3: Apply the "catch by const reference" rule and explain why catching by value slices

**Solution - Slicing Demonstration:**

Exception slicing is a subtle bug: you throw a derived exception with extra fields, but the catch clause copies just the base portion of the object. The derived part - including overridden virtual functions - silently disappears. Catching by const reference avoids the copy entirely and preserves the full dynamic type:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

// A derived exception with extra data
class AppError : public std::runtime_error {
    int code_;
public:
    AppError(const std::string& msg, int code)
        : std::runtime_error(msg), code_(code) {}
    int code() const { return code_; }

    // Override what() to include the code
    const char* what() const noexcept override {
        // Note: in real code, cache this in a member
        static thread_local std::string buf;
        buf = std::string(std::runtime_error::what()) + " [code=" + std::to_string(code_) + "]";
        return buf.c_str();
    }
};

void throw_app_error() {
    throw AppError("file corruption detected", 42);
}

int main() {
    // Catch by VALUE - SLICES the exception
    std::cout << "=== Catch by value (SLICING) ===\n";
    try {
        throw_app_error();
    } catch (std::runtime_error e) {  // by value: copies and SLICES
        std::cout << "Type: " << typeid(e).name() << "\n";
        std::cout << "what(): " << e.what() << "\n";
        // Lost: AppError::code_, AppError::what() override
        // e is now a plain runtime_error!
    }

    // Catch base by value - even more slicing
    std::cout << "\n=== Catch std::exception by value ===\n";
    try {
        throw_app_error();
    } catch (std::exception e) {  // slices down to std::exception
        std::cout << "what(): " << e.what() << "\n";
        // May even lose the error message on some implementations!
    }

    // Catch by CONST REFERENCE - preserves everything
    std::cout << "\n=== Catch by const reference (CORRECT) ===\n";
    try {
        throw_app_error();
    } catch (const std::runtime_error& e) {
        std::cout << "Type: " << typeid(e).name() << "\n";
        std::cout << "what(): " << e.what() << "\n";
        // Dynamic type preserved - calls AppError::what()!

        // Can even downcast if needed:
        if (auto* app_err = dynamic_cast<const AppError*>(&e)) {
            std::cout << "code(): " << app_err->code() << "\n";
        }
    }

    // Catch ordering: most derived FIRST
    std::cout << "\n=== Correct catch ordering ===\n";
    try {
        throw_app_error();
    } catch (const AppError& e) {          // Most derived - checked FIRST
        std::cout << "AppError: " << e.what() << ", code=" << e.code() << "\n";
    } catch (const std::runtime_error& e) { // Base - checked second
        std::cout << "runtime_error: " << e.what() << "\n";
    } catch (const std::exception& e) {     // Most general - last
        std::cout << "exception: " << e.what() << "\n";
    }
}
// Expected output:
//   === Catch by value (SLICING) ===
//   Type: St13runtime_error       (sliced to runtime_error!)
//   what(): file corruption detected   (lost the [code=42] suffix!)
//
//   === Catch std::exception by value ===
//   what(): file corruption detected
//
//   === Catch by const reference (CORRECT) ===
//   Type: 8AppError               (dynamic type preserved!)
//   what(): file corruption detected [code=42]
//   code(): 42
//
//   === Correct catch ordering ===
//   AppError: file corruption detected [code=42], code=42
```

**Why slicing happens:**

```cpp
AppError object in exception storage:
+------------------------------------+
| std::exception  { vtable, what }   |
| std::runtime_error { message }     |
| AppError { code_ = 42 }           | <- full object
+------------------------------------+

catch (std::runtime_error e):         // by VALUE -> copy constructor
+--------------------------------+
| std::exception  { vtable }     |
| std::runtime_error { message } |    <- AppError portion GONE
+--------------------------------+    <- vtable points to runtime_error

catch (const std::runtime_error& e):  // by REFERENCE -> no copy
-> reference to the ORIGINAL object
-> vtable still points to AppError
-> dynamic dispatch works correctly
```

---

## Notes

- **CppCoreGuidelines E.15:** Catch exceptions from a hierarchy by reference.
- **CppCoreGuidelines E.17:** Don't try to catch every exception in every function - catch where you can handle it.
- **Catch `...` only at boundaries:** `main()`, thread entry points, callback dispatchers.
- **"Catch-log-rethrow"** is acceptable: `catch (const E& e) { log(e); throw; }` - preserves the original exception. `throw;` rethrows without slicing.
- **Function-try-blocks** exist for constructors: `C::C() try : member(...) { } catch (...) { }` - rarely used, useful for logging init failures.
- **`noexcept` vs try/catch:** If a function can't meaningfully handle an error, mark it `noexcept` rather than wrapping in try/catch.
