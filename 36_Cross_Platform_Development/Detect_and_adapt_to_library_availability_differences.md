# Detect and Adapt to Library Availability Differences

**Category:** Cross Platform Development  
**Standard:** C++17, C++20  
**Reference:** <https://cmake.org/cmake/help/latest/command/find_package.html>  

---

## Topic Overview

Cross-platform C++ projects must handle the reality that libraries available on one platform may be absent, differently versioned, or API-incompatible on another. The two complementary strategies are **feature detection** (testing what is available at compile time or build time) and **graceful degradation** (providing reduced functionality when a dependency is missing).

| Strategy | Level | Mechanism | Example |
| --- | --- | --- | --- |
| Build-time detection | CMake / build system | `find_package`, `check_symbol_exists` | Detect OpenSSL, use fallback TLS |
| Preprocess-time detection | Preprocessor | `__has_include`, `#ifdef HAS_LIBFOO` | Include `<format>` or `fmt/format.h` |
| Compile-time adaptation | Templates / `if constexpr` | SFINAE, concepts, tag dispatch | Use `std::from_chars` or `strtod` |
| Link-time selection | Linker / CMake | `target_link_libraries(... OPTIONAL)` | Link `liburing` if present |
| Runtime detection | `dlopen` / `LoadLibrary` | Dynamic loading + function pointers | GPU driver capabilities |

An important principle: feature detection is strictly superior to platform detection for library availability. Testing `find_package(Threads REQUIRED)` succeeds on any platform with a threading library, whereas hardcoding `if(UNIX) target_link_libraries(... pthread)` breaks on MinGW, Cygwin, or exotic POSIX systems. Test what you actually need, not what OS you think you are on.

The **polyfill pattern** provides a local implementation of a facility that may not be available: if the platform has `<span>`, use it; otherwise, compile a bundled `span.hpp`. This is the C++ equivalent of JavaScript polyfills and is widely used in header-only libraries. The flow diagram below shows how a CMake-based detection pipeline propagates availability flags all the way into your C++ source:

```cpp
Build-time detection flow:

  CMakeLists.txt
       │
  find_package(OpenSSL)     --- Found? --► #define HAS_OPENSSL 1
       │                                     -> link OpenSSL::SSL
       │-- Not found --► option: USE_BORINGSSL? --► link BoringSSL
       │                    │
       │                    └-- Not found --► #define USE_BUILTIN_TLS 1
       │                                      -> compile embedded TLS
       ▼
  configure_file(config.h.in -> config.h)   <- propagates defines
```

---

## Self-Assessment

### Q1: Write a CMake module that detects optional libraries and generates a C++ config header with availability flags

The pattern here has three moving parts: `find_package` detects libraries, `check_include_file_cxx` / `check_cxx_symbol_exists` probe for specific headers and symbols, and `configure_file` bakes all the results into a generated `config.h` that your C++ code can include. This keeps all the detection logic in the build system where it belongs:

```cmake
# CMakeLists.txt - Feature detection and config generation

cmake_minimum_required(VERSION 3.20)
project(portable_app LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# Optional dependency detection
find_package(OpenSSL QUIET)
find_package(ZLIB QUIET)
find_package(fmt QUIET)
find_package(Threads REQUIRED)

# Check for specific symbols / headers in the C++ compiler
include(CheckIncludeFileCXX)
include(CheckCXXSymbolExists)

check_include_file_cxx("filesystem" HAS_FILESYSTEM_HEADER)
check_include_file_cxx("format"     HAS_FORMAT_HEADER)
check_include_file_cxx("execution"  HAS_EXECUTION_HEADER)

check_cxx_symbol_exists(posix_memalign "cstdlib" HAS_POSIX_MEMALIGN)
check_cxx_symbol_exists(_aligned_malloc "malloc.h" HAS_ALIGNED_MALLOC)

# Generate config header
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/config.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/config.h"
)

# Build target
add_executable(app main.cpp)
target_include_directories(app PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
target_link_libraries(app PRIVATE Threads::Threads)

if(OpenSSL_FOUND)
    target_link_libraries(app PRIVATE OpenSSL::SSL OpenSSL::Crypto)
endif()

if(ZLIB_FOUND)
    target_link_libraries(app PRIVATE ZLIB::ZLIB)
endif()

if(fmt_FOUND)
    target_link_libraries(app PRIVATE fmt::fmt)
endif()
```

The template file uses `#cmakedefine01`, which generates either `1` or `0` - always a defined token. This is safer than plain `#cmakedefine`, which generates either a `#define` or nothing, making `#if` vs `#ifdef` usage confusing:

```cpp
// config.h.in - Template for CMake configure_file
#pragma once

// Library availability (set by CMake)
#cmakedefine01 OpenSSL_FOUND
#cmakedefine01 ZLIB_FOUND
#cmakedefine01 fmt_FOUND

// Standard library features
#cmakedefine01 HAS_FILESYSTEM_HEADER
#cmakedefine01 HAS_FORMAT_HEADER
#cmakedefine01 HAS_EXECUTION_HEADER

// Platform capabilities
#cmakedefine01 HAS_POSIX_MEMALIGN
#cmakedefine01 HAS_ALIGNED_MALLOC
```

The resulting `config.h` flows into your application code, where you use ordinary `#if` to choose between available implementations. The key benefit is that the selection logic is in one place and driven by actual detection results rather than hardcoded platform assumptions:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// main.cpp - Uses generated config
#include "config.h"
#include <iostream>

#if HAS_FORMAT_HEADER && defined(__cpp_lib_format)
    #include <format>
    #define USE_STD_FORMAT 1
#elif fmt_FOUND
    #include <fmt/format.h>
    #define USE_FMT 1
#endif

int main() {
    std::cout << "Feature report:\n";
    std::cout << "  OpenSSL:    " << (OpenSSL_FOUND ? "yes" : "no") << "\n";
    std::cout << "  zlib:       " << (ZLIB_FOUND ? "yes" : "no") << "\n";
    std::cout << "  std::format:" << (HAS_FORMAT_HEADER ? "yes" : "no") << "\n";

    #if defined(USE_STD_FORMAT)
    std::cout << std::format("  Using: {}\n", "std::format");
    #elif defined(USE_FMT)
    std::cout << fmt::format("  Using: {}\n", "fmt::format");
    #else
    std::cout << "  Using: printf fallback\n";
    #endif

    return 0;
}
```

### Q2: Implement a polyfill for `std::span` that is used when the standard header is unavailable

The polyfill pattern gives you a single include that automatically falls back to a bundled implementation when the standard one is unavailable. `__has_include` checks whether the header exists, and the feature-test macro `__cpp_lib_span` confirms that the implementation is complete. If either check fails, the bundled `polyfill::span` is compiled instead, and the `compat::span` alias points to whichever version is active:

```cpp
// portable_span.hpp - Use std::span if available, otherwise provide a polyfill
#pragma once

#if defined(__has_include)
    #if __has_include(<span>) && defined(__cpp_lib_span) && __cpp_lib_span >= 202002L
        #include <span>
        #define HAS_STD_SPAN 1
    #endif
#endif

#if !defined(HAS_STD_SPAN)
// Minimal polyfill matching the std::span API subset
#include <cstddef>
#include <type_traits>
#include <iterator>
#include <array>

namespace polyfill {

inline constexpr std::size_t dynamic_extent = static_cast<std::size_t>(-1);

template <typename T, std::size_t Extent = dynamic_extent>
class span {
    T*          data_  = nullptr;
    std::size_t size_  = 0;

public:
    using element_type    = T;
    using value_type      = std::remove_cv_t<T>;
    using size_type       = std::size_t;
    using pointer         = T*;
    using reference       = T&;
    using iterator        = T*;
    using const_iterator  = const T*;

    constexpr span() noexcept = default;
    constexpr span(T* ptr, std::size_t count) : data_(ptr), size_(count) {}
    constexpr span(T* first, T* last) : data_(first), size_(last - first) {}

    template <std::size_t N>
    constexpr span(T (&arr)[N]) noexcept : data_(arr), size_(N) {}

    template <std::size_t N>
    constexpr span(std::array<value_type, N>& arr) noexcept
        : data_(arr.data()), size_(N) {}

    constexpr T*          data()  const noexcept { return data_; }
    constexpr std::size_t size()  const noexcept { return size_; }
    constexpr bool        empty() const noexcept { return size_ == 0; }

    constexpr T& operator[](std::size_t idx) const { return data_[idx]; }
    constexpr T& front() const { return data_[0]; }
    constexpr T& back()  const { return data_[size_ - 1]; }

    constexpr iterator begin() const noexcept { return data_; }
    constexpr iterator end()   const noexcept { return data_ + size_; }

    constexpr span<T> subspan(std::size_t offset, std::size_t count = dynamic_extent) const {
        return span<T>(data_ + offset,
                       count == dynamic_extent ? size_ - offset : count);
    }

    constexpr span<T> first(std::size_t count) const { return {data_, count}; }
    constexpr span<T> last(std::size_t count)  const { return {data_ + size_ - count, count}; }
};

// Deduction guides
template <typename T, std::size_t N>
span(T (&)[N]) -> span<T, N>;

template <typename T, std::size_t N>
span(std::array<T, N>&) -> span<T, N>;

} // namespace polyfill
#endif

// Unified namespace
namespace compat {
    #if defined(HAS_STD_SPAN)
    using std::span;
    #else
    using polyfill::span;
    #endif
}

// Usage
#include <iostream>
#include <vector>
#include <numeric>

void process(compat::span<const int> data) {
    int sum = 0;
    for (int v : data) sum += v;
    std::cout << "Sum of " << data.size() << " elements: " << sum << "\n";
}

// Example main
/*
int main() {
    std::vector<int> vec(100);
    std::iota(vec.begin(), vec.end(), 1);

    process(compat::span<const int>(vec.data(), vec.size()));

    int arr[] = {10, 20, 30};
    process(arr);

    return 0;
}
*/
```

Notice that `process` accepts `compat::span<const int>` - it does not know or care whether that resolves to `std::span` or `polyfill::span`. This is the whole point: the polyfill disappears once you upgrade your toolchain.

### Q3: Demonstrate runtime library detection using `dlopen`/`LoadLibrary` for optional GPU acceleration

Runtime detection is appropriate when you cannot know at build time whether a library will be present on the user's machine - a GPU compute library is the classic example. The `DynamicLibrary` wrapper below hides the `LoadLibraryA`/`dlopen` difference, and `ComputeEngine` tries a list of GPU libraries in order before falling back to a CPU implementation:

```cpp
#include <iostream>
#include <functional>
#include <string>
#include <memory>

#ifdef _WIN32
    #define WIN32_LEAN_AND_MEAN
    #include <windows.h>
    using LibHandle = HMODULE;
#else
    #include <dlfcn.h>
    using LibHandle = void*;
#endif

// Platform-agnostic dynamic library loader
class DynamicLibrary {
    LibHandle handle_ = nullptr;
    std::string name_;

public:
    explicit DynamicLibrary(const std::string& path) : name_(path) {
        #ifdef _WIN32
        handle_ = ::LoadLibraryA(path.c_str());
        #else
        handle_ = ::dlopen(path.c_str(), RTLD_LAZY | RTLD_LOCAL);
        #endif
    }

    ~DynamicLibrary() {
        if (handle_) {
            #ifdef _WIN32
            ::FreeLibrary(handle_);
            #else
            ::dlclose(handle_);
            #endif
        }
    }

    DynamicLibrary(const DynamicLibrary&) = delete;
    DynamicLibrary& operator=(const DynamicLibrary&) = delete;

    bool is_loaded() const { return handle_ != nullptr; }

    template <typename FuncPtr>
    FuncPtr get_function(const char* name) const {
        if (!handle_) return nullptr;
        #ifdef _WIN32
        return reinterpret_cast<FuncPtr>(::GetProcAddress(handle_, name));
        #else
        return reinterpret_cast<FuncPtr>(::dlsym(handle_, name));
        #endif
    }

    std::string error() const {
        #ifdef _WIN32
        return "LoadLibrary error: " + std::to_string(::GetLastError());
        #else
        const char* err = ::dlerror();
        return err ? err : "unknown error";
        #endif
    }
};

// GPU acceleration with graceful fallback
class ComputeEngine {
    using MatMulFn = void(*)(const float*, const float*, float*, int);

    std::unique_ptr<DynamicLibrary> gpu_lib_;
    MatMulFn gpu_matmul_ = nullptr;

    // CPU fallback (naive)
    static void cpu_matmul(const float* a, const float* b, float* c, int n) {
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j) {
                float sum = 0;
                for (int k = 0; k < n; ++k)
                    sum += a[i * n + k] * b[k * n + j];
                c[i * n + j] = sum;
            }
    }

public:
    ComputeEngine() {
        // Try loading GPU library at runtime
        const char* lib_names[] = {
            #ifdef _WIN32
            "cuda_compute.dll", "opencl_compute.dll",
            #elif defined(__APPLE__)
            "libmetal_compute.dylib",
            #else
            "libcuda_compute.so", "libopencl_compute.so",
            #endif
        };

        for (const char* name : lib_names) {
            gpu_lib_ = std::make_unique<DynamicLibrary>(name);
            if (gpu_lib_->is_loaded()) {
                gpu_matmul_ = gpu_lib_->get_function<MatMulFn>("gpu_matmul");
                if (gpu_matmul_) {
                    std::cout << "GPU acceleration: " << name << "\n";
                    return;
                }
            }
        }
        std::cout << "GPU acceleration: not available, using CPU fallback\n";
    }

    void matmul(const float* a, const float* b, float* c, int n) {
        if (gpu_matmul_) {
            gpu_matmul_(a, b, c, n);
        } else {
            cpu_matmul(a, b, c, n);
        }
    }

    bool has_gpu() const { return gpu_matmul_ != nullptr; }
};

int main() {
    ComputeEngine engine;

    constexpr int N = 4;
    float a[N * N] = {1,0,0,0, 0,1,0,0, 0,0,1,0, 0,0,0,1}; // identity
    float b[N * N] = {1,2,3,4, 5,6,7,8, 9,10,11,12, 13,14,15,16};
    float c[N * N] = {};

    engine.matmul(a, b, c, N);

    std::cout << "Result[0][0] = " << c[0] << " (expected: 1)\n";
    std::cout << "Result[1][1] = " << c[5] << " (expected: 6)\n";
    std::cout << "Backend: " << (engine.has_gpu() ? "GPU" : "CPU") << "\n";

    return 0;
}
```

Notice the pattern: check that the library loaded, then check that the specific function symbol exists. A library can load successfully but be missing specific symbols - always verify both.

---

## Notes

- **Feature detection > platform detection**: `check_cxx_symbol_exists` and `__has_include` test the actual capability, avoiding stale platform-to-feature mappings.
- CMake's `configure_file` with `#cmakedefine01` generates `0` or `1`, safe for use in both `#if` and `if constexpr` contexts. Prefer it over `#cmakedefine` which generates `#define` or nothing.
- Polyfills should match the standard API exactly (same member functions, deduction guides, namespace conventions) so the `#if` switch between polyfill and standard is seamless.
- For `dlopen`/`LoadLibrary`, always check both the library load AND each function pointer - a library may load successfully but lack specific symbols.
- Use `RTLD_LOCAL` (not `RTLD_GLOBAL`) for optional libraries to avoid symbol conflicts with other loaded libraries.
- Provide a clear "capability report" at startup (which optional features are active) to simplify debugging deployment issues.
