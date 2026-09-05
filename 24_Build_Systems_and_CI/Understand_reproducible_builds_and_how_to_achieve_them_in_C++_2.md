# Debug build non-determinism to achieve reproducible C++ builds

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://reproducible-builds.org>  

---

## Topic Overview

This file focuses on **diffoscope-based debugging** of non-determinism and the `-ffile-prefix-map` flag family in depth. While companion files cover the overall reproducibility workflow and SOURCE_DATE_EPOCH, this file digs into how to find and fix the last remaining sources of non-determinism.

The reason this matters is that once you have applied the obvious fixes, there are often a few subtle sources of non-determinism left over. `diffoscope` is the tool that makes those visible - it understands binary formats well enough to tell you not just that two binaries differ, but which DWARF section, which archive entry, or which section header is the culprit.

### Non-Determinism Debugging Workflow

Here is the general workflow you follow when chasing down non-determinism. You start with a hash comparison, and if that fails you drill in with `diffoscope` until you identify the specific source:

```cpp
// Debugging flow:
// Build #1 -> binary_a
// Build #2 -> binary_b
//   |
// sha256sum -> different?
//   | yes
// diffoscope binary_a binary_b --html report.html
//   |
// Report shows:
//   |-- ELF section .debug_info -> absolute paths -> fix: -ffile-prefix-map
//   |-- ELF section .comment -> compiler build ID -> fix: pin compiler
//   |-- .note.gnu.build-id -> random -> fix: --build-id=sha1
//   `-- archive timestamps -> fix: ar Dcr
```

---

## Self-Assessment

### Q1: Use -ffile-prefix-map=old=new to strip absolute paths from debug info for reproducibility

**Answer:**

`-ffile-prefix-map` is actually a shorthand for two more specific flags combined. Understanding the distinction helps when you need to apply the fix selectively:

```bash
# The -ffile-prefix-map flag family

# -ffile-prefix-map=OLD=NEW
# Combines -fdebug-prefix-map AND -fmacro-prefix-map
# Replaces OLD prefix with NEW in:
#   1. DWARF debug info (DW_AT_comp_dir, DW_AT_name)
#   2. __FILE__ macro expansion
#   3. __builtin_FILE() expansion (C++20)

# Example: strip absolute source path
g++ -g -ffile-prefix-map=/home/alice/myproject=. -o app main.cpp

# Example: strip build directory too
g++ -g \
    -ffile-prefix-map=/home/alice/myproject=. \
    -ffile-prefix-map=/home/alice/myproject/build=build \
    -o app main.cpp

# Multiple maps can be chained - applied in order

# Verify with readelf
readelf --debug-dump=info app | grep -E "comp_dir|DW_AT_name"
# Before: DW_AT_comp_dir: /home/alice/myproject
# After:  DW_AT_comp_dir:

# Also affects __FILE__
```

Notice that the flag also affects `__FILE__` and `std::source_location::current()`. That means assertion failure messages and log output also become reproducible, which is a nice bonus:

```cpp
#include <iostream>
#include <source_location>

void log_location() {
    auto loc = std::source_location::current();
    std::cout << loc.file_name() << ":" << loc.line() << "\n";
    // Without -ffile-prefix-map: /home/alice/myproject/src/main.cpp:5
    // With -ffile-prefix-map:    ./src/main.cpp:5
}

int main() {
    log_location();
    // __FILE__ is also affected:
    std::cout << __FILE__ << "\n";
    // Without: /home/alice/myproject/src/main.cpp
    // With:    ./src/main.cpp
}
```

In CMake, guard the flags behind a compiler ID check so they are not accidentally passed to MSVC:

```cmake
# CMake best practice:
cmake_minimum_required(VERSION 3.21)
project(MyProject CXX)

# Strip all absolute paths from binaries
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(
        -ffile-prefix-map=${CMAKE_SOURCE_DIR}=.
        -ffile-prefix-map=${CMAKE_BINARY_DIR}=build
    )
endif()
```

### Q2: Explain how embedded `__DATE__` / `__TIME__` macros break reproducibility and how to avoid them

**Answer:**

The reason `__DATE__` and `__TIME__` are so dangerous is that they are invisible at the source level - it just looks like a string constant, and you would never know from reading the code that it changes on every compilation. Once any `.cpp` file uses them, that translation unit's `.o` file will be different every build, and that difference propagates into the final binary.

```cpp
// How __DATE__/__TIME__ break reproducibility
//
// __DATE__ expands to: "Jun 15 2024" (compilation date)
// __TIME__ expands to: "14:32:01"    (compilation time)
//
// These are different for EVERY compilation, even from the same source.
// If any .cpp file uses them, the .o file changes, the final binary changes.

// Common offenders:
// 1. Version strings: const char* ver = "v1.0 built " __DATE__;
// 2. Logging: LOG_INFO("Starting server (compiled " __DATE__ ")");
// 3. Third-party libraries that embed build timestamps

// Detection
```

Start by auditing your codebase and turning on the compiler's built-in warning:

```bash
# GCC/Clang warning for __DATE__/__TIME__:
g++ -Wdate-time -c main.cpp
# warning: macro "__DATE__" might prevent reproducible builds [-Wdate-time]

# Find all usages in codebase:
grep -rn '__DATE__\|__TIME__\|__TIMESTAMP__' src/ include/
# src/version.cpp:5: const char* build_date = __DATE__;
# include/config.h:12: #define BUILD_STAMP __DATE__ " " __TIME__

# Fixes

# Fix 1: SOURCE_DATE_EPOCH (transparent, recommended)
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
g++ -c main.cpp
# __DATE__ now uses the git commit timestamp (deterministic)

# Fix 2: Replace with build-system-injected version
```

The build-system injection approach is more explicit and communicates intent clearly - the version info comes from git, not from the clock:

```cmake
# Generate a version header from CMake:
execute_process(COMMAND git describe --tags --always
    OUTPUT_VARIABLE GIT_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND git log -1 --format=%ci
    OUTPUT_VARIABLE GIT_DATE OUTPUT_STRIP_TRAILING_WHITESPACE)

configure_file(
    ${CMAKE_SOURCE_DIR}/cmake/version.h.in
    ${CMAKE_BINARY_DIR}/generated/version.h
    @ONLY
)
```

```cpp
// cmake/version.h.in
#pragma once
#define APP_VERSION "@GIT_VERSION@"
#define APP_BUILD_DATE "@GIT_DATE@"

// Usage (instead of __DATE__):
#include "version.h"
std::cout << "Version: " << APP_VERSION << " (" << APP_BUILD_DATE << ")\n";
// Deterministic! Only changes when git state changes.
```

### Q3: Use diffoscope to compare two builds and identify sources of non-determinism

**Answer:**

`diffoscope` is the key diagnostic tool here. Unlike a plain `diff`, it understands binary formats - ELF, archives, zips, and more - and can tell you exactly what the difference means in human-readable terms. The workflow is: build twice, compare with `diffoscope`, read the report to find the source, apply a fix, and repeat until the builds match.

```bash
# Install diffoscope
sudo apt-get install diffoscope
# Or: pip install diffoscope

# Build twice with reproducibility flags
export SOURCE_DATE_EPOCH=0

cmake -B build1 -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$(pwd)=."
cmake --build build1

cmake -B build2 -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$(pwd)=."
cmake --build build2

# Compare with diffoscope
diffoscope build1/myapp build2/myapp

# Output if NOT reproducible:
# --- build1/myapp
# +++ build2/myapp
# |-- readelf --debug-dump=info {}
# | @@ -100,1 +100,1 @@
# | -  DW_AT_comp_dir (strp): /home/user/build1
# | +  DW_AT_comp_dir (strp): /home/user/build2
# |   debug info has absolute build directory paths
# |-- readelf --notes {}
# |   .note.gnu.build-id differs
# |   GCC's build-id uses random input by default

# Generate HTML report
diffoscope build1/myapp build2/myapp --html report.html
# Open report.html in browser for detailed side-by-side comparison

# Compare static libraries
diffoscope build1/libmylib.a build2/libmylib.a
# May show: archive timestamps differ
# Fix: use ar with D flag (deterministic mode)
# CMAKE_C_ARCHIVE_CREATE: "<CMAKE_AR> Dcr <TARGET> <OBJECTS>"

# diffoscope for individual object files
diffoscope build1/CMakeFiles/myapp.dir/main.cpp.o \
           build2/CMakeFiles/myapp.dir/main.cpp.o
# Pinpoints exactly which compilation unit introduces non-determinism
```

This table maps `diffoscope` findings directly to root causes and fixes - work through it top to bottom when reading a diffoscope report:

**Common diffoscope findings and fixes:**

| diffoscope finding | Cause | Fix |
| --- | --- | --- |
| `DW_AT_comp_dir` differs | Absolute build path | `-ffile-prefix-map` |
| `.note.gnu.build-id` differs | Random build ID | `--build-id=sha1` (content-based) |
| Archive timestamp differs | `ar` embeds timestamp | `ARFLAGS=Dcr` or `ar D` |
| `__DATE__` string differs | Compile-time timestamp | `SOURCE_DATE_EPOCH` |
| Section ordering differs | Parallel compilation | Link order in build files |

---

## Notes

- **diffoscope** understands 100+ file formats: ELF, archives, zip, PE, Mach-O, PDF, images, etc. - it is the right tool for any binary comparison, not just C++ builds.
- **`--build-id=sha1`** in GCC generates a build-id based on file content (deterministic) instead of a random value, which eliminates one common source of non-determinism.
- **Bazel** achieves reproducible builds by default through sandboxing and content-addressed caching - if you can tolerate the migration cost, it removes most of this manual work.
- **LLVM's `--build-id=tree`** uses a Merkle tree hash for fast, content-based build IDs.
- **Windows:** MSVC's `/Brepro` flag zeros out the PE timestamp header for reproducibility, since MSVC does not support `SOURCE_DATE_EPOCH`.
