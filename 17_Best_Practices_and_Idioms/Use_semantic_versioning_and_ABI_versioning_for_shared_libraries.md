# Use semantic versioning and ABI versioning for shared libraries

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://semver.org>  

---

## Topic Overview

**Semantic Versioning (SemVer)** defines what a version number change means to the people using your library. **ABI versioning** is the complementary runtime concern: it ensures that existing compiled programs keep working after you ship an update, without requiring them to be recompiled.

The two concepts work together. SemVer tells humans how serious a change is; ABI versioning tells the operating system's dynamic linker which binary versions are compatible at load time.

Here is the three-part version number and what each segment signals:

```cpp
Version:  MAJOR . MINOR . PATCH
           |       |       |
           |       |       +-- Bug fixes, no API/ABI change
           |       +---------- New features, backward compatible
           +------------------ Breaking changes (API or ABI)
```

### ABI vs API

It is easy to confuse API compatibility with ABI compatibility. An API break means source code that used to compile will no longer compile. An ABI break means code that used to run will crash or misbehave at the binary level - even if it still compiles. For example, adding a data member to a class does not break your header, but it changes the object's size, so any code that was compiled against the old layout is now using wrong offsets.

| Change | API Break? | ABI Break? | SemVer |
| --- | --- | --- | --- |
| Add new function | No | No | MINOR |
| Add virtual method at end | No | Yes | MAJOR |
| Add data member to class | No* | Yes | MAJOR |
| Change parameter type | Yes | Yes | MAJOR |
| Fix bug in function body | No | No | PATCH |
| Remove deprecated function | Yes | Yes | MAJOR |

---

## Self-Assessment

### Q1: Explain semantic versioning components and compatibility implications

Here is a concrete example of a library version and what each digit says about the change:

```cpp
Example:  libfoo 2.3.1
                 | | |
                 | | +-- Patch: fixed memory leak in parse()
                 | +---- Minor: added format_json() function
                 +------ Major: removed deprecated XML methods
```

The rules for what each bump promises to your users:

1. **PATCH** (2.3.0 -> 2.3.1): Bug fixes only. No API/ABI change. Drop-in replacement.
2. **MINOR** (2.3.1 -> 2.4.0): New features added. Old code still compiles and links.
3. **MAJOR** (2.4.0 -> 3.0.0): Breaking changes. Recompilation or code changes required.

**Compatibility matrix:**

| Upgrade | Source compatible? | Binary compatible? |
| --- | --- | --- |
| PATCH bump | Yes | Yes |
| MINOR bump | Yes | Usually yes |
| MAJOR bump | No | No |

On Linux, the dynamic linker uses the **soname** - which encodes only the MAJOR version - to decide which library to load at runtime. That is why symlinks are set up like this:

```bash
libfoo.so.2.3.1     # real file (MAJOR.MINOR.PATCH)
libfoo.so.2         # soname symlink (MAJOR only = ABI version)
libfoo.so           # linker name (for -lfoo)

# The dynamic linker uses soname:
# If ABI is compatible, only MAJOR matters
```

As long as only the MINOR or PATCH digits change, all programs linked against `libfoo.so.2` can pick up the new file transparently.

### Q2: Use symbol versioning scripts for backward ABI compatibility

Sometimes you need to ship a library that old programs (linked against v1.0 symbols) and new programs (expecting v2.0 symbols) can both use - from the same `.so` file. Symbol versioning scripts make that possible. The library exports each symbol tagged with the version that introduced it, and the linker records which version tag a program was built against.

Here is the C++ source for a library with two generations of API:

```cpp
// === libmath.h ===
#pragma once

namespace math {
    // Version 1: original API
    int add(int a, int b);

    // Version 2: added multiply
    int multiply(int a, int b);
}

// === libmath.cpp ===
#include "libmath.h"

namespace math {
    int add(int a, int b) { return a + b; }
    int multiply(int a, int b) { return a * b; }
}
```

The **version script** (libmath.map) controls which symbols are visible under which version tag:

```cpp
# Symbol versioning: controls which symbols are visible per version
LIBMATH_1.0 {
    global:
        extern "C++" {
            math::add*;       # v1.0 exports add
        };
    local:
        *;                    # hide everything else
};

LIBMATH_2.0 {
    global:
        extern "C++" {
            math::multiply*;  # v2.0 adds multiply
        };
} LIBMATH_1.0;               # inherits from 1.0
```

Then build and create the expected symlinks:

```bash
# Build with version script
g++ -shared -fPIC -o libmath.so.2.0.0 libmath.cpp \
    -Wl,--version-script=libmath.map -Wl,-soname,libmath.so.2

# Create symlinks
ln -sf libmath.so.2.0.0 libmath.so.2
ln -sf libmath.so.2 libmath.so

# Old programs linked against LIBMATH_1.0 still work
# New programs can use LIBMATH_2.0 symbols too
```

The key insight is that the old program's object file records `math::add@@LIBMATH_1.0` as its dependency. The new library still exports that exact versioned symbol, so the old program keeps working.

### Q3: Use inline namespaces to version ABI-breaking changes

Inline namespaces give you a clean C++ mechanism for the same idea. When you declare a namespace `inline`, names inside it are accessible without the namespace prefix - so `mylib::Config` silently refers to `mylib::v2::Config`. Meanwhile, old binary code that was compiled against v1 still resolves to the mangled name `mylib::v1::Config const&`, which the library continues to export.

```cpp
#include <iostream>
#include <cstdint>

namespace mylib {
    // Inline namespace = current ABI version
    // Old code links to v1 symbols, new code links to v2 symbols

    namespace v1 {
        struct Config {
            int width;
            int height;
            // ABI: sizeof = 8 bytes
        };

        void init(const Config& c) {
            std::cout << "v1::init(" << c.width << "x" << c.height << ")\n";
        }
    }

    // v2 is the CURRENT version (inline)
    inline namespace v2 {
        struct Config {
            int width;
            int height;
            int depth;        // NEW: breaks ABI (sizeof changed!)
            uint32_t flags;   // NEW
            // ABI: sizeof = 16 bytes
        };

        void init(const Config& c) {
            std::cout << "v2::init(" << c.width << "x" << c.height
                      << "x" << c.depth << " flags=" << c.flags << ")\n";
        }
    }
}

int main() {
    // Unqualified: uses inline v2 automatically
    mylib::Config cfg{800, 600, 32, 0x01};
    mylib::init(cfg);  // calls v2::init

    // Explicit v1 access still works:
    mylib::v1::Config old_cfg{640, 480};
    mylib::v1::init(old_cfg);  // calls v1::init

    std::cout << "v1 Config size: " << sizeof(mylib::v1::Config) << '\n';
    std::cout << "v2 Config size: " << sizeof(mylib::v2::Config) << '\n';
}
// Expected output:
// v2::init(800x600x32 flags=1)
// v1::init(640x480)
// v1 Config size: 8
// v2 Config size: 16
```

The reason this works at the binary level is that name mangling embeds the full namespace path. A program compiled with the old header gets mangled names containing `v1`, a program compiled with the new header gets names containing `v2`, and the library exports both. Both programs coexist at runtime without conflict:

```cpp
Program compiled with libfoo 1.x:
  Links to symbol: mylib::v1::init(mylib::v1::Config const&)

Program compiled with libfoo 2.x:
  Links to symbol: mylib::v2::init(mylib::v2::Config const&)

Both symbols exist in the library -> both programs work!
```

---

## Notes

- `inline namespace` is used by the standard library: `std::__1::`, `std::__cxx11::`.
- Never add or remove virtual methods without bumping the MAJOR version.
- Use the Pimpl idiom to insulate ABI from implementation changes.
- CMake: set `VERSION` and `SOVERSION` properties on shared library targets.
