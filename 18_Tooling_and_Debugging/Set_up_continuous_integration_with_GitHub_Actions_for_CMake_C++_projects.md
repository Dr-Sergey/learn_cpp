# Set up continuous integration with GitHub Actions for CMake C++ projects

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://docs.github.com/en/actions>  

---

## Topic Overview

GitHub Actions gives you free CI/CD for open-source projects and a generous free tier for private repos. For C++ projects, a typical workflow looks like this: checkout the code, install any dependencies, run `cmake` to configure, build, then run your test suite with CTest. Automating this means every push and every pull request gets verified before it can merge.

The high-level picture of what a well-structured C++ CI pipeline looks like is shown below:

```cpp
Workflow Trigger (push/PR)
  │
  ├─ Job: Linux GCC
  │    ├─ cmake -B build -DCMAKE_CXX_COMPILER=g++
  │    ├─ cmake --build build
  │    └─ ctest --test-dir build --output-on-failure
  │
  ├─ Job: macOS Clang
  │    └─ (same steps, different runner)
  │
  └─ Job: Sanitizers
       ├─ cmake -B build -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined"
       └─ ctest ...
```

The reason you run sanitizer jobs separately is that ASAN and TSAN are mutually exclusive - they conflict at the memory instrumentation level. More on that in Q3.

---

## Self-Assessment

### Q1: Write a multi-platform workflow (Linux GCC + macOS Clang)

GitHub Actions matrix builds let you declare what varies between jobs (OS, compiler, extra flags) and the platform runs the same steps for every combination. Here is a workflow that builds on both Linux with GCC and macOS with Clang:

```yaml
# .github/workflows/ci.yml
name: C++ CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: ubuntu-latest
            compiler: g++-13
            cmake_args: -DCMAKE_CXX_COMPILER=g++-13
          - os: macos-latest
            compiler: clang++
            cmake_args: -DCMAKE_CXX_COMPILER=clang++

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Release ${{ matrix.cmake_args }}

      - name: Build
        run: cmake --build build --parallel $(nproc 2>/dev/null || sysctl -n hw.ncpu)

      - name: Test
        run: ctest --test-dir build --output-on-failure --parallel 4
```

A few things worth noting here. `fail-fast: false` means that if the macOS job fails, the Linux job keeps running - you see all failures at once instead of stopping at the first one. The `nproc 2>/dev/null || sysctl -n hw.ncpu` trick is just a portable way to pass the CPU count on both Linux and macOS.

**Key points:**

- `strategy.matrix` runs the same steps on multiple OS/compiler combinations.
- `fail-fast: false` lets all matrix jobs finish even if one fails.
- `actions/checkout@v4` checks out the repo.

### Q2: Add CTest with failure reporting

Testing only pays off if failures are actually visible. Here is a minimal CMakeLists.txt that registers a test executable with CTest, followed by the corresponding test source:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(myapp src/main.cpp)

# Enable testing
enable_testing()
add_executable(test_math tests/test_math.cpp)
add_test(NAME MathTests COMMAND test_math)
```

```cpp
// tests/test_math.cpp
#include <cassert>
#include <iostream>

int add(int a, int b) { return a + b; }

int main() {
    assert(add(2, 3) == 5);
    assert(add(-1, 1) == 0);
    assert(add(0, 0) == 0);
    std::cout << "All math tests passed!\n";
    return 0;  // CTest treats exit code 0 as PASS
}
// Expected output:
// All math tests passed!
```

CTest interprets a non-zero exit code as a test failure. The `--output-on-failure` flag is what makes test output visible in the CI log - without it you just see "FAILED" with no details. The `--timeout` guard catches tests that hang and would otherwise stall the whole pipeline:

```yaml
      - name: Test
        run: |
          ctest --test-dir build \
            --output-on-failure \
            --parallel 4 \
            --timeout 120
        # --output-on-failure: prints stdout/stderr only for failed tests
        # --timeout 120: kills tests that hang for 2+ minutes
```

### Q3: Add a sanitizer CI job

ASAN and TSAN cannot run together because they both replace the memory allocator with incompatible implementations. The solution is to run them in separate matrix entries so they execute in parallel and each gets a clean build. Here is the sanitizer portion of the workflow:

```yaml
  sanitizers:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        sanitizer:
          - name: ASAN+UBSAN
            flags: "-fsanitize=address,undefined -fno-omit-frame-pointer"
          - name: TSAN
            flags: "-fsanitize=thread"
    name: ${{ matrix.sanitizer.name }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure
        run: |
          cmake -B build \
            -DCMAKE_CXX_COMPILER=clang++-17 \
            -DCMAKE_BUILD_TYPE=Debug \
            -DCMAKE_CXX_FLAGS="${{ matrix.sanitizer.flags }}"

      - name: Build
        run: cmake --build build --parallel $(nproc)

      - name: Test with sanitizers
        env:
          ASAN_OPTIONS: "detect_leaks=1:halt_on_error=1"
          UBSAN_OPTIONS: "halt_on_error=1:print_stacktrace=1"
          TSAN_OPTIONS: "halt_on_error=1"
        run: ctest --test-dir build --output-on-failure
```

The `halt_on_error=1` option is essential: without it a sanitizer might print an error report but the process continues and exits with code 0, so the CI job would appear to pass. With it, the process terminates immediately with a non-zero code, and CTest (and then GitHub Actions) correctly marks the job as failed.

**Key points:**

- ASAN and TSAN are **mutually exclusive** - they need separate jobs.
- `halt_on_error=1` makes sanitizer findings fail the build.
- `-fno-omit-frame-pointer` produces better stack traces.
- Use `Debug` build type - optimizations can hide bugs.

---

## Notes

- Always set `CMAKE_EXPORT_COMPILE_COMMANDS=ON` for clang-tidy integration.
- Cache `~/.cache/ccache` or the CMake build directory for faster CI.
- Use the `concurrency` key to cancel in-progress runs on new pushes.
- Add a `clang-format` check job to enforce code style.
- Matrix builds multiply the number of jobs - keep them fast.
