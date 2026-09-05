# Plan Incremental Migration from Headers to Modules

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** [cppreference — Modules](https://en.cppreference.com/w/cpp/language/modules)  

---

## Topic Overview

Migrating a large codebase from headers to C++20 modules is a multi-phase effort that cannot happen atomically. You cannot flip a switch and have it all work. The standard provides **header units** and the **global module fragment** as bridging mechanisms, enabling gradual migration without a flag-day rewrite. A well-planned migration preserves build correctness at every step while progressively reaping module benefits (faster builds, stronger encapsulation, no macro leakage).

The fundamental challenge is **dependency ordering**: modules impose a strict compilation DAG (importers must wait for the imported module's BMI), whereas headers are order-independent at the file level. Migration therefore must proceed **bottom-up** - leaf libraries first, high-level application code last. Attempting top-down migration forces every downstream consumer to change simultaneously, which is usually impractical.

**Migration phases overview:**

| Phase | Action | Mechanism | Risk |
| --- | --- | --- | --- |
| 0 - Audit | Identify macro usage, include cycles, build graph | Static analysis tools | None (read-only) |
| 1 - Header units | Import third-party and stable internal headers as header units | `import <header>;` | Build system must support header unit scanning |
| 2 - Wrapper modules | Create thin module wrappers around existing headers | Global module fragment + `export` | Dual maintenance of wrapper + header |
| 3 - Leaf module conversion | Convert leaf libraries to named modules | Replace `.h`/`.cpp` with `.cppm`/`.cpp` | Consumers must switch to `import` |
| 4 - Propagate upward | Convert mid-level and high-level components | Same as phase 3 | Increasing coordination required |
| 5 - Remove legacy headers | Delete header files, remove include guards | Cleanup | Downstream breakage if external consumers exist |

The **global module fragment** (`module;` ... `export module M;`) is the key migration tool - it lets a module interface include traditional headers before the module declaration, quarantining macro-dependent code outside the module's purview. Think of it as a controlled airlock between the legacy world and the module world.

```cpp
┌──────────────────────────────────────┐
│  Migration Direction (bottom-up)     │
│                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────┐  │
│  │ App Code │->>│ Mid-tier │->>│ Leaf │  │
│  │ (last)   │ │ (middle) │ │(1st) │  │
│  └──────────┘ └──────────┘ └──────┘  │
│                                      │
│  Phase 5 <---- Phase 4 <---- Phase 3 │
└──────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: How do you create a wrapper module around an existing header-based library without modifying the original headers

The global module fragment is designed exactly for this. You put the `#include` inside the fragment so that all the macros and declarations are available to the module body, without any of it leaking to consumers.

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// ---------- Original header: legacy/math_utils.h ----------
#pragma once
#include <cmath>
#include <vector>

#define MATH_PI 3.14159265358979323846

namespace legacy {
    struct Vec2 { double x, y; };
    double length(Vec2 v) { return std::sqrt(v.x*v.x + v.y*v.y); }
    Vec2 normalize(Vec2 v) {
        double len = length(v);
        return {v.x/len, v.y/len};
    }
    std::vector<Vec2> generate_circle(int n, double r);
}

// ---------- Wrapper module: math_utils_mod.cppm ----------
module;                          // global module fragment - preprocessor land

#include "legacy/math_utils.h"   // include here: macros and all

export module math_utils;        // enter module purview

// Re-export selected entities
export namespace legacy {
    using legacy::Vec2;          // export the struct
    using legacy::length;
    using legacy::normalize;
    using legacy::generate_circle;
}

// Wrap the macro as a constexpr (macros don't cross module boundaries)
export constexpr double pi = 3.14159265358979323846;

// ---------- consumer.cpp ----------
import math_utils;

int main() {
    legacy::Vec2 v{3.0, 4.0};
    double len = legacy::length(v);  // OK
    // MATH_PI;                       // ERROR: macros don't travel through modules
    double circumference = 2.0 * pi * len;  // OK: use the constexpr
}
```

**Key takeaway:** The global module fragment absorbs all `#include` dependencies and macros. The `export module` declaration starts the module purview where you selectively re-export what consumers need. Macros must be replaced with `constexpr` or `consteval` alternatives - which forces a cleaner API anyway.

---

### Q2: How do you handle a library that relies heavily on macros during migration

This is where migration gets genuinely tricky. The strategy is to provide a clean module interface for new code while keeping a compatibility header around for code that is not yet migrated. You run both in parallel during the transition period.

```cpp
// ---------- Problem: config.h uses macros consumed by user code ----------
// config.h
#pragma once
#define APP_VERSION_MAJOR 2
#define APP_VERSION_MINOR 5
#define APP_DEBUG_MODE 1
#define APP_LOG(msg) do { if (APP_DEBUG_MODE) std::cerr << msg << '\n'; } while(0)

// ---------- Strategy: provide both macro header and module ----------

// config_module.cppm - module for non-macro parts
export module config;

export struct AppVersion {
    static constexpr int major = 2;
    static constexpr int minor = 5;
};

export constexpr bool debug_mode = true;

export void app_log(const char* msg);  // replace macro with function

// config_module.cpp - implementation
module config;
#include <iostream>

void app_log(const char* msg) {
    if (debug_mode) {
        std::cerr << msg << '\n';
    }
}

// ---------- Transition header: config_compat.h ----------
// For code not yet migrated - include this header
#pragma once
#include "config.h"              // legacy macros still available

// ---------- Migrated consumer ----------
import config;

void startup() {
    app_log("Starting up");      // replaces APP_LOG("Starting up")
    if constexpr (AppVersion::major >= 2) {
        // new feature path
    }
}
```

The table below covers the most common macro patterns and their idiomatic module replacements. Each row is a concrete migration step.

**Migration table for macro patterns:**

| Macro Pattern | Module Replacement | Notes |
| --- | --- | --- |
| `#define CONSTANT 42` | `export constexpr int constant = 42;` | Direct replacement |
| `#define TYPE_ALIAS(x) std::vector<x>` | `export template<typename T> using Alias = std::vector<T>;` | Template alias |
| `#define LOG(msg) ...` | `export void log(std::string_view msg);` | Function (possibly `constexpr`) |
| `#define PLATFORM_WIN 1` | `export constexpr bool platform_win = true;` | Build-system sets value |
| `#define FOREACH(c, body) ...` | Range-based for / ranges | Language feature replacement |

---

### Q3: What does a CMakeLists.txt look like for a mixed header/module project during migration

A real migration project will have native modules, wrapper modules, and legacy header-only libraries all linked together. CMake 3.28 can handle this combination as long as each target is declared correctly.

```cmake
# CMakeLists.txt - mixed migration project
cmake_minimum_required(VERSION 3.28)  # CMake 3.28+ for C++ module support
project(migration_example CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ---------- Legacy header-only library (unchanged) ----------
add_library(legacy_math INTERFACE)
target_include_directories(legacy_math INTERFACE ${CMAKE_SOURCE_DIR}/include)

# ---------- Module wrapper around legacy library ----------
add_library(math_module)
target_sources(math_module
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/modules/math_utils_mod.cppm
)
target_link_libraries(math_module PRIVATE legacy_math)

# ---------- New module-native library ----------
add_library(graphics_module)
target_sources(graphics_module
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/modules/graphics.cppm
            src/modules/graphics-shapes.cppm    # partition
            src/modules/graphics-render.cppm    # partition
    PRIVATE
        src/modules/graphics_impl.cpp
)

# ---------- Application (mixed: some imports, some includes) ----------
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE math_module graphics_module legacy_math)
```

The application source shows what a migration-era file looks like: some things are imported from modules, others are still included from headers that have not been migrated yet.

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// src/main.cpp - mixed consumer
#include "legacy/string_utils.h"  // not yet migrated
import math_utils;                // wrapper module
import graphics;                  // native module

int main() {
    // Use module-imported APIs
    legacy::Vec2 v{1, 0};
    auto n = legacy::normalize(v);

    // Use still-header-based code
    auto trimmed = legacy::trim("  hello  ");  // from string_utils.h
}
```

**Critical CMake notes:** `FILE_SET CXX_MODULES` tells CMake to perform dependency scanning on `.cppm` files. CMake 3.28+ generates the P1689 dependency info and orchestrates compilation order. Modules must be in `PUBLIC` or `INTERFACE` file sets to be importable by dependents.

---

## Notes

- **Always migrate bottom-up**: leaf dependencies first, application code last. Each step must leave the build green.
- **Header units** (`import <vector>;`) are a zero-effort first step - they modularize standard library and third-party headers without source changes.
- **Global module fragment** is the bridge: place `#include` directives before `export module M;` to use headers inside a module.
- **Macros do not cross module boundaries** - this is by design. Replace them with `constexpr`, `consteval`, `inline` functions, or template aliases.
- Keep a **compatibility header** during transition so unmigrated code can still `#include` the old interface.
- Use **feature-test macros** (`__cpp_modules >= 201907L`) in headers that must work both as includes and behind module wrappers.
- Monitor **build times** at each phase - wrapper modules add indirection, but named modules should show compile-time improvements once fully adopted.
- **Do not rush**: a partially migrated codebase with wrapper modules is stable and shippable. Full migration can span multiple release cycles.
- Test with at least two compilers (MSVC, GCC, or Clang) - module support maturity varies and catching portability issues early saves pain.
