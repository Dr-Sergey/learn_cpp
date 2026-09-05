# Use CMakePresets.json for reproducible, shareable build configurations

**Category:** Build Systems and CI  
**Standard:** CMake 3.21+  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html>  

---

## Topic Overview

`CMakePresets.json` replaces long, error-prone `cmake -D...` command lines with named, version-controlled presets. Every developer and CI system uses `cmake --preset <name>` to get identical configurations. It supports configure, build, test, and package presets.

Here is why this matters in practice: without presets, each developer has to remember - or copy from a wiki page - the exact set of `-D` flags needed to configure the project correctly. Someone inevitably forgets one, gets a silently different build, and wonders why their tests behave differently from CI. Presets solve this by making the correct configuration a single, discoverable command.

### Preset Architecture

There are two files involved. The first is committed to git and shared by the whole team. The second is personal and gitignored:

```cpp
// CMakePresets.json (committed to git - shared by team)
// |-- configurePresets: [debug, release, ci-linux, ci-windows, asan]
// |-- buildPresets:     [debug-build, release-build]
// |-- testPresets:      [debug-test, release-test]
// `-- packagePresets:   [release-package]

// CMakeUserPresets.json (gitignored - per-developer)
// |-- inherits from shared presets
// |-- overrides: custom install path, local toolchain
// `-- developer-specific settings
```

---

## Self-Assessment

### Q1: Write a CMakePresets.json with configure, build, and test presets for debug and release builds

**Answer:**

The pattern here is to put all shared settings in a hidden `base` preset and then have `debug` and `release` inherit from it. That way there is no duplication, and changing a shared setting like the C++ standard only needs to happen in one place.

```json
{
    "version": 6,
    "cmakeMinimumRequired": {
        "major": 3,
        "minor": 25,
        "patch": 0
    },
    "configurePresets": [
        {
            "name": "base",
            "hidden": true,
            "generator": "Ninja",
            "cacheVariables": {
                "CMAKE_CXX_STANDARD": "20",
                "CMAKE_CXX_STANDARD_REQUIRED": "ON",
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
            }
        },
        {
            "name": "debug",
            "displayName": "Debug",
            "inherits": "base",
            "binaryDir": "${sourceDir}/build/debug",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "BUILD_TESTING": "ON",
                "CMAKE_CXX_FLAGS": "-Wall -Wextra -Wpedantic"
            }
        },
        {
            "name": "release",
            "displayName": "Release",
            "inherits": "base",
            "binaryDir": "${sourceDir}/build/release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "BUILD_TESTING": "ON",
                "CMAKE_CXX_FLAGS": "-Wall -Wextra -DNDEBUG"
            }
        },
        {
            "name": "asan",
            "displayName": "Address Sanitizer",
            "inherits": "debug",
            "binaryDir": "${sourceDir}/build/asan",
            "cacheVariables": {
                "CMAKE_CXX_FLAGS": "-Wall -Wextra -fsanitize=address,undefined -fno-omit-frame-pointer",
                "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined"
            }
        }
    ],
    "buildPresets": [
        {
            "name": "debug-build",
            "configurePreset": "debug"
        },
        {
            "name": "release-build",
            "configurePreset": "release",
            "configuration": "Release"
        }
    ],
    "testPresets": [
        {
            "name": "debug-test",
            "configurePreset": "debug",
            "output": {
                "outputOnFailure": true
            },
            "execution": {
                "timeout": 120,
                "jobs": 4
            }
        },
        {
            "name": "release-test",
            "configurePreset": "release",
            "output": {
                "outputOnFailure": true
            }
        }
    ]
}
```

With the presets defined, the daily workflow becomes clean three-command sequences. `cmake --list-presets` shows what is available so a new contributor does not have to read documentation to get started:

```bash
# Usage:
cmake --preset debug          # Configure debug build
cmake --build --preset debug-build   # Build
ctest --preset debug-test     # Run tests

# List all available presets:
cmake --list-presets
# Available configure presets:
#   "debug"   - Debug
#   "release" - Release
#   "asan"    - Address Sanitizer
```

### Q2: Explain how CMakeUserPresets.json extends shared presets without polluting version control

**Answer:**

`CMakeUserPresets.json` is the personal layer. It inherits from the shared presets but adds or overrides things that are specific to one developer's machine - install paths, local vcpkg locations, a preferred compiler. Because it is gitignored, it never affects anyone else.

```json
// CMakeUserPresets.json - NOT committed to git
// .gitignore must contain: CMakeUserPresets.json
{
    "version": 6,
    "configurePresets": [
        {
            "name": "my-debug",
            "inherits": "debug",
            "displayName": "My Debug (custom paths)",
            "cacheVariables": {
                "CMAKE_INSTALL_PREFIX": "D:/my-install",
                "CMAKE_PREFIX_PATH": "D:/vcpkg/installed/x64-windows"
            },
            "environment": {
                "CC": "C:/Program Files/LLVM/bin/clang-cl.exe",
                "CXX": "C:/Program Files/LLVM/bin/clang-cl.exe"
            }
        },
        {
            "name": "my-cross",
            "inherits": "release",
            "displayName": "My ARM Cross Build",
            "toolchainFile": "D:/toolchains/aarch64-linux.cmake",
            "binaryDir": "${sourceDir}/build/arm64"
        }
    ]
}
```

The gitignore entry is the critical part - without it, personal paths leak into version control:

```bash
# .gitignore
CMakeUserPresets.json

# Alice's workflow:
cmake --preset my-debug        # Uses her custom paths
cmake --build --preset debug-build

# Bob's workflow:
cmake --preset debug           # Uses shared defaults
cmake --build --preset debug-build

# Both produce correct builds - Bob doesn't see Alice's paths
```

**How inheritance works:**

- `"inherits": "debug"` copies all settings from the `debug` preset in `CMakePresets.json`
- UserPresets can override specific values (install prefix, toolchain, etc.)
- UserPresets cannot modify shared presets - only extend them
- CMake merges both files at configuration time

### Q3: Use cmake --preset release to reproduce an exact build configuration on any machine

**Answer:**

This is the core benefit of presets: one command, identical configuration everywhere. No flags to remember, no wiki page to consult. The same command works for a new hire's first build and for the CI pipeline.

```bash
# The problem (without presets)
# Developer must know the exact flags:
cmake -B build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=20 \
    -DCMAKE_CXX_STANDARD_REQUIRED=ON \
    -DCMAKE_INSTALL_PREFIX=/opt/myapp \
    -DBUILD_TESTING=ON \
    -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
    -DCMAKE_CXX_FLAGS="-Wall -Wextra -DNDEBUG"
# -> Error-prone, different developers use different flags

# The solution (with presets)
cmake --preset release
# -> Identical configuration on every machine
# -> No need to remember flags
# -> CI and developers use the same command

cmake --build --preset release-build
ctest --preset release-test

# CI integration
```

In CI, matrix builds over presets are cleaner than duplicating cmake commands for each configuration. The preset name is the only variable:

```yaml
# .github/workflows/ci.yml - clean and simple
name: CI
on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        preset: [debug, release, asan]
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Install Ninja

        run: sudo apt-get install -y ninja-build

      - name: Configure

        run: cmake --preset ${{ matrix.preset }}

      - name: Build

        run: cmake --build --preset ${{ matrix.preset }}-build

      - name: Test

        run: ctest --preset ${{ matrix.preset }}-test
```

**Benefits over manual `-D` flags:**

- **Reproducible:** Same preset -> same configuration, guaranteed
- **Discoverable:** `cmake --list-presets` shows what is available
- **Version-controlled:** Changes to build configuration are tracked in git
- **IDE integration:** VS Code, CLion, Visual Studio all read CMakePresets.json
- **CI-friendly:** Matrix over presets instead of duplicating cmake command lines

---

## Notes

- **Preset version compatibility:** version 6 requires CMake 3.25+. Use version 3 for CMake 3.21+ compatibility.
- **Condition expressions** (version 6+): presets can be conditionally enabled based on OS, compiler, etc. - useful for platform-specific presets.
- **Toolchain presets:** set `"toolchainFile"` in a preset for cross-compilation, so developers don't have to remember the toolchain path.
- **Package presets** (version 6): generate CPack packages from presets, keeping the packaging configuration alongside the build configuration.
- **Inheritance** can chain: `asan` inherits `debug` which inherits `base` - each level adds settings, and you can have as many levels as makes sense.
