# Understand reproducible builds and how to achieve them in C++

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://reproducible-builds.org/>  

---

## Topic Overview

This file focuses on the **SOURCE_DATE_EPOCH standard** and **SHA256 verification workflow** for achieving reproducible C++ builds. A build is reproducible when every developer and CI runner produces the exact same binary from the same source and toolchain. The goal is not just a nice property - it is a security and reliability guarantee: if the binary matches, you know exactly what source produced it.

### The Reproducibility Checklist

Think of this as the minimum set of things you need to get right before you can claim a build is reproducible. Each item addresses a specific source of non-determinism:

```cpp
// Reproducibility checklist - work through these top to bottom:
// [ ] Pin compiler version (GCC 13.2.0, not "latest")
// [ ] Pin all dependency versions (vcpkg baseline hash)
// [ ] Set SOURCE_DATE_EPOCH (deterministic timestamps)
// [ ] Use -ffile-prefix-map (strip absolute paths)
// [ ] Sort all file lists (no glob order dependency)
// [ ] Use deterministic ar (ARFLAGS=Dcr)
// [ ] Remove __DATE__/__TIME__ (or use SOURCE_DATE_EPOCH)
// [ ] Verify with sha256sum or diffoscope
```

---

## Self-Assessment

### Q1: Use -ffile-prefix-map=$(pwd)=. to strip absolute paths from debug info for reproducibility

**Answer:**

The `-ffile-prefix-map` flag tells the compiler to rewrite path prefixes before embedding them in the binary. Without it, the absolute path to your source directory ends up in the DWARF debug info, and that path is different on every developer's machine.

```bash
# Compiler flags for path stripping

# GCC/Clang: replace source directory path with "."
g++ -g -ffile-prefix-map=$(pwd)=. -o app main.cpp

# The flag family:
# -ffile-prefix-map=OLD=NEW      -> replace in ALL embedded paths (debug + __FILE__)
# -fdebug-prefix-map=OLD=NEW     -> replace only in DWARF debug info
# -fmacro-prefix-map=OLD=NEW     -> replace only in __FILE__ macro expansion
# -ffile-prefix-map is equivalent to both of the above

# CMake integration
```

Putting it into CMake ensures the flag is applied to every target automatically:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(MyApp CXX)

# Strip both source and build tree paths
add_compile_options(
    -ffile-prefix-map=${CMAKE_SOURCE_DIR}=.
    -ffile-prefix-map=${CMAKE_BINARY_DIR}=build
)

# Also fix __FILE__ in assertion messages:
# Before: /home/alice/project/src/main.cpp:42: assertion failed
# After:  ./src/main.cpp:42: assertion failed
```

After applying the flag, verify that no absolute paths are left in the binary:

```bash
# Verify paths are stripped
# Check what paths are embedded in the binary:
strings app | grep /home
# (should be empty after -ffile-prefix-map)

readelf --debug-dump=info app | grep comp_dir
# DW_AT_comp_dir: .    <- relative, not absolute

# MSVC equivalent:
# cl /d1trimfile:C:\Users\alice\project=. main.cpp
```

### Q2: Use SOURCE_DATE_EPOCH to make __DATE__ and __TIME__ macros deterministic

**Answer:**

`SOURCE_DATE_EPOCH` is a standard environment variable defined by the reproducible-builds.org project. When it is set, GCC and Clang use it as the build timestamp instead of the current wall-clock time. The result is that `__DATE__` and `__TIME__` expand to a fixed, deterministic string - the same on every machine that builds the same commit.

```bash
# SOURCE_DATE_EPOCH standard
# Defined by https://reproducible-builds.org/specs/source-date-epoch/
# When this environment variable is set, compilers use it instead of "now"
# for __DATE__, __TIME__, and embedded timestamps

# Set to the last git commit time:
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
echo $SOURCE_DATE_EPOCH
# 1718451121

# Or set to a fixed value for absolute reproducibility:
export SOURCE_DATE_EPOCH=0
# __DATE__ becomes "Jan  1 1970", __TIME__ becomes "00:00:00"

# Build with GCC:
SOURCE_DATE_EPOCH=$(git log -1 --format=%ct) g++ -o app main.cpp

# Build with Clang:
SOURCE_DATE_EPOCH=$(git log -1 --format=%ct) clang++ -o app main.cpp
# Both honor SOURCE_DATE_EPOCH since GCC 7 and Clang 6
```

Here is what it looks like from the C++ side - the same code behaves differently depending on whether `SOURCE_DATE_EPOCH` is set:

```cpp
// Demonstrating the effect
#include <iostream>

int main() {
    // Without SOURCE_DATE_EPOCH: changes every build
    // With SOURCE_DATE_EPOCH=1718451121: always "Jun 15 2024" "12:32:01"
    std::cout << "Build date: " << __DATE__ << "\n";
    std::cout << "Build time: " << __TIME__ << "\n";
    return 0;
}
```

You can set it from CMake by querying git at configure time:

```cmake
# CMake integration: set SOURCE_DATE_EPOCH from git
find_package(Git REQUIRED)
execute_process(
    COMMAND ${GIT_EXECUTABLE} log -1 --format=%ct
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_COMMIT_TIMESTAMP
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
set(ENV{SOURCE_DATE_EPOCH} ${GIT_COMMIT_TIMESTAMP})
message(STATUS "SOURCE_DATE_EPOCH = ${GIT_COMMIT_TIMESTAMP}")
```

### Q3: Verify build reproducibility by comparing SHA256 of two independently built binaries

**Answer:**

The verification workflow is straightforward: build from scratch in two separate directories, hash the outputs, and compare. If the hashes match, the build is reproducible. If they differ, `diffoscope` will tell you exactly which sections of the binary differ and why.

```bash
# Verification workflow

# Step 1: Set reproducibility flags
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
REPRO_FLAGS="-ffile-prefix-map=$(pwd)=. -g"

# Step 2: Clean build #1
rm -rf build1
cmake -B build1 \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_FLAGS="$REPRO_FLAGS"
cmake --build build1 --parallel $(nproc)

# Record hashes
find build1 -name '*.o' -o -name 'myapp' | sort | xargs sha256sum > hashes1.txt

# Step 3: Clean build #2
rm -rf build2
cmake -B build2 \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_FLAGS="$REPRO_FLAGS"
cmake --build build2 --parallel $(nproc)

# Record hashes
find build2 -name '*.o' -o -name 'myapp' | sort | xargs sha256sum > hashes2.txt

# Step 4: Compare (adjust paths so they match)
sed 's|build1/|BUILD/|g' hashes1.txt > norm1.txt
sed 's|build2/|BUILD/|g' hashes2.txt > norm2.txt
diff norm1.txt norm2.txt

# If empty: reproducible
# If different: investigate which object files differ

# Quick one-liner check
sha256sum build1/myapp build2/myapp
# abc123...  build1/myapp
# abc123...  build2/myapp  <- same hash = reproducible

# Deep investigation with diffoscope
diffoscope build1/myapp build2/myapp --html diff-report.html
# Opens HTML showing exactly which bytes differ and why
```

Once you have verified it locally, automate the check in CI so regressions are caught immediately:

```yaml
# CI automation: reproducibility gate
name: Verify Reproducible Build
on: [push]
jobs:
  reproducible:
    runs-on: ubuntu-latest
    env:
      SOURCE_DATE_EPOCH: "0"
    steps:

      - uses: actions/checkout@v4

      - name: Build 1

        run: |
          cmake -B b1 -DCMAKE_BUILD_TYPE=Release \
            -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$PWD=."
          cmake --build b1
          sha256sum b1/myapp | cut -d' ' -f1 > hash1

      - name: Build 2

        run: |
          cmake -B b2 -DCMAKE_BUILD_TYPE=Release \
            -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$PWD=."
          cmake --build b2
          sha256sum b2/myapp | cut -d' ' -f1 > hash2

      - name: Compare

        run: |
          if ! diff -q hash1 hash2; then
            echo "::error::Builds are NOT reproducible!"
            diff hash1 hash2
            exit 1
          fi
          echo "Builds are reproducible!"
```

---

## Notes

- **`SOURCE_DATE_EPOCH`** is supported by GCC (7+), Clang (6+), and many other tools (tar, gzip, zip, javac) - it is a cross-ecosystem standard, not a compiler-specific extension.
- **MSVC** does not support `SOURCE_DATE_EPOCH` - use `/Brepro` flag for deterministic PE timestamps instead.
- **Static libraries (.a)** embed timestamps by default - use `ar Dcr` or set `CMAKE_C_ARCHIVE_CREATE` with the `D` flag to disable this.
- **Python in build scripts** can introduce non-determinism via `dict` ordering - use `PYTHONHASHSEED=0` to make it deterministic.
- **Reproducible builds are a supply-chain security measure** - they allow third parties to verify that a binary was built from the claimed source code, which is essential for package managers and open-source distribution.
