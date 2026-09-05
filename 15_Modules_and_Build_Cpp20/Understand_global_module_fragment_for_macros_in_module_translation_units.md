# Understand global module fragment for macros in module translation units

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/modules>  

---

## Topic Overview

The **global module fragment** is a section at the top of a module interface or implementation unit, between the bare `module;` keyword and the actual module declaration, where you can `#include` legacy headers. This is the **only place** where `#include` is permitted in a module unit while preserving full macro access.

The reason this exists is pragmatic: the real world is full of headers that define macros your code needs - platform headers, C standard library headers, legacy project headers. Modules cannot export macros, and placing `#include` after the module declaration has implementation-defined behavior. The global module fragment is the solution: include your legacy headers there, use what they provide inside the module body, and let the module boundary stop everything from leaking outward.

### Structure of a Module Unit with Global Module Fragment

```cpp
┌──────────────────────────────────────────────┐
│ module;                                      │  <- starts global module fragment
│                                              │
│ #include <cstdio>         // legacy C header │
│ #include "legacy_lib.h"   // project header  │
│ #define MY_MACRO 42       // macros OK here  │
│                                              │
├──────────────────────────────────────────────┤
│ export module my_module;                     │  <- module declaration ends fragment
│                                              │
│ // MY_MACRO is available here                │
│ // But nothing from this section leaks to    │
│ // importers of my_module                    │
│                                              │
│ export int get_value() { return MY_MACRO; }  │
└──────────────────────────────────────────────┘
```

### Why the Global Module Fragment Exists

| Problem | Solution |
| --- | --- |
| Module can't `#include` after module declaration (implementation-defined behavior) | Place `#include` in global module fragment |
| Legacy headers define macros the module code needs | `#include` them in the fragment |
| Don't want macros to leak to importers | Module boundary blocks macro export |

---

## Self-Assessment

### Q1: Use the global module fragment (`module;` before the module declaration) to include legacy headers with macros

Here is a practical logging module that needs `printf`, `assert`, and platform-specific headers. All of those go in the fragment. Notice that `assert` and `printf` are available inside the module body even though nothing from the fragment ever reaches the consumer.

**Module interface: logging.cppm**

```cpp
module;  // <- global module fragment begins

// Include legacy headers that define macros we need
#include <cstdio>      // for printf, FILE*, etc.
#include <cstdlib>     // for EXIT_SUCCESS, EXIT_FAILURE
#include <cassert>     // for assert() macro

// Platform-specific macros
#ifdef _WIN32
#  include <windows.h>  // defines NOMINMAX, WIN32_LEAN_AND_MEAN, etc
#endif

export module logging;  // <- module declaration (ends global module fragment)

// Now we can use macros from the included headers
export enum class LogLevel { Info, Warning, Error };

export void log_message(LogLevel level, const char* msg) {
    // printf from <cstdio> is available because we included it
    // in the global module fragment
    switch (level) {
    case LogLevel::Info:    std::printf("[INFO]    %s\n", msg); break;
    case LogLevel::Warning: std::printf("[WARNING] %s\n", msg); break;
    case LogLevel::Error:   std::printf("[ERROR]   %s\n", msg); break;
    }
}

export [[nodiscard]] int safe_divide(int a, int b) {
    assert(b != 0 && "Division by zero!");  // assert macro works
    return a / b;
}
```

**Consumer: main.cpp**

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
import logging;
#include <iostream>

int main() {
    log_message(LogLevel::Info, "Application started");
    log_message(LogLevel::Warning, "Low memory");
    log_message(LogLevel::Error, "File not found");

    int result = safe_divide(10, 3);
    std::cout << "10 / 3 = " << result << '\n';
}
// Expected output:
// [INFO]    Application started
// [WARNING] Low memory
// [ERROR]   File not found
// 10 / 3 = 3
```

**How this works:**

- `module;` starts the global module fragment. All `#include` directives go here.
- `export module logging;` ends the fragment and begins the module proper.
- Macros like `assert` and functions like `printf` are available **inside** the module but NOT exported to importers.

### Q2: Show that macros defined in the global module fragment do not leak into importers

This example makes the firewall behavior explicit by checking for macro presence in the consumer with `#ifdef`. None of the macros should be visible there.

**Module: config.cppm**

```cpp
module;

// Define macros in global module fragment
#define CONFIG_VERSION 3
#define MAX_BUFFER 1024
#define DEBUG_MODE 1

export module config;

// Use macros internally
export int get_version() { return CONFIG_VERSION; }
export int get_max_buffer() { return MAX_BUFFER; }
export bool is_debug() { return DEBUG_MODE != 0; }

// Even macros defined AFTER module declaration don't leak
#define INTERNAL_DETAIL 999
export int get_detail() { return INTERNAL_DETAIL; }
```

**Consumer: main.cpp**

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
import config;
#include <iostream>

int main() {
    // Functions work - they return values computed from macros
    std::cout << "Version: " << get_version() << '\n';
    std::cout << "Buffer:  " << get_max_buffer() << '\n';
    std::cout << "Debug:   " << std::boolalpha << is_debug() << '\n';
    std::cout << "Detail:  " << get_detail() << '\n';

    // But the MACROS themselves are NOT visible:
    #ifdef CONFIG_VERSION
        std::cout << "CONFIG_VERSION leaked!\n";  // will NOT print
    #else
        std::cout << "CONFIG_VERSION not visible (correct!)\n";
    #endif

    #ifdef MAX_BUFFER
        std::cout << "MAX_BUFFER leaked!\n";  // will NOT print
    #else
        std::cout << "MAX_BUFFER not visible (correct!)\n";
    #endif

    #ifdef INTERNAL_DETAIL
        std::cout << "INTERNAL_DETAIL leaked!\n";  // will NOT print
    #else
        std::cout << "INTERNAL_DETAIL not visible (correct!)\n";
    #endif
}
// Expected output:
// Version: 3
// Buffer:  1024
// Debug:   true
// Detail:  999
// CONFIG_VERSION not visible (correct!)
// MAX_BUFFER not visible (correct!)
// INTERNAL_DETAIL not visible (correct!)
```

**Key insight:** Modules create a **firewall** for macros. No matter where you define a macro in a module unit - global fragment or after the module declaration - it cannot leak to importers.

### Q3: Explain the difference between `import <header>` and `#include <header>` in terms of macro visibility

This is a subtle but important distinction. Header units (the `import <header>;` form) are not named modules, and they do behave differently with respect to macros.

| Aspect | `#include <header>` | `import <header>;` (header unit) |
| --- | --- | --- |
| Mechanism | Textual insertion | Compiled header unit import |
| Macros | Exported - all macros visible | Exported - macros ARE visible (exception!) |
| Multiple inclusion | Re-parsed each time (unless PCH) | Parsed once, reused |
| Interaction with modules | Works in global module fragment | Works anywhere in module |

**Important exception: Header unit imports DO export macros!**

```cpp
// Unlike named modules, header units preserve macro behavior

// With #include:
#include <cassert>    // assert macro is now defined
assert(true);         // OK

// With import (header unit):
import <cassert>;     // assert macro is ALSO defined!
assert(true);         // OK - header units export macros

// With named module:
import my_module;     // macros from my_module are NOT defined
// assert(true);      // ERROR if only imported via named module
```

The reason header units export macros is intentional: they are a compatibility bridge, not a clean break from the preprocessor. Their whole purpose is to let you import existing headers without rewriting them, and those headers may rely on macro definitions being visible to consumers.

**Comparison in a module context:**

```cpp
module;
#include <cassert>       // Method 1: include in global fragment
export module example;

// vs.

export module example;
import <cassert>;        // Method 2: import header unit

// Both make assert() available inside the module.
// But importers of "example" see different behavior:
//   - #include in global fragment: assert NOT visible to importers
//   - import <cassert> in module: assert depending on implementation
```

**Rule of thumb:**

- Use `#include` in the **global module fragment** for macros needed inside the module.
- Use `import <header>;` for **header units** when you want a faster, module-like include.
- Named modules **never** export macros - this is by design.

---

## Notes

- The global module fragment can **only** contain preprocessor directives (`#include`, `#define`, `#ifdef`, etc.).
- Code (function definitions, variable declarations) is NOT allowed in the global module fragment.
- The global module fragment is optional - only needed when the module depends on macro-based headers.
- Header units (`import <header>;`) are a middle ground: faster than `#include`, but still export macros.
