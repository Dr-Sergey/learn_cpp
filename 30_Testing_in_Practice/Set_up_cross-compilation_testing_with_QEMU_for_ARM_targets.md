# Set up cross-compilation testing with QEMU for ARM targets

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**QEMU user-mode emulation** lets you run ARM (or other architecture) binaries directly on an x86 host machine, without any physical ARM hardware. Combined with cross-compilation, this means you can build your firmware test suite for ARM and run it in CI on standard Linux runners - catching architecture-specific bugs like endianness issues, alignment faults, and integer size assumptions before you ever touch real hardware.

The thing to understand up front is that there are two very different QEMU modes, and they serve different purposes. User-mode QEMU (`qemu-arm`) just translates ARM instructions to x86 and proxies Linux syscalls - it's fast and transparent for test executables. System-mode QEMU (`qemu-system-arm`) emulates a full board including memory-mapped peripherals, which is much slower and more complex to configure. For running unit tests, user-mode is almost always what you want.

### QEMU Modes

| Mode | Emulates | Speed | Use Case |
| --- | --- | --- | --- |
| **User-mode** (`qemu-arm`) | CPU + syscalls | Fast (10-50x native) | Run test binaries |
| **System** (`qemu-system-arm`) | Full board | Slow (100x+) | Boot firmware, test peripherals |
| **Docker multiarch** | Transparent via binfmt | Moderate | CI containers |

---

## Self-Assessment

### Q1: Set up ARM cross-compilation and QEMU test execution

**Answer:**

First, install the cross-compilation toolchain and QEMU user-mode emulator. On Ubuntu these are standard packages:

```bash
# === Install cross-compilation toolchain + QEMU ===
# Ubuntu/Debian:
sudo apt-get install -y \
    gcc-arm-linux-gnueabihf \
    g++-arm-linux-gnueabihf \
    qemu-user \
    qemu-user-static \
    libc6-armhf-cross

# For AArch64:
sudo apt-get install -y \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    qemu-user

# Verify
arm-linux-gnueabihf-g++ --version
qemu-arm --version
```

The CMake toolchain file is where the magic happens. The key line is `CMAKE_CROSSCOMPILING_EMULATOR` - when this is set, CTest automatically prepends the emulator command to every test invocation. Your `ctest` commands look exactly the same whether you're running natively or via QEMU; CMake handles the wrapping transparently.

```cmake
# === arm-toolchain.cmake ===
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)

# Sysroot for cross-compiled libraries
set(CMAKE_SYSROOT /usr/arm-linux-gnueabihf)
set(CMAKE_FIND_ROOT_PATH /usr/arm-linux-gnueabihf)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

# Tell CTest to run tests via QEMU
set(CMAKE_CROSSCOMPILING_EMULATOR
    "qemu-arm;-L;/usr/arm-linux-gnueabihf")
```

The `-L /usr/arm-linux-gnueabihf` argument tells QEMU where to find the ARM shared libraries (glibc etc.) when running a dynamically linked executable. Without it, QEMU would try to load the host's x86 libraries into an ARM binary, which fails immediately.

The main `CMakeLists.txt` is completely standard - it doesn't need to know anything about QEMU:

```cmake
# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.20)
project(embedded_tests)

set(CMAKE_CXX_STANDARD 20)

add_library(firmware_logic
    src/protocol_parser.cpp
    src/state_machine.cpp
    src/crc_calculator.cpp
)

enable_testing()
find_package(GTest REQUIRED)

add_executable(firmware_tests
    tests/test_protocol.cpp
    tests/test_state_machine.cpp
    tests/test_crc.cpp
)
target_link_libraries(firmware_tests firmware_logic GTest::gtest_main)
add_test(NAME firmware_tests COMMAND firmware_tests)
# CMAKE_CROSSCOMPILING_EMULATOR automatically prepends qemu-arm
```

Building and running looks like this - the `file` command is useful to confirm you actually produced an ARM binary and not an x86 one by accident:

```bash
# === Build and test ===
# Cross-compile for ARM
cmake -B build-arm --toolchain arm-toolchain.cmake
cmake --build build-arm

# Verify it's an ARM binary
file build-arm/firmware_tests
# Output: ELF 32-bit LSB executable, ARM, EABI5

# Run via QEMU (automatic with CTest)
ctest --test-dir build-arm --output-on-failure

# Or run manually
qemu-arm -L /usr/arm-linux-gnueabihf ./build-arm/firmware_tests
```

### Q2: Build Google Test for ARM cross-compilation

**Answer:**

When using `FetchContent` with a cross-compilation toolchain, googletest automatically gets cross-compiled as part of your project build - you don't need to cross-compile it separately. CMake propagates the toolchain settings to all `FetchContent` dependencies.

```cmake
# === Cross-compile GTest as part of the project ===
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0
)
# Prevent GTest from being installed with the project
set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# GTest is automatically cross-compiled with the project's toolchain
add_executable(arm_tests tests/test_all.cpp)
target_link_libraries(arm_tests gtest_main gmock)
add_test(NAME arm_tests COMMAND arm_tests)
```

Here's the equivalent toolchain file for 64-bit ARM (AArch64), which covers Raspberry Pi 4, Apple M1/M2 Macs (under Linux), and modern server ARM chips:

```bash
# === aarch64-toolchain.cmake (64-bit ARM) ===
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

set(CMAKE_SYSROOT /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

set(CMAKE_CROSSCOMPILING_EMULATOR
    "qemu-aarch64;-L;/usr/aarch64-linux-gnu")
```

### Q3: CI pipeline with QEMU cross-compilation testing

**Answer:**

The GitHub Actions workflow below tests both 32-bit ARM and AArch64 in parallel using a matrix. It also shows the Docker multiarch approach as an alternative - useful when you want to test in an environment that more closely resembles a real ARM Linux system (with the full distribution's package set) rather than just running a cross-compiled binary through QEMU.

```yaml
# === .github/workflows/cross-test.yml ===
name: Cross-Platform Tests

on: [push, pull_request]

jobs:
  cross-test:
    strategy:
      matrix:
        arch:

          - name: arm32

            toolchain: arm-toolchain.cmake
            packages: gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf qemu-user

          - name: aarch64

            toolchain: aarch64-toolchain.cmake
            packages: gcc-aarch64-linux-gnu g++-aarch64-linux-gnu qemu-user

    runs-on: ubuntu-latest
    name: test-${{ matrix.arch.name }}

    steps:

      - uses: actions/checkout@v4

      - name: Install cross toolchain

        run: |
          sudo apt-get update
          sudo apt-get install -y ${{ matrix.arch.packages }} ninja-build

      - name: Cross-compile

        run: |
          cmake -B build -G Ninja \
            --toolchain cmake/${{ matrix.arch.toolchain }} \
            -DCMAKE_BUILD_TYPE=Release
          cmake --build build

      - name: Verify binary architecture

        run: file build/firmware_tests

      - name: Run tests via QEMU

        run: ctest --test-dir build --output-on-failure --parallel 4
        timeout-minutes: 10

  # === Docker multiarch (alternative approach) ===
  docker-arm:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Set up QEMU for Docker

        uses: docker/setup-qemu-action@v3

      - name: Build and test in ARM container

        run: |
          docker run --rm --platform linux/arm/v7 \
            -v ${{ github.workspace }}:/src -w /src \
            gcc:13 \
            bash -c "
              apt-get update && \
              apt-get install -y cmake ninja-build && \
              cmake -B build -G Ninja && \
              cmake --build build && \
              ctest --test-dir build --output-on-failure
            "
```

The Docker multiarch approach (`docker/setup-qemu-action`) is simpler to configure because you don't need to manage a sysroot - the container already has everything the ARM binary needs. The trade-off is that it's slower and less flexible than direct cross-compilation with QEMU user-mode.

---

## Notes

- `CMAKE_CROSSCOMPILING_EMULATOR` is the key - CTest automatically prepends QEMU to all test commands.
- Use `-L /path/to/sysroot` with QEMU to resolve dynamic libraries for cross-compiled binaries.
- User-mode QEMU handles CPU emulation well but does not emulate peripherals (timers, GPIO, DMA).
- For peripheral-dependent code, use QEMU system mode with a machine definition (much more complex).
- Static linking (`-static`) avoids sysroot issues: `set(CMAKE_EXE_LINKER_FLAGS "-static")`.
- QEMU user-mode runs at roughly 10-50x slowdown vs native - fast enough for test suites.
- Docker multiarch with `setup-qemu-action` is simpler but slower than direct cross-compilation.
- Test both 32-bit ARM and AArch64 if your firmware targets both.
