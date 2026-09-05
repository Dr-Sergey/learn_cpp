# Build shared library architecture with proper symbol visibility

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Symbol visibility** controls which functions and classes are accessible from outside a shared library (`.so`/`.dll`). By default, GCC exports everything; MSVC exports nothing. Proper visibility reduces library size, improves load times, prevents symbol collisions, and enforces a clean public API boundary.

The reason this matters more than it might seem: every exported symbol has a cost. The dynamic linker has to process it at load time. Two libraries that export a symbol with the same name can silently conflict. And from a design standpoint, exporting a symbol is making a promise to your users - you cannot easily change or remove it later without breaking ABI compatibility. Controlling visibility carefully forces you to be intentional about what your library's public surface actually is.

### Visibility Defaults

| Compiler | Default | Result |
| --- | --- | --- |
| GCC/Clang (Linux) | All symbols visible | Everything exported (bad) |
| MSVC (Windows) | Nothing exported | Must `__declspec(dllexport)` each symbol |
| GCC with `-fvisibility=hidden` | All hidden | Must explicitly mark exports |

---

## Self-Assessment

### Q1: Implement cross-platform export macros

**Answer:**

The standard approach is a single header with platform-detection macros that map to the right compiler annotation. On MSVC you use `dllexport`/`dllimport`; on GCC/Clang you use visibility attributes. The `MYLIB_BUILDING` define is set by CMake only when building the library itself - consumers never define it:

```cpp
// === mylib_export.h - cross-platform visibility macros ===
#pragma once

#if defined(_MSC_VER)
    // MSVC: dllexport when building, dllimport when consuming
    #ifdef MYLIB_BUILDING
        #define MYLIB_API __declspec(dllexport)
    #else
        #define MYLIB_API __declspec(dllimport)
    #endif
    #define MYLIB_HIDDEN
#elif defined(__GNUC__) || defined(__clang__)
    // GCC/Clang: use visibility attributes
    #define MYLIB_API __attribute__((visibility("default")))
    #define MYLIB_HIDDEN __attribute__((visibility("hidden")))
#else
    #define MYLIB_API
    #define MYLIB_HIDDEN
#endif

// === CMakeLists.txt ===
// add_library(mylib SHARED src/mylib.cpp)
// target_compile_definitions(mylib PRIVATE MYLIB_BUILDING)
// if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
//     target_compile_options(mylib PRIVATE -fvisibility=hidden)
// endif()

// === Using the macros ===
// mylib.h
#include "mylib_export.h"

namespace mylib {

// Exported: visible to consumers
class MYLIB_API Widget {
public:
    Widget();
    void do_something();
    int get_value() const;

private:
    // Private members are part of the binary layout but not the API
    class Impl;  // PIMPL hides implementation details
    std::unique_ptr<Impl> impl_;
};

MYLIB_API void initialize();
MYLIB_API void shutdown();
MYLIB_API const char* version();

// NOT exported: internal helper
MYLIB_HIDDEN void internal_helper();

}  // namespace mylib
```

The `dllimport` annotation on the consumer side is not just cosmetic - it tells the compiler the symbol lives in a DLL and generates more efficient import thunk code. This is why the same header needs different macros depending on whether you are building or consuming the library.

### Q2: Use CMake's GenerateExportHeader for automated macros

**Answer:**

Writing the visibility macros by hand is error-prone. CMake's built-in `GenerateExportHeader` module generates them for you based on the target name, and it handles the static library case automatically:

```cmake
# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.20)
project(mylib VERSION 1.0.0)

add_library(mylib SHARED
    src/widget.cpp
    src/init.cpp
)

# Automatically generate export macros
include(GenerateExportHeader)
generate_export_header(mylib
    EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/include/mylib_export.h
    EXPORT_MACRO_NAME MYLIB_API
    NO_EXPORT_MACRO_NAME MYLIB_HIDDEN
    STATIC_DEFINE MYLIB_STATIC
)

# Set hidden visibility by default (GCC/Clang)
set_target_properties(mylib PROPERTIES
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN YES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
)

target_include_directories(mylib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

# Support both shared and static builds
# When built as STATIC, MYLIB_STATIC is defined and macros expand to nothing
```

```cpp
// === Generated mylib_export.h looks like: ===
// #ifdef MYLIB_STATIC
//   #define MYLIB_API
//   #define MYLIB_HIDDEN
// #else
//   #ifdef mylib_EXPORTS
//     #define MYLIB_API __declspec(dllexport)  // or __attribute__((visibility("default")))
//   #else
//     #define MYLIB_API __declspec(dllimport)
//   #endif
//   #define MYLIB_HIDDEN
// #endif
```

The `SOVERSION` property is worth understanding. It sets the `libmylib.so.1` symlink that clients link against. If you make breaking API changes, you bump `SOVERSION`, and the old `.so.1` can coexist with the new `.so.2` on the same system - existing binaries keep working.

### Q3: Design ABI-stable library with PIMPL

**Answer:**

Symbol visibility tells the linker what to export. ABI stability is a harder problem: it means that a binary compiled against version 1.0 of your library still works correctly when loaded against version 1.1, without recompilation. The PIMPL (pointer to implementation) idiom is the standard C++ solution. The public header declares only the pointer; the actual data layout lives in the `.cpp` file and can change freely:

```cpp
// === Public header: ABI-stable, never changes layout ===
// include/mylib/engine.h
#include "mylib_export.h"
#include <memory>
#include <string>

namespace mylib {

class MYLIB_API Engine {
public:
    Engine();
    ~Engine();  // Must be in .cpp where Impl is complete

    // Move only (PIMPL)
    Engine(Engine&&) noexcept;
    Engine& operator=(Engine&&) noexcept;

    // Public API: stable across SO versions
    bool initialize(const std::string& config_path);
    void process();
    int status() const;

private:
    class Impl;  // Defined in .cpp only
    std::unique_ptr<Impl> impl_;
};

}  // namespace mylib

// === Implementation: free to change without ABI break ===
// src/engine.cpp
namespace mylib {

class Engine::Impl {
public:
    bool initialize(const std::string& config_path) {
        // Can add/remove/change fields freely
        config_ = load_config(config_path);
        db_ = open_database(config_.db_path);
        return true;
    }
    void process() { /* ... */ }
    int status() const { return status_; }

private:
    // These can change between library versions
    Config config_;
    Database db_;
    int status_ = 0;
    // Adding fields here does NOT break ABI
};

Engine::Engine() : impl_(std::make_unique<Impl>()) {}
Engine::~Engine() = default;
Engine::Engine(Engine&&) noexcept = default;
Engine& Engine::operator=(Engine&&) noexcept = default;

bool Engine::initialize(const std::string& p) {
    return impl_->initialize(p);
}
void Engine::process() { impl_->process(); }
int Engine::status() const { return impl_->status(); }

}  // namespace mylib
```

The reason `~Engine()` must be defined in the `.cpp` file even though it is `= default`: at the point where the destructor runs, `std::unique_ptr<Impl>` needs to know the complete type of `Impl` to call its destructor. If the destructor were inlined in the header, the compiler would see an incomplete `Impl` type and refuse to compile. Moving `= default` to the `.cpp` fixes this.

---

## Notes

- **Always use `-fvisibility=hidden`** (GCC/Clang) - export only what you explicitly intend to.
- CMake's `GenerateExportHeader` handles cross-platform macros automatically and is the recommended approach.
- **PIMPL idiom** is essential for ABI stability: changing `Impl` does not change the header layout that consumers compiled against.
- Exported classes should not have inline functions that use non-exported types.
- Template classes cannot be exported - they must be in headers (or explicitly instantiated and exported).
- Use `nm -D libmylib.so` or `dumpbin /exports mylib.dll` to inspect which symbols are actually exported.
- Version your SO: `libmylib.so.1.0.0` with `SOVERSION 1` so compatible updates can coexist with older binaries.
