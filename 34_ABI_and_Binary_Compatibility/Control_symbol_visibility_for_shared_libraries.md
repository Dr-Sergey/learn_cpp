# Control Symbol Visibility for Shared Libraries

**Category:** ABI and Binary Compatibility  
**Standard:** C++11 and later (compiler-specific attributes)  
**Reference:** https://gcc.gnu.org/wiki/Visibility  

---

## Topic Overview

Symbol visibility controls which symbols in a shared library are accessible to external users and which remain internal. By default, GCC exports **all** symbols from a shared library, creating a bloated dynamic symbol table that slows down load time, increases memory usage, and exposes implementation details that could break if users depend on them. Controlling visibility is essential for professional library development.

On Linux/macOS, the primary mechanisms are `__attribute__((visibility("default")))` for exported symbols and `-fvisibility=hidden` to hide everything by default. On Windows, `__declspec(dllexport)` and `__declspec(dllimport)` serve a similar purpose - but Windows hides symbols by default, the opposite of Unix. A cross-platform library needs a unified export macro that handles both worlds.

Here is a summary of the four visibility levels available on GCC/Clang and what each one does:

| Visibility Level | GCC/Clang Attribute              | Effect                                        |
| --- | --- | --- |
| `default`       | `visibility("default")`         | Exported, available to users of the shared lib |
| `hidden`        | `visibility("hidden")`          | Not exported, internal to the shared lib       |
| `internal`      | `visibility("internal")`        | Like hidden + cannot be called even via pointer from outside (rare) |
| `protected`     | `visibility("protected")`       | Exported but cannot be preempted by another DSO |

To make the practical impact concrete, here is what the symbol count and load performance typically look like once you apply `-fvisibility=hidden` and annotate your public API:

```cpp
// Typical symbol counts for a large library:
//
//   Default visibility (all exported):    ~12,000 symbols
//   With -fvisibility=hidden + macros:    ~800 symbols
//
//   Load time improvement:  ~40% faster
//   Library size reduction: ~15-20%
//   Fewer ODR violation risks across DSOs
```

Those numbers are not just aesthetics. Fewer exported symbols mean the dynamic linker has less work to do at startup, and it means fewer opportunities for **symbol interposition** - a subtle bug where a user accidentally defines a function with the same name as an internal library function, causing the linker to redirect calls. With hidden visibility, internal symbols cannot be interposed.

---

## Self-Assessment

### Q1: Build a cross-platform export macro system and apply it to a library API

A real library needs to compile correctly on both Windows (where you use `__declspec`) and Linux/macOS (where you use GCC/Clang visibility attributes). The standard approach is a single `export.h` header that detects the platform and defines consistent macros. Here is how that looks in practice:

```cpp
// === mylib/export.h ===
#ifndef MYLIB_EXPORT_H
#define MYLIB_EXPORT_H

// Detect platform and set up export/import macros
#if defined(_WIN32) || defined(__CYGWIN__)
    // Windows: symbols hidden by default, explicit export needed
    #ifdef MYLIB_BUILDING_DLL
        #define MYLIB_API __declspec(dllexport)
    #else
        #define MYLIB_API __declspec(dllimport)
    #endif
    #define MYLIB_HIDDEN
#elif defined(__GNUC__) && __GNUC__ >= 4
    // GCC/Clang: use visibility attributes
    // Library MUST be compiled with -fvisibility=hidden
    #define MYLIB_API    __attribute__((visibility("default")))
    #define MYLIB_HIDDEN __attribute__((visibility("hidden")))
#else
    // Fallback: no visibility control
    #define MYLIB_API
    #define MYLIB_HIDDEN
#endif

// Template export (needed for explicit instantiations)
#if defined(_WIN32)
    #ifdef MYLIB_BUILDING_DLL
        #define MYLIB_TEMPLATE_API __declspec(dllexport)
    #else
        #define MYLIB_TEMPLATE_API __declspec(dllimport)
    #endif
#else
    #define MYLIB_TEMPLATE_API __attribute__((visibility("default")))
#endif

// Deprecation markers combined with export
#define MYLIB_DEPRECATED_API  [[deprecated]] MYLIB_API

#endif  // MYLIB_EXPORT_H


// === mylib/logger.h ===
// #include "export.h"
#include <string>
#include <cstdio>

namespace mylib {

// Entire class exported - all members accessible
class MYLIB_API Logger {
public:
    enum class Level { Debug, Info, Warning, Error };

    Logger(const char* name);
    ~Logger();

    void log(Level level, const char* message);
    void set_level(Level min_level);

    // Static factory - also exported because class is MYLIB_API
    static Logger& instance();

private:
    struct Impl;           // Pimpl - internal layout hidden from users
    Impl* impl_;
};

// Free function - explicitly exported
MYLIB_API void configure_logging(const char* config_path);

// Internal helper - hidden even on non-Windows platforms
MYLIB_HIDDEN void internal_log_rotate();

// Deprecated but still exported for backward compat
MYLIB_DEPRECATED_API void init_logging();

}  // namespace mylib
```

The key idea is that `MYLIB_BUILDING_DLL` is defined only when compiling the library itself (you set it in your CMake or build system). When users include the header, that macro is not defined and `MYLIB_API` expands to `dllimport` on Windows or `visibility("default")` on Unix - both meaning "this symbol lives in the shared library, please import it."

### Q2: Use version scripts on Linux to precisely control which symbols are exported, including versioned symbol sets

Version scripts give you even finer control than visibility attributes - you can match symbol patterns, assign symbols to named version nodes, and have multiple implementations of the same function coexist in one `.so`. This is the mechanism glibc uses to maintain backward compatibility across decades.

```cpp
// === Build command ===
// g++ -shared -fvisibility=hidden -Wl,--version-script=mylib.map \
//     -o libmylib.so.2.0.0 mylib.cpp

// === mylib.map (version script) ===
/*
MYLIB_1.0 {
    global:
        # Export C++ mangled names for public API
        extern "C++" {
            mylib::Logger::Logger*;
            mylib::Logger::~Logger*;
            mylib::Logger::log*;
            mylib::Logger::set_level*;
            mylib::Logger::instance*;
            mylib::configure_logging*;
            typeinfo?for?mylib::Logger;
            vtable?for?mylib::Logger;
        };
    local:
        *;  # Hide everything else
};

MYLIB_2.0 {
    global:
        extern "C++" {
            mylib::Logger::flush*;
            mylib::Logger::set_output*;
            mylib::AsyncLogger::*;
        };
} MYLIB_1.0;  # Inherits from 1.0
*/

// === mylib.cpp - implementation with .symver directives ===
#include <cstdio>

namespace mylib {

// Original v1.0 implementation
void configure_logging_v1(const char* path) {
    std::printf("v1 config: %s\n", path);
}

// New v2.0 implementation with extended functionality
void configure_logging_v2(const char* path) {
    std::printf("v2 config: %s (with validation)\n", path);
    // Additional validation logic
}

// GNU extension: symbol versioning at the source level
// Bind the v1 implementation to the MYLIB_1.0 version
__asm__(".symver configure_logging_v1,_ZN5mylib19configure_loggingEPKc@MYLIB_1.0");
// Bind the v2 implementation as the default (@@) for MYLIB_2.0
__asm__(".symver configure_logging_v2,_ZN5mylib19configure_loggingEPKc@@MYLIB_2.0");

// Old binaries linked against MYLIB_1.0 call configure_logging_v1
// New binaries linked against MYLIB_2.0 call configure_logging_v2
// Both coexist in the same .so file!

}  // namespace mylib


// === Verify exported symbols ===
// $ nm -D libmylib.so | c++filt | grep configure
// T mylib::configure_logging(char const*)@@MYLIB_2.0
// T mylib::configure_logging(char const*)@MYLIB_1.0
//
// $ readelf -s --wide libmylib.so | grep GLOBAL | wc -l
// 12   (instead of hundreds with default visibility)
```

Notice that `@@` marks the default version (used by newly compiled code) while `@` marks a compat version (used by old binaries). Both functions live side by side in the same `.so`, and the linker picks the right one based on what was recorded at link time.

### Q3: Diagnose and fix common visibility-related issues: typeinfo across DSO boundaries, template instantiation, and vtable emission

Visibility mistakes rarely cause linker errors - they cause subtle runtime failures like `dynamic_cast` returning null, or two DSOs silently using different copies of the same class. Here are the three most common traps and their fixes:

```cpp
#include <cstdio>
#include <typeinfo>
#include <stdexcept>

// === Problem 1: typeinfo not exported -> dynamic_cast fails ===

// BAD: class hidden by -fvisibility=hidden
// If a plugin tries dynamic_cast<Base*>(ptr), it fails because
// the typeinfo for Base in the main DSO has different address
// than typeinfo for Base in the plugin DSO.
class /* MYLIB_API missing! */ Base {
public:
    virtual ~Base() = default;
    virtual void work() = 0;
};

// FIX: Export the class so typeinfo and vtable are shared
class MYLIB_API Base_Fixed {
public:
    virtual ~Base_Fixed() = default;
    virtual void work() = 0;
};
// Now both DSOs reference the same typeinfo -> dynamic_cast works


// === Problem 2: Template instantiation in headers ===

// With -fvisibility=hidden, template instantiations in different
// DSOs get separate copies with hidden visibility -> ODR violation

template <typename T>
class MYLIB_HIDDEN Stack {  // hidden by default
public:
    void push(const T& val) { data_[top_++] = val; }
    T pop() { return data_[--top_]; }
private:
    T data_[128];
    int top_ = 0;
};

// FIX: Explicitly instantiate and export in the library
// In header:
extern template class MYLIB_API Stack<int>;
extern template class MYLIB_API Stack<double>;

// In .cpp:
template class MYLIB_API Stack<int>;
template class MYLIB_API Stack<double>;
// Users get the single exported instantiation instead of
// generating their own hidden copy


// === Problem 3: vtable emission location ===

// GCC emits the vtable in the TU that defines the first
// non-inline virtual function. If ALL virtuals are inline,
// vtable is emitted in every TU -> weak symbol.
// With -fvisibility=hidden, each DSO gets its own vtable copy!

class MYLIB_API Interface {
public:
    virtual ~Interface() = default;
    virtual void method() = 0;          // pure virtual - no definition
    virtual void another() { /* inline */ }
};
// WARNING: No non-inline virtual -> vtable emitted as weak in every TU

// FIX: Provide at least one non-inline virtual function defined in the .cpp
// This anchors the vtable to a single TU in the library.
class MYLIB_API Interface_Fixed {
public:
    virtual ~Interface_Fixed();           // defined out-of-line in .cpp
    virtual void method() = 0;
    virtual void another() { /* inline ok now */ }
};
// In .cpp: Interface_Fixed::~Interface_Fixed() = default;
// Vtable and typeinfo emitted once -> correct visibility


int main() {
    std::printf("Visibility examples compiled successfully\n");
    return 0;
}
```

The destructor anchoring trick in Problem 3 is easy to forget. The reason this trips people up is that pure-virtual classes often have nothing obvious to put out of line. The destructor is the natural choice - just declare it in the header and define it as `= default` in the `.cpp` file. That is enough to anchor the vtable.

---

## Notes

- Always compile shared libraries with `-fvisibility=hidden` and then explicitly export only the public API with `MYLIB_API`. This is the single most impactful thing you can do for load time and ODR safety.
- On Windows, symbols are hidden by default - the inverse of Unix. `__declspec(dllexport)` makes Windows behave like Unix's `visibility("default")`.
- The vtable and typeinfo must be exported for any class used with `dynamic_cast` or `typeid` across DSO boundaries - missing this causes subtle runtime failures, not linker errors.
- Template instantiations with hidden visibility in different DSOs violate ODR - use explicit instantiation with export macros to prevent this.
- Anchor the vtable by having at least one non-inline virtual function defined in a `.cpp` file - this prevents duplicate vtable emission across translation units.
- Use `nm -D` (dynamic symbols) to audit your library's actual exports - aim to export only what's in your public headers.
- Version scripts (`-Wl,--version-script=`) provide finer control than visibility attributes alone - they can match patterns, version symbols, and strip specific internal symbols.
