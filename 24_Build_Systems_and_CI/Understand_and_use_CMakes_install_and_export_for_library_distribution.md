# Understand and use CMake's install() and export() for library distribution

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://cmake.org/cmake/help/latest/command/install.html>  

---

## Topic Overview

When distributing a C++ library, you want downstream projects to use `find_package(MyLib)` and automatically get proper targets with include paths, link libraries, and compile features attached. CMake's `install()` and `export()` commands generate the config files that make this possible. Without these, a consumer of your library has to manually specify include paths and linker flags, which is fragile and error-prone.

### Package Distribution Flow

Here is the full picture from your library's `CMakeLists.txt` to a downstream project successfully using it:

```cpp
Your library project:
  install(TARGETS mylib EXPORT MyLibTargets ...)
  install(EXPORT MyLibTargets ...)
  configure_package_config_file(...)

  cmake --install build --prefix /usr/local
  |
Installed files:
  /usr/local/lib/libmylib.a
  /usr/local/include/mylib/mylib.h
  /usr/local/lib/cmake/MyLib/MyLibConfig.cmake
  /usr/local/lib/cmake/MyLib/MyLibTargets.cmake
  |
Downstream project:
  find_package(MyLib REQUIRED)
  target_link_libraries(app PRIVATE MyLib::mylib)
  # Gets include dirs, link libs, compile features automatically!
```

### install() vs export() vs configure_package_config_file()

If the command names feel confusing at first, here is what each one does:

| Command | Purpose |
| --- | --- |
| `install(TARGETS)` | Copy libraries/executables to install prefix |
| `install(DIRECTORY)` | Copy headers to install prefix |
| `install(EXPORT)` | Generate `*Targets.cmake` with imported targets |
| `export(EXPORT)` | Generate build-tree `*Targets.cmake` (for `add_subdirectory` use) |
| `configure_package_config_file()` | Generate `*Config.cmake` for `find_package()` |
| `write_basic_package_version_file()` | Generate `*ConfigVersion.cmake` for version checking |

---

## Self-Assessment

### Q1: Write install(TARGETS, EXPORT, INCLUDES DESTINATION) for a library with headers

**Answer:**

The key detail here is the relationship between `install(TARGETS)` with `EXPORT` and the later `install(EXPORT)` call. The first call registers the target with a named export set. The second call writes the export set to a `.cmake` file in the install prefix. You need both.

```cmake
# CMakeLists.txt for a distributable library
cmake_minimum_required(VERSION 3.21)
project(MyLib VERSION 1.2.3 LANGUAGES CXX)

# ======= Define the library =======
add_library(mylib
    src/mylib.cpp
    src/parser.cpp
)

# Public headers are in include/mylib/
target_include_directories(mylib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

target_compile_features(mylib PUBLIC cxx_std_20)

# ======= Install the library binary =======
install(TARGETS mylib
    EXPORT MyLibTargets           # Name the export set
    LIBRARY DESTINATION lib       # .so on Linux
    ARCHIVE DESTINATION lib       # .a on Linux
    RUNTIME DESTINATION bin       # .dll on Windows
    INCLUDES DESTINATION include  # Adds include/ to INTERFACE_INCLUDE_DIRECTORIES
)

# ======= Install headers =======
install(DIRECTORY include/mylib
    DESTINATION include
    FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)

# ======= Install the export set =======
install(EXPORT MyLibTargets
    FILE MyLibTargets.cmake
    NAMESPACE MyLib::            # Creates MyLib::mylib target
    DESTINATION lib/cmake/MyLib
)
```

After running `cmake --install`, the result looks like this:

```bash
# Build and install
cmake -B build -DCMAKE_INSTALL_PREFIX=/opt/mylib
cmake --build build
cmake --install build

# Result:
# /opt/mylib/
# +-- include/mylib/mylib.h
# +-- lib/libmylib.a
# +-- lib/cmake/MyLib/
#     +-- MyLibTargets.cmake
#     +-- MyLibTargets-release.cmake
#     +-- MyLibConfig.cmake
#     +-- MyLibConfigVersion.cmake
```

**Key details:**

- `EXPORT MyLibTargets` registers the target with the named export set. This is just a name you choose - it shows up as the filename of the generated `.cmake` file.
- `NAMESPACE MyLib::` creates namespaced imported targets. Downstream code uses `MyLib::mylib`, not just `mylib`. The namespace prevents name collisions between libraries.
- `INCLUDES DESTINATION include` adds the include path to the imported target's interface properties, so downstream code automatically gets the right `-I` flag without having to specify it manually.
- Generator expressions (`$<BUILD_INTERFACE:...>` / `$<INSTALL_INTERFACE:...>`) handle the fact that the include path is different during building (pointing to the source tree) versus after installation (pointing to the install prefix).

### Q2: Generate a CMake config file so downstream projects can find_package(MyLib) after install

**Answer:**

The `install(EXPORT)` call from Q1 generates `MyLibTargets.cmake`, but downstream projects also need `MyLibConfig.cmake` as the entry point for `find_package`. This is the file CMake actually searches for. You generate it from a template using `CMakePackageConfigHelpers`.

```cmake
# ======= Add to CMakeLists.txt (after install(TARGETS)) =======
include(CMakePackageConfigHelpers)

# Generate MyLibConfigVersion.cmake
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion  # 1.x.y compatible with 1.a.b
)

# Generate MyLibConfig.cmake from template
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/MyLibConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfig.cmake"
    INSTALL_DESTINATION lib/cmake/MyLib
)

# Install the config files
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/MyLibConfigVersion.cmake"
    DESTINATION lib/cmake/MyLib
)
```

The template file is minimal - it just includes the targets file and checks for required components:

```cmake
# cmake/MyLibConfig.cmake.in - the template
@PACKAGE_INIT@

include("${CMAKE_CURRENT_LIST_DIR}/MyLibTargets.cmake")

# Check that all required components are available
check_required_components(MyLib)

# Optional: find dependencies that MyLib needs
# include(CMakeFindDependencyMacro)
# find_dependency(Boost 1.74 REQUIRED COMPONENTS filesystem)
# find_dependency(fmt 10.0 REQUIRED)
```

Here is how a downstream project uses the installed library:

```cmake
# ======= Downstream project using MyLib =======
# downstream/CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(MyApp CXX)

# find_package looks in:
#   /opt/mylib/lib/cmake/MyLib/MyLibConfig.cmake
find_package(MyLib 1.2 REQUIRED)

add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE MyLib::mylib)
# Automatically gets:
#   - Include path: /opt/mylib/include
#   - Link library: /opt/mylib/lib/libmylib.a
#   - C++20 standard (from compile features)
```

Building the downstream project is straightforward - just point `CMAKE_PREFIX_PATH` at the install location:

```bash
# Build downstream project
cmake -B build -DCMAKE_PREFIX_PATH=/opt/mylib
cmake --build build

# CMake finds MyLib:
# -- Found MyLib: /opt/mylib (found version "1.2.3")
```

**How find_package works after install:**

1. `find_package(MyLib 1.2 REQUIRED)` searches for `MyLibConfig.cmake` in standard paths plus any paths in `CMAKE_PREFIX_PATH`.
2. `MyLibConfig.cmake` includes `MyLibTargets.cmake`, which defines the `MyLib::mylib` imported target with all its properties.
3. `MyLibConfigVersion.cmake` checks version compatibility. `SameMajorVersion` means 1.2.3 satisfies a request for 1.2 because the major version matches.

### Q3: Explain the difference between build-tree exports and install-tree exports

**Answer:**

This is a distinction that confuses almost everyone the first time. There are two separate workflows for consuming a library: you can either install it and use `find_package`, or you can use it directly from the build tree with `add_subdirectory` or `FetchContent`. The two export commands exist to serve these two different consumers.

```cmake
# ======= Install-tree export (for installed packages) =======
# Generated by: install(EXPORT ...)
# File goes to: ${CMAKE_INSTALL_PREFIX}/lib/cmake/MyLib/MyLibTargets.cmake
# Contains: paths relative to the install prefix
#
# MyLibTargets.cmake (installed) contains:
# set_target_properties(MyLib::mylib PROPERTIES
#   INTERFACE_INCLUDE_DIRECTORIES "${_IMPORT_PREFIX}/include"
#   IMPORTED_LOCATION "${_IMPORT_PREFIX}/lib/libmylib.a"
# )
# -> Portable: works after the package is installed anywhere

# install(EXPORT MyLibTargets
#     FILE MyLibTargets.cmake
#     NAMESPACE MyLib::
#     DESTINATION lib/cmake/MyLib
# )


# ======= Build-tree export (for add_subdirectory / FetchContent) =======
# Generated by: export(EXPORT ...)
# File goes to: ${CMAKE_CURRENT_BINARY_DIR}/MyLibTargets.cmake
# Contains: absolute paths to the BUILD tree
#
# MyLibTargets.cmake (build tree) contains:
# set_target_properties(MyLib::mylib PROPERTIES
#   INTERFACE_INCLUDE_DIRECTORIES "/home/user/mylib/include"
#   IMPORTED_LOCATION "/home/user/mylib/build/libmylib.a"
# )
# -> NOT portable: only works on this specific machine

# export(EXPORT MyLibTargets
#     FILE "${CMAKE_CURRENT_BINARY_DIR}/MyLibTargets.cmake"
#     NAMESPACE MyLib::
# )
```

In practice, a well-maintained library's `CMakeLists.txt` typically provides both, so it works for all use cases:

```cmake
# ======= Practical comparison =======

# The full CMakeLists.txt typically has BOTH:

# 1. Build-tree export: allows FetchContent / add_subdirectory users
#    to find your targets without installing
export(EXPORT MyLibTargets
    FILE "${CMAKE_CURRENT_BINARY_DIR}/cmake/MyLibTargets.cmake"
    NAMESPACE MyLib::
)

# 2. Install-tree export: allows installed package users
#    to find your targets via find_package
install(EXPORT MyLibTargets
    FILE MyLibTargets.cmake
    NAMESPACE MyLib::
    DESTINATION lib/cmake/MyLib
)

# ======= When to use each =======
#
# Build-tree export:
#   - Parent project uses add_subdirectory(mylib)
#   - FetchContent_Declare(mylib GIT_REPOSITORY ...)
#   - Developer workflow: edit mylib and app simultaneously
#   - NOT for distribution (absolute paths!)
#
# Install-tree export:
#   - Library is installed: cmake --install build --prefix /usr/local
#   - Package manager distributes it (vcpkg, Conan, apt)
#   - Downstream uses: find_package(MyLib REQUIRED)
#   - Portable: relative paths resolved at find_package time
```

| Aspect | Build-tree export | Install-tree export |
| --- | --- | --- |
| Generated by | `export(EXPORT ...)` | `install(EXPORT ...)` |
| Path type | Absolute (machine-specific) | Relative (portable) |
| Location | Build directory | Install prefix |
| Consumer | `add_subdirectory` / FetchContent | `find_package` after install |
| Distributable | No | Yes |
| Use case | Development / monorepo | Packaging / distribution |

---

## Notes

- **`GNUInstallDirs`** module provides portable install directories: `include(GNUInstallDirs)` then use `CMAKE_INSTALL_LIBDIR`, `CMAKE_INSTALL_INCLUDEDIR` instead of hardcoded `lib/` and `include/`. This matters on distros that install to `lib64/` instead of `lib/`.
- **`find_dependency`** in `Config.cmake.in` propagates transitive dependencies - if MyLib depends on Boost, including `find_dependency(Boost)` in your config file means downstream consumers get Boost automatically without having to add it themselves.
- **Version compatibility modes:** `SameMajorVersion`, `SameMinorVersion`, `ExactVersion`, `AnyNewerVersion` - choose based on your ABI stability guarantees. If you promise semantic versioning, `SameMajorVersion` is the right choice.
- **CPack** can create installers (DEB, RPM, NSIS) from the same `install()` rules, turning a properly configured CMake project into a distributable package with minimal extra work.
