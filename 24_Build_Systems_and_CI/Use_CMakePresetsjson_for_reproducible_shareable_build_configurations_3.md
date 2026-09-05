# Design platform-specific CMakePresets.json configurations

**Category:** Build Systems and CI  
**Standard:** Not version-specific  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html>  

---

## Topic Overview

This third CMakePresets file focuses on **platform-specific presets** with condition expressions, **toolchain file selection** via presets, and the **workflow preset** feature introduced in version 6.

The condition feature is what makes presets scale to cross-platform projects. Without it, a Windows preset would show up on Linux and fail confusingly. With conditions, `cmake --list-presets` only shows what is actually usable on the current machine.

### Platform-Specific Preset Design

Here is the typical structure for a cross-platform project. The hidden base preset holds everything shared, and each platform-specific preset inherits from it and adds what is needed:

```cpp
// CMakePresets.json
// |-- ci-base (hidden) -- common settings
// |-- debug-linux     -- inherits ci-base, condition: Linux
// |-- debug-windows   -- inherits ci-base, condition: Windows
// |-- release-linux   -- inherits ci-base, condition: Linux, LTO
// |-- release-windows -- inherits ci-base, condition: Windows, MSVC
// `-- cross-arm64     -- inherits ci-base, toolchainFile
```

Here is a quick reference for the preset features used in this file:

| Feature | Version Required | Purpose |
| --- | --- | --- |
| `condition` | 3 | OS/compiler-specific presets |
| `toolchainFile` | 3 | Embed toolchain selection |
| `workflowPresets` | 6 | Chain configure->build->test |
| `$env{}` macro | 3 | Read environment variables |
| `$penv{}` macro | 3 | Read parent environment |

---

## Self-Assessment

### Q1: Write a CMakePresets.json with configure, build, and test presets for debug and release with platform conditions

**Answer:**

The `condition` field uses `${hostSystemName}` to filter presets by platform. This is what makes the list tidy: on Linux you see Linux presets, on Windows you see Windows presets, and cross-compilation presets appear everywhere because they have no condition. The workflow preset at the end chains configure, build, and test into one command - very convenient for CI:

```json
{
    "version": 6,
    "configurePresets": [
        {
            "name": "ci-base",
            "hidden": true,
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_CXX_STANDARD": "20",
                "CMAKE_CXX_STANDARD_REQUIRED": "ON",
                "BUILD_TESTING": "ON"
            }
        },
        {
            "name": "debug-linux",
            "displayName": "Debug (Linux/GCC)",
            "inherits": "ci-base",
            "condition": {
                "type": "equals",
                "lhs": "${hostSystemName}",
                "rhs": "Linux"
            },
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_FLAGS": "-g -O0 -Wall -Wextra -Wpedantic -fsanitize=address,undefined",
                "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined"
            }
        },
        {
            "name": "release-linux",
            "displayName": "Release (Linux/GCC + LTO)",
            "inherits": "ci-base",
            "condition": {
                "type": "equals",
                "lhs": "${hostSystemName}",
                "rhs": "Linux"
            },
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_CXX_FLAGS": "-O2 -Wall -Wextra -DNDEBUG",
                "CMAKE_INTERPROCEDURAL_OPTIMIZATION": "ON"
            }
        },
        {
            "name": "debug-windows",
            "displayName": "Debug (Windows/MSVC)",
            "inherits": "ci-base",
            "condition": {
                "type": "equals",
                "lhs": "${hostSystemName}",
                "rhs": "Windows"
            },
            "generator": "Ninja",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_FLAGS": "/W4 /WX /EHsc /Zi"
            }
        },
        {
            "name": "release-windows",
            "displayName": "Release (Windows/MSVC + LTO)",
            "inherits": "ci-base",
            "condition": {
                "type": "equals",
                "lhs": "${hostSystemName}",
                "rhs": "Windows"
            },
            "generator": "Ninja",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_CXX_FLAGS": "/W4 /WX /EHsc /O2 /GL /DNDEBUG",
                "CMAKE_EXE_LINKER_FLAGS": "/LTCG"
            }
        },
        {
            "name": "cross-arm64",
            "displayName": "Cross-compile ARM64",
            "inherits": "ci-base",
            "toolchainFile": "${sourceDir}/cmake/toolchains/aarch64-linux-gnu.cmake",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release"
            }
        }
    ],
    "buildPresets": [
        { "name": "debug-linux",     "configurePreset": "debug-linux" },
        { "name": "release-linux",   "configurePreset": "release-linux" },
        { "name": "debug-windows",   "configurePreset": "debug-windows" },
        { "name": "release-windows", "configurePreset": "release-windows" },
        { "name": "cross-arm64",     "configurePreset": "cross-arm64" }
    ],
    "testPresets": [
        {
            "name": "debug-linux",
            "configurePreset": "debug-linux",
            "output": { "outputOnFailure": true },
            "execution": { "timeout": 120 }
        },
        {
            "name": "release-linux",
            "configurePreset": "release-linux",
            "output": { "outputOnFailure": true }
        }
    ],
    "workflowPresets": [
        {
            "name": "ci-linux",
            "displayName": "Full CI (Linux)",
            "steps": [
                { "type": "configure", "name": "release-linux" },
                { "type": "build",     "name": "release-linux" },
                { "type": "test",      "name": "release-linux" }
            ]
        }
    ]
}
```

The `condition` field ensures `cmake --list-presets` only shows presets valid for the current host OS.

### Q2: Show that cmake --preset=release-linux selects the right toolchain and flags without manual -D flags

**Answer:**

Watch what happens on a Linux machine. The conditions filter the list automatically, and the preset picks up all the right flags without you specifying a single `-D`:

```bash
# On a Linux machine
$ cmake --list-presets
Available configure presets:

  "debug-linux"      - Debug (Linux/GCC)
  "release-linux"    - Release (Linux/GCC + LTO)
  "cross-arm64"      - Cross-compile ARM64
# Note: Windows presets NOT listed (condition filtered them out)

$ cmake --preset release-linux
# Output:
# Preset CMake variables:
#   BUILD_TESTING="ON"
#   CMAKE_BUILD_TYPE="Release"
#   CMAKE_CXX_FLAGS="-O2 -Wall -Wextra -DNDEBUG"
#   CMAKE_CXX_STANDARD="20"
#   CMAKE_INTERPROCEDURAL_OPTIMIZATION="ON"
# -- The CXX compiler identification is GNU 13.2.0
# -- Configuring done (0.4s)
# -- Generating done (0.1s)
# -- Build files have been written to: /project/build/release-linux

$ cmake --build --preset release-linux -- -j$(nproc)
# [42/42] Linking CXX executable myapp

$ ctest --preset release-linux
# Test project /project/build/release-linux
#   1/8 Test #1: unit_core ............   Passed  0.05 sec
#
#   8/8 Test #8: integration_api .....   Passed  1.23 sec
# 100% tests passed, 0 tests failed

# Or run the full workflow in one command
$ cmake --workflow --preset ci-linux
# Executes configure -> build -> test sequentially
# If any step fails, the workflow stops immediately

# Toolchain preset for cross-compilation
$ cmake --preset cross-arm64
# Reads toolchainFile from the preset - no -DCMAKE_TOOLCHAIN_FILE needed
# The preset encodes the toolchain path relative to ${sourceDir}
```

### Q3: Explain how CMakeUserPresets.json extends the shared presets without committing machine-specific paths

**Answer:**

The inheritance model between the two files is intentionally one-directional: user presets can extend shared presets, but shared presets can never depend on user presets. This guarantees that the shared configuration is always self-contained and correct for any machine that does not have a user preset file.

Here is an example of a user preset that adds ccache integration and a custom vcpkg path - both of which are machine-specific and should never be in version control:

```json
// CMakeUserPresets.json - MUST be in .gitignore
{
    "version": 6,
    "configurePresets": [
        {
            "name": "dev-alice",
            "inherits": "debug-linux",
            "displayName": "Alice's Local Dev",
            "cacheVariables": {
                "CMAKE_PREFIX_PATH": "/home/alice/libs/fmt;/home/alice/libs/spdlog",
                "CMAKE_INSTALL_PREFIX": "/home/alice/opt/myproject"
            },
            "environment": {
                "CC":  "/usr/lib/ccache/gcc-13",
                "CXX": "/usr/lib/ccache/g++-13",
                "VCPKG_ROOT": "/home/alice/vcpkg"
            }
        },
        {
            "name": "dev-alice-clang",
            "inherits": "debug-linux",
            "cacheVariables": {
                "CMAKE_CXX_COMPILER": "/usr/bin/clang++-17",
                "CMAKE_CXX_FLAGS": "-stdlib=libc++ -g -O0"
            }
        }
    ],
    "buildPresets": [
        {
            "name": "dev-alice",
            "configurePreset": "dev-alice",
            "jobs": 16
        }
    ]
}
```

**How inheritance works:**

```cpp
// CMakePresets.json (committed)         CMakeUserPresets.json (gitignored)
// +---------------------+               +----------------------+
// | ci-base (hidden)    |               | dev-alice             |
// | debug-linux         |<--inherits----| inherits: debug-linux |
// | release-linux       |               | + custom paths        |
// +---------------------+               | + ccache compiler     |
//                                       | dev-alice-clang       |
//                                       | inherits: debug-linux |
//                                       | + clang++-17          |
//                                       +----------------------+
```

**Rules:**

- `CMakeUserPresets.json` **can** inherit from `CMakePresets.json` presets
- `CMakePresets.json` **cannot** reference `CMakeUserPresets.json` presets
- Use `$env{HOME}` or `$penv{VCPKG_ROOT}` to avoid hardcoded absolute paths where possible
- `$penv{}` reads from the parent environment (shell), `$env{}` reads from preset-defined environment

---

## Notes

- **Condition types:** `equals`, `notEquals`, `inList`, `matches` (regex), `allOf`, `anyOf`, `not` - you can combine these for complex platform targeting, such as "Linux and Clang" or "not Windows".
- **`cmake --workflow`** (version 6) chains configure->build->test->package - ideal for CI one-liners where you want a single command to represent the whole pipeline.
- **`${hostSystemName}`** returns `Linux`, `Windows`, or `Darwin` - it maps to `CMAKE_HOST_SYSTEM_NAME` and is evaluated when presets are loaded, not at build time.
- **Toolchain in presets** avoids the `-DCMAKE_TOOLCHAIN_FILE=...` flag - the toolchain becomes part of the reproducible configuration and is stored relative to `${sourceDir}`, so it works from any checkout location.
