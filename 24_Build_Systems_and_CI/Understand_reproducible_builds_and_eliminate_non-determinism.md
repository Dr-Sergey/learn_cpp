# Understand reproducible builds and eliminate non-determinism

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://reproducible-builds.org/>  

---

## Topic Overview

A **reproducible build** produces bit-for-bit identical output given the same source code, build environment, and build instructions. The whole point is that if two different people - or two different CI runners - build the same commit with the same toolchain, they should get exactly the same bytes in the final binary.

Why does this matter? Because if the binary changes every time you build it, you cannot tell whether a change in the output came from a code change or just from noise. Reproducibility is what makes build caches trustworthy, what makes supply-chain audits possible, and what makes distributed caching systems like remote build caches actually correct.

Non-determinism sneaks in through several sources that are easy to overlook: absolute paths baked into debug info, timestamps embedded by `__DATE__`/`__TIME__`, and randomized data structure iteration. None of these affect runtime behavior, but they all make the binary different on every machine and every build.

### Sources of Non-Determinism in C++ Builds

Here is a quick map of the most common culprits and what to do about each one:

| Source | What changes | Fix |
| --- | --- | --- |
| Absolute paths in DWARF debug | `/home/alice/project/` vs `/home/bob/project/` | `-ffile-prefix-map=$(pwd)=.` |
| `__DATE__` / `__TIME__` macros | Compile timestamp | `SOURCE_DATE_EPOCH`, or ban usage |
| File ordering (glob) | Directory listing order varies | Sort file lists explicitly |
| Uninitialized padding | Struct padding bytes are random | `-ftrivial-auto-var-init=zero` |
| Parallel compilation | Object file link order | Use explicit link order in build system |
| Randomized hash seeds | `unordered_map` iteration order | Use deterministic containers in codegen |
| Compiler version | Different codegen | Pin exact compiler version |

---

## Self-Assessment

### Q1: Use -ffile-prefix-map=$(pwd)=. to strip absolute paths from debug info for reproducibility

**Answer:**

The core problem is that when you compile with `-g`, the compiler embeds the absolute path to your source directory into the DWARF debug info. That path is different on every developer's machine, so two builds of the same code produce different binaries even though the source is identical.

The fix is to tell the compiler to rewrite those paths before embedding them.

```bash
# The problem: absolute paths in debug info

# When you compile with debug info:
g++ -g -o hello hello.cpp

# The DWARF debug info contains absolute paths:
readelf --debug-dump=info hello | grep DW_AT_comp_dir
# DW_AT_comp_dir: /home/alice/projects/hello
# Different on every developer's machine

# Two developers building the same code get different binaries:
# Alice: DW_AT_comp_dir: /home/alice/projects/hello
# Bob:   DW_AT_comp_dir: /home/bob/projects/hello
# sha256sum differs

# The fix: -ffile-prefix-map

# Replace absolute path with relative:
g++ -g -ffile-prefix-map=$(pwd)=. -o hello hello.cpp

readelf --debug-dump=info hello | grep DW_AT_comp_dir
# DW_AT_comp_dir:
# Same on every machine

# More specific variants:
# -fdebug-prefix-map=$(pwd)=.     # Only DWARF debug info
# -fmacro-prefix-map=$(pwd)=.     # Only __FILE__ macro expansion
# -ffile-prefix-map=$(pwd)=.      # Both (recommended)
```

You can bake this into CMake so the flag is applied automatically for every target:

```cmake
# CMakeLists.txt integration
add_compile_options(
    -ffile-prefix-map=${CMAKE_SOURCE_DIR}=.
    -ffile-prefix-map=${CMAKE_BINARY_DIR}=build
)

# Or with CMake 3.26+ built-in support:
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffile-prefix-map=${CMAKE_SOURCE_DIR}=.")
```

Once the flag is in place, you can verify that two independent builds actually match:

```bash
# Verify the fix:
# Build 1 (in /home/alice/project/)
cd /home/alice/project && g++ -g -ffile-prefix-map=$(pwd)=. -o hello hello.cpp
sha256sum hello
# abc123

# Build 2 (in /home/bob/project/)
cd /home/bob/project && g++ -g -ffile-prefix-map=$(pwd)=. -o hello hello.cpp
sha256sum hello
# abc123...  <- IDENTICAL
```

`-ffile-prefix-map=OLD=NEW` replaces every occurrence of the `OLD` path prefix with `NEW` in the generated object files. Setting `OLD=$(pwd)` and `NEW=.` converts absolute paths to relative, making the binary identical regardless of where it was built.

### Q2: Explain how __DATE__ and __TIME__ macros break reproducibility and how to replace them

**Answer:**

`__DATE__` and `__TIME__` expand to the exact moment the compiler ran. Every build - even from the same source - produces a different string, which means a different object file, which means a different binary. They are probably the most common hidden source of non-determinism in real codebases.

```cpp
// The problem
// __DATE__ and __TIME__ expand to the compilation timestamp:

#include <iostream>

void print_version() {
    std::cout << "Built on: " << __DATE__ << " " << __TIME__ << "\n";
    // Output: Built on: Jun 15 2024 14:32:01
    // Changes every time you compile!
    // Two builds from the same source -> different binaries
}
```

The cleanest fix is `SOURCE_DATE_EPOCH`, a standard environment variable that GCC and Clang both honor. When it is set, the compiler uses that timestamp instead of the current time:

```bash
# Fix 1: SOURCE_DATE_EPOCH (recommended)
# GCC and Clang honor this environment variable:
# When set, __DATE__ and __TIME__ use this timestamp instead of "now"

export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
# Uses the last git commit timestamp (deterministic!)

g++ -g -o hello hello.cpp
# __DATE__ is now the commit date, same on every build of this commit

# In CMake:
# set(ENV{SOURCE_DATE_EPOCH} "1718451121")  # Unix timestamp
```

If you want to be more aggressive about eliminating `__DATE__`/`__TIME__` entirely, you can redefine them to a placeholder string that makes the problem visible:

```cpp
// Fix 2: Ban __DATE__/__TIME__ with compiler errors

// .clang-tidy
// Checks: bugprone-suspicious-include,cert-dcl58-cpp,...

// Or use a macro that fails compilation:
#ifdef REPRODUCIBLE_BUILD
    #undef __DATE__
    #define __DATE__ "REDACTED"
    #undef __TIME__
    #define __TIME__ "REDACTED"
#endif

// Fix 3: Use version info from build system

// CMakeLists.txt:
// configure_file(version.h.in version.h @ONLY)

// version.h.in:
// #define BUILD_VERSION "@PROJECT_VERSION@"
// #define BUILD_GIT_HASH "@GIT_HASH@"

#include "version.h"

void print_version_safe() {
    std::cout << "Version: " << BUILD_VERSION << "\n";
    std::cout << "Commit:  " << BUILD_GIT_HASH << "\n";
    // Deterministic! Changes only when version/commit changes
}
```

GCC also has a warning flag that catches any use of these macros in your codebase:

```bash
# Detect __DATE__/__TIME__ usage
# GCC warning (since GCC 4.9):
g++ -Wdate-time -Werror=date-time -c hello.cpp
# error: macro "__DATE__" might prevent reproducible builds [-Werror=date-time]

# Search codebase for usage:
grep -rn '__DATE__\|__TIME__' src/ include/
```

### Q3: Verify reproducibility with diffoscope by comparing two independent builds byte-for-byte

**Answer:**

The verification workflow is: build twice in separate directories, compare hashes, and if they differ, use `diffoscope` to pinpoint exactly which bytes are different and why. This is how you actually debug non-determinism - you don't guess, you let the tool show you.

```bash
# Step 1: Build twice independently

# Build 1
mkdir build1 && cd build1
cmake .. -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_FLAGS="-ffile-prefix-map=$(pwd)=." \
    -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$(pwd)=."
cmake --build . --parallel $(nproc)
sha256sum myapp > ../checksums1.txt
cd ..

# Build 2 (clean directory)
mkdir build2 && cd build2
cmake .. -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_FLAGS="-ffile-prefix-map=$(pwd)=." \
    -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$(pwd)=."
cmake --build . --parallel $(nproc)
sha256sum myapp > ../checksums2.txt
cd ..

# Step 2: Compare checksums
diff checksums1.txt checksums2.txt
# If empty (no diff) -> reproducible
# If different -> investigate with diffoscope

# Step 3: Use diffoscope for detailed comparison
# Install:
sudo apt-get install diffoscope

# Compare binaries:
diffoscope build1/myapp build2/myapp --html report.html

# diffoscope output example:
# -- myapp
# | -- readelf --debug-dump=info {}
# | | @@ -42,1 +42,1 @@
# | | -  DW_AT_comp_dir: /home/user/build1
# | | +  DW_AT_comp_dir: /home/user/build2
# | | Found the source of non-determinism

# Step 4: Fix and re-verify
# Add -ffile-prefix-map, re-build, re-compare
# Iterate until diffoscope reports "no differences"

# CI automation
```

Once you can reproduce the issue, you automate the check so it runs on every push. A CI job that builds twice and compares hashes is a cheap but very effective reproducibility gate:

```yaml
# .github/workflows/reproducible.yml
name: Reproducibility Check
on: [push]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Build 1

        env:
          SOURCE_DATE_EPOCH: "0"
        run: |
          cmake -B build1 -DCMAKE_BUILD_TYPE=Release \
            -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$PWD=."
          cmake --build build1
          sha256sum build1/myapp > hash1.txt

      - name: Build 2 (clean)

        env:
          SOURCE_DATE_EPOCH: "0"
        run: |
          cmake -B build2 -DCMAKE_BUILD_TYPE=Release \
            -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$PWD=."
          cmake --build build2
          sha256sum build2/myapp > hash2.txt

      - name: Verify identical

        run: |
          diff hash1.txt hash2.txt
          echo "Builds are reproducible!"
```

---

## Notes

- **`strip-nondeterminism`** is a Debian tool that removes timestamps from archives (.a, .jar, .zip) - useful as a post-processing step if you cannot fix the root cause.
- **Link order** can be non-deterministic with CMake globs - always list source files explicitly rather than using `file(GLOB ...)`.
- **ASLR** doesn't affect binary reproducibility (it's a runtime feature), but PIE/non-PIE builds differ in the binary layout.
- **`ar` archives** embed timestamps by default - use `ARFLAGS=Dcr` for deterministic `.a` files.
- **Reproducible builds are required** for: package manager verification, supply chain security, audit compliance (ISO 27001), and efficient remote caching.
