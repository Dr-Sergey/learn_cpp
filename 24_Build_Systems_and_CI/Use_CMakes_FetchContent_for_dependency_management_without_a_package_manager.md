# Use CMake's FetchContent for dependency management without a package manager

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://cmake.org/cmake/help/latest/module/FetchContent.html>  

---

## Topic Overview

`FetchContent` downloads and builds dependencies at **configure time** directly from Git repositories, URLs, or archives - no package manager (Conan, vcpkg) needed. It's the simplest way to add third-party libraries to a CMake project, which makes it a great starting point for smaller projects or teams that do not want to manage a separate package manager setup.

The important thing to understand about `FetchContent` is *when* it runs. Downloading happens during the CMake configure step (when you run `cmake -B build`), not during the build step. After the first configure, CMake caches the downloaded sources and reuses them - so subsequent configures are fast even offline.

### FetchContent vs find_package

The table below captures when each approach makes sense. As a rough rule of thumb: use `FetchContent` when you want "it just works on any machine," and use `find_package` when the library is large, slow to compile, or already installed on your target machines:

| Aspect | `FetchContent` | `find_package` |
| --- | --- | --- |
| Source | Downloads & builds from source | Uses pre-installed library |
| Speed | Slower (compiles deps every clean build) | Fast (already compiled) |
| Version control | Pinned via GIT_TAG/URL_HASH | Whatever is installed |
| Offline builds | Needs network (or cached) | Works offline |
| Install pollution | Can pollute install target | No issue |
| Use case | Small-medium deps, CI | System libs, large deps |

### FetchContent Workflow

Here is the sequence of events during a configure when you use `FetchContent`:

```bash
cmake -B build
  ├── FetchContent_Declare()  ── records URL/tag
  ├── FetchContent_MakeAvailable()
  │     ├── Downloads source (if not cached)
  │     ├── Adds as subdirectory
  │     └── Targets now available
  └── target_link_libraries(myapp PRIVATE gtest_main)
```

---

## Self-Assessment

### Q1: Use FetchContent_Declare and FetchContent_MakeAvailable to pull googletest from GitHub

**Answer:**

The pattern is always: `include(FetchContent)`, then `FetchContent_Declare()` for each dependency to record where it lives and what version to fetch, then one call to `FetchContent_MakeAvailable()` that triggers the actual download and makes the targets available. After that, you use the targets just like any normal `find_package` result:

```cmake
cmake_minimum_required(VERSION 3.24)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# FetchContent setup
include(FetchContent)

# Declare googletest — pin to a specific commit for reproducibility
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0          # Always pin a tag or commit hash
    GIT_SHALLOW    TRUE             # Don't download full history
    FIND_PACKAGE_ARGS               # Try find_package first (CMake 3.24+)
)

# Declare fmt library
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
    GIT_SHALLOW    TRUE
)

# Declare from a URL archive (faster than git clone)
FetchContent_Declare(
    nlohmann_json
    URL      https://github.com/nlohmann/json/releases/download/v3.11.3/json.tar.xz
    URL_HASH SHA256=d6c65aca6b1ed68e7a182f4757f0f1c0e0a8c4e09dce5efec41dadb23b71a79b
)

# Download and make available (downloads only on first configure)
FetchContent_MakeAvailable(googletest fmt nlohmann_json)

# Use the fetched libraries
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE fmt::fmt nlohmann_json::nlohmann_json)

# Tests
enable_testing()
add_executable(myapp_test tests/test_main.cpp)
target_link_libraries(myapp_test PRIVATE GTest::gtest_main fmt::fmt)

include(GoogleTest)
gtest_discover_tests(myapp_test)
```

The second configure is much faster because CMake reuses the cached downloads:

```bash
# Build and test:
cmake -B build -G Ninja
# -- [fetchcontent] googletest: downloading
# -- [fetchcontent] fmt: downloading
cmake --build build
ctest --test-dir build --output-on-failure
# Second configure reuses cached downloads — no re-download
```

A few options you will want to know for `FetchContent_Declare`:

- `GIT_TAG` - always pin to a tag or commit SHA, never use `main` or `master` since that will download different code every time
- `GIT_SHALLOW TRUE` - downloads only the single commit, which is much faster than cloning the full history
- `URL_HASH SHA256=...` - verifies the archive's integrity, preventing corrupted or tampered downloads
- `FIND_PACKAGE_ARGS` (CMake 3.24+) - tries to use a system-installed version first, and only falls back to downloading if nothing is found

### Q2: Explain the difference between FetchContent (builds from source) and find_package (uses installed)

**Answer:**

Understanding the difference is important because picking the wrong strategy can waste significant build time or break reproducibility. The code comments below capture the tradeoffs concisely, but the key insight is the hybrid approach with `FIND_PACKAGE_ARGS` - it gives you speed on machines with dependencies pre-installed while gracefully handling machines that have nothing:

```cmake
# Strategy 1: FetchContent (always builds from source)
include(FetchContent)
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
)
FetchContent_MakeAvailable(fmt)
# Downloads fmt source, adds as subdirectory, compiles it with your project
# Pros: Always correct version, works on any machine, no pre-install needed
# Cons: Adds to build time, may conflict with other fetched deps

# Strategy 2: find_package (uses pre-installed)
find_package(fmt 10.2 REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)
# Looks for FmtConfig.cmake or Findfmt.cmake in CMAKE_PREFIX_PATH
# Pros: Fast (already compiled), works offline
# Cons: Must be installed first (apt, vcpkg, manual build)

# Strategy 3: Hybrid (CMake 3.24+) — best of both
include(FetchContent)
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
    FIND_PACKAGE_ARGS 10.2    # Try find_package(fmt 10.2) first
)
FetchContent_MakeAvailable(fmt)
# If fmt 10.2+ is installed -> uses it (fast)
# If not installed -> downloads and builds from source (reliable)
# Best for CI: pre-install deps in Docker image for speed,
# but FetchContent handles missing deps gracefully

# When to use which
# FetchContent:
#   - Small/medium libraries (googletest, fmt, nlohmann_json)
#   - Projects that must clone-and-build with no prerequisites
#   - CI environments without pre-installed deps
#
# find_package:
#   - Large libraries (Boost, Qt, OpenCV)
#   - System libraries (OpenSSL, zlib)
#   - When build time matters
#
# FIND_PACKAGE_ARGS (hybrid):
#   - Default choice for modern CMake 3.24+
```

### Q3: Show how EXCLUDE_FROM_ALL prevents fetched dependencies from polluting the install target

**Answer:**

This is a subtle gotcha that catches people when they first ship a library using `FetchContent`. By default, when you run `cmake --install`, CMake installs *all* targets - including the ones from your fetched dependencies. That means users installing your library also get `libfmt.a` and the fmt headers dumped into their system, which is not what you want. The fix is `EXCLUDE_FROM_ALL`:

```cmake
# Problem: FetchContent installs deps alongside your library
include(FetchContent)
FetchContent_Declare(fmt GIT_REPOSITORY ... GIT_TAG 10.2.1)
FetchContent_MakeAvailable(fmt)

# Running 'cmake --install build' installs:
#   /usr/local/lib/libfmt.a         <- Unwanted
#   /usr/local/include/fmt/...      <- Unwanted
#   /usr/local/lib/libmylib.a       <- Your library (wanted)

# Solution 1: EXCLUDE_FROM_ALL (CMake 3.28+)
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
    EXCLUDE_FROM_ALL                # <-- Prevents install of fmt targets
)
FetchContent_MakeAvailable(fmt)

# Now 'cmake --install build' only installs YOUR targets
# fmt is compiled and linked, but not installed

# Solution 2: Manual EXCLUDE_FROM_ALL (older CMake)
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
)
FetchContent_GetProperties(fmt)
if(NOT fmt_POPULATED)
    FetchContent_Populate(fmt)
    add_subdirectory(${fmt_SOURCE_DIR} ${fmt_BINARY_DIR} EXCLUDE_FROM_ALL)
endif()
# Same effect, but works with CMake < 3.28

# Solution 3: Override install() in fetched project
# Some projects check for this variable:
set(FMT_INSTALL OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(fmt)

# Verify no pollution
# cmake --install build --prefix /tmp/test-install
# ls /tmp/test-install/lib/
#   libmylib.a       # Only your library
# ls /tmp/test-install/include/
#   mylib/           # Only your headers
```

Solution 1 is the cleanest for CMake 3.28+ - one extra word in the declare block and you are done. For older CMake you fall back to the manual `FetchContent_Populate` + `add_subdirectory(... EXCLUDE_FROM_ALL)` pattern, which is more verbose but achieves the same result.

---

## Notes

- **Cache directory:** FetchContent stores downloads in `_deps/` inside the build tree - add to `.gitignore`
- **`FETCHCONTENT_FULLY_DISCONNECTED=ON`** - forces offline builds using previously cached sources
- **`FETCHCONTENT_UPDATES_DISCONNECTED=ON`** - skips update checks for already-downloaded deps
- **Override source:** `FETCHCONTENT_SOURCE_DIR_<NAME>=/path/to/local/clone` - use a local checkout instead of downloading (great for development of a dependency)
