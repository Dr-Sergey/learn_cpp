# Use Conan 2.0 for C++ package management

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://docs.conan.io/2/>  

---

## Topic Overview

Conan 2.0 is a decentralized C++ package manager that downloads, builds, and caches pre-compiled binaries for your dependencies. If a pre-built binary matches your exact compiler, OS, architecture, and settings, Conan grabs it instantly. If not, it builds from source with `--build=missing`. The integration with CMake is intentionally clean: Conan generates a **CMakeToolchain** file that sets your compiler and standard, and a set of **CMakeDeps** config files that make `find_package()` work for every dependency you declared. From CMake's perspective, it looks exactly like the libraries were installed on the system.

### Conan 2.0 Workflow

Here is the overall pipeline from your recipe file to a working build:

```cpp
conanfile.py (declares deps)
     │
     ▼
conan install . --build=missing
     │
     ├── Downloads binaries from ConanCenter (if available)
     ├── Builds from source (if --build=missing and no binary)
     ├── Generates: conan_toolchain.cmake
     └── Generates: FindBoost.cmake, Findfmt.cmake, etc.
          │
          ▼
cmake --preset conan-release
     └── find_package(Boost) ── works via generated files
```

### Conan 2.0 vs 1.x

If you have used Conan before, the 2.0 changes are significant enough that existing recipes will not just work. The generator names changed, and the layout system is smarter:

| Feature | Conan 1.x | Conan 2.0 |
| --- | --- | --- |
| Config file | `conanfile.txt` or `.py` | `conanfile.py` preferred |
| Generators | `cmake`, `cmake_find_package` | `CMakeDeps` + `CMakeToolchain` |
| Profile system | Basic | Full graph resolution |
| Layout | Manual | `cmake_layout()` auto-detects |

---

## Self-Assessment

### Q1: Create a conanfile.py that requires Boost and generates CMake find modules

**Answer:**

A `conanfile.py` is a Python class that describes your project to Conan. The `requires` field lists your runtime dependencies, `generate()` creates the CMake integration files, and `build()` / `package()` drive the actual compilation. Notice that the `CMakeLists.txt` side is completely ordinary - it just calls `find_package` and links targets. Conan handles making those targets available:

```python
# conanfile.py
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout, CMakeDeps, CMakeToolchain


class MyProjectConan(ConanFile):
    name = "myproject"
    version = "1.0.0"
    settings = "os", "compiler", "build_type", "arch"

    # Dependencies
    requires = (
        "boost/1.84.0",
        "fmt/10.2.1",
        "spdlog/1.13.0",
    )
    tool_requires = "cmake/3.28.1"   # Build tool dependency
    test_requires = "gtest/1.14.0"   # Test-only dependency

    def layout(self):
        cmake_layout(self)           # Auto: build/Release, build/Debug

    def generate(self):
        # CMakeDeps: generates Find*.cmake / *Config.cmake for each dep
        deps = CMakeDeps(self)
        deps.generate()

        # CMakeToolchain: generates conan_toolchain.cmake
        tc = CMakeToolchain(self)
        tc.variables["BUILD_TESTING"] = True
        tc.generate()

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()

    def package(self):
        cmake = CMake(self)
        cmake.install()
```

The `CMakeLists.txt` that consumes these generated files looks exactly like normal CMake - no Conan-specific calls needed here:

```cmake
# CMakeLists.txt — uses the generated find modules
cmake_minimum_required(VERSION 3.24)
project(MyProject LANGUAGES CXX)

find_package(Boost REQUIRED COMPONENTS filesystem system)
find_package(fmt REQUIRED)
find_package(spdlog REQUIRED)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE
    Boost::filesystem
    Boost::system
    fmt::fmt
    spdlog::spdlog
)
```

### Q2: Run conan install --build=missing to download and build dependencies

**Answer:**

The workflow is: detect your compiler profile once, then `conan install` to get your dependencies, then build through CMake presets. The `--build=missing` flag tells Conan to compile from source anything for which a pre-built binary does not exist for your exact configuration:

```bash
# Step 1: Create a Conan profile
conan profile detect  # Auto-detect compiler, OS, arch
conan profile show
# Output:
# [settings]
# os=Linux
# compiler=gcc
# compiler.version=13
# compiler.libcxx=libstdc++11
# build_type=Release
# arch=x86_64

# Step 2: Install dependencies
conan install . --build=missing
# --build=missing: build from source if no pre-built binary exists
# Downloads:
#   - boost/1.84.0: Found pre-built binary
#   - fmt/10.2.1: Found pre-built binary
#   - spdlog/1.13.0: Building from source (no binary for this config)

# Output:
# ======== Computing dependency graph ========
# boost/1.84.0: Already installed
# fmt/10.2.1: Already installed
# spdlog/1.13.0: Building from source
# ======== Installing packages ========
# spdlog/1.13.0: Calling build()
# spdlog/1.13.0: Package built
# ======== Finalizing install ========
# Generated: conan_toolchain.cmake
# Generated: BoostConfig.cmake, fmtConfig.cmake, spdlogConfig.cmake

# Step 3: Build with CMake using Conan preset
cmake --preset conan-release    # Uses conan_toolchain.cmake
cmake --build --preset conan-release
ctest --preset conan-release

# Debug build
conan install . --build=missing -s build_type=Debug
cmake --preset conan-debug
cmake --build --preset conan-debug

# Useful flags
conan install . --build=missing --output-folder=build
conan install . --build="boost/*"    # Force rebuild specific package
conan install . --build=*            # Rebuild everything from source
```

### Q3: Create a custom Conan recipe (ConanFile class) for a header-only library

**Answer:**

Header-only libraries need a slightly different recipe because there is no compilation step - you just need to copy the headers into the package folder and describe the include path. The `package_id(self)` override that calls `self.info.clear()` is important: it tells Conan that this package produces the same result regardless of OS, compiler, or build type, so a single package binary works for everyone:

```python
# conanfile.py for a header-only library "mymath"
from conan import ConanFile
from conan.tools.files import copy
from conan.tools.layout import basic_layout
import os


class MyMathConan(ConanFile):
    name = "mymath"
    version = "1.0.0"
    license = "MIT"
    description = "A header-only C++20 math library"
    url = "https://github.com/myorg/mymath"

    # Header-only: no settings needed (no compilation)
    no_copy_source = True   # Source doesn't need to be copied to build dir

    # Export the headers and license
    exports_sources = "include/*", "LICENSE"

    def layout(self):
        basic_layout(self, src_folder=".")

    def package(self):
        copy(self, "*.hpp", src=os.path.join(self.source_folder, "include"),
             dst=os.path.join(self.package_folder, "include"))
        copy(self, "LICENSE", src=self.source_folder,
             dst=os.path.join(self.package_folder, "licenses"))

    def package_info(self):
        # Header-only: no libs to link, just include path
        self.cpp_info.bindirs = []
        self.cpp_info.libdirs = []
        # Include path is set automatically from package_folder/include

    def package_id(self):
        # Header-only: same package for all configurations
        self.info.clear()
```

To test and publish the package locally and then upload it to a remote:

```bash
# Test locally:
conan create . --build-folder=tmp
# Output:
# mymath/1.0.0: Package created
# mymath/1.0.0: Package size: 12 KB

# Use in another project's conanfile.py:
# requires = "mymath/1.0.0"

# Upload to a remote:
conan remote add myremote https://myartifactory.com/conan
conan upload "mymath/1.0.0" -r myremote --confirm
```

---

## Notes

- **Conan cache:** `~/.conan2/` stores all downloaded/built packages - `conan cache path boost/1.84.0` shows the path
- **Lockfiles:** `conan lock create .` generates `conan.lock` for fully reproducible dependency resolution
- **Profiles:** create custom profiles for cross-compilation, sanitizers, etc. in `~/.conan2/profiles/`
- **ConanCenter:** <https://conan.io/center> - the public repository with 1500+ packages
