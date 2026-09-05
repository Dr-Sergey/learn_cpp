# Use CMake modern targets and avoid directory-level commands

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://cmake.org/cmake/help/latest/>  

---

## Topic Overview

The single most important thing to know about modern CMake (3.0 and later) is that it is **target-based**, not directory-based. Properties like include paths, compile definitions, and link libraries are attached to specific targets - not to the directory you happen to be in when you call a command. This distinction matters enormously once your project has more than one target or depends on any third-party libraries.

The old-style, directory-level commands affect every target in the current CMakeLists.txt scope. The modern, target-level commands are precise and propagate dependencies correctly through the build graph:

```cpp
OLD (directory-level):           MODERN (target-level):
  include_directories(...)         target_include_directories(lib PUBLIC ...)
  add_definitions(...)             target_compile_definitions(lib PRIVATE ...)
  add_compile_options(...)         target_compile_options(lib PRIVATE ...)
  link_libraries(...)              target_link_libraries(app PRIVATE lib)

  // BAD: Affects ALL targets       // GOOD: Only affects specified target
  // BAD: No dependency propagation // GOOD: PUBLIC/PRIVATE/INTERFACE propagation
```

---

## Self-Assessment

### Q1: Target-based CMakeLists.txt with PRIVATE/PUBLIC/INTERFACE

The three visibility keywords - `PRIVATE`, `PUBLIC`, and `INTERFACE` - are the heart of modern CMake. They control whether a property stays inside a target, propagates to consumers, or exists only for consumers. Here is a realistic project structure that demonstrates all three:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(ModernCMakeDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# === Library target ===
add_library(mathlib src/math.cpp)

# PUBLIC: consumers also need these
target_include_directories(mathlib PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# PRIVATE: only mathlib needs this internally
target_compile_definitions(mathlib PRIVATE MATHLIB_BUILDING)

# PUBLIC: consumers inherit this flag
target_compile_definitions(mathlib PUBLIC MATHLIB_VERSION=2)

# === Another library that depends on mathlib ===
add_library(physics src/physics.cpp)
target_include_directories(physics PUBLIC include)
target_link_libraries(physics PUBLIC mathlib)  # physics users also get mathlib

# === Application target ===
add_executable(app src/main.cpp)

# PRIVATE: only app links these, not propagated
target_link_libraries(app PRIVATE physics)

# PRIVATE: only app sees these options
target_compile_options(app PRIVATE -Wall -Wextra -Wpedantic)

# === INTERFACE library (header-only) ===
add_library(concepts_lib INTERFACE)
target_include_directories(concepts_lib INTERFACE include/concepts)
target_compile_features(concepts_lib INTERFACE cxx_std_20)
```

The propagation rules are worth memorizing. If you get them wrong, consumers of your library will either fail to compile (missing includes) or pick up settings they should not see (your internal compile flags):

| Keyword | Compiling target | Consumers of target |
| --- | --- | --- |
| PRIVATE | Yes | No |
| PUBLIC | Yes | Yes |
| INTERFACE | No | Yes |

### Q2: Why `target_compile_options()` is better than `add_compile_options()`

The reason this matters in practice is third-party code. If you use `add_compile_options(-Werror)` in your root `CMakeLists.txt` and then pull in a third-party library with `add_subdirectory`, that library will also be compiled with `-Werror`. If it has any warnings - and many do - your build breaks, even though the problem is not in your code.

With target-level commands, the flags stay exactly where you put them:

```cmake
# BAD: directory-level - affects EVERY target in this directory
add_compile_options(-Wall -Wextra -Werror)
# Problem: third-party subdirectories also get -Werror
# add_subdirectory(third_party/noisy_lib)  # fails to compile

# GOOD: target-level - only your targets get strict warnings
target_compile_options(mylib PRIVATE -Wall -Wextra -Werror)
target_compile_options(myapp PRIVATE -Wall -Wextra -Werror)
# third_party/noisy_lib is unaffected

# For libraries: PUBLIC flags propagate to consumers
# BAD: forces YOUR warnings onto all consumers
target_compile_options(mylib PUBLIC -Werror)  # Don't do this!

# GOOD: keep strictness PRIVATE
target_compile_options(mylib PRIVATE -Werror)
```

The last two entries are particularly important for library authors. Publishing a library with `PUBLIC -Werror` is considered impolite - you are forcing every consumer of your library to also have zero warnings under your compiler version, which may not be achievable for them.

| Approach | Scope | Third-party safe? | Recommendation |
| --- | --- | --- | --- |
| `add_compile_options()` | All targets in directory | No | Avoid |
| `target_compile_options(PRIVATE)` | Single target | Yes | Use this |
| `target_compile_options(PUBLIC)` | Target + consumers | Depends | Rarely needed |

### Q3: Proper `install()` rules and export sets

Once you want other projects to consume your library via `find_package()`, you need to set up export sets. This is the mechanism that lets downstream projects write `find_package(MyProject REQUIRED)` and immediately get the right include paths, compile flags, and link targets - without knowing anything about where you installed the files.

Here is a complete install setup:

```cmake
# CMakeLists.txt (continued)
include(GNUInstallDirs)

# Install the library and headers
install(TARGETS mathlib physics
    EXPORT MyProjectTargets   # Create an export set
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Generate and install the config file for find_package()
install(EXPORT MyProjectTargets
    FILE MyProjectTargets.cmake
    NAMESPACE MyProject::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/MyProject
)

# Create MyProjectConfig.cmake
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/MyProjectConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/MyProject
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfig.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/MyProject
)
```

The config template that goes with this is minimal:

```cmake
# cmake/MyProjectConfig.cmake.in
@PACKAGE_INIT@
include("${CMAKE_CURRENT_LIST_DIR}/MyProjectTargets.cmake")
check_required_components(MyProject)
```

Once installed, a downstream consumer gets everything they need automatically:

```cmake
# Consumer's CMakeLists.txt
find_package(MyProject REQUIRED)
target_link_libraries(consumer_app PRIVATE MyProject::mathlib)
# Automatically gets: include dirs, compile defs, link flags
```

---

## Notes

- Never use `include_directories()`, `link_libraries()`, `add_definitions()` in modern CMake.
- Use `target_compile_features(lib PUBLIC cxx_std_20)` instead of `set(CMAKE_CXX_STANDARD 20)`.
- Generator expressions (`$<BUILD_INTERFACE:...>`) handle build vs install paths.
- `INTERFACE` libraries are perfect for header-only libraries.
- Use `cmake --install build --prefix /install/path` to test install rules locally.
