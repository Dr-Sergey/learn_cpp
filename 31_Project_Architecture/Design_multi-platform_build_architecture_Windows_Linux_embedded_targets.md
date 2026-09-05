# Design multi-platform build architecture (Windows, Linux, embedded targets)

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

A **multi-platform build architecture** compiles the same codebase for different operating systems and hardware targets. In C++ this requires abstracting platform-specific code, managing toolchains, and configuring CMake to handle cross-compilation. Modern projects target Windows, Linux, macOS, and embedded ARM/RISC-V from a single source tree.

The goal is that the vast majority of your codebase is platform-agnostic, and the platform-specific parts are isolated in clearly marked locations. That way a developer on Windows and a developer on Linux see the same public API - only the implementation files differ, and CMake selects the right ones automatically.

### Platform Abstraction Strategies

| Strategy | Complexity | Flexibility | Overhead |
| --- | --- | --- | --- |
| `#ifdef` blocks | Low | Low (messy at scale) | Zero |
| Platform source files | Medium | High | Zero |
| Abstract interface + factory | High | Highest | vtable |
| CMake generator expressions | Build-level | N/A | Zero |

---

## Self-Assessment

### Q1: Structure platform-specific code with CMake

The cleanest approach at scale is not `#ifdef` soup in your source files - it's having separate `.cpp` files per platform, all implementing the same public header. CMake's `if(WIN32)` / `if(UNIX)` conditions select which source files to compile. The consumer of the library never sees the platform difference; they just include the public header.

The directory layout block shows the physical file structure that pairs with the CMakeLists. Each platform gets its own subdirectory under `src/platform/`, and embedded targets get their own as well. CMake adds the right subdirectory's files depending on what you're building for.

**Answer:**

```cmake
# === CMakeLists.txt: Platform-specific sources ===
add_library(platform STATIC
    src/platform/common.cpp
)

# Platform-specific source selection
if(WIN32)
    target_sources(platform PRIVATE
        src/platform/windows/filesystem.cpp
        src/platform/windows/threading.cpp
        src/platform/windows/network.cpp
    )
    target_link_libraries(platform PRIVATE ws2_32 shlwapi)
elseif(UNIX AND NOT APPLE)
    target_sources(platform PRIVATE
        src/platform/linux/filesystem.cpp
        src/platform/linux/threading.cpp
        src/platform/linux/network.cpp
    )
    target_link_libraries(platform PRIVATE pthread)
elseif(APPLE)
    target_sources(platform PRIVATE
        src/platform/macos/filesystem.cpp
        src/platform/macos/threading.cpp
        src/platform/macos/network.cpp
    )
    target_link_libraries(platform PRIVATE "-framework Foundation")
endif()

# Cross-compilation for embedded ARM
if(CMAKE_SYSTEM_NAME STREQUAL "Generic")
    target_sources(platform PRIVATE
        src/platform/embedded/filesystem_fatfs.cpp
        src/platform/embedded/threading_freertos.cpp
    )
    target_compile_definitions(platform PUBLIC PLATFORM_EMBEDDED)
endif()

target_include_directories(platform PUBLIC include/platform)
```

```cpp
# Directory layout:
src/platform/
+-- common.cpp             # Shared implementation
+-- windows/
|   +-- filesystem.cpp     # Win32 API
|   +-- threading.cpp      # Windows threads
|   +-- network.cpp        # Winsock
+-- linux/
|   +-- filesystem.cpp     # POSIX
|   +-- threading.cpp      # pthreads
|   +-- network.cpp        # Berkeley sockets
+-- macos/
|   +-- ...
+-- embedded/
    +-- filesystem_fatfs.cpp
    +-- threading_freertos.cpp
```

The `CMAKE_SYSTEM_NAME STREQUAL "Generic"` condition is the conventional signal that you're targeting a bare-metal embedded system with no OS. At that point you swap in RTOS and filesystem drivers instead of POSIX equivalents.

### Q2: Set up cross-compilation toolchain files

A toolchain file is a CMake script that tells the build system *which compiler to use* and *what flags to pass* when targeting a different platform. You pass it via `-DCMAKE_TOOLCHAIN_FILE=...` at configure time. The key settings are the compiler executables, the CPU-specific flags, and `CMAKE_TRY_COMPILE_TARGET_TYPE` - without that last one, CMake will try to *run* test executables during configuration, which fails immediately when you're cross-compiling for ARM.

`CMakePresets.json` is the modern way to standardize these configurations across your whole team. Instead of everyone memorizing the right flags and paths, they just run `cmake --preset arm-embedded`.

**Answer:**

```cmake
# === cmake/toolchains/arm-none-eabi.cmake ===
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(TOOLCHAIN_PREFIX arm-none-eabi-)

set(CMAKE_C_COMPILER   ${TOOLCHAIN_PREFIX}gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}g++)
set(CMAKE_ASM_COMPILER ${TOOLCHAIN_PREFIX}gcc)
set(CMAKE_OBJCOPY      ${TOOLCHAIN_PREFIX}objcopy)
set(CMAKE_SIZE         ${TOOLCHAIN_PREFIX}size)

# Don't try to run executables during configure
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# MCU-specific flags
set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16")
set(CMAKE_C_FLAGS_INIT   "${MCU_FLAGS} -ffunction-sections -fdata-sections")
set(CMAKE_CXX_FLAGS_INIT "${MCU_FLAGS} -ffunction-sections -fdata-sections -fno-rtti -fno-exceptions")
set(CMAKE_EXE_LINKER_FLAGS_INIT "-Wl,--gc-sections -specs=nosys.specs")

# === cmake/toolchains/linux-aarch64.cmake ===
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER   aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

```json
// === CMakePresets.json: unified build presets ===
{
  "version": 6,
  "configurePresets": [
    {
      "name": "windows-release",
      "generator": "Ninja",
      "binaryDir": "build/windows-release",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release"
      },
      "condition": { "type": "equals", "lhs": "${hostSystemName}", "rhs": "Windows" }
    },
    {
      "name": "linux-release",
      "generator": "Ninja",
      "binaryDir": "build/linux-release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    },
    {
      "name": "arm-embedded",
      "generator": "Ninja",
      "binaryDir": "build/arm-embedded",
      "toolchainFile": "cmake/toolchains/arm-none-eabi.cmake",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "MinSizeRel" }
    }
  ]
}
```

The `-ffunction-sections -fdata-sections` + `-Wl,--gc-sections` combination is standard practice for embedded targets: it lets the linker strip unused functions and data, which can dramatically reduce binary size on a microcontroller with limited flash.

### Q3: Write platform-abstracted code with compile-time selection

This is the end result: a `FileSystem` class with the same public header on all platforms, and completely different implementations underneath. On Linux it delegates to `std::filesystem`, on the embedded target it calls into the FatFS library. The consumer code just includes `platform/filesystem.h` and calls `FileSystem::read_file()` - no ifdefs, no platform checks.

The reason this is better than scattering ifdefs everywhere is maintainability: when you add a new platform, you create a new `.cpp` file and update CMakeLists - you don't have to hunt through every source file for `#ifdef PLATFORM_X` blocks.

**Answer:**

```cpp
// === include/platform/filesystem.h (public interface) ===
#pragma once
#include <string>
#include <vector>
#include <optional>

namespace platform {

class FileSystem {
public:
    static bool exists(const std::string& path);
    static std::optional<std::string> read_file(const std::string& path);
    static bool write_file(const std::string& path, const std::string& data);
    static std::vector<std::string> list_directory(const std::string& path);
    static std::string temp_directory();
};

}  // namespace platform

// === src/platform/linux/filesystem.cpp ===
#include "platform/filesystem.h"
#include <fstream>
#include <filesystem>

namespace platform {

bool FileSystem::exists(const std::string& path) {
    return std::filesystem::exists(path);
}

std::optional<std::string> FileSystem::read_file(const std::string& path) {
    std::ifstream f(path, std::ios::binary);
    if (!f) return std::nullopt;
    return std::string{std::istreambuf_iterator<char>(f), {}};
}

std::string FileSystem::temp_directory() {
    return std::filesystem::temp_directory_path().string();
}

}  // namespace platform

// === src/platform/embedded/filesystem_fatfs.cpp ===
#include "platform/filesystem.h"
#include "fatfs/ff.h"

namespace platform {

bool FileSystem::exists(const std::string& path) {
    FILINFO fno;
    return f_stat(path.c_str(), &fno) == FR_OK;
}

std::optional<std::string> FileSystem::read_file(const std::string& path) {
    FIL fil;
    if (f_open(&fil, path.c_str(), FA_READ) != FR_OK)
        return std::nullopt;
    auto size = f_size(&fil);
    std::string buf(size, '\0');
    UINT br;
    f_read(&fil, buf.data(), size, &br);
    f_close(&fil);
    return buf;
}

}  // namespace platform
```

Both implementations return `std::optional<std::string>` on failure - no exceptions needed. The embedded version uses `f_open`/`f_read`/`f_close` from FatFS, while the desktop version uses standard C++ streams. The interface contract is identical.

---

## Notes

- **Platform-specific source files** are cleaner than `#ifdef` blocks at scale.
- CMake selects the correct sources; consumers only see the public header.
- **Toolchain files** configure the compiler, linker, and sysroot for cross-compilation.
- `CMakePresets.json` standardizes build configurations across developers and CI.
- `CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY` prevents CMake from trying to run executables during configure (essential for embedded cross-compilation).
- Use `std::filesystem` on desktop; FatFS, LittleFS, or SPIFFS on embedded.
- CI matrix: build for all targets on every PR to catch platform-specific regressions early.
