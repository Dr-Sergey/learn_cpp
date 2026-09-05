# Understand C interop: extern C, name mangling, and calling conventions

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/language_linkage>  

---

## Topic Overview

C++ compilers **mangle** function names to encode parameter types into the symbol name - that's the mechanism that makes function overloading possible. C compilers don't do any of that; they just use the raw function name. `extern "C"` tells the C++ compiler to suppress mangling for a specific declaration, producing a C-compatible symbol that C code (or any other language linking against a C ABI) can find by name.

This is the foundation of all C/C++ interop. If you're exposing a C++ library to Python, Rust, C, or any other language, `extern "C"` is almost certainly involved.

### Name Mangling - What Happens

Here's the concrete difference between what a C++ compiler produces by default versus with `extern "C"`:

```cpp
C++ source:                      Mangled symbol (GCC):
──────────                       ─────────────────────
int add(int, int)                _Z3addii
double add(double, double)       _Z3adddd
void MyClass::run(int)           _ZN7MyClass3runEi

extern "C" int add(int, int)     add        <- C-compatible
```

The mangled names encode the types: `ii` means two `int` parameters, `dd` means two `double` parameters. This is what allows overloads to coexist - each has a unique symbol. With `extern "C"`, the symbol is just `add`, which is what a C linker expects.

### Key Concepts

Here's a summary of what `extern "C"` does and doesn't change:

| Feature | C++ (default) | `extern "C"` |
| --- | :---: | :---: |
| Name mangling | Yes | **No** |
| Function overloading | Yes | **No** |
| Templates | Yes | **No** |
| Default parameters | Yes | Yes (C++ side only) |
| `noexcept` | Yes | **Required** (safety) |
| Namespace | Yes | Ignored for linkage |

### Calling Convention Defaults

On modern 64-bit platforms, calling conventions are not something you usually have to think about - both C and C++ use the same platform convention. The concern is mainly historical, for 32-bit Windows:

| Platform | C default | C++ default | Notes |
| --- | --- | --- | --- |
| Linux x86-64 | System V AMD64 | System V AMD64 | Same - interop is easy |
| Windows x64 | `__cdecl` | `__cdecl` | Same since MSVC x64 |
| Windows x86 | `__cdecl` | `__cdecl` | But COM uses `__stdcall` |

---

## Self-Assessment

### Q1: Wrap a C++ function with `extern "C"` and verify the unmangled symbol with nm

**Answer:**

The standard pattern for a C-compatible header is to wrap the declarations in `extern "C"` using a preprocessor guard, so the header works correctly whether it's included from C or C++ code:

```cpp
// mathlib.h - C-compatible header
#ifndef MATHLIB_H
#define MATHLIB_H

#ifdef __cplusplus
extern "C" {
#endif

// These functions will have unmangled names
int add(int a, int b);
double multiply(double a, double b);

#ifdef __cplusplus
}  // extern "C"
#endif

#endif // MATHLIB_H
```

```cpp
// mathlib.cpp - C++ implementation
#include "mathlib.h"
#include <cmath>

// extern "C" suppresses mangling for these definitions
extern "C" int add(int a, int b) {
    return a + b;
}

extern "C" double multiply(double a, double b) {
    return a * b;
}
```

You can verify the result with `nm`. Without `extern "C"`, GCC produces mangled symbols; with it, you get clean C-compatible names:

**Verifying with nm:**

```bash
# Compile to object file
g++ -c mathlib.cpp -o mathlib.o

# Without extern "C" - mangled:
nm mathlib.o
# T _Z3addii              <- mangled (encodes int, int)
# T _Z8multiplydd         <- mangled (encodes double, double)

# With extern "C" - unmangled:
nm mathlib.o
# T add                   <- clean C symbol
# T multiply              <- clean C symbol

# Demangle to verify:
nm mathlib.o | c++filt
# Same output with extern "C" since names are already clean
```

**Why this matters:** a C program can now link against `mathlib.o` because it looks for the symbol `add`, not `_Z3addii`.

### Q2: Explain why `extern "C"` functions cannot be overloaded and cannot throw C++ exceptions safely

**Answer:**

Both limitations follow directly from what `extern "C"` does. Overloading is impossible because the mechanism that makes overloading work - mangled names - is exactly what `extern "C"` turns off. Exceptions are dangerous because C code has no concept of stack unwinding, so if an exception propagates through a C stack frame, the destructors for objects on that frame never run.

**No overloading:** Name mangling IS the mechanism that enables overloading. Without it, all overloads produce the same symbol - a linker error:

```cpp
extern "C" int process(int x);       // symbol: "process"
extern "C" int process(double x);    // symbol: "process" - COLLISION!
// error: conflicting declaration of C function 'int process(double)'
```

**No safe exceptions:** C calling conventions have no stack unwinding support. If a C++ exception propagates through C code, the C stack frames are not unwound - resource leaks and undefined behavior follow. The right fix is to catch everything at the boundary and translate to an error code:

```cpp
// WRONG: exception escapes into C
extern "C" int dangerous(int x) {
    std::vector<int> v(x);  // might throw std::bad_alloc
    return v.size();         // if throw, C caller has no handler -> UB
}

// CORRECT: catch all exceptions at the boundary
extern "C" int safe(int x) noexcept {
    try {
        std::vector<int> v(x);
        return static_cast<int>(v.size());
    } catch (const std::exception& e) {
        // Log or set error code - NEVER let exception escape
        return -1;
    } catch (...) {
        return -1;
    }
}

// Error reporting pattern
struct ErrorInfo {
    int code;
    char message[256];
};

extern "C" int robust_api(int input, ErrorInfo* err) noexcept {
    try {
        if (input < 0) throw std::invalid_argument("negative input");
        return input * 2;
    } catch (const std::exception& e) {
        if (err) {
            err->code = -1;
            strncpy(err->message, e.what(), sizeof(err->message) - 1);
            err->message[sizeof(err->message) - 1] = '\0';
        }
        return -1;
    } catch (...) {
        if (err) {
            err->code = -2;
            strncpy(err->message, "unknown error", sizeof(err->message) - 1);
        }
        return -1;
    }
}
```

### Q3: Write a C-compatible header using `#ifdef __cplusplus` to expose a C++ class as a C API

**Answer:**

The standard pattern here is the **opaque handle** - C gets a pointer to a type it knows nothing about, and all operations go through functions that take that pointer. The full C++ class lives entirely in the `.cpp` file, invisible to C. Every function in the C API wraps its body in a `try/catch` so no exceptions can escape:

```cpp
// image_processor.h - C-compatible header
#ifndef IMAGE_PROCESSOR_H
#define IMAGE_PROCESSOR_H

#include <stddef.h>  // size_t (C header, not <cstddef>)

#ifdef __cplusplus
extern "C" {
#endif

// Opaque handle - C sees a pointer to incomplete type
typedef struct ImageProcessor ImageProcessor;

// Lifecycle
ImageProcessor* image_processor_create(int width, int height);
void image_processor_destroy(ImageProcessor* ip);

// Operations
int image_processor_load(ImageProcessor* ip, const char* path);
int image_processor_blur(ImageProcessor* ip, float radius);
int image_processor_save(ImageProcessor* ip, const char* path);

// Query
int image_processor_width(const ImageProcessor* ip);
int image_processor_height(const ImageProcessor* ip);
const char* image_processor_last_error(const ImageProcessor* ip);

#ifdef __cplusplus
}  // extern "C"
#endif

#endif // IMAGE_PROCESSOR_H
```

```cpp
// image_processor.cpp - C++ implementation
#include "image_processor.h"
#include <string>
#include <vector>
#include <cstring>

// The actual C++ class - never exposed to C
struct ImageProcessor {
    int width_;
    int height_;
    std::vector<uint8_t> pixels_;
    std::string last_error_;

    ImageProcessor(int w, int h)
        : width_(w), height_(h), pixels_(w * h * 4, 0) {}

    bool load(const char* path) {
        // ... real implementation ...
        return true;
    }

    bool blur(float radius) {
        if (radius < 0) {
            last_error_ = "negative radius";
            return false;
        }
        // ... real implementation ...
        return true;
    }

    bool save(const char* path) {
        // ... real implementation ...
        return true;
    }
};

// C API implementation - catch ALL exceptions
extern "C" {

ImageProcessor* image_processor_create(int width, int height) {
    try {
        return new ImageProcessor(width, height);
    } catch (...) {
        return nullptr;
    }
}

void image_processor_destroy(ImageProcessor* ip) {
    delete ip;  // safe if ip is nullptr
}

int image_processor_load(ImageProcessor* ip, const char* path) {
    try {
        return ip->load(path) ? 0 : -1;
    } catch (const std::exception& e) {
        ip->last_error_ = e.what();
        return -1;
    }
}

int image_processor_blur(ImageProcessor* ip, float radius) {
    try {
        return ip->blur(radius) ? 0 : -1;
    } catch (const std::exception& e) {
        ip->last_error_ = e.what();
        return -1;
    }
}

int image_processor_save(ImageProcessor* ip, const char* path) {
    try {
        return ip->save(path) ? 0 : -1;
    } catch (const std::exception& e) {
        ip->last_error_ = e.what();
        return -1;
    }
}

int image_processor_width(const ImageProcessor* ip) { return ip->width_; }
int image_processor_height(const ImageProcessor* ip) { return ip->height_; }
const char* image_processor_last_error(const ImageProcessor* ip) {
    return ip->last_error_.c_str();
}

}  // extern "C"
```

From the C consumer's perspective, the API is completely transparent - it just uses a pointer and function calls, with no knowledge of what's inside:

```c
// main.c - Pure C consumer
#include "image_processor.h"
#include <stdio.h>

int main(void) {
    ImageProcessor* ip = image_processor_create(1920, 1080);
    if (!ip) {
        fprintf(stderr, "Failed to create processor\n");
        return 1;
    }

    if (image_processor_load(ip, "photo.png") != 0) {
        fprintf(stderr, "Load error: %s\n", image_processor_last_error(ip));
    }

    image_processor_blur(ip, 2.5f);
    image_processor_save(ip, "blurred.png");

    printf("Size: %dx%d\n", image_processor_width(ip), image_processor_height(ip));

    image_processor_destroy(ip);
    return 0;
}
```

---

## Notes

- The **opaque handle pattern** is the gold standard for C-API design: C gets a pointer, never sees struct internals.
- `extern "C"` does NOT change the calling convention - it only changes name mangling.
- On Windows, COM interfaces use `__stdcall`; regular C APIs use `__cdecl`. `extern "C"` gives you `__cdecl`.
- Always mark `extern "C"` functions `noexcept` or wrap their bodies in `try/catch`.
- Use `nm -C` (demangle) to debug symbol issues; on Windows use `dumpbin /exports`.
- Prefer `#pragma once` for modern compilers, but include guards are more portable when C code also includes the header.
