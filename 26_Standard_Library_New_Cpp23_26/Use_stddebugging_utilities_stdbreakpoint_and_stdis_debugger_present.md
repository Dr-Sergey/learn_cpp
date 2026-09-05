# Use std::debugging utilities: std::breakpoint and std::is_debugger_present

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/utility/breakpoint>  

---

## Topic Overview

This file focuses on **cross-platform comparison and migration** from vendor-specific debug intrinsics to `std::breakpoint()` and `std::is_debugger_present()`. (See also file #577 for core usage patterns.)

### Platform Intrinsics vs C++26 Standard

| Platform | Breakpoint Intrinsic | Debugger Check | Header |
| --- | --- | --- | --- |
| MSVC | `__debugbreak()` | `IsDebuggerPresent()` | `<windows.h>` |
| GCC | `__builtin_trap()` | ptrace-based check | N/A |
| Clang | `__builtin_debugtrap()` | ptrace-based check / sysctl | N/A |
| Apple | `__builtin_trap()` | `AmIBeingDebugged()` | N/A |
| **C++26** | **`std::breakpoint()`** | **`std::is_debugger_present()`** | `<debugging>` |

### Key Differences

One of the most important things to know before migrating: `__builtin_trap()` is **not** a debug breakpoint. It emits an undefined instruction that terminates the program, and the optimizer is allowed to treat everything after it as dead code. This is a common mistake when porting GCC code:

```cpp
__debugbreak()        -> generates int 3 (x86 debug interrupt) - MSVC only
__builtin_trap()      -> generates ud2 (undefined instruction) - terminates program!
__builtin_debugtrap() -> generates int 3 (Clang only, not GCC)
std::breakpoint()     -> portable, implementation picks the right instruction
```

---

## Self-Assessment

### Q1: Call std::breakpoint() to trigger a debugger breakpoint programmatically

This example shows a side-by-side migration: the old `#ifdef` chain versus the single C++26 call. Pay attention to the GCC comment - the old code had a subtle bug where `__builtin_trap()` would terminate the program rather than pause it:

<!-- compile: needs `debugging` (not in the CI standard library yet) -->
```cpp
#include <debugging>  // C++26
#include <iostream>

// Migration: old platform code -> C++26

// OLD approach (fragile, error-prone):
void old_debug_break() {
#if defined(_MSC_VER)
    __debugbreak();                  // MSVC: int 3
#elif defined(__clang__)
    __builtin_debugtrap();           // Clang: int 3 equivalent
#elif defined(__GNUC__)
    __builtin_trap();                // GCC: ud2 - WARNING: terminates!
    // This is a common bug: __builtin_trap() is NOT a breakpoint,
    // it's an unconditional abort. There is no GCC equivalent to debugtrap.
#else
    #error "No debugger break available"
#endif
}

// NEW approach (C++26 - works everywhere):
void new_debug_break() {
    std::breakpoint();  // Compiler picks the right instruction
    // On x86: int 3
    // On ARM: brk #0xf000
    // On platforms without debugger support: no-op or SIGTRAP
}

int main() {
    int x = compute_something();

    // Use case: conditional breakpoint in production diagnostic mode
    if (x < 0) {
        std::cout << "Unexpected negative value: " << x << '\n';
        std::breakpoint();  // Developer can inspect x in debugger
    }

    std::cout << "Continuing after potential break\n";
}

int compute_something() { return -1; }  // Simulated error
```

### Q2: Use std::is_debugger_present() to enable extra logging only when running under a debugger

Here is a practical pattern: an adaptive logger that automatically upgrades its verbosity when a debugger is attached. You do not need to rebuild or change a config file - the presence of the debugger is enough:

<!-- compile: needs `debugging` (not in the CI standard library yet) -->
```cpp
#include <debugging>  // C++26
#include <iostream>
#include <fstream>
#include <string>

// Adaptive logging: verbose under debugger

enum class LogLevel { Error, Warning, Info, Debug, Trace };

class AdaptiveLogger {
    LogLevel threshold_;
public:
    AdaptiveLogger() {
        // Automatically upgrade log level when debugger is attached
        threshold_ = std::is_debugger_present() ? LogLevel::Trace
                                                 : LogLevel::Warning;
    }

    void log(LogLevel level, const std::string& msg) {
        if (static_cast<int>(level) > static_cast<int>(threshold_))
            return;

        const char* label[] = {"ERROR", "WARN", "INFO", "DEBUG", "TRACE"};
        std::cerr << "[" << label[static_cast<int>(level)] << "] " << msg << '\n';
    }
};

// Expensive validation only under debugger
class SortedContainer {
    std::vector<int> data_;
public:
    void insert(int val) {
        auto it = std::lower_bound(data_.begin(), data_.end(), val);
        data_.insert(it, val);

        // Expensive O(n) validation - only when debugging
        if (std::is_debugger_present()) {
            for (size_t i = 1; i < data_.size(); ++i) {
                if (data_[i] < data_[i-1]) {
                    std::breakpoint();  // Invariant broken - stop here
                }
            }
        }
    }
};

int main() {
    AdaptiveLogger logger;
    logger.log(LogLevel::Info,   "Starting application");
    logger.log(LogLevel::Debug,  "Initializing subsystems");
    logger.log(LogLevel::Trace,  "Memory pool allocated 4096 bytes");

    // Under debugger: all 3 messages print
    // Without debugger: only Error and Warning would print (none here)

    SortedContainer sc;
    sc.insert(3);
    sc.insert(1);
    sc.insert(5);
}
```

The `SortedContainer` example shows a pattern that is hard to achieve cleanly without this API: an expensive invariant check that runs only when you are actively debugging, costing nothing in production.

### Q3: Compare std::breakpoint() with `__debugbreak()` (MSVC) and `__builtin_debugtrap()` (Clang/GCC)

The most important row in this table is the `__builtin_trap()` column - notice how it differs on every key property:

| Feature | `std::breakpoint()` | `__debugbreak()` (MSVC) | `__builtin_debugtrap()` (Clang) | `__builtin_trap()` (GCC) |
| --- | --- | --- | --- | --- |
| **Standard** | C++26 | MSVC extension | Clang extension | GCC/Clang extension |
| **Portable** | Yes | No (MSVC only) | No (Clang only) | Partially |
| **x86 instruction** | `int 3` | `int 3` | `int 3` | `ud2` |
| **Behavior without debugger** | Impl-defined | SIGTRAP / STATUS_BREAKPOINT | SIGTRAP | Terminates program |
| **Can continue after** | Yes (in debugger) | Yes (in debugger) | Yes (in debugger) | No - undefined inst. |
| **Code after is reachable** | Yes | Yes | Yes | No - UB assumed |
| **Optimizer impact** | None | None | None | Treats as `noreturn` |

The optimizer impact of `__builtin_trap()` is the reason this trips people up. Because GCC treats it as `noreturn`, it is allowed to eliminate all code that follows it as unreachable dead code - including the debugging output you were trying to get:

```cpp
// Critical difference: __builtin_trap() vs __builtin_debugtrap()

void test() {
    int x = 42;
    __builtin_trap();      // <- GCC treats as noreturn!
    std::cout << x << '\n'; // <- GCC may REMOVE this code entirely
    // Optimizer assumes __builtin_trap() never returns -> dead code elimination

    // __builtin_debugtrap() (Clang only) and std::breakpoint() do NOT
    // have this problem - code after them is still reachable
}

// Migration checklist:
// 1. Replace __debugbreak()         -> std::breakpoint()
// 2. Replace __builtin_debugtrap()  -> std::breakpoint()
// 3. Replace __builtin_trap()       -> std::breakpoint()  <- BEHAVIOR CHANGE!
//    (trap = terminate, breakpoint = pause)
// 4. Replace IsDebuggerPresent()    -> std::is_debugger_present()
// 5. Replace ptrace(PTRACE_TRACEME) -> std::is_debugger_present()
```

Item 3 in the migration checklist is flagged for good reason: replacing `__builtin_trap()` with `std::breakpoint()` is a behavior change, not just a refactor. If your code was relying on `__builtin_trap()` to terminate on a fatal error, `std::breakpoint()` will pause under a debugger and potentially continue otherwise. Make sure that distinction is intentional.

---

## Notes

- `__builtin_trap()` is **not** a breakpoint - it emits `ud2` (x86) which **terminates** the program; never use it as a debug break.
- `std::breakpoint()` generates the same instruction as `__debugbreak()` / `__builtin_debugtrap()` on each platform.
- `is_debugger_present()` on Linux typically reads `/proc/self/status` for `TracerPid`; on Windows it reads `PEB.BeingDebugged`.
- Both functions have minimal overhead - suitable for use in hot paths behind `if (is_debugger_present())`.
- For conditional-only breaks, prefer `std::breakpoint_if_debugging()` over a manual `if` + `breakpoint` combination.
