# Use std::terminate_handler and std::unexpected_handler correctly

**Category:** Error Handling  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/error/terminate_handler>  

---

## Topic Overview

`std::terminate_handler` is a function pointer type for the function called by `std::terminate()`. By installing a custom handler via `std::set_terminate()`, you can log diagnostic information before the program aborts. `std::unexpected_handler` was its counterpart for dynamic exception specifications, **removed in C++17**.

### When `std::terminate()` Is Called

It is worth memorizing these triggers, because some of them are surprising - in particular, a `noexcept` function that throws goes directly to `std::terminate`, skipping any outer `try/catch`:

```cpp
Situation                                    Example
-------------------------------------------- ------------------------------
Uncaught exception                           throw with no matching catch
Exception escapes noexcept function          void f() noexcept { throw X; }
Exception during stack unwinding             Destructor throws while
                                             another exception is in flight
Exception from thread with no catch         std::thread([] { throw X; })
Violated std::unexpected (pre-C++17)         throw() spec violated
std::nested_exception::rethrow_nested()      called on null exception_ptr
std::rethrow_exception(nullptr)              bad exception_ptr
```

### `terminate_handler` API

```cpp
#include <exception>

// Type: void (*)() noexcept
using terminate_handler = void (*)() noexcept;

// Install a custom handler - returns the previous one
std::terminate_handler old = std::set_terminate(my_handler);

// Get current handler
std::terminate_handler current = std::get_terminate();
```

### Important Rules

| Rule | Explanation |
| --- | --- |
| Handler must not return | If it returns, `std::abort()` is called |
| Handler must not throw | It's `noexcept` - throwing calls `std::abort()` |
| Handler should call `std::abort()` | Or `std::_Exit()` to terminate the process |
| One handler per program | Global - last `set_terminate()` call wins |

---

## Self-Assessment

### Q1: Install a terminate handler that logs the exception type before calling abort

**Solution - Diagnostic Terminate Handler:**

The most valuable thing a terminate handler can do is tell you what exception was in flight when the program died. `std::current_exception()` gives you access to it even from inside the handler. Note the unconditional `std::abort()` at the end - the handler must never return:

```cpp
#include <iostream>
#include <exception>
#include <cstdlib>
#include <typeinfo>

#if __has_include(<cxxabi.h>)
  #include <cxxabi.h>
#endif

// Custom terminate handler that logs diagnostics
void my_terminate_handler() noexcept {
    std::cerr << "=== FATAL: std::terminate() called ===\n";

    // Try to get the current exception (if any)
    if (auto eptr = std::current_exception()) {
        try {
            std::rethrow_exception(eptr);
        } catch (const std::exception& e) {
            std::cerr << "  Exception type: " << typeid(e).name() << "\n";
            std::cerr << "  what(): " << e.what() << "\n";
        } catch (const char* msg) {
            std::cerr << "  C-string exception: " << msg << "\n";
        } catch (int code) {
            std::cerr << "  Integer exception: " << code << "\n";
        } catch (...) {
            std::cerr << "  Unknown exception type\n";
        }
    } else {
        std::cerr << "  No active exception\n";
    }

    // MUST terminate - handler cannot return
    std::abort();
}

void dangerous_function() noexcept {
    // BUG: throwing from noexcept -> calls std::terminate()
    throw std::runtime_error("noexcept violation!");
}

int main() {
    // Install our handler
    std::terminate_handler old_handler = std::set_terminate(my_terminate_handler);
    std::cout << "Custom terminate handler installed.\n";

    // This will trigger std::terminate because of noexcept violation
    dangerous_function();

    // Never reached
    return 0;
}
// Expected output:
//   Custom terminate handler installed.
//   === FATAL: std::terminate() called ===
//     Exception type: St13runtime_error    (mangled name, platform-dependent)
//     what(): noexcept violation!
//   (program aborts)
```

**Production-quality handler with file logging:**

In a real application you typically want crash details written to a file that survives the process. Here the key concern is keeping allocations minimal - the heap may be in an inconsistent state when `terminate` fires:

```cpp
#include <iostream>
#include <fstream>
#include <exception>
#include <cstdlib>
#include <ctime>

void production_terminate_handler() noexcept {
    // Write to a crash log file
    // Note: limited operations are safe in a terminate handler
    // Avoid allocations; use pre-opened file descriptors if possible
    std::ofstream log("crash.log", std::ios::app);
    if (log) {
        std::time_t now = std::time(nullptr);
        log << "\n=== CRASH at " << std::ctime(&now);

        if (auto eptr = std::current_exception()) {
            try {
                std::rethrow_exception(eptr);
            } catch (const std::exception& e) {
                log << "Exception: " << e.what() << "\n";
            } catch (...) {
                log << "Unknown exception\n";
            }
        }
    }
    std::abort();  // generates core dump on POSIX
}
```

---

### Q2: Explain that `std::unexpected` was removed in C++17 and what replaced it

**History and Evolution:**

```cpp
C++98/03                          C++11                    C++17
-----------------                 ----------------         ----------------
Dynamic exception specs:          Deprecated:              REMOVED:
  void f() throw(X, Y);            throw()                  throw(X, Y)
  void g() throw();                 throw(X, Y)             std::unexpected()
  -> violating calls                                         std::unexpected_handler
    std::unexpected()              Added:                    std::set_unexpected()
                                    noexcept
                                    noexcept(expr)         Kept:
                                                             noexcept
                                                             noexcept(expr)
                                                             std::terminate()
```

**Why Dynamic Exception Specs Failed:**

| Problem | Explanation |
| --- | --- |
| **Runtime overhead** | Compiler must check thrown exception types at runtime |
| **Not checked at compile time** | Unlike Java, violations are detected only at runtime |
| **Unexpected behavior** | Violating `throw(X)` calls `std::unexpected()` -> `std::terminate()` |
| **Composability** | Templates can't express "throws whatever T throws" |
| **Performance** | Pessimizes optimization - compiler can't assume no exception |

The reason this trips people up is the three-step chain in the old model: a `throw()` violation did not go directly to `terminate` - it went through `unexpected()` first, which could itself throw, which could then reach `terminate`. This was confusing and hard to reason about. `noexcept` cuts straight to `terminate` with no intermediate step.

What replaced them:

```cpp
// C++98: dynamic exception specification (REMOVED in C++17)
void old_style() throw(std::runtime_error, std::logic_error);  // removed

// C++98: no-throw specification (evolved)
void old_nothrow() throw();  // deprecated in C++11, removed in C++20

// C++11+: noexcept (the modern replacement)
void modern_nothrow() noexcept;              // never throws
void conditional() noexcept(sizeof(int) > 2); // conditionally noexcept

// C++11+: for functions that DO throw -> no annotation needed
void may_throw();  // implicitly "can throw anything"
```

`std::unexpected` vs `std::terminate`:

```cpp
Pre-C++17:
  throw(X) violated -> std::unexpected() -> by default calls std::terminate()

C++17+:
  noexcept violated -> std::terminate() directly

  std::unexpected()        -> REMOVED
  std::unexpected_handler  -> REMOVED
  std::set_unexpected()    -> REMOVED
```

---

### Q3: Show why terminate handlers should not throw and what happens if they do

**Solution - Throwing from a Terminate Handler:**

The reason you cannot escape `terminate` by throwing is that the handler itself is declared `noexcept`. Any attempt to throw from it immediately triggers `std::abort()` - there is no recovery path. The same applies if the handler simply returns without calling `abort` or `_Exit`:

```cpp
#include <iostream>
#include <exception>
#include <cstdlib>

// BAD: terminate handler that throws
void bad_terminate_handler() noexcept {
    std::cerr << "bad_terminate_handler entered\n";

    // This throw violates noexcept -> calls std::abort() immediately
    // You can't "escape" terminate by throwing!
    throw std::runtime_error("trying to escape");  // undefined behavior / abort

    std::cerr << "This line never reached\n";
}

// BAD: terminate handler that returns
void returning_handler() noexcept {
    std::cerr << "returning_handler: I'll just return...\n";
    return;  // if handler returns, std::abort() is called
}

// GOOD: proper terminate handler
void good_terminate_handler() noexcept {
    std::cerr << "good_terminate_handler: logging and aborting\n";

    // Log diagnostics...
    if (auto eptr = std::current_exception()) {
        try {
            std::rethrow_exception(eptr);
        } catch (const std::exception& e) {
            std::cerr << "  Exception: " << e.what() << "\n";
        } catch (...) {
            std::cerr << "  Unknown exception\n";
        }
    }

    std::abort();  // always terminate
}

int main() {
    // Demonstrate the constraint:
    // terminate_handler signature is: void() noexcept
    //
    // If it throws -> noexcept is violated -> std::abort() called
    // If it returns -> std::abort() called
    // Only valid actions: log + abort/exit/_Exit

    std::set_terminate(good_terminate_handler);

    // Trigger terminate via noexcept violation
    []() noexcept {
        throw std::runtime_error("boom");
    }();
}
// Expected output:
//   good_terminate_handler: logging and aborting
//     Exception: boom
//   (abort)
```

**Summary of terminate handler rules:**

```cpp
+--------------------------------------------------------+
| Terminate Handler Rules                                |
+--------------------------------------------------------+
| Yes: Log exception info via current_exception()        |
| Yes: Write to log file (pre-opened if possible)        |
| Yes: Call std::abort() or std::_Exit()                 |
| Yes: Flush important buffers                           |
|                                                        |
| No: Do NOT throw (noexcept -> abort)                   |
| No: Do NOT return (-> abort)                           |
| No: Do NOT allocate heap memory (may fail)             |
| No: Do NOT call complex functions (may throw)          |
| No: Do NOT try to "recover"                            |
+--------------------------------------------------------+
```

---

## Notes

- **`std::abort()` vs `std::_Exit()`:** `abort()` may generate a core dump (useful for debugging). `_Exit()` terminates without cleanup (no destructors, no atexit handlers).
- **Thread safety:** `std::set_terminate()` is not thread-safe - install your handler early in `main()` before spawning threads.
- **`std::current_exception()` in terminate handler** works when terminate was called due to an uncaught exception - it returns the active exception.
- **Signal handlers vs terminate handlers:** Terminate handlers have fewer restrictions than signal handlers but should still avoid non-trivial operations.
- **`noexcept` replaced `throw()`:** The C++11 `noexcept` specifier is the modern way to declare non-throwing functions, and its violation goes directly to `std::terminate()` - cleaner than the old `throw()` -> `unexpected()` -> `terminate()` chain.
