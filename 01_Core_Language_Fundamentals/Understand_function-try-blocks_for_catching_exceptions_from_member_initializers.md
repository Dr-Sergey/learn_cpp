# Understand function-try-blocks for catching exceptions from member initializers

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/function-try-block>  

---

## Topic Overview

A **function-try-block** wraps the entire function body (including the member initializer list) in a try-catch. This is the **only** way to catch exceptions thrown by base class or member constructors during initialization.

### Syntax

The key difference from a regular try-catch is where the `try` keyword goes - it moves before the colon of the initializer list, not inside the constructor body:

```cpp
// Normal try-catch: CANNOT catch member initializer exceptions
MyClass(int x) : member(x) {
    try { /* body */ } catch (...) { /* only catches body exceptions */ }
}

// Function-try-block: catches EVERYTHING including member initializers
MyClass(int x)
try : member(x) {           // try covers the initializer list AND body
    // constructor body
}
catch (const std::exception& e) {
    // Can catch exceptions from member(x) initialization
    std::cerr << "Init failed: " << e.what() << "\n";
    // IMPORTANT: for constructors, the exception is ALWAYS rethrown
}
```

### Critical Rule: Constructor Function-Try-Blocks Always Rethrow

When a constructor's function-try-block catches an exception, the catch handler **implicitly rethrows** the exception when it exits - you cannot swallow it. This is because:

1. The object is only partially constructed - member lifetimes haven't begun or have been destroyed.
2. Returning a "valid" object from a half-constructed state would be undefined behavior.
3. The standard mandates: if control reaches the end of a constructor's catch handler, the current exception is rethrown.

You **can** throw a different exception from the catch handler, but you cannot prevent an exception from propagating.

### What You CAN Do in the Catch Handler

- Log the error
- Clean up external resources (files, network connections)
- Throw a **different** exception (wrapping the original)
- **NOT**: access member variables (they may be destroyed)

Here is a small example showing the allowed pattern - catch the member init failure and translate it into a higher-level exception:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

class FileLogger {
    std::string filename;
public:
    FileLogger(const std::string& f) : filename(f) {
        if (f.empty()) throw std::runtime_error("empty filename");
    }
};

class Application
try : logger("") {   // will throw!
    // constructor body
}
catch (const std::exception& e) {
    std::cerr << "Application init failed: " << e.what() << "\n";
    // Cannot access 'logger' here -- it was never fully constructed
    // Exception is automatically rethrown when catch exits
    // Or throw a different exception:
    throw std::runtime_error("Application startup failed");
}
```

---

## Self-Assessment

### Q1: Write a constructor with a function-try-block to catch exceptions from member initializers

This example initializes two members in sequence - if either one throws, the function-try-block catches it and can re-wrap it before propagating:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>

class Database {
public:
    Database(const std::string& connStr) {
        if (connStr.empty()) throw std::runtime_error("empty connection string");
        std::cout << "Database connected: " << connStr << "\n";
    }
};

class Cache {
public:
    Cache(int maxSize) {
        if (maxSize <= 0) throw std::invalid_argument("invalid cache size");
        std::cout << "Cache created with size " << maxSize << "\n";
    }
};

class Server
{
    Database db;
    Cache cache;

public:
    // Function-try-block -- catches exceptions from db and cache initialization
    Server(const std::string& dbConn, int cacheSize)
    try : db(dbConn), cache(cacheSize) {
        std::cout << "Server initialized successfully\n";
    }
    catch (const std::invalid_argument& e) {
        std::cerr << "Invalid config: " << e.what() << "\n";
        throw std::runtime_error("Server config error: " + std::string(e.what()));
    }
    catch (const std::runtime_error& e) {
        std::cerr << "Runtime init error: " << e.what() << "\n";
        // Implicit rethrow if we don't throw explicitly
    }
};

int main() {
    try {
        Server s1("localhost:5432", 100);   // OK
    } catch (...) {}

    try {
        Server s2("", 100);    // Database throws -- caught by function-try-block
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    try {
        Server s3("localhost", -1);   // Cache throws -- caught and re-wrapped
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }
}
```

**How it works:**

- The `try` keyword appears before the `:` initializer list, covering both member initialization and the body.
- If `db(dbConn)` throws, the catch handler executes. `cache` was never constructed, so it's not destroyed.
- The catch handler can throw a different (wrapped) exception or let the original implicitly rethrow.

### Q2: Explain why a function-try-block in a constructor always rethrows the exception on exit

**Answer:**

When a constructor throws during member initialization:

1. **Already-constructed members are destroyed** (in reverse order).
2. **The object's lifetime never began** - the memory is allocated but no valid object exists.
3. **Referencing members in the catch handler is undefined behavior** - they've been destroyed.

If the language allowed swallowing the exception:

- The constructor would "succeed" with a partially constructed, invalid object.
- Calling any member function or accessing any data member would be UB.
- Destructors would run on objects that were never fully constructed.

The standard comment in the code below explains exactly what the spec says:

```cpp
struct Bad {
    std::vector<int> data;

    Bad(int n)
    try : data(n, 0) {
    }
    catch (...) {
        // If we could swallow the exception:
        // - data is already destroyed (its dtor ran after the throw)
        // - 'this' points to garbage
        // - the caller would have a Bad object with destroyed members

        // The standard prevents this by implicitly rethrowing.
        // [except.handle]/15: "If control reaches the end of a handler
        //   of a function-try-block of a constructor, the current exception
        //   is rethrown."
    }
};
```

The same rule applies to **destructor** function-try-blocks - if a member destructor throws and you catch it, the exception is rethrown (though throwing destructors should be avoided entirely).

### Q3: Show a case where a function-try-block in a destructor catches an exception from a member destructor

```cpp
#include <iostream>
#include <stdexcept>

// WARNING: Throwing destructors are extremely dangerous.
// This example is educational only -- never do this in production.

struct ThrowingMember {
    ~ThrowingMember() noexcept(false) {   // must opt out of noexcept
        std::cout << "ThrowingMember destructor -- throwing!\n";
        throw std::runtime_error("member dtor threw");
    }
};

class Container {
    ThrowingMember mem;
public:
    Container() { std::cout << "Container constructed\n"; }

    // Function-try-block on destructor
    ~Container()
    try {
        std::cout << "Container destructor body\n";
        // mem.~ThrowingMember() is called after the body
    }
    catch (const std::exception& e) {
        std::cerr << "Caught in dtor: " << e.what() << "\n";
        // Like constructors, the exception is implicitly rethrown.
        // If THIS destructor was called during stack unwinding,
        // rethrowing causes std::terminate().
    }
};

int main() {
    try {
        Container c;
        // c goes out of scope -> Container's dtor function-try-block runs
    }
    catch (const std::exception& e) {
        std::cout << "Main caught: " << e.what() << "\n";
    }
}
```

**How it works:**

- `ThrowingMember`'s destructor throws an exception (declared `noexcept(false)`).
- The Container's function-try-block catches the exception from the member's destructor.
- However, like constructors, the exception is rethrown from the catch handler.

**Why this is dangerous:**

- If the destructor is called during stack unwinding (another exception is active), the rethrow calls `std::terminate()`.
- This is why destructors should **always** be `noexcept` (the default since C++11).
- Function-try-blocks on destructors are useful only for logging - they cannot prevent the exception from propagating.

---

## Notes

- Function-try-blocks work on any function (not just constructors/destructors), but for non-constructor functions they behave like a normal try-catch around the entire body.
- In a constructor's catch handler, you **cannot** access `this`, base subobjects, or member variables - their lifetimes have ended.
- In practice, function-try-blocks are rare. Prefer RAII and `noexcept` destructors to avoid needing them.
- The only practical use case: **wrapping/translating exceptions** from member constructors (e.g., converting a low-level DB exception into a higher-level application exception).
