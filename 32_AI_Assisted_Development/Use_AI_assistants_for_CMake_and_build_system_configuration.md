# Use AI assistants for CMake and build system configuration

**Category:** AI Assisted Development  
**Standard:** Not version-specific  

---

## Topic Overview

CMake has a reputation for being one of the more unpleasant parts of C++ development, and that reputation is at least partially deserved. The documentation is dense, the patterns are non-obvious, and the difference between "modern CMake" (target-based, circa 2016+) and legacy CMake (global include/link commands from the CMake 2.x era) is enough to make tutorials actively misleading depending on their age. AI assistants can generate **CMakeLists.txt** from scratch, debug build issues, add dependencies correctly, configure cross-compilation, and translate between build systems (Makefile -> CMake, Bazel -> CMake). AI is especially valuable here because the cost of getting CMake wrong is high - you can waste hours on dependency wiring and linker errors that have nothing to do with your actual code.

### AI CMake Assistance Areas

Here's where AI genuinely saves time. "Excellent" tasks are the ones where AI can essentially template a correct solution from a project description; "Good" tasks need some back-and-forth but AI still accelerates them significantly.

| Task | AI Effectiveness | Example Prompt |
| --- | --- | --- |
| Generate CMakeLists.txt | Excellent | "Create CMake for this project structure" |
| Add vcpkg/Conan dependency | Excellent | "Add gRPC with vcpkg to this CMake" |
| Fix build errors | Good | "Why does this CMake error occur?" |
| Cross-compilation toolchain | Good | "CMake toolchain for ARM Cortex-M4" |
| CTest integration | Excellent | "Add GTest with CTest to this project" |
| CPack packaging | Good | "Generate DEB/RPM packages" |
| CMake presets | Good | "Create presets for debug/release/CI" |

---

## Self-Assessment

### Q1: Generate project CMake from scratch

**Answer:**

The key to a good CMake generation prompt is showing the actual directory layout and listing the dependencies explicitly. The more concrete the description, the less the AI has to guess.

```cpp
=== PROMPT ===

"Generate a modern CMakeLists.txt (CMake 3.25+) for this
project structure. Use target-based commands only (no
global include_directories or link_libraries).
Dependencies: fmt, spdlog, GTest, protobuf.
Package manager: vcpkg.

my_project/
  src/
    main.cpp
    server/
      server.cpp
      server.hpp
    proto/
      messages.proto
  include/
    my_project/
      config.hpp
      types.hpp
  tests/
    server_test.cpp
    integration_test.cpp
  CMakeLists.txt"
```

The AI produces a CMakeLists.txt that follows modern conventions: `find_package` for all dependencies, a library target that's shared between the executable and tests, and generator expressions for compiler-specific flags. Notice that test discovery uses `gtest_discover_tests` rather than manually listing test executables.

```cmake
# === AI-generated CMakeLists.txt ===
cmake_minimum_required(VERSION 3.25)
project(my_project
    VERSION 1.0.0
    LANGUAGES CXX
)

# -- Standards --
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# -- Dependencies (vcpkg) --
find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(Protobuf CONFIG REQUIRED)
find_package(GTest CONFIG REQUIRED)

# -- Protobuf generation --
set(PROTO_FILES src/proto/messages.proto)
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${PROTO_FILES})

# -- Main library (shared between app and tests) --
add_library(my_project_lib
    src/server/server.cpp
    ${PROTO_SRCS}
)
target_include_directories(my_project_lib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
        ${CMAKE_CURRENT_BINARY_DIR}  # For generated proto headers
)
target_link_libraries(my_project_lib
    PUBLIC
        fmt::fmt
        spdlog::spdlog
        protobuf::libprotobuf
)
target_compile_options(my_project_lib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

# -- Executable --
add_executable(my_project src/main.cpp)
target_link_libraries(my_project PRIVATE my_project_lib)

# -- Tests --
enable_testing()
add_executable(my_project_tests
    tests/server_test.cpp
    tests/integration_test.cpp
)
target_link_libraries(my_project_tests PRIVATE
    my_project_lib
    GTest::gtest_main
)

include(GoogleTest)
gtest_discover_tests(my_project_tests)
```

The `$<BUILD_INTERFACE:...>` / `$<INSTALL_INTERFACE:...>` generator expression on the include path is the right way to handle headers that live in different locations during build versus install. Many hand-written CMake files get this wrong.

### Q2: Debug CMake build errors

**Answer:**

When CMake fails, the error messages can be cryptic if you don't know what you're looking for. AI is good at translating these into the root cause and showing multiple fix options.

```cpp
=== PROMPT ===

"Explain this CMake error and show the fix:

CMake Error at CMakeLists.txt:18 (find_package):
  By not providing 'FindBoost.cmake' in
  CMAKE_MODULE_PATH this project has asked CMake to
  find a package configuration file provided by 'Boost',
  but CMake did not find one.

  Could not find a package configuration file provided by
  'Boost' (requested version 1.82) with any of the
  following names:
    BoostConfig.cmake
    boost-config.cmake

I installed Boost via vcpkg. Using vcpkg toolchain file."


=== AI DIAGNOSIS ===
```

There are three separate issues that could produce this error, and the AI presents all of them. The most common culprit with vcpkg is the toolchain ordering issue - if you set `CMAKE_TOOLCHAIN_FILE` after `project()`, vcpkg hasn't initialized yet when `find_package` runs.

```cmake
# Problem: vcpkg's Boost doesn't always provide config-mode files
# The error means CMake can't find BoostConfig.cmake

# Fix 1: Use MODULE mode (FindBoost.cmake ships with CMake)
find_package(Boost 1.82 REQUIRED
    COMPONENTS filesystem system
    # No CONFIG keyword - uses FindBoost.cmake
)

# Fix 2: If using vcpkg, ensure toolchain is set BEFORE project()
cmake_minimum_required(VERSION 3.25)

# This MUST be before project():
if(DEFINED ENV{VCPKG_ROOT})
    set(CMAKE_TOOLCHAIN_FILE
        "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
        CACHE STRING "")
endif()

project(my_project LANGUAGES CXX)

# Fix 3: Use individual Boost components via vcpkg
find_package(boost_filesystem CONFIG REQUIRED)
find_package(boost_system CONFIG REQUIRED)
target_link_libraries(my_target PRIVATE
    Boost::filesystem
    Boost::system
)
```

Fix 2 is the one most people don't know about. CMake processes the toolchain file during the `project()` call, so if you set `CMAKE_TOOLCHAIN_FILE` after `project()`, it effectively does nothing.

### Q3: Generate CMake presets and cross-compilation

**Answer:**

CMake presets are the modern replacement for having a collection of shell scripts that run CMake with different flags. They're shareable, versionable, and IDE-friendly. AI generates them well when you describe the configurations you need.

```cpp
=== PROMPT ===

"Generate CMakePresets.json with:

- Debug and Release configurations
- GCC and Clang compiler options
- Sanitizer builds (ASan, TSan, UBSan)
- CI build preset (Release + LTO + static analysis)
- vcpkg integration"
```

The generated presets use inheritance (`"inherits"`) to avoid repeating the vcpkg toolchain setup in every configuration. Each named preset becomes a `cmake --preset <name>` command you can run directly.

```json
{
    "version": 6,
    "cmakeMinimumRequired": { "major": 3, "minor": 25, "patch": 0 },
    "configurePresets": [
        {
            "name": "default",
            "hidden": true,
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_TOOLCHAIN_FILE": {
                    "value": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
                    "type": "FILEPATH"
                },
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
            }
        },
        {
            "name": "debug",
            "inherits": "default",
            "displayName": "Debug",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug"
            }
        },
        {
            "name": "release",
            "inherits": "default",
            "displayName": "Release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_INTERPROCEDURAL_OPTIMIZATION": "ON"
            }
        },
        {
            "name": "asan",
            "inherits": "debug",
            "displayName": "Debug + ASan",
            "cacheVariables": {
                "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer"
            }
        },
        {
            "name": "tsan",
            "inherits": "debug",
            "displayName": "Debug + TSan",
            "cacheVariables": {
                "CMAKE_CXX_FLAGS": "-fsanitize=thread -fno-omit-frame-pointer"
            }
        },
        {
            "name": "ci",
            "inherits": "release",
            "displayName": "CI Build",
            "cacheVariables": {
                "CMAKE_CXX_CLANG_TIDY": "clang-tidy;--warnings-as-errors=*"
            }
        }
    ],
    "buildPresets": [
        { "name": "debug", "configurePreset": "debug" },
        { "name": "release", "configurePreset": "release" },
        { "name": "ci", "configurePreset": "ci", "jobs": 4 }
    ],
    "testPresets": [
        {
            "name": "debug",
            "configurePreset": "debug",
            "output": { "outputOnFailure": true }
        }
    ]
}
```

With this file in place, your CI script becomes `cmake --preset ci && cmake --build --preset ci && ctest --preset debug` - three readable lines that capture all the build configuration without magic shell flags.

---

## Notes

- Always use **target-based CMake** (`target_link_libraries`, `target_include_directories`), never the global commands - modern CMake's whole design philosophy is that dependencies flow through targets, and mixing in global commands breaks that model.
- AI is excellent at **translating between build systems** - "convert this Makefile to CMake" works surprisingly well as a prompt.
- For **vcpkg**, the toolchain file must be set **before** `project()` - this is a common AI-generated bug to watch for in generated code.
- Ask AI to generate **CMakePresets.json** instead of bash scripts for build configurations - presets are IDE-friendly and don't require the reader to decode shell syntax.
- AI can generate **FetchContent** blocks for header-only libraries without a package manager, which is useful for dependencies that aren't in vcpkg yet.
- For cross-compilation, provide the **target triple** and **sysroot path** in the prompt - AI needs that context to generate a correct toolchain file.
- `CMAKE_EXPORT_COMPILE_COMMANDS` is essential for IDE integration and tools like clangd and clang-tidy - always enable it in generated files.
