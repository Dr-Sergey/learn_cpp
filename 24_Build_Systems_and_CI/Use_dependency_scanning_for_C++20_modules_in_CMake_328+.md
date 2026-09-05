# Use dependency scanning for C++20 modules in CMake 3.28+

**Category:** Build Systems and CI  
**Standard:** C++20  
**Reference:** <https://cmake.org/cmake/help/latest/prop_tgt/CXX_SCAN_FOR_MODULES.html>  

---

## Topic Overview

This second file on C++20 module scanning focuses on the **practical end-to-end workflow**: enabling scanning on targets, how CMake 3.28 automatically resolves compilation order, and the details of the two-phase scan+compile pipeline.

If the first module scanning file explained *what* scanning is, this one is about *how it actually runs under the hood*. Understanding the scan+compile pipeline is genuinely useful because it explains why Ninja is required, why you cannot just throw module files into `target_sources()` the old way, and what to look at when a module build goes wrong.

### Why Build Systems Need Scanning

The core problem is that `#include` and `import` work completely differently from the build system's perspective. With `#include`, the compiler opens the header inline - the build system does not need to know anything upfront. With `import`, the build system must compile the module interface first and *then* compile the importer. There is no way to figure this out from file names alone - you have to look inside the source files:

```cpp
Traditional C++ (#include):
  Compiler sees #include "foo.h" -> opens foo.h immediately
  Build system doesn't need to know about header dependencies upfront
  -> Parallel compilation of all .cpp files works fine

C++20 Modules (import):
  import math;  -> needs math.cppm compiled FIRST (produces math.pcm)
  Build system MUST know "main.cpp depends on math module"
  -> Cannot discover this by just looking at file names
  -> Need to SCAN source files to find import/export declarations
```

---

## Self-Assessment

### Q1: Enable CXX_SCAN_FOR_MODULES on a CMake target using C++20 modules

**Answer:**

There are three ways to enable scanning depending on your situation. Method 1 (via `FILE_SET CXX_MODULES`) is the recommended approach for new code because it sets everything up automatically. Method 2 (explicit property) is an escape hatch for unusual cases. Method 3 (global default) is handy when you have many targets all using modules:

```cmake
cmake_minimum_required(VERSION 3.28)
project(ScanDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Method 1: FILE_SET CXX_MODULES (recommended)
# This automatically sets CXX_SCAN_FOR_MODULES=ON
add_library(geometry)
target_sources(geometry
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/geometry.cppm           # export module geometry;
            src/geometry-shapes.cppm    # export module geometry:shapes;
            src/geometry-transform.cppm # export module geometry:transform;
    PRIVATE
        src/geometry_impl.cpp           # module geometry; (implementation)
)

# Method 2: Explicit property (for edge cases)
add_library(legacy_module src/old_module.cppm src/old_impl.cpp)
set_target_properties(legacy_module PROPERTIES
    CXX_SCAN_FOR_MODULES ON          # Enable scanning explicitly
)

# Global default (apply to all targets)
set(CMAKE_CXX_SCAN_FOR_MODULES ON)   # Every target gets scanning

# Consuming target
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE geometry)
# main.cpp can now: import geometry;
```

Here is the module interface and one of its partitions. Notice that the primary interface re-exports the partitions, and the partition files use the `module name:partition` syntax:

```cpp
// src/geometry.cppm
export module geometry;

export import :shapes;       // re-export shapes partition
export import :transform;    // re-export transform partition

export namespace geo {
    constexpr double pi = 3.14159265358979323846;
}
```

```cpp
// src/geometry-shapes.cppm
export module geometry:shapes;

export namespace geo {
    struct Point { double x, y; };
    struct Circle { Point center; double radius; };
    struct Rect { Point top_left; double width, height; };
}
```

After building with `--verbose`, you should see the scan steps explicitly listed before the compilation steps:

```bash
# Verify scanning is active:
cmake -B build -G Ninja
cmake --build build --verbose 2>&1 | grep -i "scan"
# [1/6] Scanning src/geometry.cppm for CXX dependencies
# [2/6] Scanning src/geometry-shapes.cppm for CXX dependencies
# [3/6] Scanning src/geometry-transform.cppm for CXX dependencies
```

### Q2: Show how CMake 3.28 automatically determines module compilation order without manual dependency specification

**Answer:**

This is one of the genuinely nice things about CMake 3.28's module support: you just list the files, and CMake figures out the order. You do not write `add_dependencies()` calls or specify which partition must be compiled before which other partition. The scan results drive all of that automatically. The comments in the first block make this explicit:

```cmake
# CMakeLists.txt — just list the files, CMake figures out the order
add_library(mathlib)
target_sources(mathlib
    PUBLIC FILE_SET CXX_MODULES FILES
        src/math.cppm               # export module math;
        src/math-arithmetic.cppm    # export module math:arithmetic;
        src/math-trig.cppm          # export module math:trig; — imports :arithmetic
)

add_executable(calculator src/main.cpp)  # import math;
target_link_libraries(calculator PRIVATE mathlib)

# You do NOT write:
#   add_dependencies(math.cppm math-arithmetic.cppm)   <- NOT needed
# CMake scans files and discovers the dependency graph automatically
```

Here is what CMake actually does internally when you build. The four-step sequence is worth understanding because it shows why scanning must precede compilation and why Ninja's dynamic dependency support is essential:

```cpp
What CMake does internally:
═══════════════════════════════

Step 1: Generate scan rules (during configure)
  Ninja build.ninja contains:
    rule CXX_SCAN
      command = clang-scan-deps -format=p1689 ...

Step 2: Scan all module sources (during build, phase 1)
  clang-scan-deps math.cppm -> provides: ["math"], requires: [":arithmetic", ":trig"]
  clang-scan-deps math-arithmetic.cppm -> provides: ["math:arithmetic"], requires: []
  clang-scan-deps math-trig.cppm -> provides: ["math:trig"], requires: [":arithmetic"]
  clang-scan-deps main.cpp -> provides: [], requires: ["math"]

Step 3: Ninja collects scan results -> builds DAG:
  math-arithmetic.cppm ──► math-trig.cppm ──► math.cppm ──► main.cpp
       (no deps)           (needs :arithmetic)  (needs partitions)  (needs math)

Step 4: Compile in topological order (phase 2):
  [1] math-arithmetic.cppm -> math-arithmetic.pcm + .o
  [2] math-trig.cppm       -> math-trig.pcm + .o     (uses arithmetic.pcm)
  [3] math.cppm            -> math.pcm + .o           (uses both partitions)
  [4] main.cpp             -> main.o                   (uses math.pcm)
  [5] Link -> calculator
```

### Q3: Explain the two-phase module build: scan phase (extract imports) then compile phase (ordered by deps)

**Answer:**

This is the conceptual heart of C++20 module builds. The reason this two-phase approach works is that scanning is much cheaper than compiling - it only parses `import` and `export` declarations, not the full translation unit. So you can scan all files in parallel, collect the dependency information, and then start a carefully ordered compilation phase. No file is compiled before its dependencies are ready:

```cpp
PHASE 1: DEPENDENCY SCANNING
═════════════════════════════
Purpose: Discover which modules each file imports/exports
Tool: clang-scan-deps (Clang), or compiler-specific scanner
Format: P1689 JSON

For each source file -> produce a .ddi (dep discovery info):

  math.cppm.ddi:
  {
    "rules": [{
      "primary-output": "math.cppm.o",
      "provides": [{"logical-name": "math", "is-interface": true}],
      "requires": [
        {"logical-name": "math:arithmetic"},
        {"logical-name": "math:trig"}
      ]
    }]
  }

  main.cpp.ddi:
  {
    "rules": [{
      "primary-output": "main.cpp.o",
      "provides": [],
      "requires": [{"logical-name": "math"}]
    }]
  }

Scanning is FAST — it only parses import/export lines, not full compilation.
All files can be scanned in parallel (no ordering constraint for scanning).


PHASE 2: ORDERED COMPILATION
═════════════════════════════
Ninja reads all .ddi files -> builds dependency DAG -> compiles in order:

  [parallel] Scan all files -> .ddi
  [serial]   Build DAG from .ddi files
  [ordered]  Compile: arithmetic -> trig -> math -> main
  [final]    Link all .o files

Key insight: Phase 1 (scan) is parallel, Phase 2 (compile) is ordered.
Without scanning, the build system would try to compile main.cpp before
math.pcm exists -> compilation error.
```

**Why Ninja is required (not Make):**

The reason Ninja works and Make does not comes down to a feature called `dyndep` (dynamic dependencies). The build plan for module builds is not fully known at configure time - it depends on what the scan phase discovers. Ninja supports updating its build graph at runtime based on scan outputs. Make requires all dependencies to be known before the build starts:

```cpp
Ninja has "dyndep" (dynamic dependencies):

  - Build rules can declare "I'll discover more dependencies at build time"
  - Scan results are consumed as dyndep files
  - Ninja re-plans the build order after scanning

Make has NO equivalent:

  - All dependencies must be known before build starts
  - Cannot incorporate scan results into build ordering
  - -> Make-based module builds require manual dependency specification
```

This is not a theoretical limitation - in practice it means Make-based builds with C++20 modules require you to manually specify every dependency ordering, which defeats the whole point of having a scanner. Use Ninja.

---

## Notes

- **File extensions:** CMake recognizes `.cppm`, `.ixx`, `.ccm`, `.cxxm` as module interface files when added to `FILE_SET CXX_MODULES`
- **`import std;`** requires CMake 3.30+ with `CMAKE_CXX_MODULE_STD=ON`
- **Debugging:** `cmake --build build --verbose` shows scan and compile steps with commands
- **Performance:** scanning adds ~1-2 seconds for small projects - negligible for medium/large projects where correct ordering prevents failed builds
