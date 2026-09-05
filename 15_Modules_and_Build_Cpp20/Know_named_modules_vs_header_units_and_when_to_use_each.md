# Know Named Modules vs Header Units and When to Use Each

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** [cppreference — Modules](https://en.cppreference.com/w/cpp/language/modules)  

---

## Topic Overview

C++20 gives you two modular mechanisms: **named modules** and **header units**. They serve different roles and have distinct semantic and build-system properties. Picking the wrong one wastes migration effort or leaves performance on the table, so it is worth understanding the difference clearly before you start.

A **named module** is an explicitly declared module (`export module name;`) that defines a clear interface/implementation boundary. It does not export macros, enforces ownership, introduces module linkage, and produces a Binary Module Interface (BMI) that encodes its exported declarations. Named modules are the target end-state for your own code.

A **header unit** is an existing header file imported via `import <header>;` or `import "header";`. The compiler processes the header, caches the result as a BMI, and makes all its declarations - including macros - available. Header units are a **compatibility bridge** - they let you import third-party or standard library headers without rewriting them, while still gaining compilation speed from caching the BMI.

The reason this distinction matters so much is that the two mechanisms have fundamentally different semantics around macros and encapsulation. Named modules are a clean break from the preprocessor era. Header units are a pragmatic step on the way there.

| Property | Named Module | Header Unit |
| --- | --- | --- |
| Declaration syntax | `export module M;` | `import <header>;` / `import "header";` |
| Macros exported | No | Yes |
| Encapsulation (visibility control) | Yes (export vs non-export) | No (everything is visible) |
| Module linkage | Yes | No |
| ODR isolation | Yes (per-module ownership) | Partial (same as headers) |
| Build order dependency | Must compile before importers | Must compile before importers |
| Source modification required | Yes (new `.cppm` files) | No (import existing `.h`) |
| Ideal for | Your own code, first-party libs | Third-party headers, std library |

The semantic difference is profound. Named modules **own** their declarations: the compiler knows that `class Foo` in `module M` belongs to `M`, enabling stronger diagnostics, better encapsulation, and module linkage. Header units merely cache a header's preprocessed output - they carry no ownership semantics and leak all macros.

```text
┌────────────────────────────────────────────────────────────────┐
│                Build System Perspective                        │
│                                                                │
│  Source files ->> Dependency scan ->> Compilation DAG          │
│                                                                │
│  Named Module:  .cppm --scan-->> "provides module M"           │
│                                 "imports module N"             │
│                 Result: compile M.cppm, produce M.bmi          │
│                         then compile importers of M            │
│                                                                │
│  Header Unit:   <vector> -->> compile header unit once          │
│                              produce vector.bmi                │
│                              importers use cached BMI          │
│                                                                │
│  Key difference: header units require NO source changes,       │
│  named modules require new module declaration files.           │
└────────────────────────────────────────────────────────────────┘
```

In practice, the decision matrix is straightforward: use named modules for code you control, and header units for code you don't (or can't yet) modify. That one heuristic covers the vast majority of cases.

---

## Self-Assessment

### Q1: Demonstrate the macro behavior difference between named modules and header units

This is the single most important semantic difference to internalize. Watch what happens to the macros in each approach.

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// ---------- config.h (traditional header) ----------
#pragma once
#define CONFIG_MAX_THREADS 16
#define CONFIG_VERSION "2.0"
inline int get_max_buffer() { return 4096; }

// ---------- Approach A: Header unit ----------
// consumer_a.cpp
import "config.h";               // header unit import

int main() {
    int threads = CONFIG_MAX_THREADS;   // OK: macros ARE exported from header units
    const char* ver = CONFIG_VERSION;   // OK
    int buf = get_max_buffer();         // OK
}

// ---------- Approach B: Named module wrapper ----------
// config_module.cppm
module;
#include "config.h"              // global module fragment
export module config;

export constexpr int max_threads = CONFIG_MAX_THREADS;  // capture macro value
export constexpr const char* version = CONFIG_VERSION;
export using ::get_max_buffer;

// consumer_b.cpp
import config;

int main() {
    int threads = max_threads;          // OK: constexpr
    // CONFIG_MAX_THREADS;              // ERROR: macro NOT available
    const char* ver = version;          // OK
    int buf = get_max_buffer();         // OK
}
```

**Answer:** Header units preserve macro semantics - `CONFIG_MAX_THREADS` is available after `import "config.h";`. Named modules block macros entirely - you must convert them to `constexpr` or function alternatives. This macro isolation is a **feature** of named modules, not a limitation. It forces macro-based APIs to become proper typed constants, which is almost always an improvement.

---

### Q2: When should you choose header units over named modules for a third-party library

The scenario below shows both options for the same library. You will often start with option 1 and move to option 2 only when you need tighter control over the visible API.

```cpp
// ---------- Scenario: Using a header-only library (e.g., nlohmann/json) ----------

// Option 1: Header unit (recommended for third-party)
// consumer.cpp
import <nlohmann/json.hpp>;      // header unit: zero changes to library

void parse_config(const std::string& text) {
    auto j = nlohmann::json::parse(text);
    int val = j["key"].get<int>();
}

// Option 2: Wrapper module (more effort, cleaner interface)
// json_wrapper.cppm
module;
#include <nlohmann/json.hpp>     // in global module fragment
export module json_wrapper;

export namespace json_api {
    using json = nlohmann::json;

    // Expose only what your project needs
    json parse(std::string_view text) {
        return nlohmann::json::parse(text);
    }

    template<typename T>
    T get(const json& j, const std::string& key) {
        return j[key].get<T>();
    }
}
```

Use the decision matrix below to choose between them based on your project's needs.

**Decision matrix:**

| Criterion | Use Header Unit | Use Wrapper Module |
| --- | --- | --- |
| Library update frequency | High (want zero maintenance) | Low (stable API) |
| Macro dependencies | Consumers need the macros | Can replace macros |
| API surface exposure | Full API is fine | Want to restrict visible API |
| Build system maturity | Supports header unit scanning | Supports named modules |
| Compilation speed priority | Moderate improvement | Better improvement (smaller BMI) |
| Number of consumers | Few | Many (amortize wrapper cost) |

---

### Q3: How does build system handling differ between named modules and header units, and what are the pitfalls

Both mechanisms require P1689 dependency scanning and both produce BMIs that importers must wait for. The differences show up in how much build system configuration you need and which failure modes to watch for.

```cpp
// ---------- CMake example showing both ----------
// CMakeLists.txt
cmake_minimum_required(VERSION 3.28)
project(mixed_example CXX)
set(CMAKE_CXX_STANDARD 20)

# Named module library
add_library(core_module)
target_sources(core_module
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/core.cppm
            src/core-types.cppm
    PRIVATE
        src/core_impl.cpp
)

# Application using both named modules and header units
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE core_module)

# For header units, compiler flags may be needed:
# MSVC:   /headerUnit:angle <vector>=vector.ifc
# GCC:    -fmodule-header=<vector>
# Clang:  -fmodule-header <vector>
```

The consumer can mix both styles freely within the same file, though not all headers work reliably as header units.

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
// src/main.cpp
import core;                     // named module - CMake handles compilation order
import <vector>;                 // header unit - requires compiler/build system support
import <string>;                 // header unit
#include <iostream>              // still a regular include (not all headers work as HU)
#include <vector>

int main() {
    std::vector<core::Item> items = core::load_items();
    for (const auto& item : items) {
        std::cout << item.name << '\n';  // iostream often problematic as header unit
    }
}
```

**Build system pitfalls:**

| Pitfall | Named Modules | Header Units |
| --- | --- | --- |
| Dependency scanning | Required (P1689) | Required (P1689) |
| Circular dependencies | Impossible (by design) | N/A (same as headers) |
| Compilation ordering | Build system must respect DAG | Must compile HU before importers |
| BMI compatibility | ABI-sensitive, compiler-version locked | Same issue |
| Parallel build impact | Reduces parallelism on DAG edges | Minimal (few unique HUs) |
| `<iostream>` as HU | N/A | Often fails - too many macros/globals |

**Key pitfall:** Not all standard library headers work well as header units. Headers with heavy macro usage, `#pragma` dependencies, or complex conditional compilation may fail. Test each header unit individually before committing to it in production.

---

## Notes

- **Named modules** = your code. **Header units** = other people's code. This heuristic covers 90% of cases.
- Header units export macros; named modules do not. This is the single most important semantic difference.
- Header units are compiled once and cached as BMIs - this alone can significantly speed up builds that include the same heavy headers in many TUs.
- Standard library header units (`import <vector>;`, `import <string>;`) are well-supported on MSVC and improving on GCC/Clang. `import <iostream>;` remains problematic on some implementations.
- The **global module fragment** (`module; #include ... export module M;`) is the mechanism for named modules to consume traditional headers internally.
- Named modules enable **module linkage** - a third linkage kind for encapsulating implementation details. Header units provide no such encapsulation.
- BMI files are **not portable** across compiler versions, optimization levels, or sometimes even flag changes. Build systems must invalidate BMIs when compiler settings change.
- If your build system does not yet support P1689 dependency scanning, header units are significantly easier to adopt than named modules (some compilers can auto-detect them).
- Consider `import std;` (C++23) as the ultimate header-unit replacement for the entire standard library - a single named module for std.
