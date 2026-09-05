# Design CI matrices for cross-platform testing

**Category:** Cross Platform Development  
**Standard:** C++17  
**Reference:** <https://docs.github.com/en/actions/using-jobs/using-a-matrix-strategy-for-your-jobs>  

---

## Topic Overview

### Why CI Matrices Matter

If you test your code on a single platform with a single compiler, you are catching only a fraction of the bugs your users will encounter. Different compilers have genuinely different behavior, not just different warnings. The key differences to appreciate:

- **Warning sets** (Clang catches things GCC misses and vice versa)
- **Standard library implementations** (libstdc++, libc++, MSVC STL)
- **ABI and padding** differences
- **Platform-specific behavior** (path separators, endianness, `wchar_t` size)

A CI matrix runs the same tests across multiple configurations automatically, so you catch cross-platform regressions the moment they are introduced rather than when a user on a different OS files a bug report.

### GitHub Actions Matrix - Complete Example

The core idea is a `matrix` strategy that expands a single job definition into multiple parallel jobs. Each entry in the `include` list becomes one build-and-test run. Setting `fail-fast: false` is important - it tells GitHub Actions to keep running all matrix jobs even if one fails, so you get the full picture in a single CI run:

```yaml
name: CI
on: [push, pull_request]

jobs:
  build-and-test:
    strategy:
      fail-fast: false  # Don't cancel other jobs if one fails
      matrix:
        include:
          # Linux GCC
          - os: ubuntu-24.04
            compiler: gcc-13
            cxx: g++-13
            cc: gcc-13
            build_type: Release
            sanitizer: ""

          # Linux Clang
          - os: ubuntu-24.04
            compiler: clang-17
            cxx: clang++-17
            cc: clang-17
            build_type: Release
            sanitizer: ""

          # Linux ASan + UBSan
          - os: ubuntu-24.04
            compiler: clang-17-asan
            cxx: clang++-17
            cc: clang-17
            build_type: Debug
            sanitizer: "-fsanitize=address,undefined -fno-omit-frame-pointer"

          # Linux TSan
          - os: ubuntu-24.04
            compiler: clang-17-tsan
            cxx: clang++-17
            cc: clang-17
            build_type: Debug
            sanitizer: "-fsanitize=thread"

          # macOS
          - os: macos-14
            compiler: apple-clang
            cxx: clang++
            cc: clang
            build_type: Release
            sanitizer: ""

          # Windows MSVC
          - os: windows-2022
            compiler: msvc
            cxx: cl
            cc: cl
            build_type: Release
            sanitizer: ""

    runs-on: ${{ matrix.os }}
    name: ${{ matrix.compiler }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure
        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=${{ matrix.build_type }}
          -DCMAKE_CXX_COMPILER=${{ matrix.cxx }}
          -DCMAKE_C_COMPILER=${{ matrix.cc }}
          -DCMAKE_CXX_FLAGS="${{ matrix.sanitizer }}"
          -DCMAKE_EXE_LINKER_FLAGS="${{ matrix.sanitizer }}"

      - name: Build
        run: cmake --build build --config ${{ matrix.build_type }} -j $(nproc 2>/dev/null || echo 4)

      - name: Test
        run: ctest --test-dir build --build-config ${{ matrix.build_type }} --output-on-failure -j4
```

### ARM Cross-Compilation in CI

You can test on ARM architecture without any ARM hardware by using QEMU in user-mode emulation. Cross-compile for AArch64, then run the resulting binary under `qemu-aarch64`. This is slower than native execution but reliably catches alignment, ABI, and weak-memory-model bugs that would never surface on x86:

```yaml
  arm64-cross:
    runs-on: ubuntu-24.04
    name: ARM64 Cross-Compile

    steps:
      - uses: actions/checkout@v4

      - name: Install cross-compiler
        run: sudo apt-get install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu qemu-user

      - name: Configure
        run: cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/aarch64-linux.cmake

      - name: Build
        run: cmake --build build -j$(nproc)

      - name: Test under QEMU
        run: |
          cd build
          ctest --output-on-failure
        env:
          QEMU_LD_PREFIX: /usr/aarch64-linux-gnu
```

### Multi-Standard Matrix

You can also expand the matrix across C++ standard versions. This is useful to ensure your library works on C++17 while also confirming it takes advantage of C++20 and C++23 features correctly, and to catch any deprecated-feature warnings before they become errors in a future standard:

```yaml
    strategy:
      matrix:
        cxx_standard: [17, 20, 23]
        compiler: [g++-13, clang++-17]

    steps:
      - name: Configure
        run: >
          cmake -B build
          -DCMAKE_CXX_STANDARD=${{ matrix.cxx_standard }}
          -DCMAKE_CXX_COMPILER=${{ matrix.compiler }}
```

### Caching Dependencies

CI build times add up fast. Caching both the package dependencies and the build artifacts can cut overall CI time dramatically. The cache key includes a hash of the relevant input files, so the cache is automatically invalidated when your dependencies or build configuration changes:

```yaml
      - name: Cache vcpkg
        uses: actions/cache@v4
        with:
          path: build/vcpkg_installed
          key: vcpkg-${{ matrix.os }}-${{ hashFiles('vcpkg.json') }}

      - name: Cache build
        uses: actions/cache@v4
        with:
          path: build
          key: build-${{ matrix.compiler }}-${{ hashFiles('CMakeLists.txt', 'src/**') }}
```

### What Each Config Catches

If the number of configurations feels overwhelming, it helps to know that each one catches a genuinely different class of problem. This table maps each configuration to the bugs it is best at exposing:

| Configuration | Catches |
| --- | --- |
| GCC Release | Warnings GCC-specific, optimization bugs |
| Clang Release | Warnings Clang-specific, different code gen |
| ASan + UBSan | Buffer overflows, use-after-free, UB, signed overflow |
| TSan | Data races, deadlocks, lock-order violations |
| MSVC | Windows-specific issues, different ABI, permissive mode violations |
| macOS | Apple Clang quirks, framework compatibility |
| ARM / QEMU | Endianness, alignment, weak memory model bugs |
| C++17/20/23 | Feature availability, deprecated features, breaking changes |

---

## Self-Assessment

### Q1: Why use `fail-fast: false`

By default, GitHub Actions cancels all matrix jobs when one fails. With `fail-fast: false`, all configurations run to completion. This is important because:

- A bug might only reproduce on one platform - you need all results to diagnose it
- Different sanitizers catch different bugs - you want all sanitizer results even if one job fails
- A platform-specific fix might break another platform - you need the full picture to review safely

### Q2: What is the minimum CI matrix for a cross-platform C++ library

At minimum: **GCC + Clang + MSVC** (3 compilers) with **ASan+UBSan** (1 sanitizer config). This gives you 4 jobs total and catches the vast majority of portability issues. Add TSan if you have concurrent code, and ARM cross-compilation if you target embedded.

### Q3: How do you test on ARM without ARM hardware

Use `qemu-user` (user-mode QEMU) which translates ARM instructions to x86 on the fly. Cross-compile with `aarch64-linux-gnu-g++`, then run the test binary with `qemu-aarch64`. Set `QEMU_LD_PREFIX` to point to the ARM sysroot for shared libraries. This is slower than native but catches alignment, endianness, and ABI issues.

---

## Notes

- Cache build artifacts to speed up CI - `actions/cache@v4` can cut build time by 50%+.
- Run sanitizers in Debug mode (`-O0` or `-O1`) for better error messages.
- Use `ccache` in CI to cache compilation across runs.
- Consider using `act` (https://github.com/nektos/act) to test GitHub Actions locally.
- Add a coverage job with `--coverage` and upload to Codecov or Coveralls.
