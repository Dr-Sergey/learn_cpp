# Use CMake toolchain files and presets for multi-platform builds

**Category:** Cross Platform Development  
**Standard:** C++17  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html>  

---

## Topic Overview

### Toolchain Files — What and Why

A **toolchain file** tells CMake how to compile for a specific target platform. For native builds, CMake auto-detects the compiler. For cross-compilation or custom configurations - like building for ARM from an x86 Linux machine - you provide a toolchain file so CMake knows which compiler to use, where to find target headers and libraries, and how to separate host tools from target tools.

Here is a minimal toolchain file for cross-compiling to ARM64 Linux:

```cmake
# aarch64-linux.cmake — Cross-compile for ARM64 Linux
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER   aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

# Sysroot for target headers and libraries
set(CMAKE_SYSROOT /opt/sysroots/aarch64-linux)

# Where to search for libraries/headers (target only, not host)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

The `CMAKE_FIND_ROOT_PATH_MODE_*` settings are crucial: they prevent CMake from accidentally picking up host libraries when you are cross-compiling. Without them, `find_package` might link against your development machine's libraries instead of the target's. You pass the toolchain file at configure time:

```bash
cmake -B build-arm64 -DCMAKE_TOOLCHAIN_FILE=cmake/aarch64-linux.cmake
cmake --build build-arm64
```

### Windows MSVC Toolchain

For Windows builds, you often need to control the C runtime linkage. This file forces static CRT linking, which produces a self-contained binary that does not depend on a specific Visual C++ Redistributable being installed:

```cmake
# windows-msvc.cmake — Explicit MSVC configuration
set(CMAKE_SYSTEM_NAME Windows)
set(CMAKE_C_COMPILER cl)
set(CMAKE_CXX_COMPILER cl)

# Use static CRT for standalone binaries
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
```

### Embedded ARM Cortex-M Toolchain

Bare-metal embedded targets need special handling because there is no OS to run executables during CMake's compiler detection step. Notice the `CMAKE_TRY_COMPILE_TARGET_TYPE` setting at the bottom - that is the key that makes this work:

```cmake
# arm-cortex-m4.cmake
set(CMAKE_SYSTEM_NAME Generic)  # Bare-metal, no OS
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER   arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)

set(CMAKE_C_FLAGS_INIT
    "-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16")
set(CMAKE_CXX_FLAGS_INIT
    "-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16 -fno-exceptions -fno-rtti")

set(CMAKE_EXE_LINKER_FLAGS_INIT
    "-T${CMAKE_SOURCE_DIR}/linker.ld --specs=nosys.specs --specs=nano.specs")

# No standard library test — bare metal can't run executables
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
```

### CMake Presets — Modern Configuration Management

`CMakePresets.json` (CMake 3.19+) replaces ad-hoc shell scripts with a standardized, shareable configuration that lives in version control alongside your code. Instead of everyone on the team maintaining their own build scripts, you define named configurations once and anyone can reproduce an exact build with a single command.

Here is a realistic preset file covering debug, release, sanitizer, Windows, and cross-compile configurations:

```json
{
    "version": 6,
    "cmakeMinimumRequired": { "major": 3, "minor": 25, "patch": 0 },

    "configurePresets": [
        {
            "name": "linux-gcc-debug",
            "displayName": "Linux GCC Debug",
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_STANDARD": "20",
                "CMAKE_CXX_COMPILER": "g++-13",
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
            }
        },
        {
            "name": "linux-gcc-release",
            "inherits": "linux-gcc-debug",
            "displayName": "Linux GCC Release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_INTERPROCEDURAL_OPTIMIZATION": "ON"
            }
        },
        {
            "name": "linux-clang-asan",
            "displayName": "Linux Clang + ASan",
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_COMPILER": "clang++-17",
                "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer"
            }
        },
        {
            "name": "windows-msvc-release",
            "displayName": "Windows MSVC Release",
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "condition": { "type": "equals", "lhs": "${hostSystemName}", "rhs": "Windows" },
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_CXX_STANDARD": "20"
            }
        },
        {
            "name": "arm64-cross",
            "displayName": "ARM64 Cross-Compile",
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "toolchainFile": "${sourceDir}/cmake/aarch64-linux.cmake",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_CXX_STANDARD": "20"
            }
        }
    ],

    "buildPresets": [
        { "name": "linux-gcc-debug",   "configurePreset": "linux-gcc-debug" },
        { "name": "linux-gcc-release", "configurePreset": "linux-gcc-release" },
        { "name": "linux-clang-asan",  "configurePreset": "linux-clang-asan" },
        { "name": "arm64-cross",       "configurePreset": "arm64-cross" }
    ],

    "testPresets": [
        {
            "name": "linux-gcc-debug",
            "configurePreset": "linux-gcc-debug",
            "output": { "outputOnFailure": true },
            "execution": { "noTestsAction": "error", "jobs": 8 }
        }
    ]
}
```

With this file in place, building is a two-command process:

```bash
cmake --preset linux-gcc-release
cmake --build --preset linux-gcc-release
ctest --preset linux-gcc-debug
```

The `inherits` field is particularly useful: `linux-gcc-release` inherits everything from `linux-gcc-debug` and only overrides what changes. This keeps the file DRY and reduces the chance of configurations drifting apart.

### User-Local Overrides with `CMakeUserPresets.json`

Developers can add their own presets without modifying the shared file. `CMakeUserPresets.json` works exactly like `CMakePresets.json` and can inherit from the shared presets, making it easy to point at a non-standard local compiler or add personal debugging flags:

```json
{
    "version": 6,
    "configurePresets": [
        {
            "name": "my-local-debug",
            "inherits": "linux-gcc-debug",
            "cacheVariables": {
                "CMAKE_CXX_COMPILER": "/opt/gcc-14/bin/g++"
            }
        }
    ]
}
```

Add `CMakeUserPresets.json` to `.gitignore` so personal overrides never pollute the shared configuration.

### Multi-Platform CMakeLists.txt Patterns

The companion to presets is writing `CMakeLists.txt` in a way that handles platform differences cleanly at the target level rather than via global variables. The key practices are: add platform sources with `target_sources`, link platform libraries with `target_link_libraries`, and use generator expressions for compiler-specific flags:

```cmake
cmake_minimum_required(VERSION 3.25)
project(mylib CXX)

add_library(mylib src/core.cpp)
target_compile_features(mylib PUBLIC cxx_std_20)

# Platform-specific sources
if(WIN32)
    target_sources(mylib PRIVATE src/platform_win32.cpp)
    target_link_libraries(mylib PRIVATE ws2_32)  # Winsock
elseif(APPLE)
    target_sources(mylib PRIVATE src/platform_macos.cpp)
    target_link_libraries(mylib PRIVATE "-framework CoreFoundation")
else()
    target_sources(mylib PRIVATE src/platform_linux.cpp)
    target_link_libraries(mylib PRIVATE pthread)
endif()

# Compiler-specific warnings
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive->
)
```

This approach keeps platform-specific logic encapsulated in one place. Everything downstream that links against `mylib` automatically gets the right sources and libraries without needing to know about the platform.

---

## Self-Assessment

### Q1: What is the difference between a toolchain file and a preset

A **toolchain file** is a CMake script that configures the compiler, sysroot, and cross-compilation settings. It tells CMake *how* to build for a specific target - which compiler binary to use, where the target headers live, and how `find_package` should search for libraries.

A **preset** is a JSON configuration that bundles the toolchain file, build type, cache variables, generator, and other settings into a named, shareable profile. Presets are higher-level - they can reference a toolchain file as one part of the configuration, alongside things like build type and compiler flags. Think of the toolchain file as the low-level "what compiler" and the preset as the high-level "how to build a named configuration."

### Q2: Why use `CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY` for embedded

CMake's compiler detection builds and runs a small test program. On bare-metal, there is no OS to run executables, so the test fails and CMake reports that the compiler is broken. Setting `CMAKE_TRY_COMPILE_TARGET_TYPE` to `STATIC_LIBRARY` tells CMake to only compile the test program (not link or run it), which works even without a runtime environment.

### Q3: How do you handle platform-specific code in a CMake project

Use `if(WIN32)`, `if(APPLE)`, `if(UNIX)` guards with `target_sources()` to add platform-specific source files. Use generator expressions for compiler-specific flags: `$<$<CXX_COMPILER_ID:MSVC>:/W4>`. Never set global variables like `CMAKE_CXX_FLAGS` directly - use target-level commands so that flags only affect the targets that need them and do not leak into dependencies.

---

## Notes

- Always use `CMakePresets.json` for new projects - it replaces ad-hoc build scripts with a shareable, version-controlled configuration.
- Add `CMakeUserPresets.json` to `.gitignore` for developer-local customization without polluting the shared file.
- Use the `Ninja` generator for fastest builds on all platforms - it parallelizes more aggressively than Make and has better dependency tracking.
- `CMAKE_EXPORT_COMPILE_COMMANDS=ON` generates `compile_commands.json` for IDE/LSP integration - clangd, clang-tidy, and most editors can use this file for accurate code intelligence.
- Test with multiple compilers in CI: GCC, Clang, and MSVC at minimum - each exposes different bugs and has different default warning sets.
