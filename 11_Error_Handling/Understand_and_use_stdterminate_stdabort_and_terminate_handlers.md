# Understand and use std::terminate, std::abort, and terminate handlers

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/error/terminate>  

---

## Topic Overview

C++ provides several mechanisms for **abnormal program termination**. Knowing when each is called - and the differences between them - is essential for writing robust error handling and for diagnosing production crashes. Getting this wrong means either losing important cleanup (like flushing files) or, worse, letting a corrupt program keep running.

### Termination Functions Comparison

There are more ways to end a C++ program than most people realize. This table shows which cleanup steps run for each one:

| Function | Destructors Run? | `atexit` Handlers Run? | Signal Raised? | Core Dump? | When Called |
| --- | --- | --- | --- | --- | --- |
| `return` from `main()` | Yes (stack) | Yes | No | No | Normal exit |
| `std::exit(code)` | **Static** only | Yes | No | No | Controlled shutdown |
| `std::quick_exit(code)` | No | `at_quick_exit` only | No | No | Fast shutdown |
| `std::abort()` | **No** | **No** | `SIGABRT` | Yes (if enabled) | Emergency stop |
| `std::terminate()` | **No** (usually) | **No** | Calls handler -> `abort()` | Yes | Unrecoverable error |
| `std::_Exit(code)` | **No** | **No** | No | No | Immediate exit |

The key surprise for many developers is that `std::abort()` and `std::terminate()` skip destructors entirely - your RAII cleanup does not run. That is intentional: these functions are for situations where the program state is so broken that running destructors might make things worse.

### When `std::terminate()` Is Called

`std::terminate()` is not something you call yourself most of the time - the runtime calls it for you when the exception system hits a wall. Here are all the situations where that happens:

```cpp
// std::terminate() is invoked automatically when:

// 1. An exception escapes a noexcept function
void f() noexcept { throw 42; }  // -> terminate()

// 2. An uncaught exception (no matching catch)
int main() { throw 42; }  // -> terminate()

// 3. Exception during stack unwinding (double exception)
~Foo() { throw 42; }  // if already unwinding -> terminate()

// 4. A std::thread destructor with joinable() == true
{ std::thread t(f); }  // -> terminate()

// 5. Exception from std::async / future destructor
//    (implementation-defined)

// 6. Violation of std::unexpected (deprecated in C++17)

// 7. A std::condition_variable::wait with broken invariant

// 8. Calling std::rethrow_exception(nullptr)
```

The double-exception case (item 3) is the one that trips people up most often. If your destructor throws while stack unwinding is already in progress from another exception, C++ has no way to handle two simultaneous exceptions, so it calls `std::terminate()`. This is the main reason destructors should never throw.

### `std::terminate` vs `std::abort`

These two are closely related but not the same. `std::terminate()` calls a handler first (which you can customize), and that handler then typically calls `std::abort()`. The diagram shows the relationship:

```cpp
// std::terminate():
//   Call terminate_handler (customizable!)
//   (default: std::abort)
//
// std::abort():
//   Raise SIGABRT (signal handlers can intercept)
//   NO destructors
//   NO atexit handlers
//   Flush: implementation-defined
//   Core dump (if ulimit allows)
```

The customizable handler in `std::terminate()` is your opportunity to log diagnostics before the process dies. That hook does not exist with a direct call to `std::abort()`.

### Important Notes

- `std::terminate()` calls the **terminate handler**, which defaults to `std::abort()`.
- You can install a custom handler with `std::set_terminate()`.
- `std::abort()` raises `SIGABRT` - you can install a signal handler, but you **must not** return from it (call `_Exit` or `abort` again).
- Destructors of local objects are **not** called by `terminate()` or `abort()` - RAII cleanup is skipped.
- `std::exit()` **does** call destructors of static/thread_local objects and `atexit` handlers.

---

## Self-Assessment

### Q1: Show a case where a `noexcept` function throws and `std::terminate` is invoked

The moment you throw from a `noexcept` function, the compiler does not even attempt stack unwinding. It goes straight to `std::terminate()`. Here is a complete example with a custom handler so you can see exactly what gets reported:

**Solution - `noexcept` Violation:**

```cpp
#include <iostream>
#include <exception>
#include <stdexcept>

// Custom terminate handler - called when terminate() fires
void my_terminate_handler() {
    std::cerr << "=== MY TERMINATE HANDLER ===\n";

    // Try to identify the current exception
    if (auto eptr = std::current_exception()) {
        try {
            std::rethrow_exception(eptr);
        } catch (const std::exception& e) {
            std::cerr << "Exception: " << e.what() << "\n";
        } catch (...) {
            std::cerr << "Unknown exception\n";
        }
    }

    std::abort();  // MUST terminate - cannot continue
}

// noexcept function that throws -> std::terminate()
void risky() noexcept {
    throw std::runtime_error("oops - threw from noexcept!");
    // Compiler knows this is noexcept, so it doesn't generate
    // unwind tables. The throw triggers std::terminate().
}

int main() {
    std::set_terminate(my_terminate_handler);

    std::cout << "About to call risky()...\n";
    risky();  // This calls std::terminate()
    std::cout << "This line is never reached\n";
}
// Expected output:
//   About to call risky()...
//   === MY TERMINATE HANDLER ===
//   Exception: oops - threw from noexcept!
//   (then SIGABRT / core dump)
```

Notice that `std::current_exception()` inside the handler actually returns the exception that caused `terminate()`. That is very handy for logging - you can tell the difference between "threw from noexcept" and "terminate() called directly."

The other common scenarios that trigger `std::terminate()` are worth seeing too:

```cpp
// Scenario 2: Uncaught exception
void uncaught() {
    throw std::logic_error("nobody catches me");
    // main() has no try/catch -> std::terminate()
}

// Scenario 3: Destructor throws during unwinding
struct Bad {
    ~Bad() noexcept(false) {
        throw std::runtime_error("destructor throw");
    }
};

void double_exception() {
    try {
        Bad b;
        throw std::runtime_error("first exception");
        // During unwinding, ~Bad() throws -> terminate()
    } catch (...) {}
}

// Scenario 4: Joinable thread destroyed
#include <thread>
void forgotten_thread() {
    std::thread t([] { /* work */ });
    // t goes out of scope while joinable -> terminate()
}
```

Each of these calls `terminate()` for the same core reason: the exception machinery is in a state from which it cannot recover.

---

### Q2: Install a custom terminate handler that logs a stack trace before aborting

This is the pattern you actually want in production. A good terminate handler logs enough information that you can diagnose the crash after the fact, even without a debugger attached.

**Solution - Custom Terminate Handler with Diagnostics:**

```cpp
#include <iostream>
#include <exception>
#include <cstdlib>
#include <ctime>
#include <string>

#ifdef __has_include
#  if __has_include(<stacktrace>)
#    include <stacktrace>
#    define HAS_STACKTRACE 1
#  endif
#endif

#ifdef __GNUC__
#include <cxxabi.h>   // for demangling on GCC/Clang
#include <execinfo.h> // for backtrace()
#endif

void production_terminate_handler() {
    // 1. Timestamp
    auto now = std::time(nullptr);
    std::cerr << "\n=== FATAL: std::terminate() called at "
              << std::ctime(&now);

    // 2. Current exception info
    if (auto eptr = std::current_exception()) {
        try {
            std::rethrow_exception(eptr);
        } catch (const std::exception& e) {
            std::cerr << "Exception type: " << typeid(e).name() << "\n";
            std::cerr << "What: " << e.what() << "\n";
        } catch (...) {
            std::cerr << "Unknown non-std::exception thrown\n";
        }
    } else {
        std::cerr << "No active exception (terminate called directly)\n";
    }

    // 3. Stack trace (C++23 or platform-specific)
#ifdef HAS_STACKTRACE
    std::cerr << "Stack trace:\n" << std::stacktrace::current() << "\n";
#elif defined(__GNUC__)
    void* frames[64];
    int count = backtrace(frames, 64);
    char** symbols = backtrace_symbols(frames, count);
    std::cerr << "Backtrace (" << count << " frames):\n";
    for (int i = 0; i < count; ++i)
        std::cerr << "  [" << i << "] " << symbols[i] << "\n";
    free(symbols);
#else
    std::cerr << "(Stack trace not available on this platform)\n";
#endif

    // 4. Abort (generates core dump if enabled)
    std::cerr << "=== Aborting ===\n";
    std::abort();
}

void crashy() noexcept {
    throw std::runtime_error("something went very wrong");
}

int main() {
    // Install before any code that might crash
    std::set_terminate(production_terminate_handler);

    crashy();  // triggers our custom handler
}
// Expected output (GCC/Linux):
//   === FATAL: std::terminate() called at Wed Jan  1 12:00:00 2025
//   Exception type: St13runtime_error
//   What: something went very wrong
//   Backtrace (8 frames):
//     [0] ./a.out(+0x1234)
//     [1] ./a.out(+0x5678)
//     ...
//   === Aborting ===
```

**Key Points:**

- `std::set_terminate()` returns the **previous** handler - can be chained.
- The handler **must not return** - it must call `std::abort()`, `std::_Exit()`, or loop forever.
- `std::current_exception()` inside the handler returns the exception that caused `terminate()`, if any.
- Install the handler **as early as possible** in `main()`.

---

### Q3: Explain why `std::abort` is different from `std::exit` in terms of destructors

The distinction matters most when you have important cleanup to do - database connections, temp files, network sockets. Let's watch what each function actually does:

**Complete Comparison:**

```cpp
#include <iostream>
#include <cstdlib>

struct Resource {
    std::string name;
    Resource(const char* n) : name(n) {
        std::cout << "  Construct: " << name << "\n";
    }
    ~Resource() {
        std::cout << "  Destroy: " << name << "\n";
    }
};

Resource global_r("global");  // static storage duration

void at_exit_fn() { std::cout << "  atexit handler called\n"; }

void demo_exit() {
    Resource local_r("local");       // automatic storage
    static Resource static_r("static_local"); // static storage
    std::atexit(at_exit_fn);

    std::cout << "Calling std::exit(0):\n";
    std::exit(0);
    // What happens:
    //   1. atexit handlers run (reverse order)   -> "atexit handler called"
    //   2. Static destructors run                -> "Destroy: static_local"
    //                                            -> "Destroy: global"
    //   3. Local destructors DO NOT RUN          -> "Destroy: local" MISSING!
    //   4. Streams flushed, files closed
}

void demo_abort() {
    Resource local_r("local");
    static Resource static_r("static_local");
    std::atexit(at_exit_fn);

    std::cout << "Calling std::abort():\n";
    std::abort();
    // What happens:
    //   1. SIGABRT raised
    //   2. NO destructors (local OR static)
    //   3. NO atexit handlers
    //   4. Streams NOT guaranteed flushed
    //   5. Core dump (if enabled)
}

int main() {
    // Uncomment ONE:
    // demo_exit();
    // demo_abort();
}
```

**Summary Table:**

| Behavior | `std::exit()` | `std::abort()` | `return` from `main()` |
| --- | --- | --- | --- |
| Local destructors | **No** | **No** | **Yes** |
| Static destructors | **Yes** | **No** | **Yes** |
| `atexit` handlers | **Yes** | **No** | **Yes** |
| Stream flush | **Yes** | **No** (impl-defined) | **Yes** |
| Signal raised | None | `SIGABRT` | None |
| Core dump | No | Yes (if enabled) | No |
| Exit code | Specified | Implementation-defined (usually 134 on Linux) | Specified |

To make the practical consequences concrete, here is the scenario that matters most in real codebases:

```cpp
// Scenario: Application with database connection and temp files

// std::exit():
//   Static DatabaseConnection destructor runs -> clean disconnect
//   atexit handler deletes temp files
//   Local RAII objects (open file handles in current function) leak

// std::abort():
//   Database connection dropped abruptly (corrupted transaction?)
//   Temp files not cleaned up
//   Core dump for post-mortem debugging  (this is the upside)

// return from main():
//   ALL destructors run (local + static)
//   All atexit handlers run
//   Clean shutdown
```

The reason to choose `abort()` over `exit()` is not cleanup - you are giving that up. You choose `abort()` specifically because you want the core dump for post-mortem debugging, and because you suspect the heap or other state is already corrupted so running destructors might cause more damage.

**Best Practices:**

- **Prefer `return` from `main()`** - cleanest shutdown.
- **Use `std::exit()` only** from deep call stacks where returning is impractical (and only if local RAII leaks are acceptable).
- **Use `std::abort()`** for unrecoverable failures where you need a core dump and destructors might fail (e.g., corrupted heap).
- **Never call `std::exit()` from destructors** - leads to infinite recursion if a static destructor calls `exit()`.

---

## Notes

- **`std::quick_exit()`** (C++11) is even faster - runs `at_quick_exit` handlers but no destructors, no `atexit`, no signal. Designed for when `exit()` hangs due to deadlocked destructors.
- **`std::_Exit()`** is the most brutal - no handlers, no destructors, no signals, no flush. Direct kernel exit.
- **`SIGABRT` handler:** You can install a handler for `SIGABRT`, but it **must not return**. Use it only for logging before the process dies.
- **`std::terminate_handler` is thread-safe:** `std::set_terminate` is not thread-safe to call concurrently, but the installed handler can be called from any thread.
- **Compile with `-g`** and enable core dumps (`ulimit -c unlimited` on Linux) to get useful post-mortem debugging from `abort()`.
