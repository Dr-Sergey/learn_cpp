# Use Conan 2.0 for dependency management with CMake integration

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://docs.conan.io/2/>  

---

## Topic Overview

This third Conan file focuses on the **CMake integration details**: how `CMakeDeps` and `CMakeToolchain` generators work, the `conan install` -> `conan build` workflow for reproducible builds, and how Conan resolves transitive dependency conflicts in the graph.

The reason this integration is worth understanding deeply is that it is the bridge between Conan's world and CMake's world. Once you understand what `CMakeToolchain` and `CMakeDeps` generate and why, the "magic preset" workflow makes complete sense instead of feeling like a black box.

### Integration Architecture

Here is what actually happens when `conan install` runs. Two types of files get generated, and each plays a distinct role in the subsequent CMake build:

```cpp
conanfile.py
     │
     ▼  conan install .
     │
     ├── CMakeToolchain -> conan_toolchain.cmake
     │     Sets: CMAKE_CXX_STANDARD, CMAKE_BUILD_TYPE, compiler flags
     │     Creates: CMakePresets.json (conan-release, conan-debug)
     │
     └── CMakeDeps -> Find*.cmake / *-config.cmake
           For each dependency: Boost, fmt, spdlog, etc.
           Enables: find_package(Boost REQUIRED)

     ▼  cmake --preset conan-release
     │     Reads conan_toolchain.cmake via preset
     │     find_package() finds generated config files
     │
     ▼  cmake --build --preset conan-release
```

`CMakeToolchain` is about *how* to compile (compiler settings, standard, flags). `CMakeDeps` is about *what* to link against (the targets and include paths for each dependency). Together they give CMake everything it needs.

---

## Self-Assessment

### Q1: Create a conanfile.py that declares dependencies and integrates with CMakeDeps and CMakeToolchain

**Answer:**

The `generate()` method is where the CMake integration is wired up. `CMakeToolchain` produces the toolchain file and the preset entries; `CMakeDeps` produces a config file for every dependency. After this, the `CMakeLists.txt` looks completely ordinary - it has no idea Conan is involved:

```python
# conanfile.py
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout, CMakeDeps, CMakeToolchain


class HttpServerConan(ConanFile):
    name = "httpserver"
    version = "1.0.0"
    settings = "os", "compiler", "build_type", "arch"

    requires = (
        "boost/1.84.0",
        "fmt/10.2.1",
        "nlohmann_json/3.11.3",
        "openssl/3.2.1",
    )
    test_requires = "gtest/1.14.0"

    exports_sources = "CMakeLists.txt", "src/*", "include/*", "tests/*"

    def layout(self):
        cmake_layout(self)
        # Auto-detects:
        # - source folder: where CMakeLists.txt is
        # - build folder: build/Release or build/Debug
        # - generators folder: build/generators/

    def generate(self):
        # CMakeToolchain
        # Generates conan_toolchain.cmake with:
        #   - CMAKE_CXX_STANDARD
        #   - CMAKE_BUILD_TYPE
        #   - Compiler flags from Conan profile
        #   - CMakePresets.json entries (conan-release, conan-debug)
        tc = CMakeToolchain(self)
        tc.variables["BUILD_TESTING"] = True
        tc.preprocessor_definitions["APP_VERSION"] = self.version
        tc.generate()

        # CMakeDeps
        # Generates for each dependency:
        #   - boost-config.cmake, boost-targets.cmake
        #   - fmt-config.cmake, fmt-targets.cmake
        #   - etc.
        # These enable find_package() in CMakeLists.txt
        deps = CMakeDeps(self)
        deps.generate()

    def build(self):
        cmake = CMake(self)
        cmake.configure()    # cmake --preset conan-{build_type}
        cmake.build()        # cmake --build --preset conan-{build_type}
        cmake.test()         # ctest --preset conan-{build_type}

    def package(self):
        cmake = CMake(self)
        cmake.install()      # cmake --install
```

The `CMakeLists.txt` side is pure CMake with no Conan-specific calls. This is intentional - your build system description stays portable:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.24)
project(HttpServer LANGUAGES CXX)

# find_package works because CMakeDeps generated the config files
find_package(Boost REQUIRED COMPONENTS system beast)
find_package(fmt REQUIRED)
find_package(nlohmann_json REQUIRED)
find_package(OpenSSL REQUIRED)

add_executable(httpserver
    src/main.cpp
    src/server.cpp
    src/router.cpp
)
target_link_libraries(httpserver PRIVATE
    Boost::beast
    Boost::system
    fmt::fmt
    nlohmann_json::nlohmann_json
    OpenSSL::SSL
    OpenSSL::Crypto
)

# Tests
if(BUILD_TESTING)
    find_package(GTest REQUIRED)
    enable_testing()
    add_executable(httpserver_test tests/test_router.cpp)
    target_link_libraries(httpserver_test PRIVATE GTest::gtest_main)
    include(GoogleTest)
    gtest_discover_tests(httpserver_test)
endif()
```

### Q2: Run conan install and conan build to reproduce a build from scratch

**Answer:**

The `conan install` -> `conan build` sequence is designed to be fully reproducible. On any machine with the same Conan profile, you will get the same build. The "from scratch" path at the end is especially useful to put in your project README so new contributors can be up and running with a single sequence of commands:

```bash
# Full reproducible workflow

# Step 1: Install dependencies (generates toolchain + find modules)
conan install . --build=missing -s build_type=Release
# Output:
# ======== Computing dependency graph ========
# Requirements:
#     boost/1.84.0        - Cache
#     fmt/10.2.1          - Cache
#     nlohmann_json/3.11.3 - Cache
#     openssl/3.2.1       - Cache
# Test requirements:
#     gtest/1.14.0        - Cache
# ======== Finalizing install ========
# Generated:
#   build/Release/generators/conan_toolchain.cmake
#   build/Release/generators/boost-config.cmake
#   build/Release/generators/fmt-config.cmake
#   build/Release/generators/CMakePresets.json (merged into root)

# Step 2: Configure + Build + Test using Conan's CMake helper
conan build .
# Internally runs:
#   cmake --preset conan-release
#   cmake --build --preset conan-release
#   ctest --preset conan-release

# Or manually with cmake presets
cmake --preset conan-release
cmake --build --preset conan-release
ctest --preset conan-release

# Debug build
conan install . --build=missing -s build_type=Debug
conan build . -s build_type=Debug

# From scratch on a new machine
git clone https://github.com/myorg/httpserver.git
cd httpserver
pip install conan                     # Install Conan
conan profile detect                  # Detect compiler
conan install . --build=missing       # Download + build deps
conan build .                         # Configure + build + test
# Done! Fully reproducible from zero to running tests
```

### Q3: Explain the Conan 2.0 graph model and how it resolves transitive dependency conflicts

**Answer:**

Transitive dependencies are where package managers earn their keep. You directly depend on `spdlog`, and `spdlog` depends on `fmt`. Your project also directly depends on `fmt`. Now you have two paths to `fmt` in the graph, potentially at different versions. Conan has explicit rules for resolving these situations, and understanding them saves you a lot of head-scratching when a build breaks.

The dependency graph for a realistic project looks like this. The conflict cases are annotated inline:

```cpp
Dependency Graph for httpserver:
═══════════════════════════════
httpserver/1.0.0
├── boost/1.84.0
│   └── zlib/1.3.1       <- transitive dep
├── fmt/10.2.1
├── spdlog/1.13.0
│   └── fmt/10.2.1       <- CONFLICT? Same version -> merged
├── openssl/3.2.1
│   └── zlib/1.3.1       <- CONFLICT? Same version -> merged
└── gtest/1.14.0 (test)
```

The resolution rules map to very common real-world scenarios. Rule 2 ("closest to root wins") is the one that trips people up the most because it means your top-level `requires` silently overrides a transitive dependency's preference - which is usually what you want, but occasionally causes a downstream package to break if it relied on a specific version:

```python
# Rule 1: Same version -> merged (no conflict)
# httpserver requires fmt/10.2.1
# spdlog requires fmt/10.2.1
# -> One copy of fmt/10.2.1 used everywhere

# Rule 2: Different versions -> "closest to root" wins
# httpserver requires fmt/10.2.1      (depth 1 — wins)
# spdlog requires fmt/[>=9.0 <11.0]   (depth 2 — range satisfied)
# -> fmt/10.2.1 selected

# Rule 3: Incompatible version ranges -> ERROR
# httpserver requires fmt/10.2.1
# some_lib requires fmt/9.1.0         (exact, not a range)
# -> ERROR: Version conflict! Must resolve manually

# Rule 4: Override in consumer (force a version)
def requirements(self):
    self.requires("fmt/10.2.1")
    self.requires("spdlog/1.13.0")
    # Force fmt version even for spdlog's transitive dep:
    self.requires("fmt/10.2.1", override=True)

# Rule 5: Version ranges for flexibility
def requirements(self):
    self.requires("fmt/[>=10.0 <11.0]")    # Accept any 10.x
    self.requires("spdlog/[>=1.12 <2.0]")  # Accept any 1.12+
    # Conan finds a compatible fmt version for both
```

When you hit a conflict you cannot understand just from reading the recipe, the graph visualization commands are invaluable:

```bash
# Visualize the resolved graph:
conan graph info . --format=html > graph.html
# Shows all deps, versions, which conflicts were resolved, and how

conan graph info . --format=dot | dot -Tpng > graph.png
# Graphviz visualization
```

---

## Notes

- **`cmake_layout()`** auto-generates the folder structure: `build/<build_type>/generators/` for generated files
- **CMakeToolchain auto-generates CMakePresets.json** entries - `conan-release` and `conan-debug` presets appear automatically
- **`conan lock create .`** freezes the entire dependency graph into `conan.lock` for CI reproducibility
- **Profiles for cross-compilation:** `conan install . -pr:h=arm64 -pr:b=default` - host profile for target, build profile for tools
