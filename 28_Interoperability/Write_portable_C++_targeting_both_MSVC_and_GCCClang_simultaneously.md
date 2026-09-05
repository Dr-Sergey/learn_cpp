# Write portable C++ targeting both MSVC and GCC/Clang simultaneously

**Category:** Interoperability  
**Standard:** C++17  
**Reference:** <https://docs.microsoft.com/en-us/cpp/cpp/>  

---

## Topic Overview

Writing C++ that compiles cleanly on **MSVC**, **GCC**, and **Clang** requires careful handling of compiler-specific attributes, platform macros, header differences, and ABI/calling convention quirks. The goal is one codebase, three compilers, zero `#ifdef` spaghetti.

The reason this is harder than it sounds is that the three compilers have accumulated decades of incompatible extensions, and each has slightly different interpretations of the standard in edge cases. The good news is that if you stick to a handful of well-known portability rules and run CI against all three, the surprises become rare and predictable.

### Compiler Detection Macros

Before you can write portable code, you need to know which compiler you are talking to. These are the canonical detection macros.

```cpp
// Compiler identification
#if defined(_MSC_VER)
    // MSVC (or clang-cl): _MSC_VER = 1930+ for VS 2022
#elif defined(__clang__)
    // Clang: __clang_major__, __clang_minor__
#elif defined(__GNUC__)
    // GCC: __GNUC__, __GNUC_MINOR__
#endif

// Platform identification
#if defined(_WIN32)          // Windows (32 and 64-bit)
#elif defined(__linux__)     // Linux
#elif defined(__APPLE__)     // macOS / iOS
#endif

// Architecture
#if defined(_M_X64) || defined(__x86_64__)     // x64
#elif defined(_M_ARM64) || defined(__aarch64__) // ARM64
#endif
```

### Common Portability Pitfalls

These are the issues that catch people most often when they first try to port code between compilers.

| Issue | MSVC | GCC/Clang | Solution |
| --- | --- | --- | --- |
| `min`/`max` macros | `<windows.h>` defines them | Not defined | `#define NOMINMAX` |
| `__attribute__` | Not supported | Supported | Use `[[...]]` attributes |
| `__declspec` | Supported | Not supported | Portable macros |
| `#pragma once` | Supported | Supported | Works on all three |
| `__forceinline` | MSVC keyword | Not available | `__attribute__((always_inline))` |
| Signed overflow | Defined (wraps) | UB (optimized away) | Avoid relying on it |
| `/std:c++20` vs `-std=c++20` | Flag syntax differs | Flag syntax differs | Handle in CMake |

---

## Self-Assessment

### Q1: Replace __declspec(noinline) with a portable macro that maps to [[gnu::noinline]] on GCC/Clang

**Answer:**

The general strategy is to wrap every compiler-specific annotation in a `#if` chain and give the result a project-wide name that reads as intent rather than implementation. You write `PORTABLE_NOINLINE` and the reader immediately knows what it does, regardless of which compiler is underneath.

Note that in C++14 and later, standard attributes like `[[deprecated]]`, `[[nodiscard]]`, and `[[maybe_unused]]` work on all three compilers - prefer those where they exist. The macros below cover the cases where no standard attribute exists yet.

```cpp
// portable_attrs.h - Portable attribute macros
#pragma once

// No-inline
#if defined(_MSC_VER)
    #define PORTABLE_NOINLINE    __declspec(noinline)
#elif defined(__GNUC__) || defined(__clang__)
    #define PORTABLE_NOINLINE    __attribute__((noinline))
#else
    #define PORTABLE_NOINLINE
#endif

// Force inline
#if defined(_MSC_VER)
    #define PORTABLE_FORCEINLINE __forceinline
#elif defined(__GNUC__) || defined(__clang__)
    #define PORTABLE_FORCEINLINE inline __attribute__((always_inline))
#else
    #define PORTABLE_FORCEINLINE inline
#endif

// DLL export/import
#if defined(_WIN32)
    #ifdef MYLIB_EXPORTS
        #define MYLIB_API __declspec(dllexport)
    #else
        #define MYLIB_API __declspec(dllimport)
    #endif
#elif defined(__GNUC__)
    #define MYLIB_API __attribute__((visibility("default")))
#else
    #define MYLIB_API
#endif

// Deprecated (C++14 has [[deprecated]], but for older code:)
#if defined(_MSC_VER)
    #define PORTABLE_DEPRECATED(msg) __declspec(deprecated(msg))
#elif defined(__GNUC__) || defined(__clang__)
    #define PORTABLE_DEPRECATED(msg) __attribute__((deprecated(msg)))
#else
    #define PORTABLE_DEPRECATED(msg)
#endif

// Unreachable (C++23 has std::unreachable)
#if defined(_MSC_VER)
    #define PORTABLE_UNREACHABLE() __assume(false)
#elif defined(__GNUC__) || defined(__clang__)
    #define PORTABLE_UNREACHABLE() __builtin_unreachable()
#else
    #define PORTABLE_UNREACHABLE() ((void)0)
#endif

// Likely/Unlikely (pre-C++20)
#if defined(__GNUC__) || defined(__clang__)
    #define PORTABLE_LIKELY(x)   __builtin_expect(!!(x), 1)
    #define PORTABLE_UNLIKELY(x) __builtin_expect(!!(x), 0)
#else
    #define PORTABLE_LIKELY(x)   (x)
    #define PORTABLE_UNLIKELY(x) (x)
#endif
// In C++20, prefer [[likely]] and [[unlikely]] - supported on all three.

// Usage
#include <iostream>

PORTABLE_NOINLINE void debug_break_here(int value) {
    std::cerr << "Debug: " << value << "\n";
}

PORTABLE_FORCEINLINE int fast_abs(int x) {
    return x >= 0 ? x : -x;
}

class MYLIB_API Widget {  // Exported from DLL on Windows, visible on Linux
public:
    void process();
    PORTABLE_DEPRECATED("Use process() instead")
    void old_process();
};

int categorize(int x) {
    if (x > 0) return 1;
    if (x < 0) return -1;
    if (x == 0) return 0;
    PORTABLE_UNREACHABLE();  // Optimizer hint: can't reach here
}
// Compiles on: MSVC, GCC, Clang - same behavior on all three
```

### Q2: Handle Windows-specific _WIN32 / NOMINMAX guards in a cross-platform header

**Answer:**

The `NOMINMAX` problem is probably the most notorious Windows portability trap. When you include `<windows.h>`, it defines `min` and `max` as preprocessor macros. Those macros then expand anywhere those names appear - including inside `<algorithm>` - and break `std::min` and `std::max` with a compile error. The fix is to define `NOMINMAX` before any `<windows.h>` include, which tells Windows to skip those macro definitions.

The reason this trips people up is that the `<windows.h>` include is sometimes buried inside a third-party header you did not expect, so the macros appear without you explicitly including Windows headers yourself.

```cpp
// platform.h - Cross-platform foundation header
#pragma once

// STEP 1: Prevent Windows.h min/max macros
// Must be defined BEFORE any #include <windows.h>
#ifdef _WIN32
    #ifndef NOMINMAX
        #define NOMINMAX
    #endif
    #ifndef WIN32_LEAN_AND_MEAN
        #define WIN32_LEAN_AND_MEAN  // Exclude rarely-used APIs
    #endif
#endif

// STEP 2: Include platform headers
#ifdef _WIN32
    #include <windows.h>
#else
    #include <unistd.h>
    #include <sys/stat.h>
    #include <dlfcn.h>
#endif

#include <algorithm>  // std::min, std::max - NOW safe on Windows
#include <string>
#include <filesystem>
#include <cstdint>

// STEP 3: Portable type aliases
namespace platform {

#ifdef _WIN32
    using NativeHandle = HANDLE;
    constexpr char path_sep = '\\';
#else
    using NativeHandle = int;  // file descriptor
    constexpr char path_sep = '/';
#endif

// STEP 4: Portable API wrappers
inline std::string get_temp_dir() {
#ifdef _WIN32
    char buf[MAX_PATH];
    GetTempPathA(MAX_PATH, buf);
    return buf;
#else
    const char* tmp = std::getenv("TMPDIR");
    return tmp ? tmp : "/tmp";
#endif
}

inline uint64_t get_file_size(const std::string& path) {
    // Use std::filesystem - works on all platforms!
    return std::filesystem::file_size(path);
}

inline void sleep_ms(unsigned int ms) {
#ifdef _WIN32
    Sleep(ms);
#else
    usleep(ms * 1000);
#endif
}

// Dynamic library loading
class SharedLibrary {
    void* handle_ = nullptr;

public:
    explicit SharedLibrary(const std::string& path) {
#ifdef _WIN32
        handle_ = LoadLibraryA(path.c_str());
#else
        handle_ = dlopen(path.c_str(), RTLD_LAZY);
#endif
        if (!handle_) throw std::runtime_error("Failed to load: " + path);
    }

    template<typename Func>
    Func get_function(const std::string& name) {
#ifdef _WIN32
        auto ptr = GetProcAddress(static_cast<HMODULE>(handle_), name.c_str());
#else
        auto ptr = dlsym(handle_, name.c_str());
#endif
        if (!ptr) throw std::runtime_error("Symbol not found: " + name);
        return reinterpret_cast<Func>(ptr);
    }

    ~SharedLibrary() {
        if (handle_) {
#ifdef _WIN32
            FreeLibrary(static_cast<HMODULE>(handle_));
#else
            dlclose(handle_);
#endif
        }
    }

    SharedLibrary(const SharedLibrary&) = delete;
    SharedLibrary& operator=(const SharedLibrary&) = delete;
};

}  // namespace platform

// STEP 5: Verify min/max work
int main() {
    int a = 5, b = 10;
    int lo = std::min(a, b);   // Works on Windows after NOMINMAX
    int hi = std::max(a, b);   // Without NOMINMAX: compile error

    auto tmp = platform::get_temp_dir();
    platform::sleep_ms(100);

    return 0;
}
```

**NOMINMAX explained:**

```cpp
Without NOMINMAX on Windows:
  <windows.h> -> #define min(a,b) ...  #define max(a,b) ...
  <algorithm> -> std::min(a, b) -> macro-expanded -> COMPILE ERROR

With NOMINMAX:
  <windows.h> -> min/max macros suppressed
  <algorithm> -> std::min(a, b) -> works as expected

Alternative workaround (when you can't control the include order):
  (std::min)(a, b)   // Extra parentheses prevent macro expansion
  (std::max)(a, b)   // But NOMINMAX is the proper solution
```

### Q3: Use CI with both MSVC (windows-latest) and GCC (ubuntu-latest) to enforce portability

**Answer:**

The most reliable way to guarantee portability is to make it impossible to merge code that does not compile on all three compilers. A CI matrix that runs MSVC, GCC, and Clang in parallel catches problems immediately - before they are buried under subsequent commits.

Notice the `-Werror` / `/WX` flags: they treat warnings as errors, which is critical because both GCC and Clang will often warn about code that MSVC silently accepts, and vice versa. Portability issues frequently show up first as warnings on the other compiler.

```yaml
# .github/workflows/portable-ci.yml
name: Portable Build

on: [push, pull_request]

jobs:
  build:
    strategy:
      fail-fast: false  # Don't cancel other builds if one fails
      matrix:
        include:
          # MSVC on Windows
          - os: windows-latest
            compiler: msvc
            cmake_args: "-G \"Visual Studio 17 2022\" -A x64"
            cxx_flags: "/W4 /WX /std:c++20 /permissive-"

          # GCC on Linux
          - os: ubuntu-latest
            compiler: gcc-13
            cmake_args: "-G Ninja"
            cxx_flags: "-std=c++20 -Wall -Wextra -Werror -pedantic"
            install: "sudo apt-get install -y g++-13 ninja-build"

          # Clang on Linux
          - os: ubuntu-latest
            compiler: clang-17
            cmake_args: "-G Ninja -DCMAKE_CXX_COMPILER=clang++-17"
            cxx_flags: "-std=c++20 -Wall -Wextra -Werror -pedantic"
            install: >
              wget https://apt.llvm.org/llvm.sh &&
              chmod +x llvm.sh && sudo ./llvm.sh 17 &&
              sudo apt-get install -y ninja-build

          # Clang on macOS
          - os: macos-latest
            compiler: apple-clang
            cmake_args: "-G Ninja"
            cxx_flags: "-std=c++20 -Wall -Wextra -Werror"
            install: "brew install ninja"

    runs-on: ${{ matrix.os }}
    name: ${{ matrix.compiler }}

    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        if: matrix.install
        run: ${{ matrix.install }}

      - name: Configure
        run: >
          cmake -B build
          ${{ matrix.cmake_args }}
          -DCMAKE_CXX_FLAGS="${{ matrix.cxx_flags }}"
          -DCMAKE_BUILD_TYPE=Release

      - name: Build
        run: cmake --build build --config Release

      - name: Test
        run: ctest --test-dir build --config Release --output-on-failure
```

The CMake configuration is equally important. `CMAKE_CXX_EXTENSIONS OFF` is the flag most people miss - without it, CMake passes `-std=gnu++20` instead of `-std=c++20`, which silently enables GNU extensions and masks portability issues that would fail on MSVC or with `-pedantic`.

```cmake
# CMakeLists.txt - Portable CMake configuration
cmake_minimum_required(VERSION 3.20)
project(mylib LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)  # Critical: disable GNU extensions for portability

# Compiler-specific warning flags handled via CI matrix
# But we can also set sensible defaults:
if(MSVC)
    add_compile_options(/W4 /permissive-)
    add_compile_definitions(NOMINMAX WIN32_LEAN_AND_MEAN)
else()
    add_compile_options(-Wall -Wextra -Wpedantic)
endif()

add_library(mylib src/mylib.cpp)
target_include_directories(mylib PUBLIC include)

# Tests
enable_testing()
add_executable(tests tests/test_main.cpp)
target_link_libraries(tests mylib)
add_test(NAME unit_tests COMMAND tests)
```

Here is a reference for the most important flags on each toolchain:

```cpp
MSVC mandatory:
  /permissive-   Force standards-conforming behavior
  /Zc:__cplusplus Report correct __cplusplus value
  /W4            High warning level (not /Wall - too noisy)
  /WX            Treat warnings as errors

GCC/Clang mandatory:
  -std=c++20     (not -std=gnu++20 - avoids GNU extensions)
  -Wall -Wextra  Comprehensive warnings
  -Werror        Treat warnings as errors
  -pedantic      Reject non-standard extensions

CMake critical:
  CMAKE_CXX_EXTENSIONS OFF  // prevents -std=gnu++20
```

---

## Notes

- Always test with `/permissive-` on MSVC - the default permissive mode silently accepts non-conforming code that will fail on GCC/Clang.
- Use `std::filesystem` instead of platform-specific `stat`/`FindFirstFile` wherever you can - it is standardized and works on all three compilers since C++17.
- `#pragma once` works on all major compilers but is not technically standard. Combine it with traditional include guards if you need maximum portability guarantees.
- Prefer C++17/20 standard attributes (`[[nodiscard]]`, `[[maybe_unused]]`, `[[likely]]`) over compiler-specific `__attribute__` forms - the standard versions work everywhere.
- Use `std::byte` instead of `char`/`unsigned char` for binary data - it avoids the signed/unsigned char mismatch that plagues cross-platform code.
- Add `-Wsign-conversion` and MSVC's `/W4` to your build early - catching implicit narrowing conversions at the start is much easier than hunting them down later.
