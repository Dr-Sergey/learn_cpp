# Set up a complete GitHub Actions CI pipeline for a C++ CMake project

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://docs.github.com/en/actions>  

---

## Topic Overview

GitHub Actions provides free CI/CD for open-source C++ projects. A well-structured pipeline builds on all three major platforms (Linux/macOS/Windows), runs tests with sanitizers, and caches builds for speed. The idea is that every push or pull request automatically verifies your code compiles, passes tests, and is free of sanitizer errors - across the full range of compilers your users might have.

### Pipeline Architecture

Here is the shape of a complete C++ CI pipeline. It has three layers: the main matrix build across platforms, sanitizer jobs on Linux, and static analysis.

```cpp
Push / PR
  |
+---------------------------------------------+
| Matrix Strategy                              |
|  +-- ubuntu-latest   + GCC 13               |
|  +-- ubuntu-latest   + Clang 17             |
|  +-- macos-latest    + Apple Clang          |
|  +-- windows-latest  + MSVC 2022           |
+---------------------------------------------+
| Sanitizer Jobs (Linux only)                  |
|  +-- ASAN + UBSAN                           |
|  +-- TSAN                                   |
+---------------------------------------------+
| Static Analysis (clang-tidy, cppcheck)      |
+---------------------------------------------+
```

### Key GitHub Actions Concepts

If you're new to GitHub Actions, these are the four concepts you'll use constantly in a C++ pipeline:

| Concept | Purpose |
| --- | --- |
| `strategy.matrix` | Run same steps across multiple OS/compiler combos |
| `actions/cache` | Persist build dir / vcpkg between runs |
| `cmake --preset` | Reproducible configure from CMakePresets.json |
| `ctest --output-on-failure` | Show test output only on failures |

### Minimal Workflow File

Before diving into the full setup, here's the simplest possible workflow that builds and tests on all three platforms. Notice how the `${{ matrix.os }}` placeholder is the only thing that changes between runs - the rest of the steps are identical.

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Release
      - name: Build
        run: cmake --build build --config Release -j $(nproc 2>/dev/null || echo 2)
      - name: Test
        run: ctest --test-dir build --output-on-failure -C Release
```

This is enough to catch the most common portability problems. The `nproc 2>/dev/null || echo 2` fallback handles macOS and Windows where `nproc` is not available.

---

## Self-Assessment

### Q1: Write a matrix workflow that builds on ubuntu-latest (GCC), macos-latest (Clang), and windows-latest (MSVC)

**Answer:**

The trick here is using `matrix.include` instead of a simple `matrix.os` list, because each platform needs different compiler installation steps and different compiler names.

```yaml
# .github/workflows/ci.yml
name: C++ CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    name: ${{ matrix.os }} / ${{ matrix.compiler }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false      # Don't cancel other jobs on first failure
      matrix:
        include:
          # -- Linux GCC --
          - os: ubuntu-latest
            compiler: gcc-13
            cc: gcc-13
            cxx: g++-13
            install: sudo apt-get install -y gcc-13 g++-13

          # -- Linux Clang --
          - os: ubuntu-latest
            compiler: clang-17
            cc: clang-17
            cxx: clang++-17
            install: |
              wget https://apt.llvm.org/llvm.sh
              chmod +x llvm.sh
              sudo ./llvm.sh 17

          # -- macOS Apple Clang --
          - os: macos-latest
            compiler: apple-clang
            cc: clang
            cxx: clang++
            install: ""

          # -- Windows MSVC --
          - os: windows-latest
            compiler: msvc
            cc: cl
            cxx: cl
            install: ""

    steps:
      - uses: actions/checkout@v4

      - name: Install compiler
        if: matrix.install != ''
        run: ${{ matrix.install }}

      - name: Configure CMake
        env:
          CC: ${{ matrix.cc }}
          CXX: ${{ matrix.cxx }}
        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=Release
          -DCMAKE_CXX_STANDARD=20
          -DBUILD_TESTING=ON

      - name: Build
        run: cmake --build build --config Release --parallel 2

      - name: Run tests
        working-directory: build
        run: ctest --output-on-failure -C Release --timeout 120
```

**Explanation:**

- `fail-fast: false` ensures all platforms report results even if one fails - you want to see all your problems at once, not just the first one.
- `matrix.include` gives explicit control over each configuration, which is more readable than combining `os` and `compiler` lists with exclusions.
- `CC`/`CXX` environment variables tell CMake which compiler to use on Linux/macOS; on Windows, selecting the runner image is enough.
- `--parallel 2` matches the 2-vCPU GitHub-hosted runners - using more parallelism than you have CPUs just wastes memory.
- `ctest --output-on-failure` only prints test output when something fails, keeping the logs readable.

### Q2: Add sanitizer jobs: one with ASAN+UBSAN and one with TSAN on Linux

**Answer:**

Sanitizers are a separate job because they require different compiler flags and can't share a build with the regular matrix job. Also note that ASAN and TSAN are mutually exclusive - you cannot run them together.

```yaml
  # Add this job alongside the build matrix job above
  sanitizers:
    name: ${{ matrix.sanitizer }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - sanitizer: ASAN+UBSAN
            cmake_flags: >
              -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined
              -fno-sanitize-recover=all -fno-omit-frame-pointer"
              -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
            env_vars: "ASAN_OPTIONS=detect_leaks=1:detect_stack_use_after_return=1"

          - sanitizer: TSAN
            cmake_flags: >
              -DCMAKE_CXX_FLAGS="-fsanitize=thread -fno-omit-frame-pointer"
              -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread"
            env_vars: "TSAN_OPTIONS=second_deadlock_stack=1"

    steps:
      - uses: actions/checkout@v4

      - name: Install Clang 17
        run: |
          wget https://apt.llvm.org/llvm.sh
          chmod +x llvm.sh
          sudo ./llvm.sh 17

      - name: Configure
        env:
          CC: clang-17
          CXX: clang++-17
        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=Debug
          -DCMAKE_CXX_STANDARD=20
          -DBUILD_TESTING=ON
          ${{ matrix.cmake_flags }}

      - name: Build
        run: cmake --build build --parallel 2

      - name: Test with ${{ matrix.sanitizer }}
        env:
          ${{ matrix.env_vars }}: ""
        working-directory: build
        run: ctest --output-on-failure --timeout 300
```

**Key details:**

- ASAN and UBSAN are compatible and combined into one job; TSAN must be separate because it's incompatible with ASAN.
- `-fno-sanitize-recover=all` makes the sanitizer abort on the first error rather than continuing - this gives you a clean failure instead of a pile of noise.
- `-fno-omit-frame-pointer` gives better stack traces, which you'll really appreciate when debugging sanitizer reports.
- Debug build type is required for useful sanitizer line information - Release builds optimize away context you need.
- `detect_leaks=1` enables the LeakSanitizer component bundled inside ASAN.
- The TSAN timeout is longer (300s) because the thread sanitizer adds roughly 5-15x runtime overhead.

### Q3: Cache the build directory and vcpkg packages between CI runs to reduce build time

**Answer:**

Caching is what takes a pipeline from "too slow to bother running" to "fast enough that you always run it." The two biggest costs in a typical C++ CI run are vcpkg dependency installation and compilation of unchanged files - caching handles both.

```yaml
  build-cached:
    name: Cached Build (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]

    steps:
      - uses: actions/checkout@v4

      # -- Cache CMake build directory --
      - name: Cache build directory
        uses: actions/cache@v4
        with:
          path: build
          key: build-${{ matrix.os }}-${{ hashFiles('CMakeLists.txt', 'src/**', 'include/**') }}
          restore-keys: |
            build-${{ matrix.os }}-

      # -- Cache vcpkg installed packages --
      - name: Cache vcpkg
        uses: actions/cache@v4
        with:
          path: |
            ~/vcpkg
            build/vcpkg_installed
          key: vcpkg-${{ matrix.os }}-${{ hashFiles('vcpkg.json', 'vcpkg-configuration.json') }}
          restore-keys: |
            vcpkg-${{ matrix.os }}-

      # -- Setup vcpkg --
      - name: Setup vcpkg
        uses: lukka/run-vcpkg@v11
        with:
          vcpkgGitCommitId: "a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6"  # Pin!

      # -- Configure + Build --
      - name: Configure
        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=Release
          -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
          -DCMAKE_CXX_STANDARD=20

      - name: Build (incremental)
        run: cmake --build build --config Release --parallel 2

      - name: Test
        run: ctest --test-dir build --output-on-failure -C Release
```

**Cache strategy explained:**

- The build cache key uses `hashFiles` on source files - it rebuilds when code changes but reuses the cache when only the CI config changes. The `restore-keys` fallback restores the most recent cache for the same OS, which enables incremental builds because CMake only recompiles the files that changed.
- The vcpkg cache key hashes `vcpkg.json` - it only reinstalls packages when your dependency list actually changes.
- Pin the vcpkg commit hash. Without this, vcpkg updates silently and can break your builds in ways that are very hard to debug because the only thing that changed was a library version you didn't ask to change.
- This setup saves 3-8 minutes per run on typical projects, with vcpkg installation being the biggest cost.

---

## Notes

- `ccache` can further speed up builds - use `lukka/run-cmake@v10` with `CMAKE_CXX_COMPILER_LAUNCHER=ccache` to add another layer of compilation caching.
- Use `actions/upload-artifact@v4` to upload test reports, coverage data, or built binaries as artifacts that you can download after a run.
- Set up branch protection rules that require all CI jobs to pass before merging PRs - this is the main point of having CI.
- GitHub-hosted runners are free for public repos; private repos get 2000 minutes/month free, after which you pay per minute.
- Self-hosted runners are worth setting up for specialized hardware (GPUs, ARM boards) or to avoid rate limits on large projects.
- The `concurrency` key can cancel in-progress runs when a new push arrives: `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }` - this prevents queuing up a dozen runs when you push rapidly.
