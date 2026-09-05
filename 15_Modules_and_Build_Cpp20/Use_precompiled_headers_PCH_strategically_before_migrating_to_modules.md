# Use precompiled headers (PCH) strategically before migrating to modules

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** <https://cmake.org/cmake/help/latest/command/target_precompile_headers.html>  

---

## Topic Overview

**Precompiled headers (PCH)** pre-parse frequently used headers into a binary format, avoiding redundant parsing in every translation unit. They are a **build-speed optimization** for header-based code and a practical stepping stone before full module adoption.

The key thing to understand upfront is that PCH is not a language feature - it's a compiler extension that every major compiler implements, but with slightly different flags and behaviors. It gives you many of the build-speed benefits of modules without requiring you to restructure your code. That's why it's a useful transitional tool.

### PCH vs Modules vs Header Units

| Feature | PCH | Header Units | Modules |
| --- | --- | --- | --- |
| Macro isolation | No | Partial | Yes |
| Parse once | Yes (per target) | Yes (per header) | Yes |
| Include order matters | Yes | No | No |
| Build tool support | Mature | Growing | Growing |
| Standard feature | No (vendor extension) | C++20 | C++20 |
| Migration effort | Low | Medium | High |

### How PCH Works

The idea is simple: instead of parsing `<vector>`, `<string>`, and `<iostream>` once per translation unit, you parse them once, save the result as a binary blob, and then load that blob for each TU. Here's the contrast:

```cpp
Without PCH:
  [a.cpp] -- parse <vector> -- parse <string> -- parse <iostream> -- compile
  [b.cpp] -- parse <vector> -- parse <string> -- parse <iostream> -- compile
  [c.cpp] -- parse <vector> -- parse <string> -- parse <iostream> -- compile
  Total header parsing: 9 headers

With PCH:
  [pch] -- parse <vector> + <string> + <iostream> -- save binary
  [a.cpp] -- load PCH (fast) -- compile
  [b.cpp] -- load PCH (fast) -- compile
  [c.cpp] -- load PCH (fast) -- compile
  Total header parsing: 3 headers (once)
```

The bigger your project, the more dramatic the savings.

---

## Self-Assessment

### Q1: Set up a CMake `target_precompile_headers` for frequently included standard library headers

CMake has had first-class PCH support since version 3.16. You just list the headers you want precompiled and CMake handles generating the PCH and passing the right compiler flags:

```cmake
cmake_minimum_required(VERSION 3.16)  # PCH support added in 3.16
project(MyApp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(app
    src/main.cpp
    src/engine.cpp
    src/renderer.cpp
    src/physics.cpp
)

# Precompile frequently used standard library headers
target_precompile_headers(app PRIVATE
    <vector>
    <string>
    <unordered_map>
    <memory>
    <algorithm>
    <iostream>
    <functional>
    <optional>
    <variant>
)

# You can also precompile project headers
target_precompile_headers(app PRIVATE
    "src/common.h"
    "src/types.h"
)
```

If you have multiple targets that all need the same headers precompiled, you can avoid building the PCH twice by using `REUSE_FROM`:

```cmake
# Create a PCH on one target, reuse on another
add_library(core src/core.cpp)
target_precompile_headers(core PRIVATE <vector> <string> <memory>)

add_executable(app src/main.cpp)
target_precompile_headers(app REUSE_FROM core)  # shares the PCH
```

What CMake generates behind the scenes:

- A header file (`cmake_pch.h` or similar) that includes all listed headers.
- A precompiled binary of that header (`.pch`, `.gch`, or `.pch.cpp.o` depending on compiler).
- Compiler flags to use the PCH for all source files in the target.

### Q2: Measure compile-time improvement from PCH on a large project

Before spending time setting up PCH, it's worth measuring whether it actually helps your specific project. Here's the approach: do a clean build without PCH, then with PCH, and compare:

```bash
# Step 1: Clean build WITHOUT PCH
cmake -B build-no-pch -DCMAKE_CXX_FLAGS="" ..
time cmake --build build-no-pch -j$(nproc) 2>&1 | tail -1

# Step 2: Clean build WITH PCH
cmake -B build-pch ..
time cmake --build build-pch -j$(nproc) 2>&1 | tail -1

# Step 3: Compare
```

For a medium project with 50+ TUs, the improvement is typically substantial:

| Metric | Without PCH | With PCH | Improvement |
| --- | --- | --- | --- |
| Clean build | 120 sec | 45 sec | ~63% faster |
| Incremental (1 file) | 8 sec | 3 sec | ~62% faster |
| PCH generation | - | 5 sec | One-time cost |
| Disk space | - | +50 MB | PCH binary |

To get per-file timing detail, you have two useful options. In CMake, you can instrument every compile command:

```cmake
# Add timing to see per-file compilation time
set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CMAKE_COMMAND} -E time")
```

With Ninja and Clang, you can get even richer data:

```bash
ninja -C build -j1 -d stats  # show detailed timing
# Or use ClangBuildAnalyzer for Clang:
# clang++ -ftime-trace ...  then merge with ClangBuildAnalyzer
```

Guidelines for what belongs in the PCH:

- Headers included by >50% of TUs
- Large headers (STL containers, algorithm)
- Stable headers (rarely change) - PCH rebuilds when any included header changes
- Do NOT include frequently-changing project headers

### Q3: Explain why PCH is a workaround and how modules supersede it for new code

PCH solves the speed problem but leaves all the other problems of headers untouched. Here's an honest accounting of its limitations:

| Problem | Description |
| --- | --- |
| **Not a language feature** | Compiler-specific behavior, not standardized |
| **Fragile** | All TUs must use the same PCH; different compile flags = different PCH |
| **Macro pollution** | All macros from PCH headers leak into every TU |
| **Include order** | PCH must be the first include - order-dependent |
| **One PCH per target** | Can't have different PCH sets for different files |
| **Coarse granularity** | All-or-nothing; can't precompile individual headers |
| **Stale builds** | Changing any header in PCH recompiles everything |

Modules address every one of these systematically:

```cpp
PCH Problem               ->  Module Solution
----------------------------  ----------------------------
Not standardized          ->  ISO C++20 standard feature
Macro pollution           ->  Modules don't export macros
Include order dependency  ->  Imports are order-independent
One PCH per target        ->  Each module compiles independently
Coarse granularity        ->  Fine-grained: import what you need
Stale builds              ->  Only recompile if module interface changes
```

The practical migration timeline looks like this - PCH is Phase 1, not the end state:

```cpp
     NOW                                FUTURE
      |                                   |
      |  Phase 1: Add PCH to existing     |
      |  header-based code (immediate      |
      |  build speed improvement)           |
      |                                   |
      |  Phase 2: Convert stable internal  |
      |  headers to modules                |
      |                                   |
      |  Phase 3: Use import std; (C++23)  |
      |  Remove PCH for std headers        |
      |                                   |
      |  Phase 4: All code uses modules    |
      |  PCH no longer needed              |
      |                                   |
```

The point is that PCH and modules can coexist peacefully. You don't have to choose one or the other right now - use PCH to get fast builds today while gradually moving to modules for new code.

---

## Notes

- PCH support in CMake requires version 3.16+. `REUSE_FROM` requires 3.18+.
- Only put **stable, frequently-used** headers in the PCH. Volatile headers defeat the purpose.
- PCH and modules can coexist: use PCH for legacy code, modules for new code.
- On MSVC, `/Yu` (use PCH) and `/Yc` (create PCH) flags are added automatically by CMake.
- Consider `target_precompile_headers(... REUSE_FROM ...)` to avoid duplicate PCH builds across targets.
