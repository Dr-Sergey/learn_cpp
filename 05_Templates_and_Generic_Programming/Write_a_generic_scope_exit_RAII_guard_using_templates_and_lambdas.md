# Write a Generic scope_exit RAII Guard Using Templates and Lambdas

**Category:** Templates and Generic Programming  
**Standard:** C++11, C++17 (basic)  
**Reference:** <https://en.cppreference.com/w/cpp/language/raii>  

---

## Topic Overview

### What Is `scope_exit`

A **scope_exit** guard is an RAII wrapper that executes a cleanup function when it goes out of scope, **regardless of how the scope exits** (normal return, exception, `break`, etc.).

The nice part is how little it takes to use it. You write the cleanup once and put it at the top of the function, next to the resource acquisition. No matter how many early returns or exception paths you add later, the cleanup runs automatically.

```cpp
{
    auto guard = scope_exit([&] { cleanup(); });
    // ... do work ...
    // cleanup() is called automatically when guard is destroyed
}
```

### The Three Scope Guards

| Guard | When Cleanup Runs | Use Case |
| --- | --- | --- |
| `scope_exit` | **Always** - normal exit and exception | General cleanup (close file, release lock) |
| `scope_success` | **Only on normal exit** (no new exception) | Commit transaction |
| `scope_fail` | **Only on exception exit** | Rollback transaction |

### How `std::uncaught_exceptions()` Enables scope_success/scope_fail

`std::uncaught_exceptions()` (plural, added in C++17) returns a count of how many exceptions are currently in flight. By snapshotting the count at construction and comparing it at destruction, you can tell whether the scope ended normally or was unwound by an exception.

```cpp
Constructor:  saved_count_ = std::uncaught_exceptions()  // e.g., 0

Destructor:   current_count = std::uncaught_exceptions()
              if (current_count > saved_count_)  -> scope is exiting via exception
              if (current_count == saved_count_) -> scope is exiting normally
```

---

## Self-Assessment

### Q1: Implement `scope_exit<F>` that calls `F` on destruction regardless of how the scope exits

There are two subtleties to get right here. First, the destructor must catch and swallow any exception thrown by the cleanup function, because throwing from a destructor during stack unwinding terminates the program. Second, a `release()` method lets callers cancel the cleanup - useful when you only want cleanup on failure.

```cpp
#include <iostream>
#include <utility>
#include <stdexcept>

// === scope_exit: calls cleanup on destruction, always ===
template <typename F>
class scope_exit {
    F func_;
    bool active_;

public:
    explicit scope_exit(F&& f) noexcept
        : func_(std::move(f)), active_(true) {}

    explicit scope_exit(const F& f) noexcept
        : func_(f), active_(true) {}

    // Move constructor - transfers ownership
    scope_exit(scope_exit&& other) noexcept
        : func_(std::move(other.func_)), active_(other.active_) {
        other.release();
    }

    // Not copyable
    scope_exit(const scope_exit&) = delete;
    scope_exit& operator=(const scope_exit&) = delete;
    scope_exit& operator=(scope_exit&&) = delete;

    ~scope_exit() {
        if (active_) {
            try { func_(); } catch (...) {}  // never throw from destructor
        }
    }

    // Disable the guard (cancel cleanup)
    void release() noexcept { active_ = false; }
};

// Deduction guide (C++17)
template <typename F>
scope_exit(F) -> scope_exit<F>;

// Helper factory function (works in C++11/14 too)
template <typename F>
auto make_scope_exit(F&& f) {
    return scope_exit<std::decay_t<F>>(std::forward<F>(f));
}

void example_normal_exit() {
    std::cout << "--- Normal exit ---\n";
    auto guard = scope_exit([&] {
        std::cout << "  Cleanup executed (normal exit)\n";
    });
    std::cout << "  Doing work...\n";
    // guard destroyed here -> cleanup runs
}

void example_exception_exit() {
    std::cout << "\n--- Exception exit ---\n";
    try {
        auto guard = scope_exit([&] {
            std::cout << "  Cleanup executed (exception exit)\n";
        });
        std::cout << "  Doing work...\n";
        throw std::runtime_error("oops");
        // guard destroyed during stack unwinding -> cleanup still runs
    } catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
    }
}

void example_early_return(bool condition) {
    std::cout << "\n--- Early return (condition=" << std::boolalpha << condition << ") ---\n";
    auto guard = scope_exit([&] {
        std::cout << "  Cleanup executed (early return)\n";
    });
    if (condition) {
        std::cout << "  Returning early\n";
        return;  // guard destroyed -> cleanup runs
    }
    std::cout << "  Reached end\n";
    // guard destroyed -> cleanup runs
}

void example_release() {
    std::cout << "\n--- Release (cancel cleanup) ---\n";
    auto guard = scope_exit([&] {
        std::cout << "  Cleanup executed (SHOULD NOT APPEAR)\n";
    });
    std::cout << "  Releasing guard...\n";
    guard.release();  // cancel the cleanup
    // guard destroyed but inactive -> cleanup does NOT run
    std::cout << "  Guard was released, cleanup skipped\n";
}

int main() {
    example_normal_exit();
    example_exception_exit();
    example_early_return(true);
    example_early_return(false);
    example_release();
    return 0;
}
```

**Expected output:**

```text
--- Normal exit ---
  Doing work...
  Cleanup executed (normal exit)

--- Exception exit ---
  Doing work...
  Cleanup executed (exception exit)
  Caught: oops

--- Early return (condition=true) ---
  Returning early
  Cleanup executed (early return)

--- Early return (condition=false) ---
  Reached end
  Cleanup executed (early return)

--- Release (cancel cleanup) ---
  Releasing guard...
  Guard was released, cleanup skipped
```

### Q2: Show that `scope_exit` replaces ad-hoc cleanup in functions with multiple return paths

The classic problem with manual cleanup is that every `return` statement needs its own copy of the cleanup code. Add a new early-return path six months later and it is easy to forget to add the `fclose`. `scope_exit` collapses all of that into a single cleanup point declared once.

```cpp
#include <iostream>
#include <cstdio>
#include <stdexcept>
#include <utility>

template <typename F>
class scope_exit {
    F func_;
    bool active_ = true;
public:
    explicit scope_exit(F f) : func_(std::move(f)) {}
    ~scope_exit() { if (active_) { try { func_(); } catch (...) {} } }
    void release() noexcept { active_ = false; }
    scope_exit(scope_exit&& o) noexcept : func_(std::move(o.func_)), active_(o.active_) { o.release(); }
    scope_exit(const scope_exit&) = delete;
};
template <typename F> scope_exit(F) -> scope_exit<F>;

// BAD: Manual cleanup with multiple return paths
bool process_file_bad(const char* filename) {
    FILE* f = fopen(filename, "r");
    if (!f) return false;

    char buf[256];
    if (!fgets(buf, sizeof(buf), f)) {
        fclose(f);               // cleanup #1
        return false;
    }

    if (buf[0] == '#') {
        fclose(f);               // cleanup #2 (duplicated!)
        return false;
    }

    // process...
    fclose(f);                   // cleanup #3 (duplicated again!)
    return true;
}

// GOOD: scope_exit eliminates duplication
bool process_file_good(const char* filename) {
    FILE* f = fopen(filename, "r");
    if (!f) return false;

    auto guard = scope_exit([&] {
        std::cout << "  [guard] Closing file\n";
        fclose(f);               // single cleanup point - always runs
    });

    char buf[256];
    if (!fgets(buf, sizeof(buf), f)) {
        std::cout << "  [early return] Failed to read\n";
        return false;            // guard closes file automatically
    }

    if (buf[0] == '#') {
        std::cout << "  [early return] Comment line\n";
        return false;            // guard closes file automatically
    }

    std::cout << "  [success] Processed file\n";
    return true;                 // guard closes file automatically
}

// === Multiple resources ===
bool multi_resource_example() {
    std::cout << "\n--- Multiple resources ---\n";

    int* data = new int[100];
    auto g1 = scope_exit([&] {
        std::cout << "  [guard1] delete[] data\n";
        delete[] data;
    });

    FILE* log = fopen("log.txt", "w");
    if (!log) return false;
    auto g2 = scope_exit([&] {
        std::cout << "  [guard2] fclose(log)\n";
        fclose(log);
    });

    // Guards are destroyed in reverse order: g2 first, then g1
    // This matches the resource acquisition order (LIFO)
    std::cout << "  [work] Using both resources...\n";
    return true;
}

int main() {
    std::cout << "=== scope_exit vs manual cleanup ===\n\n";
    std::cout << "--- process_file_good ---\n";
    process_file_good("test.txt");

    multi_resource_example();

    std::cout << "\n=== Benefits ===\n";
    std::cout << "1. Single cleanup point - no duplication\n";
    std::cout << "2. Exception-safe by construction\n";
    std::cout << "3. Resources released in reverse order (LIFO)\n";
    std::cout << "4. Works with any cleanup: close, unlock, free, rollback\n";

    return 0;
}
```

### Q3: Explain how `scope_success` and `scope_fail` guards differ using `std::uncaught_exceptions`

`std::uncaught_exceptions()` (C++17) returns the **number of uncaught exceptions currently in flight**. By comparing the count at construction vs destruction, you can tell whether a scope exited normally or via exception.

The reason this needs to be a count rather than a boolean - the old `std::uncaught_exception()` returned a bool - is that exception handlers can be nested. If your code is already inside a `catch` block and throws again, the count goes from 1 to 2. The old boolean API could not distinguish those two situations, making `scope_fail` unreliable in nested contexts. The count fixes that.

```cpp
#include <iostream>
#include <exception>
#include <utility>
#include <stdexcept>

// === scope_exit: always runs ===
template <typename F>
class scope_exit {
    F func_;
    bool active_ = true;
public:
    explicit scope_exit(F f) : func_(std::move(f)) {}
    ~scope_exit() { if (active_) try { func_(); } catch (...) {} }
    void release() noexcept { active_ = false; }
};
template <typename F> scope_exit(F) -> scope_exit<F>;

// === scope_success: runs ONLY on normal exit ===
template <typename F>
class scope_success {
    F func_;
    int uncaught_count_;

public:
    explicit scope_success(F f)
        : func_(std::move(f))
        , uncaught_count_(std::uncaught_exceptions()) {}

    ~scope_success() {
        // If no NEW exceptions since construction -> normal exit
        if (std::uncaught_exceptions() == uncaught_count_) {
            try { func_(); } catch (...) {}
        }
    }
};
template <typename F> scope_success(F) -> scope_success<F>;

// === scope_fail: runs ONLY on exception exit ===
template <typename F>
class scope_fail {
    F func_;
    int uncaught_count_;

public:
    explicit scope_fail(F f)
        : func_(std::move(f))
        , uncaught_count_(std::uncaught_exceptions()) {}

    ~scope_fail() noexcept {
        // If MORE exceptions now than at construction -> exception exit
        if (std::uncaught_exceptions() > uncaught_count_) {
            try { func_(); } catch (...) {}
        }
    }
};
template <typename F> scope_fail(F) -> scope_fail<F>;

// === Transaction example ===
void process_order(bool should_fail) {
    std::cout << "--- process_order(fail=" << std::boolalpha << should_fail << ") ---\n";

    int order_id = 42;

    // Always log
    auto log_guard = scope_exit([&] {
        std::cout << "  [scope_exit] Logging order #" << order_id << " attempt\n";
    });

    // Commit only on success
    auto commit_guard = scope_success([&] {
        std::cout << "  [scope_success] Committing order #" << order_id << "\n";
    });

    // Rollback only on failure
    auto rollback_guard = scope_fail([&] {
        std::cout << "  [scope_fail] Rolling back order #" << order_id << "\n";
    });

    std::cout << "  Processing order #" << order_id << "...\n";
    if (should_fail) {
        throw std::runtime_error("payment declined");
    }
    std::cout << "  Order processed successfully\n";
}

int main() {
    // Successful order
    process_order(false);
    // Output:
    //   Processing order #42...
    //   Order processed successfully
    //   [scope_success] Committing order #42     <- runs (normal exit)
    //   [scope_exit] Logging order #42 attempt   <- runs (always)
    //   scope_fail does NOT run

    std::cout << "\n";

    // Failed order
    try {
        process_order(true);
    } catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
    }
    // Output:
    //   Processing order #42...
    //   [scope_fail] Rolling back order #42      <- runs (exception exit)
    //   [scope_exit] Logging order #42 attempt   <- runs (always)
    //   scope_success does NOT run

    return 0;
}
```

Here is a plain-language summary of how the count is used:

```cpp
Constructor:  saved = std::uncaught_exceptions()   // typically 0

Normal exit:
  Destructor: std::uncaught_exceptions() == saved  -> scope_success fires

Exception exit:
  Destructor: std::uncaught_exceptions() > saved   -> scope_fail fires

Nested exceptions:
  If already inside a catch, saved might be 1.
  A new throw makes it 2.
  saved=1, current=2 -> scope_fail correctly detects the new exception.
```

---

## Notes

- `scope_exit` is in the Library Fundamentals TS v3 (`<experimental/scope>`), and expected in a future standard.
- The GSL (`gsl::finally`) provides a similar facility.
- `std::uncaught_exceptions()` (note: plural) was added in C++17 replacing the broken `std::uncaught_exception()` (singular, returns `bool`).
- The singular version couldn't distinguish between nested exception levels - the plural version returns a count.
- Always mark `scope_fail` destructor `noexcept` - it runs during stack unwinding.
- `scope_exit` is one of the most practical RAII patterns - use it for any ad-hoc cleanup.
- Destruction order is reverse of declaration order - declare guards in resource acquisition order.
