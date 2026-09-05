# Model Conan 2.0 dependency types: requires, tool_requires, and test_requires

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://docs.conan.io/2/>  

---

## Topic Overview

This second Conan file focuses on the **dependency graph model** (requires vs tool_requires vs test_requires), **binary package creation**, and **uploading to a private Artifactory** remote. These are essential for teams that maintain internal C++ libraries.

The distinction between dependency types matters more than it might initially seem. A tool like `protoc` (the Protocol Buffers compiler) runs on your *build* machine at build time and generates `.pb.cc` files. The protobuf *library* gets linked into your binary and runs on the *target* machine at runtime. When you cross-compile for a different architecture, these come from completely different places. Conan's separation of `requires`, `tool_requires`, and `test_requires` maps directly onto this reality.

### Conan 2.0 Dependency Types

Here is how the types relate in a typical project:

```cpp
myproject
├── requires: "fmt/10.2.1"           <- Runtime dependency (linked into binary)
├── requires: "boost/1.84.0"         <- Runtime dependency
├── tool_requires: "cmake/3.28.1"    <- Build tool (not linked, HOST machine)
├── tool_requires: "protobuf/25.2"   <- Code generator (runs at build time)
└── test_requires: "gtest/1.14.0"    <- Test-only (not in release binary)
```

The table below captures the key properties of each type. The "Linked?" and "When Needed" columns are the ones that matter most for understanding the model:

| Dependency Type | Linked? | When Needed | Example |
| --- | --- | --- | --- |
| `requires` | Yes | Runtime + build | Boost, fmt, OpenSSL |
| `tool_requires` | No | Build time only | CMake, protoc, flatc |
| `test_requires` | Test only | Test build | GoogleTest, Catch2 |
| `python_requires` | No | Recipe reuse | Base recipe classes |

---

## Self-Assessment

### Q1: Write a conanfile.py that declares dependencies and generates CMake integration

**Answer:**

This recipe shows a realistic production setup with options (shared/static, optional TLS), all three dependency types, and conditional logic in `requirements()` based on the chosen options. Notice that `build_requirements()` is where both `tool_requires` and `test_requires` go:

```python
# conanfile.py — full-featured with all dependency types
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout, CMakeDeps, CMakeToolchain


class NetworkLibConan(ConanFile):
    name = "netlib"
    version = "2.0.0"
    settings = "os", "compiler", "build_type", "arch"
    options = {"shared": [True, False], "with_tls": [True, False]}
    default_options = {"shared": False, "with_tls": True}

    exports_sources = "CMakeLists.txt", "src/*", "include/*", "tests/*"

    # Dependency graph
    def requirements(self):
        self.requires("boost/1.84.0")
        self.requires("fmt/10.2.1")
        self.requires("spdlog/1.13.0")
        if self.options.with_tls:
            self.requires("openssl/3.2.1")

    def build_requirements(self):
        self.tool_requires("cmake/3.28.1")
        self.tool_requires("protobuf/25.2")   # protoc code generator
        self.test_requires("gtest/1.14.0")

    def layout(self):
        cmake_layout(self)

    def generate(self):
        deps = CMakeDeps(self)
        deps.generate()

        tc = CMakeToolchain(self)
        tc.variables["WITH_TLS"] = self.options.with_tls
        tc.variables["BUILD_SHARED_LIBS"] = self.options.shared
        tc.generate()

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()
        if not self.conf.get("tools.build:skip_test", default=False):
            cmake.test()

    def package(self):
        cmake = CMake(self)
        cmake.install()

    def package_info(self):
        self.cpp_info.libs = ["netlib"]
        if self.options.with_tls:
            self.cpp_info.defines.append("NETLIB_TLS_ENABLED")
```

### Q2: Explain the Conan 2.0 graph model: requires, tool_requires, and test_requires

**Answer:**

The graph model is where Conan really earns its keep in larger projects. Understanding the rules for each type helps you avoid subtle problems like shipping test frameworks in release packages or breaking cross-compilation by mixing build-host and target dependencies.

The reason `tool_requires` is separate from `requires` is cross-compilation. When you compile for ARM on an x86 machine, `cmake` and `protoc` still run on your x86 machine (the *build* host). They must be built for x86. Your runtime libraries, however, need to be compiled for ARM (the *target* host). Conan tracks which machine each dependency is for, which is why the separation exists:

```python
# requires: runtime dependencies
# Linked into the final binary, propagated to consumers
def requirements(self):
    self.requires("boost/1.84.0")
    # Transitivity: if libA requires boost, consumers of libA also get boost
    # Version conflict: Conan resolves via "closest to root wins" policy

    # Version ranges:
    self.requires("fmt/[>=10.0 <11.0]")

    # Options propagation:
    self.requires("openssl/3.2.1", options={"shared": True})

# tool_requires: build tools
# NOT linked. Run on the BUILD machine (important for cross-compilation)
def build_requirements(self):
    self.tool_requires("cmake/3.28.1")
    # cmake runs on your machine, even when cross-compiling for ARM

    self.tool_requires("protobuf/25.2")
    # protoc generates .pb.cc files at build time
    # The protobuf LIBRARY is in requires (linked into binary)
    # The protoc TOOL is in tool_requires (runs at build time)

# test_requires: test-only dependencies
def build_requirements(self):
    self.test_requires("gtest/1.14.0")
    # Only available when building tests
    # NOT propagated to consumers of this package
    # NOT included in the release binary

# Dependency graph visualization
# conan graph info . --format=html > deps.html
# Opens in browser showing full graph with all dependency types
```

You can visualize the resolved graph at any time, which is invaluable when debugging version conflicts:

```bash
# Inspect the graph:
conan graph info . --format=dot | dot -Tpng > graph.png
# Shows:
# netlib ──requires──> boost, fmt, spdlog, openssl
# netlib ──tool_requires──> cmake, protobuf
# netlib ──test_requires──> gtest
#
# Transitive resolution:
# spdlog requires fmt — but netlib also requires fmt
# Conan merges them: one fmt version (closest to root wins)
```

### Q3: Create a binary package for a custom library and upload it to a private Artifactory repository

**Answer:**

This is the typical CI workflow for a team that maintains an internal library. You create packages for multiple configurations (Debug, Release, shared/static), then upload them all to Artifactory so that consumers can download the pre-built binary without compiling from source themselves:

```bash
# Step 1: Create the package
# Build for multiple configurations:
conan create . --build=missing -s build_type=Release
conan create . --build=missing -s build_type=Debug
conan create . --build=missing -s build_type=Release -o "netlib/*:shared=True"

# Output:
# netlib/2.0.0: Package created
# netlib/2.0.0: Package ID: a1b2c3d4e5f6
# Package contains: include/netlib/*.hpp, lib/libnetlib.a

# Step 2: Configure Artifactory remote
conan remote add company https://artifactory.company.com/artifactory/api/conan/cpp-local
conan remote login company alice --password <token>

# List remotes:
conan remote list
# company: https://artifactory.company.com/
# conancenter: https://center.conan.io (default)

# Step 3: Upload to Artifactory
conan upload "netlib/2.0.0" -r company --confirm
# Uploads recipe + all binary packages (Release, Debug, shared)

# Upload only specific packages:
conan upload "netlib/2.0.0:a1b2c3d4e5f6" -r company --confirm

# Step 4: Consume from another project
# In consumer's conanfile.py:
#   self.requires("netlib/2.0.0")
#
# conan install . --build=missing
# -> Downloads netlib from company remote (or builds if binary doesn't match)

# CI automation
# .github/workflows/publish.yml:
# - conan create . --build=missing
# - conan upload "netlib/*" -r company --confirm
```

The **Package ID** is worth understanding here. It is a hash derived from the settings (OS, compiler, build_type) and options (shared, with_tls). When a consumer runs `conan install`, Conan computes the expected Package ID for their environment and checks if a binary with that ID exists in the remote. If it does, they get a fast download. If not, Conan falls back to building from source (with `--build=missing`). This is why uploading for multiple configurations is useful: you are pre-building binaries for the most common consumer environments.

---

## Notes

- **Package ID:** uniquely identifies a binary - changes when settings (compiler, OS, build_type) or options change
- **`conan lock create .`** generates a lockfile for reproducible CI builds
- **Version ranges:** `[>=1.0 <2.0]` in requires allow flexible dependency resolution
- **`conan graph info .`** shows the full dependency graph with conflicts highlighted
