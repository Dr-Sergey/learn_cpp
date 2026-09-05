# Invoke a CMake toolchain file and control find_package in cross builds

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html>  

---

## Topic Overview

This file focuses on the **practical workflow** of using CMake toolchain files for cross-compilation: the `--toolchain` flag, `find_package` behavior in sysroot environments, and resolving the host-vs-target library confusion. See the companion file for ARM64 toolchain file structure and qemu-user testing.

### CMake Toolchain Invocation

There are two syntaxes for passing a toolchain file to CMake. Both do the same thing; `--toolchain` is just cleaner and available since CMake 3.21.

```bash
# Modern CMake (3.21+): --toolchain flag
cmake -B build --toolchain=cmake/arm-linux.cmake

# Older CMake: -DCMAKE_TOOLCHAIN_FILE
cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/arm-linux.cmake

# Both are equivalent. --toolchain is preferred for clarity.
```

### find_package Behavior in Cross-Compilation

The reason this is confusing is that `find_package` searches different paths depending on whether you're doing a native or cross-compilation build. Here is the comparison:

```cpp
Native build:
  find_package(Boost) -> searches /usr/lib, /usr/local/lib

Cross-compilation with sysroot:
  find_package(Boost) -> searches:
    1. ${CMAKE_SYSROOT}/usr/lib/cmake/Boost/
    2. ${CMAKE_FIND_ROOT_PATH}/lib/cmake/Boost/
    3. NEVER /usr/lib/ (host) when MODE is ONLY
```

---

## Self-Assessment

### Q1: Write a toolchain file for ARM Linux that sets CMAKE_SYSTEM_NAME, CMAKE_C_COMPILER, and sysroot

**Answer:**

This example targets the ARM32 hard-float ABI (armhf), which is what most Raspberry Pi OS 32-bit images use. The `gnueabihf` suffix in the compiler name tells you the target ABI: GNU EABI with hardware floating point.

```cmake
# cmake/arm-linux.cmake - Toolchain for ARM32 (armv7) Linux
# Target: Raspberry Pi 3/4 running 32-bit Raspberry Pi OS

# ======= Target identification =======
set(CMAKE_SYSTEM_NAME Linux)       # Triggers cross-compilation mode
set(CMAKE_SYSTEM_PROCESSOR armv7l) # ARM 32-bit (hard-float)

# ======= Compiler =======
# Install: sudo apt-get install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf
set(CMAKE_C_COMPILER   arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)

# ======= Sysroot =======
# The target filesystem root. GCC adds --sysroot= automatically.
# Create from a Pi SD card image or via debootstrap:
#   sudo debootstrap --arch=armhf bullseye /opt/pi-sysroot http://raspbian.raspberrypi.org/raspbian
set(CMAKE_SYSROOT /opt/pi-sysroot)

# ======= Find path control =======
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)  # Host tools (e.g., protoc)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)   # Target libraries
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)   # Target headers
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)   # Target CMake packages

# ======= Hardware floating point =======
set(CMAKE_C_FLAGS_INIT   "-mfpu=neon-vfpv4 -mfloat-abi=hard")
set(CMAKE_CXX_FLAGS_INIT "-mfpu=neon-vfpv4 -mfloat-abi=hard")

# ======= Test emulation =======
set(CMAKE_CROSSCOMPILING_EMULATOR "qemu-arm;-L;${CMAKE_SYSROOT}")
```

**Key points:**

- `CMAKE_SYSTEM_NAME Linux` switches CMake into cross-compilation mode - this is the signal that tells CMake to stop assuming host == target.
- `armv7l` = 32-bit ARM, `aarch64` = 64-bit ARM, `riscv64` = RISC-V.
- The `gnueabihf` suffix means hard-float ABI (hardware FPU). Using soft-float on a hard-float board produces binaries that run but are much slower.
- `CMAKE_C_FLAGS_INIT` sets flags once during initial configuration and is not overridden on reconfigure, which makes it the right place for ABI-defining flags.

### Q2: Use cmake --toolchain=arm-linux.cmake to configure a cross-compilation build

**Answer:**

Let's walk through the full workflow, from verification that the cross-compiler is installed all the way to deploying the binary on the target device.

```bash
# ======= Step-by-step cross-compilation workflow =======

# 1. Verify the cross-compiler is installed
arm-linux-gnueabihf-g++ --version
# arm-linux-gnueabihf-g++ (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0

# 2. Configure with the toolchain file
cmake -B build-arm \
  --toolchain cmake/arm-linux.cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_STANDARD=20 \
  -DBUILD_TESTING=ON

# CMake output shows cross-compilation is active:
# -- The C compiler identification is GNU 12.3.0
# -- The CXX compiler identification is GNU 12.3.0
# -- Detecting C compiler ABI info
# -- Check for working C compiler: /usr/bin/arm-linux-gnueabihf-gcc - works
# -- System is: Linux - 5.15 - armv7l

# 3. Build
cmake --build build-arm --parallel $(nproc)

# 4. Verify the output binary is ARM
file build-arm/myapp
# build-arm/myapp: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV),
#   dynamically linked, interpreter /lib/ld-linux-armhf.so.3,

# 5. Run tests via qemu (because CMAKE_CROSSCOMPILING_EMULATOR is set)
cd build-arm && ctest --output-on-failure

# 6. Deploy to target
scp build-arm/myapp pi@raspberrypi.local:/home/pi/
ssh pi@raspberrypi.local ./myapp
```

A cleaner approach for teams is to encode the toolchain in a CMake preset so developers don't have to remember the flag:

```json
// CMakePresets.json - encode the toolchain in a preset
{
  "version": 6,
  "configurePresets": [
    {
      "name": "arm-release",
      "displayName": "ARM Release",
      "toolchainFile": "cmake/arm-linux.cmake",
      "binaryDir": "${sourceDir}/build-arm",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release",
        "CMAKE_CXX_STANDARD": "20",
        "BUILD_TESTING": "ON"
      }
    },
    {
      "name": "native-debug",
      "displayName": "Native Debug",
      "binaryDir": "${sourceDir}/build-debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug"
      }
    }
  ]
}
```

With the preset in place, cross-compiling becomes a single command:

```bash
# Now cross-compile with a single command:
cmake --preset arm-release
cmake --build --preset arm-release
```

### Q3: Explain how find_package works for target libraries in a sysroot vs host libraries

**Answer:**

This is the most important thing to get right in cross-compilation. Linking a host library (x86_64) into a target binary (ARM) produces a binary that looks fine at link time but crashes at runtime with a cryptic "wrong ELF class" error. The `CMAKE_FIND_ROOT_PATH_MODE_*` settings are what prevent this.

```cmake
# ======= The problem: host vs target library confusion =======
#
# Without proper toolchain configuration:
#   find_package(Boost) might find /usr/lib/x86_64-linux-gnu/libboost_*.so
#   Linking these x86_64 libs into an ARM binary -> instant crash or link error
#
# ======= How CMAKE_FIND_ROOT_PATH_MODE controls search =======
#
# set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
#
# This makes find_library() search ONLY these paths:
#   ${CMAKE_SYSROOT}/usr/lib/arm-linux-gnueabihf/
#   ${CMAKE_SYSROOT}/usr/local/lib/
#   ${CMAKE_FIND_ROOT_PATH}/lib/
#
# And NEVER:
#   /usr/lib/x86_64-linux-gnu/     <- host libraries
#   /usr/local/lib/                 <- host libraries
#
# ======= But host tools must be found on the host! =======
#
# set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
#
# find_program(PROTOC protoc) -> searches /usr/bin/, /usr/local/bin/
# (host protoc, because you need to run it during the build)
```

Here is a practical CMakeLists.txt that shows how the search rules play out:

```cmake
# CMakeLists.txt - practical example
cmake_minimum_required(VERSION 3.21)
project(MyApp CXX)

# find_package follows the sysroot search rules
find_package(Boost 1.74 REQUIRED COMPONENTS filesystem)
# With sysroot: finds /opt/pi-sysroot/usr/lib/arm-linux-gnueabihf/libboost_filesystem.so
# Without sysroot: would wrongly find /usr/lib/x86_64.../libboost_filesystem.so

find_package(Threads REQUIRED)

# find_program uses host tools (MODE_PROGRAM = NEVER)
find_program(PROTOC protoc REQUIRED)
# Finds /usr/bin/protoc (x86_64 host binary - runs during build)

add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE Boost::filesystem Threads::Threads)
```

When something goes wrong, `--debug-find` is your best debugging tool:

```bash
# Debugging find_package issues:

# Show where CMake searches:
cmake -B build-arm --toolchain arm-linux.cmake --debug-find

# Output shows:
#   find_library(Boost_FILESYSTEM_LIBRARY_RELEASE)
#     searching: /opt/pi-sysroot/usr/lib/arm-linux-gnueabihf
#     found:     /opt/pi-sysroot/usr/lib/arm-linux-gnueabihf/libboost_filesystem.so

# If a target library is NOT in the sysroot, install it:
# Option 1: Copy from the Pi
#   scp pi@raspberrypi.local:/usr/lib/arm-linux-gnueabihf/libfoo.so /opt/pi-sysroot/usr/lib/arm-linux-gnueabihf/

# Option 2: Cross-compile the library
#   cmake -B build-foo --toolchain arm-linux.cmake /path/to/foo-source
#   cmake --build build-foo && cmake --install build-foo --prefix /opt/pi-sysroot/usr/local

# Option 3: Use vcpkg with a cross triplet
#   vcpkg install boost:arm-linux
```

`find_package` in cross-compilation respects the `CMAKE_FIND_ROOT_PATH_MODE_*` settings throughout. Setting `MODE_LIBRARY ONLY` restricts library searches to the sysroot and `CMAKE_FIND_ROOT_PATH` directories, preventing accidental linking of host (x86_64) libraries into the target (ARM) binary. `MODE_PROGRAM NEVER` ensures build tools (protoc, flex, bison) are found on the host, since they run during the build process itself.

---

## Notes

- **`--debug-find`** (CMake 3.17+) shows every search path for every `find_*` call - it's essential for diagnosing "library not found" in cross-compilation because the error message alone rarely tells you where it looked.
- **Toolchain files are processed early** - before `project()`. Some variables (like `CMAKE_CXX_STANDARD`) should go in the CMakeLists.txt, not the toolchain file, to avoid ordering surprises.
- **vcpkg cross-compilation:** set `VCPKG_TARGET_TRIPLET=arm-linux` and `VCPKG_CHAINLOAD_TOOLCHAIN_FILE=cmake/arm-linux.cmake` to use vcpkg with a cross toolchain.
- **Common error:** "cannot find crt1.o" or "ld: cannot find -lc" means the sysroot is missing or `CMAKE_SYSROOT` is not set correctly. These are the C runtime startup files and the C library - they must come from the target sysroot.
