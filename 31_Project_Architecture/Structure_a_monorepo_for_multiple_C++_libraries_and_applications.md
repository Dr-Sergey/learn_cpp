# Structure a monorepo for multiple C++ libraries and applications

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

A **monorepo** keeps multiple libraries, applications, and shared tooling in a single repository. For C++ projects this simplifies dependency management, ensures version consistency across all components, and enables atomic cross-project refactoring - if you rename a function in a shared library, you can update every caller in the same commit and know that everything compiles together.

The challenge is keeping the build system from becoming a monolith that rebuilds everything every time. Good monorepo CMake lets you build just the libraries that changed, and CI pipelines should detect which components were modified and only rebuild those.

### Monorepo vs Polyrepo

| Aspect | Monorepo | Polyrepo |
| --- | --- | --- |
| Dependency sync | Always consistent | Version pinning, diamond deps |
| Cross-project refactor | Atomic commit | Multi-repo PR coordination |
| Build time | Potentially large | Per-repo, fast |
| CI/CD | Complex (selective builds) | Simple per-repo |
| Code ownership | Need CODEOWNERS | Natural per-repo |

There is no universally correct answer here. The table's "potentially large" build time for monorepos is manageable with selective CI builds and build caching (ccache, sccache). The "diamond dependency" problem in polyrepos - where library A and library B both depend on different versions of library C - becomes genuinely painful at scale and is one of the strongest arguments for a monorepo.

---

## Self-Assessment

### Q1: Design a monorepo directory layout

The directory layout expresses the ownership model. Libraries that multiple teams depend on live in `libs/`. Applications that consume those libraries live in `apps/`. Shared build tooling lives in `cmake/`. The top-level `CMakeLists.txt` is the orchestrator that pulls everything together.

**Answer:**

```cpp
project-root/
├─ CMakeLists.txt              # Top-level: orchestrates everything
├─ cmake/
│   ├─ CompilerOptions.cmake    # Shared compiler settings
│   ├─ Coverage.cmake           # Coverage targets
│   └─ Sanitizers.cmake         # ASan/UBSan/TSan settings
├─ libs/                        # Shared libraries
│   ├─ core/
│   │   ├─ CMakeLists.txt
│   │   ├─ include/core/
│   │   ├─ src/
│   │   └─ tests/
│   ├─ networking/
│   │   ├─ CMakeLists.txt
│   │   ├─ include/networking/
│   │   ├─ src/
│   │   └─ tests/
│   └─ serialization/
│       └─ ...
├─ apps/                        # Applications
│   ├─ server/
│   │   ├─ CMakeLists.txt
│   │   ├─ src/
│   │   └─ tests/
│   └─ cli/
│       ├─ CMakeLists.txt
│       └─ src/
├─ tools/                       # Dev tools, code generators
│   ├─ proto_gen/
│   └─ bench/
├─ third_party/                 # Vendored or FetchContent deps
│   ├─ CMakeLists.txt
│   ├─ googletest/
│   └─ fmt/
├─ docs/
├─ .github/CODEOWNERS
└─ CMakePresets.json
```

```cmake
# === Top-level CMakeLists.txt ===
cmake_minimum_required(VERSION 3.25)
project(myproject VERSION 1.0.0)

# Shared settings
include(cmake/CompilerOptions.cmake)

# Options to build subsets
option(BUILD_TESTING "Build tests" ON)
option(BUILD_APPS "Build applications" ON)
option(BUILD_TOOLS "Build tools" OFF)

# Third-party dependencies
add_subdirectory(third_party)

# Libraries (always built)
add_subdirectory(libs/core)
add_subdirectory(libs/networking)
add_subdirectory(libs/serialization)

# Applications (optional)
if(BUILD_APPS)
    add_subdirectory(apps/server)
    add_subdirectory(apps/cli)
endif()

# Tools (optional)
if(BUILD_TOOLS)
    add_subdirectory(tools/proto_gen)
endif()
```

The `BUILD_TOOLS=OFF` default is deliberate. Code generators and dev utilities take time to build and most builds do not need them. CI can enable them explicitly. Individual developers enable them only when working on tooling.

### Q2: Design per-library CMake with proper target export

Each library owns its own `CMakeLists.txt` and declares its dependencies through `target_link_libraries`. The distinction between `PUBLIC` and `PRIVATE` dependencies is important: `PUBLIC` means the dependency is part of the library's interface and consumers will need it too; `PRIVATE` means it is an implementation detail that consumers do not need to know about.

**Answer:**

```cmake
# === libs/core/CMakeLists.txt ===
add_library(core
    src/logger.cpp
    src/config.cpp
    src/thread_pool.cpp
)
add_library(myproject::core ALIAS core)

target_include_directories(core
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)
target_compile_features(core PUBLIC cxx_std_20)

# === libs/networking/CMakeLists.txt ===
add_library(networking
    src/tcp_server.cpp
    src/http_handler.cpp
)
add_library(myproject::networking ALIAS networking)

target_link_libraries(networking
    PUBLIC myproject::core       # Uses core's public API
    PRIVATE asio                 # Implementation detail
)
target_include_directories(networking
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

# === apps/server/CMakeLists.txt ===
add_executable(server
    src/main.cpp
    src/routes.cpp
)
target_link_libraries(server PRIVATE
    myproject::networking
    myproject::serialization
)

# === Tests co-located with library ===
# libs/core/tests/CMakeLists.txt
if(BUILD_TESTING)
    add_executable(core_tests
        test_logger.cpp
        test_config.cpp
        test_thread_pool.cpp
    )
    target_link_libraries(core_tests PRIVATE
        myproject::core
        GTest::gtest_main
    )
    add_test(NAME core_tests COMMAND core_tests)
endif()
```

Notice that `asio` is a `PRIVATE` dependency of `networking`. Consumers of `myproject::networking` will not need to link against `asio` themselves - CMake handles the transitive dependency tracking automatically. If you mistakenly made it `PUBLIC`, consumers would get pulled into depending on `asio` even if they never use networking features directly.

### Q3: Selective CI builds for changed components

The main scalability problem with monorepos and CI is that you do not want to rebuild and retest every component every time someone touches one library. Path-based change detection solves this: if only `libs/core` changed, only build and test `core` and the components that depend on it.

**Answer:**

```yaml
# === .github/workflows/ci.yml ===
name: CI
on: [push, pull_request]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      core: ${{ steps.filter.outputs.core }}
      networking: ${{ steps.filter.outputs.networking }}
      server: ${{ steps.filter.outputs.server }}
    steps:

      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2

        id: filter
        with:
          filters: |
            core:

              - 'libs/core/**'
              - 'cmake/**'

            networking:

              - 'libs/networking/**'
              - 'libs/core/**'          # networking depends on core

            server:

              - 'apps/server/**'
              - 'libs/**'               # server depends on all libs

  build-core:
    needs: detect-changes
    if: needs.detect-changes.outputs.core == 'true'
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4
      - run: |

          cmake -B build -DBUILD_APPS=OFF -DBUILD_TOOLS=OFF
          cmake --build build
          ctest --test-dir build --output-on-failure

  build-server:
    needs: detect-changes
    if: needs.detect-changes.outputs.server == 'true'
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4
      - run: |

          cmake -B build -DBUILD_TESTING=ON
          cmake --build build --target server
          cmake --build build --target server_tests
          ctest --test-dir build -R server --output-on-failure
```

```cpp
# === .github/CODEOWNERS ===
libs/core/           @team-platform
libs/networking/     @team-networking
libs/serialization/  @team-data
apps/server/         @team-backend
apps/cli/            @team-cli
cmake/               @team-platform
```

The `CODEOWNERS` file and the path-based CI detection reinforce the same team ownership model. Changes to `libs/core/` require review from `@team-platform` and trigger the core build job. That combination of automatic enforcement is what makes a monorepo with many teams actually workable.

---

## Notes

- Use ALIAS targets (`myproject::core`) for a clean namespace and instant error detection.
- Each library should be independently testable with its own `add_test` targets.
- Selective CI avoids rebuilding the entire monorepo on every push.
- `CODEOWNERS` enforces review by the owning team.
- `CMakePresets.json` standardizes build configurations across developers.
- Use `FetchContent` or `add_subdirectory(third_party/...)` for external dependencies.
- For very large monorepos (100+ libs), consider Bazel or build caching (ccache, sccache).
