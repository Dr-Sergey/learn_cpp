# Use CMake module dependency scanning for C++20 modules

**Category:** Build Systems and CI  
**Standard:** C++20  
**Reference:** <https://cmake.org/cmake/help/latest/prop_tgt/CXX_SCAN_FOR_MODULES.html>  

---

## Topic Overview

C++20 modules break the traditional `#include`-based compilation model in a fundamental way. With headers, the build system can compile all your translation units in parallel because there are no ordering constraints - each `.cpp` file just opens its headers and goes. Modules are different. A module interface unit (`.cppm`) must be compiled to produce a Binary Module Interface file (`.pcm`, `.ifc`, etc.) *before* any file that `import`s it can be compiled. The build system has to figure out that ordering from the source code itself, which means it must actually *read* the source files before it can plan the build. That discovery step is called **module dependency scanning**, and CMake 3.28+ handles it for you automatically.

### Two-Phase Build Model

Here is what changes between a traditional header-based build and a module build. Notice that the module case introduces an explicit scan step that did not exist before:

```cpp
Traditional (#include):
  src1.cpp ──compile──► src1.o ─┐
  src2.cpp ──compile──► src2.o ─┼─► link ──► app
  src3.cpp ──compile──► src3.o ─┘
  (all TUs compile in parallel — no ordering constraint)

C++20 Modules:
  Phase 1: Scan all sources for import/export declarations
    math.cppm   -> exports "math"
    main.cpp    -> imports "math"

  Phase 2: Compile in dependency order
    math.cppm ──compile──► math.pcm + math.o
    main.cpp  ──compile──► main.o   (needs math.pcm)
    link: main.o + math.o ──► app
```

The key insight is that phase 1 is cheap - scanning just reads the `import`/`export` lines, not the full file - and can run entirely in parallel. Phase 2 is the real compilation, and now it can proceed in the correct order because the scan told us what depends on what.

### Compiler Support Status

All three major compilers have solid module support when paired with CMake 3.28+. Here is where things stand:

| Compiler | Module Support | CMake Scanning |
| --- | --- | --- |
| GCC 14+ | Good | CMake 3.28+ |
| Clang 16+ | Good | CMake 3.28+ |
| MSVC 17.4+ | Good | CMake 3.28+ |

---

## Self-Assessment

### Q1: Enable CXX_SCAN_FOR_MODULES in CMake 3.28+ and verify correct build ordering for module TUs

**Answer:**

The key change from a traditional CMake setup is using `FILE_SET CXX_MODULES` to register your module interface files. When you do that, CMake automatically enables scanning on that target. You do not need to set `CXX_SCAN_FOR_MODULES` manually:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.28)    # Module scanning requires 3.28+
project(ModulesDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Create a library with module sources
add_library(mathlib)
target_sources(mathlib
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/math.cppm           # Module interface unit
            src/math_impl.cppm      # Module implementation partition
)
# CXX_SCAN_FOR_MODULES is ON by default for targets with CXX_MODULES file set

# Executable that imports the module
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mathlib)

# Verify scanning is enabled
# CMake automatically sets CXX_SCAN_FOR_MODULES=ON when you use
# FILE_SET CXX_MODULES. You can check explicitly:
get_target_property(scan_enabled mathlib CXX_SCAN_FOR_MODULES)
message(STATUS "Module scanning enabled: ${scan_enabled}")
```

Here is the module interface unit and its implementation partition:

```cpp
// src/math.cppm — Module interface unit
export module math;

export namespace math {
    constexpr double pi = 3.14159265358979323846;

    constexpr double square(double x) { return x * x; }
    constexpr double cube(double x)   { return x * x * x; }
    double circle_area(double radius);
}
```

```cpp
// src/math_impl.cppm — Module implementation unit
module math;

namespace math {
    double circle_area(double radius) {
        return pi * square(radius);
    }
}
```

And the consuming file that just says `import math;`:

<!-- compile: fragment: one translation unit of a multi-file module example -->
```cpp
// src/main.cpp
import math;
#include <iostream>

int main() {
    std::cout << "pi = " << math::pi << '\n';
    std::cout << "5² = " << math::square(5) << '\n';
    std::cout << "Area of circle(r=3) = " << math::circle_area(3) << '\n';
}
```

When you build with Ninja, you will see the dependency-ordered steps in the verbose output - scanning happens before compilation, and the module unit compiles before `main.cpp`:

```bash
# Build with Ninja (recommended for module builds):
cmake -B build -G Ninja
cmake --build build --verbose
# Output shows dependency-ordered compilation:
# [1/4] Scanning src/math.cppm for CXX dependencies
# [2/4] Building CXX object math.cppm.o        <- compiled first
# [3/4] Building CXX object main.cpp.o          <- compiled after math.pcm exists
# [4/4] Linking CXX executable app
```

### Q2: Explain the two-phase build: dependency scanning pass then compilation in dependency order

**Answer:**

This is worth slowing down on because the mechanics here are genuinely new compared to traditional C++ builds. The scanning tool produces a structured JSON file for each source, and Ninja uses those files to dynamically update its build plan before any compilation starts.

```cpp
Phase 1: SCANNING (p1689 format)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
For each source file, the compiler produces a .ddi file (dep info):

  clang-scan-deps -format=p1689 -- clang++ -std=c++20 math.cppm
  -> { "provides": [{"logical-name": "math"}], "requires": [] }

  clang-scan-deps -format=p1689 -- clang++ -std=c++20 main.cpp
  -> { "provides": [], "requires": [{"logical-name": "math"}] }

Phase 2: BUILD GRAPH CONSTRUCTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Ninja/CMake reads all .ddi files and constructs a DAG:

  math.cppm ──(provides "math")──► math.pcm
       ↑                               ↓
       │                         main.cpp (requires "math")
       │                               ↓
       └───────────────────────────── link

Phase 3: COMPILATION (dependency-ordered)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Step 1: Compile math.cppm -> math.o + math.pcm (BMI file)
  Step 2: Compile main.cpp  -> main.o  (uses math.pcm)
  Step 3: Link math.o + main.o -> app
```

Here is the vocabulary you will encounter when working with module builds:

- **BMI (Binary Module Interface):** `.pcm` (Clang), `.ifc` (MSVC), `.gcm` (GCC) - the compiled module interface that importers consume
- **P1689:** The standard JSON format describing what each file provides and requires; this is how the build system understands module dependencies without a custom parser
- **`.ddi`:** Dependency discovery information file, CMake's internal name for the per-file scan output
- **DAG:** Directed acyclic graph - the data structure that determines the correct compilation order

Ninja is the right generator to use here, and the reason matters. Make requires all dependencies to be known at the start; it cannot adjust its build graph midway through. Ninja has a feature called `dyndep` (dynamic dependencies) that lets scan results feed back into the build plan at runtime:

```bash
# Ninja supports dynamic dependencies (dyndep) — can adjust build
# order AFTER reading scan results. Make cannot do this
cmake -B build -G Ninja     # Best for modules
cmake -B build -G "Unix Makefiles"  # Limited module support
```

### Q3: Show the CMakeLists.txt changes needed to add a .cppm module file to a target

**Answer:**

The difference from a traditional setup is that module interface files must go through `FILE_SET CXX_MODULES` rather than plain `target_sources`. That one distinction is what tells CMake to enable scanning and treat those files as module interfaces rather than ordinary source files:

```cmake
# Before modules (traditional headers)
add_library(mathlib
    src/math.cpp
    src/math.h           # header — no special treatment needed
)
target_include_directories(mathlib PUBLIC include/)

# After: adding module interface units
cmake_minimum_required(VERSION 3.28)   # MUST be 3.28+

add_library(mathlib)

# Step 1: Add module interface files via FILE_SET CXX_MODULES
target_sources(mathlib
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/math.cppm                # Primary module interface
            src/math-utils.cppm          # Module partition interface
    PRIVATE
        src/math_impl.cpp               # Regular TU (can import math)
)

# Step 2: That's it! CMake automatically:
#   - Sets CXX_SCAN_FOR_MODULES=ON
#   - Scans for import/export declarations
#   - Builds .cppm files before their dependents
#   - Installs BMI files alongside headers

# Consuming from another target
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mathlib)
# main.cpp can now write: import math;

# Module partitions
# src/math.cppm:
#   export module math;
#   export import :utils;    // re-export partition
#
# src/math-utils.cppm:
#   export module math:utils;
#   export int add(int a, int b) { return a + b; }
#
# Both partition and primary interface go in FILE_SET CXX_MODULES

# Common mistakes
# BAD: Don't add .cppm to target_sources() without FILE_SET:
#     add_library(mathlib src/math.cppm)  # no scanning
#
# BAD: Don't forget cmake_minimum_required(VERSION 3.28):
#     VERSION 3.20 won't enable scanning
#
# BAD: Don't use Makefiles:
#     -G "Unix Makefiles"  # Limited module support — use Ninja
```

The `FILE_SET CXX_MODULES` is the magic incantation. Without it, CMake treats `.cppm` files like any other source file, does no scanning, and your build breaks the moment anything tries to `import` the module before it is compiled.

---

## Notes

- **File extensions:** `.cppm` (Clang convention), `.ixx` (MSVC convention), `.ccm` (GCC) - CMake recognizes all when added via `FILE_SET CXX_MODULES`
- **Header units** (`import <iostream>;`) are not yet well supported in CMake - stick with `#include` for standard headers
- **`CMAKE_CXX_MODULE_STD`** (CMake 3.30+): enables `import std;` support for the standard library module
- **Install:** `install(TARGETS mathlib FILE_SET CXX_MODULES DESTINATION lib/cmake/mathlib)` installs BMI files for downstream consumers
