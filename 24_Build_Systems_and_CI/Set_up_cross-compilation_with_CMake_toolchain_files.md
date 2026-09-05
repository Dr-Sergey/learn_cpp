# Set up cross-compilation with CMake toolchain files

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html>  

---

## Topic Overview

**Cross-compilation** means building code on one platform (the host) that runs on a different platform (the target). A common example is compiling for a Raspberry Pi (ARM64) from a desktop Linux machine (x86_64). CMake uses **toolchain files** to configure which compiler to use, where the target's headers and libraries live, and how to find packages without accidentally picking up host libraries. This is essential for embedded Linux (ARM), RISC-V, IoT devices, or building Windows binaries on Linux.

### Cross-Compilation Model

The key mental model is that you have two worlds: the host machine where you run CMake and the compiler, and the target machine where the compiled binary will actually execute. The toolchain file bridges those two worlds.

```cpp
Host (build machine)              Target (execution platform)
x86_64 Ubuntu 22.04               ARM64 Raspberry Pi (aarch64)
+-- cmake                          +-- Application binary
+-- aarch64-linux-gnu-g++          +-- libstdc++.so (target)
+-- sysroot/ (target libs)         +-- Linux kernel (aarch64)
+-- qemu-aarch64 (for testing)
```

### Key CMake Toolchain Variables

These are the variables that control how CMake finds compilers, headers, and libraries in a cross-compilation setup. Getting them right is the entire job of a toolchain file.

| Variable | Purpose | Example |
| --- | --- | --- |
| `CMAKE_SYSTEM_NAME` | Target OS | `Linux`, `Windows`, `Generic` |
| `CMAKE_SYSTEM_PROCESSOR` | Target CPU | `aarch64`, `armv7l`, `riscv64` |
| `CMAKE_C_COMPILER` | C cross-compiler path | `aarch64-linux-gnu-gcc` |
| `CMAKE_CXX_COMPILER` | C++ cross-compiler path | `aarch64-linux-gnu-g++` |
| `CMAKE_SYSROOT` | Target root filesystem | `/opt/sysroot-aarch64` |
| `CMAKE_FIND_ROOT_PATH` | Additional search prefixes | `/opt/arm64-libs` |
| `CMAKE_FIND_ROOT_PATH_MODE_*` | Control where find_* looks | `ONLY` (target only) |

---

## Self-Assessment

### Q1: Write a toolchain file for ARM64 Linux cross-compilation from an x86_64 host

**Answer:**

A toolchain file is just a regular CMake script that sets a handful of special variables before your project is configured. The most important one is `CMAKE_SYSTEM_NAME` - setting it to something other than the host OS is what flips CMake into cross-compilation mode.

```cmake
# aarch64-linux-gnu.cmake - ARM64 cross-compilation toolchain
#
# Usage: cmake -B build --toolchain aarch64-linux-gnu.cmake

# ======= Target system description =======
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

# ======= Cross-compiler paths =======
# Install: sudo apt-get install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
set(CMAKE_C_COMPILER   aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

# ======= Sysroot (target filesystem) =======
# Contains target headers and libraries.
# Populated via: debootstrap --arch=arm64 jammy /opt/sysroot-aarch64
set(CMAKE_SYSROOT /opt/sysroot-aarch64)

# ======= Search path control =======
# ONLY  = search only in sysroot/find_root_path (not host paths)
# NEVER = search only on host (for tools like protoc)
# BOTH  = search everywhere
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)   # Use host's tools (protoc, etc.)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)    # Target libs only
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)    # Target headers only
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)    # Target packages only

# ======= Optional: qemu for test execution =======
set(CMAKE_CROSSCOMPILING_EMULATOR "qemu-aarch64;-L;${CMAKE_SYSROOT}")
```

With the toolchain file saved, here's how you set up the cross-build and verify it worked:

```bash
# Install the cross-compiler on Ubuntu
sudo apt-get install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

# Create a sysroot (minimal)
sudo debootstrap --arch=arm64 jammy /opt/sysroot-aarch64

# Configure the cross-build
cmake -B build --toolchain aarch64-linux-gnu.cmake -DCMAKE_BUILD_TYPE=Release

# Build
cmake --build build --parallel $(nproc)

# Verify the binary is ARM64
file build/myapp
# build/myapp: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV)
```

Setting `CMAKE_SYSTEM_NAME` to something other than the host triggers CMake's cross-compilation mode. The cross-compiler prefix (`aarch64-linux-gnu-`) is the standard naming convention for Debian/Ubuntu cross toolchains - the prefix encodes the target triple (architecture-vendor-OS-ABI). The sysroot provides target-architecture headers and libraries.

### Q2: Set CMAKE_SYSROOT, CMAKE_C_COMPILER, and CMAKE_FIND_ROOT_PATH correctly

**Answer:**

The reason this trips people up is the distinction between `CMAKE_SYSROOT` and `CMAKE_FIND_ROOT_PATH`. The sysroot is passed to the compiler itself (`--sysroot=`), affecting how the compiler finds headers. `CMAKE_FIND_ROOT_PATH` is for CMake's own `find_*` commands, letting you add extra search prefixes on top of the sysroot.

```cmake
# ======= Complete toolchain with fine-grained search paths =======

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

# -- Compiler --
# Must be an absolute path if not in $PATH
set(CMAKE_C_COMPILER   /usr/bin/aarch64-linux-gnu-gcc-12)
set(CMAKE_CXX_COMPILER /usr/bin/aarch64-linux-gnu-g++-12)

# -- Sysroot --
# The root of the target filesystem. CMake passes --sysroot= to GCC.
# All standard search paths (/usr/include, /usr/lib) are relative to this.
set(CMAKE_SYSROOT /opt/sysroot-aarch64)

# -- Additional search prefixes --
# CMAKE_FIND_ROOT_PATH adds extra prefixes where find_library/find_package look.
# Useful for libraries installed outside the sysroot.
set(CMAKE_FIND_ROOT_PATH
    /opt/arm64-custom-libs    # Custom-built target libraries
    /opt/arm64-vcpkg-installed # vcpkg output for arm64
)

# -- Search mode control --
#
# With CMAKE_SYSROOT=/opt/sysroot-aarch64 and
# CMAKE_FIND_ROOT_PATH=/opt/arm64-custom-libs:
#
# find_library(SSL ssl) searches:
#   /opt/sysroot-aarch64/usr/lib/
#   /opt/arm64-custom-libs/lib/
#   (NOT /usr/lib/ - that's the host!)
#
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)  # Search host for tools
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)   # Target libs only
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)   # Target headers only
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)   # Target CMake configs only

# -- Linker flags --
# Often needed for cross-compilation
set(CMAKE_EXE_LINKER_FLAGS_INIT "-static-libstdc++")
```

An example of adding a custom-built library to the find path:

```bash
# Example: adding a custom-built library to the find path
# 1. Cross-compile OpenSSL for aarch64:
#    ./Configure linux-aarch64 --prefix=/opt/arm64-custom-libs --cross-compile-prefix=aarch64-linux-gnu-
#    make -j$(nproc) && make install

# 2. Now CMake finds it via CMAKE_FIND_ROOT_PATH:
#    find_package(OpenSSL REQUIRED)  <- finds /opt/arm64-custom-libs/lib/libssl.a
```

Here is how the search works in practice:

- `CMAKE_SYSROOT` tells the compiler itself where to find standard headers/libs (`--sysroot=`).
- `CMAKE_FIND_ROOT_PATH` tells CMake's `find_*` commands additional places to search on top of the sysroot.
- `MODE_LIBRARY ONLY` means CMake prepends the sysroot and find_root_paths to every library search - it never looks in `/usr/lib` (the host). Without this, CMake may find host x86_64 libraries and silently link them into the ARM64 binary, causing an immediate crash at runtime on the target device.

### Q3: Use qemu-user to run cross-compiled tests on the host machine in CI

**Answer:**

QEMU user-mode emulation lets you run ARM64 binaries directly on an x86_64 host by translating instructions at runtime. Combined with `CMAKE_CROSSCOMPILING_EMULATOR`, CTest uses it automatically - you don't have to change your test code at all.

```cmake
# In the toolchain file:
# CMAKE_CROSSCOMPILING_EMULATOR tells ctest how to run target binaries

# For ARM64 targets:
set(CMAKE_CROSSCOMPILING_EMULATOR
    "qemu-aarch64"       # The emulator binary
    "-L" "${CMAKE_SYSROOT}"  # Where to find target's shared libs
)

# For ARM32 targets:
# set(CMAKE_CROSSCOMPILING_EMULATOR "qemu-arm;-L;${CMAKE_SYSROOT}")

# For RISC-V:
# set(CMAKE_CROSSCOMPILING_EMULATOR "qemu-riscv64;-L;${CMAKE_SYSROOT}")
```

Here is the full workflow from installation to running tests:

```bash
# Install qemu-user on the host
sudo apt-get install qemu-user qemu-user-binfmt

# Configure with the toolchain
cmake -B build --toolchain aarch64-linux-gnu.cmake -DBUILD_TESTING=ON

# Build
cmake --build build -j$(nproc)

# Run tests - ctest automatically uses qemu-aarch64
cd build && ctest --output-on-failure

# What happens when ctest runs a test:
#   Instead of: ./test_mylib
#   It runs:    qemu-aarch64 -L /opt/sysroot-aarch64 ./test_mylib
#
# qemu-user translates ARM64 instructions to x86_64 at runtime.
# ~5-10x slower than native, but works for correctness testing.

# Verify manually:
qemu-aarch64 -L /opt/sysroot-aarch64 build/test_mylib
# Output: All tests passed (runs ARM64 binary on x86_64 host!)
```

And here is how to wire this up in a GitHub Actions CI job:

```yaml
# GitHub Actions CI example with cross-compilation + qemu
name: Cross-Compile CI
on: [push]
jobs:
  arm64:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install cross toolchain + qemu
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
            qemu-user qemu-user-binfmt

      - name: Configure (cross)
        run: cmake -B build --toolchain cmake/aarch64-linux-gnu.cmake -DBUILD_TESTING=ON

      - name: Build
        run: cmake --build build -j2

      - name: Test (via qemu)
        run: ctest --test-dir build --output-on-failure
```

`CMAKE_CROSSCOMPILING_EMULATOR` integrates transparently with CTest - when it's set, every test executable is automatically prefixed with the emulator command. The `-L` flag tells qemu-user where the target's dynamic linker and shared libraries live (the sysroot). This enables running ARM64 tests in CI on standard x86_64 runners without any special hardware.

---

## Notes

- **Static linking** (`-static`) avoids sysroot issues entirely but produces larger binaries - it's a useful escape hatch when sysroot setup is painful.
- **Multiarch** on Debian/Ubuntu (`dpkg --add-architecture arm64`) provides target libraries as native packages, which is sometimes easier than maintaining a full sysroot manually.
- **vcpkg cross-compilation** uses triplets like `arm64-linux` - set `VCPKG_TARGET_TRIPLET=arm64-linux` alongside the toolchain file.
- **CMake presets** can encode toolchain paths: `"toolchainFile": "cmake/aarch64-linux-gnu.cmake"` in `CMakePresets.json`, so developers don't have to remember the flag.
- **Common mistake:** forgetting `CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY` - CMake then links host x86_64 libs into the ARM binary and the result crashes at runtime with "wrong ELF class".
