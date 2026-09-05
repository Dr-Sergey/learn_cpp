# Use package managers (Conan, vcpkg) in project architecture

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

If you've ever cloned a C++ project and then spent an hour hunting down the right version of Boost or trying to make someone's hand-rolled `FindFoo.cmake` work, you already understand the problem that package managers solve. **Package managers** automate dependency installation, version resolution, and build integration so that "clone and build" actually works on the first try.

**vcpkg** (from Microsoft) and **Conan** (from JFrog) are the two dominant C++ package managers. Both solve "dependency hell" but with different philosophies: vcpkg integrates very tightly with CMake and keeps configuration minimal, while Conan is more flexible and explicitly supports any build system through generators.

### vcpkg vs Conan

| Aspect | vcpkg | Conan |
| --- | --- | --- |
| Integration | CMake-native (toolchain file) | Generators for CMake, Meson, etc. |
| Version mgmt | manifest mode (vcpkg.json) | conanfile.txt/conanfile.py |
| Binary caching | Built-in | Built-in + remote |
| Custom packages | Overlay ports | conanfile.py recipes |
| CI integration | Simple | Flexible |
| Package count | ~2500 | ~1700 (ConanCenter) |

---

## Self-Assessment

### Q1: Set up vcpkg with manifest mode in a project

**Answer:**

Manifest mode means you describe your dependencies in a `vcpkg.json` file that lives in your project root - similar to `package.json` in the JavaScript world. vcpkg reads it, installs the right versions locally inside your build tree, and CMake picks them up automatically through the toolchain file.

```json
// === vcpkg.json (project manifest) ===
{
  "name": "my-project",
  "version-semver": "1.0.0",
  "dependencies": [
    "fmt",
    "spdlog",
    {
      "name": "boost-asio",
      "version>=": "1.83.0"
    },
    {
      "name": "gtest",
      "host": true
    },
    {
      "name": "protobuf",
      "features": ["zlib"]
    }
  ],
  "overrides": [
    { "name": "fmt", "version": "10.1.1" }
  ],
  "builtin-baseline": "a1a1cbc975528dd2758c143091a4b3246af5e221"
}
```

The `builtin-baseline` hash pins the entire set of available package versions to a specific commit of the vcpkg registry, which is how you get fully reproducible builds. Without it, "latest available" shifts under you over time.

On the CMake side, all you need is `find_package` - vcpkg handles making the packages findable:

```cmake
# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.25)
project(my-project)

# vcpkg integrates via toolchain file:
# cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake

# Or set in CMakePresets.json:
# "toolchainFile": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"

find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(Boost REQUIRED COMPONENTS system)
find_package(GTest CONFIG REQUIRED)
find_package(Protobuf CONFIG REQUIRED)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE
    fmt::fmt
    spdlog::spdlog
    Boost::system
    protobuf::libprotobuf
)
```

The cleanest way to wire the toolchain file without typing it on the command line every time is to put it in `CMakePresets.json`:

```json
// === CMakePresets.json ===
{
  "version": 6,
  "configurePresets": [
    {
      "name": "default",
      "generator": "Ninja",
      "binaryDir": "build",
      "toolchainFile": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release"
      }
    }
  ]
}
```

Now `cmake --preset default` is all anyone needs to type, and the toolchain is picked up automatically.

### Q2: Set up Conan 2.x with CMake integration

**Answer:**

Conan works slightly differently: instead of a toolchain file that CMake loads, Conan generates CMake files that describe where packages live, and you include those generated files from your `CMakeLists.txt`. The simplest way to declare dependencies is `conanfile.txt`:

```ini
# === conanfile.txt (simple) ===
[requires]
fmt/10.1.1
spdlog/1.12.0
boost/1.83.0
protobuf/3.21.12

[generators]
CMakeDeps
CMakeToolchain

[options]
boost/*:without_test=True
boost/*:without_python=True
```

When you need conditional dependencies - for example, skipping `protobuf` on a bare-metal target - you graduate to `conanfile.py`, which is a regular Python class:

```python
# === conanfile.py (advanced) ===
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout

class MyProject(ConanFile):
    name = "my-project"
    version = "1.0.0"
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"

    def requirements(self):
        self.requires("fmt/10.1.1")
        self.requires("spdlog/1.12.0")
        self.requires("boost/1.83.0")
        if self.settings.os != "baremetal":
            self.requires("protobuf/3.21.12")

    def build_requirements(self):
        self.test_requires("gtest/1.14.0")

    def layout(self):
        cmake_layout(self)

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()

    def package(self):
        cmake = CMake(self)
        cmake.install()
```

The build flow with Conan 2.x is a two-step process: first Conan installs dependencies and generates CMake files, then CMake does its normal configure-and-build:

```bash
# Build with Conan 2.x:
conan install . --output-folder=build --build=missing
cmake --preset conan-release
cmake --build --preset conan-release
```

The `--build=missing` flag tells Conan to compile from source any packages that aren't in the binary cache yet. Once built once, they'll be cached and subsequent installs are fast.

### Q3: Create a custom package/port

**Answer:**

Both tools let you package internal libraries so they can be consumed just like any public package. Here's how you write a Conan recipe for an internal library - notice that `package_info` is where you tell consumers which library to link against:

```python
# === Conan: custom recipe for internal library ===
# recipes/mylib/conanfile.py
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout
from conan.tools.files import copy

class MyLibConan(ConanFile):
    name = "mylib"
    version = "2.0.0"
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"
    exports_sources = "CMakeLists.txt", "src/*", "include/*"

    def requirements(self):
        self.requires("fmt/10.1.1")

    def layout(self):
        cmake_layout(self)

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()

    def package(self):
        copy(self, "*.h", src=os.path.join(self.source_folder, "include"),
             dst=os.path.join(self.package_folder, "include"))
        copy(self, "*.a", src=self.build_folder,
             dst=os.path.join(self.package_folder, "lib"), keep_path=False)
        copy(self, "*.so", src=self.build_folder,
             dst=os.path.join(self.package_folder, "lib"), keep_path=False)

    def package_info(self):
        self.cpp_info.libs = ["mylib"]

# Upload to private remote:
# conan create recipes/mylib/
# conan upload mylib/2.0.0 -r=my-artifactory --all
```

The vcpkg equivalent uses "overlay ports" - a small folder of CMake scripts that follow vcpkg's port conventions. You point vcpkg at the folder with `--overlay-ports` and it treats your library exactly like any registry package:

```cmake
# === vcpkg: custom overlay port ===
# ports/mylib/portfile.cmake
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH
    REPO myorg/mylib
    REF v2.0.0
    SHA512 abc123...
)

vcpkg_cmake_configure(SOURCE_PATH "${SOURCE_PATH}")
vcpkg_cmake_install()
vcpkg_cmake_config_fixup(CONFIG_PATH lib/cmake/mylib)
file(REMOVE_RECURSE "${CURRENT_PACKAGES_DIR}/debug/include")

# ports/mylib/vcpkg.json
# { "name": "mylib", "version": "2.0.0",
#   "dependencies": ["fmt"] }

# Use: vcpkg install mylib --overlay-ports=./ports
```

---

## Notes

- **vcpkg manifest mode** (vcpkg.json) is strongly preferred over classic mode - it version-locks dependencies per project so different projects can use different versions without conflict.
- **Conan 2.x** is a major redesign from 1.x with different APIs and conventions; if you find a tutorial online, check which version it targets before following it.
- Both tools support **binary caching**: CI can populate a cache the first time and skip rebuilding dependencies on subsequent runs (`VCPKG_BINARY_SOURCES` for vcpkg, `conan remote` for Conan).
- Always pin exact dependency versions in production - never rely on "latest" because upstream packages can introduce breaking changes without warning.
- For internal libraries, use Conan private remotes (such as Artifactory) or vcpkg overlay ports so they flow through the same tooling as public packages.
- `CMakePresets.json` integrates cleanly with both vcpkg and Conan and is the recommended way to keep build configurations reproducible across machines.
- Consider `FetchContent` only for small, header-only dependencies where pulling in a full package manager is genuinely more overhead than it saves.
