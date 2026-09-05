# Use std::stacktrace (C++23) for diagnostic stack capture

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/stacktrace>  

---

## Topic Overview

C++23 introduces `std::stacktrace` and `std::stacktrace_entry` (header `<stacktrace>`) for programmatic capture and inspection of call stacks. Before C++23, getting a stack trace required platform-specific APIs (`backtrace()`, `CaptureStackBackTrace`, `libunwind`) or third-party libraries like Boost.Stacktrace. Now it's standardized and works the same way everywhere.

### Key Types

| Type | Description |
| --- | --- |
| `std::stacktrace` | A sequence of `stacktrace_entry` objects representing the current call stack |
| `std::stacktrace_entry` | A single frame: source file, line number, function description |
| `std::stacktrace::current()` | Static method - captures the current stack trace |

### stacktrace_entry Methods

| Method | Return | Description |
| --- | --- | --- |
| `description()` | `std::string` | Function/symbol name (may be mangled or demangled) |
| `source_file()` | `std::string` | Source file path (empty if debug info unavailable) |
| `source_line()` | `uint_least32_t` | Line number (0 if unavailable) |

### Core Example

The simplest use is to call `std::stacktrace::current()` at the point where you want the trace. You get back a range of `stacktrace_entry` objects you can iterate, or you can just stream the whole thing directly.

```cpp
#include <stacktrace>
#include <iostream>
#include <string>

void inner_function() {
    // Capture the current stack trace
    auto trace = std::stacktrace::current();

    std::cout << "Stack trace (" << trace.size() << " frames):\n";
    for (const auto& entry : trace) {
        std::cout << "  " << entry.description();
        if (!entry.source_file().empty())
            std::cout << " at " << entry.source_file() << ":" << entry.source_line();
        std::cout << "\n";
    }

    // Or simply use the stream operator:
    std::cout << "\nFormatted:\n" << trace << "\n";
}

void outer_function() {
    inner_function();
}

int main() {
    outer_function();
}
// Example output (GCC, debug build):
// Stack trace (4 frames):
//   inner_function() at main.cpp:6
//   outer_function() at main.cpp:19
//   main at main.cpp:23
//   __libc_start_main
```

### Embedding in Exceptions

One of the most practical uses of `std::stacktrace` is storing it in a custom exception so you always know where the throw happened - even after the stack has unwound. The trick is to use the same default-parameter pattern as `std::source_location`:

```cpp
#include <stacktrace>
#include <stdexcept>
#include <string>
#include <sstream>
#include <iostream>
#include <utility>

class traced_error : public std::runtime_error {
    std::stacktrace trace_;
public:
    traced_error(const std::string& msg,
                 std::stacktrace trace = std::stacktrace::current())
        : std::runtime_error(msg), trace_(std::move(trace)) {}

    const std::stacktrace& trace() const noexcept { return trace_; }

    std::string full_message() const {
        std::ostringstream oss;
        oss << what() << "\n\nStack trace:\n" << trace_;
        return oss.str();
    }
};

void risky_operation() {
    throw traced_error("file not found: config.yaml");
}

int main() {
    try {
        risky_operation();
    } catch (const traced_error& e) {
        std::cerr << e.full_message() << "\n";
    }
    // Output:
    // file not found: config.yaml
    //
    // Stack trace:
    //  0# risky_operation() at main.cpp:20
    //  1# main at main.cpp:25
    //  ...
}
```

### Skipping Frames

When `stacktrace` is used inside a logging or error-handling helper, the top frame will be the helper itself - which is noise. Pass a skip count to strip those off:

```cpp
#include <stacktrace>
#include <iostream>

void log_with_trace(const std::string& msg) {
    // Skip 1 frame (this function) to start from the caller
    auto trace = std::stacktrace::current(1);  // skip=1
    std::cout << msg << "\n" << trace << "\n";
}

// Also: current(skip, max_depth) to limit capture depth
// auto trace = std::stacktrace::current(0, 5); // top 5 frames only
```

---

## Self-Assessment

### Q1: Capture the current stacktrace in an exception's what() message

**Answer:**

```cpp
#include <stacktrace>
#include <stdexcept>
#include <string>
#include <sstream>
#include <iostream>

class diagnostic_error : public std::runtime_error {
    std::stacktrace trace_;
    std::string full_msg_;

public:
    diagnostic_error(const std::string& msg)
        : std::runtime_error(msg)
        , trace_(std::stacktrace::current(1)) // skip this constructor
    {
        std::ostringstream oss;
        oss << msg << "\n\nCaptured at:\n" << trace_;
        full_msg_ = oss.str();
    }

    const char* what() const noexcept override {
        return full_msg_.c_str();
    }

    const std::stacktrace& where() const noexcept { return trace_; }
};

void database_query(const std::string& sql) {
    throw diagnostic_error("SQL error: syntax error near 'SELCT'");
}

void process_request(int id) {
    database_query("SELCT * FROM users WHERE id=" + std::to_string(id));
}

int main() {
    try {
        process_request(42);
    } catch (const std::exception& e) {
        std::cerr << "CAUGHT:\n" << e.what() << "\n";
    }
    // Output:
    // CAUGHT:
    // SQL error: syntax error near 'SELCT'
    //
    // Captured at:
    //  0# database_query(std::string const&) at main.cpp:22
    //  1# process_request(int) at main.cpp:26
    //  2# main at main.cpp:30
}
```

`std::stacktrace::current(1)` captures the call stack, skipping 1 frame (the constructor itself). The trace is stored as a member and formatted into `what()` during construction. When caught, the exception message includes the complete call chain leading to the error.

### Q2: Print a stacktrace to stderr on SIGSEGV using a signal handler

**Answer:**

```cpp
#include <stacktrace>
#include <iostream>
#include <csignal>
#include <cstdlib>

// WARNING: signal handlers have severe restrictions.
// Most std library functions (including I/O) are NOT async-signal-safe.
// This is for diagnostic/debugging purposes only — NOT production-safe.

void signal_handler(int sig) {
    // Capture stack trace at the point of the signal
    auto trace = std::stacktrace::current();

    // Write to stderr (technically unsafe in signal handler, but
    // useful for crash diagnostics in debug builds)
    std::cerr << "\n=== SIGNAL " << sig << " received ===\n";
    std::cerr << "Stack trace:\n" << trace << "\n";

    // Re-raise to get the default behavior (core dump, etc.)
    std::signal(sig, SIG_DFL);
    std::raise(sig);
}

void dangerous_function(int* ptr) {
    *ptr = 42; // SEGFAULT if ptr is null
}

void caller() {
    int* bad_ptr = nullptr;
    dangerous_function(bad_ptr);
}

int main() {
    // Register signal handler
    std::signal(SIGSEGV, signal_handler);

    caller(); // will trigger SIGSEGV
}
// Expected output (before crash):
// === SIGNAL 11 received ===
// Stack trace:
//  0# signal_handler(int) at main.cpp:12
//  1# dangerous_function(int*) at main.cpp:24
//  2# caller() at main.cpp:29
//  3# main at main.cpp:35
```

**Important caveats:**

- Signal handlers should only call async-signal-safe functions (`write()`, `_exit()`, etc.)
- `std::stacktrace::current()` and `std::cerr` are NOT async-signal-safe
- This approach is useful for **debugging** but not safe for **production** crash handlers
- In production, use platform-specific mechanisms (Linux: `backtrace_symbols_fd`, Windows: `MiniDumpWriteDump`)

### Q3: Explain why std::stacktrace may include or exclude inlined frames depending on optimization

When the compiler inlines a function, it copies the function's code directly into the caller - no actual function call occurs at runtime. Since stack traces are captured at **runtime** by walking the actual call stack (frame pointers or unwind tables), inlined functions have no stack frame to discover. The function's code is there, but it doesn't have its own frame in the call stack any more.

| Optimization | Inlined? | Visible in stacktrace? |
| --- | --- | --- |
| `-O0` (debug) | No - all functions have stack frames | Yes - every function appears |
| `-O1` | Some small functions inlined | Those functions disappear from trace |
| `-O2` / `-O3` | Aggressive inlining | Many functions absent, call chain compressed |
| `-Og` (debug-optimized) | Minimal inlining | Most functions present |

```cpp
#include <stacktrace>
#include <iostream>

// This function will likely be inlined at -O2
inline int add(int a, int b) { return a + b; }

// This function may or may not be inlined
int process(int x) {
    return add(x, 10);
}

void show_trace() {
    auto trace = std::stacktrace::current();
    for (const auto& entry : trace)
        std::cout << "  " << entry.description() << "\n";
}

void middle() {
    show_trace();
}

int main() {
    middle();
}

// At -O0:     main -> middle -> show_trace (all visible)
// At -O2:     main -> show_trace  (middle may be inlined away)
// At -O3:     main only          (everything inlined into main)
```

**What determines visibility:**

1. **Frame pointer (`-fno-omit-frame-pointer`):** Preserving frame pointers helps stack unwinding find all frames. Without it, the unwinder relies on `.eh_frame` / DWARF unwind tables.

2. **Debug info (`-g`):** Doesn't prevent inlining but provides source-level metadata. Some implementations can synthesize inlined frames from DWARF `.debug_line` + `.debug_info`.

3. **Link-time optimization (LTO):** Can inline across translation units, further reducing visible frames.

4. **`__attribute__((noinline))`:** Forces a function to never be inlined, guaranteeing it appears in stack traces.

**Practical advice:**

- For diagnostic builds: compile with `-O0 -g -fno-omit-frame-pointer`
- For production with traces: `-O2 -g -fno-omit-frame-pointer` - keeps debug info even with optimization
- For critical diagnostic functions: mark them `__attribute__((noinline))` or `[[gnu::noinline]]`

---

## Notes

- **Linking:** On GCC/Clang, link with `-lstdc++_libbacktrace` or `-lstdc++exp` for stacktrace support. MSVC includes it automatically.
- **Performance:** `stacktrace::current()` is relatively expensive (microseconds). Don't call it in hot paths. Use it for error reporting, not logging.
- **`current(skip, max_depth)`:** `skip` frames from the top, capture at most `max_depth` frames. Useful for wrapper functions and limiting overhead.
- **Allocator support:** `std::basic_stacktrace<Allocator>` allows custom allocators - useful in embedded/real-time contexts where heap allocation is restricted.
- **`to_string(entry)`:** Individual entries can be converted to strings for custom formatting.
- **Empty traces:** `stacktrace` may be empty if debug information is unavailable or if the platform doesn't support stack unwinding.
- Compile with `-std=c++23 -g -Wall -Wextra` and link with stacktrace library.
