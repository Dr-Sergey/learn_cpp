# Use Conditional Compilation and Feature Detection Macros

**Category:** Cross Platform Development  
**Standard:** C++17, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/feature_test>  

---

## Topic Overview

Cross-platform C++ relies on compile-time decisions: which OS APIs to call, which language features are available, and which headers exist. The traditional approach - testing platform macros like `_WIN32` or `__linux__` - tells you *where* you are but not *what the compiler supports*. Modern C++ inverts this with feature-test macros (`__cpp_*` and `__has_include`), which let you test capabilities directly rather than inferring them from compiler versions.

| Macro Category | Examples | What It Tests |
| --- | --- | --- |
| **Platform** | `_WIN32`, `__linux__`, `__APPLE__`, `__ANDROID__` | Target operating system |
| **Compiler** | `__GNUC__`, `__clang__`, `_MSC_VER`, `__INTEL_COMPILER` | Compiler identity/version |
| **Architecture** | `__x86_64__`, `__aarch64__`, `__arm__`, `_M_X64` | CPU architecture |
| **Language feature** | `__cpp_concepts`, `__cpp_constexpr`, `__cpp_modules` | C++ standard feature support |
| **Library feature** | `__cpp_lib_ranges`, `__cpp_lib_format`, `__cpp_lib_jthread` | Standard library support |
| **Header presence** | `__has_include(<optional>)`, `__has_include(<span>)` | Header availability |
| **Attribute** | `__has_cpp_attribute(nodiscard)`, `__has_cpp_attribute(likely)` | Attribute support |

The key principle here is to **prefer feature detection over platform detection**. Testing `__cpp_lib_format >= 202110L` is more precise than testing `_MSC_VER >= 1930`, because the former directly answers "can I use `std::format`?" while the latter requires you to maintain a mapping of compiler versions to feature support - a mapping that goes stale as compilers are updated.

`__has_include` (standardized in C++17) checks whether a header can be `#include`d. This is invaluable for optional dependencies and polyfills. It evaluates at preprocessing time and works inside `#if` directives.

When you have multiple detection strategies available, apply them in this order - more specific and more reliable first:

```cpp
Decision hierarchy (prefer top to bottom):

  1. __cpp_lib_xxx >= value   ->  "Does the library support feature X?"
  2. __cpp_xxx >= value       ->  "Does the language support feature X?"
  3. __has_include(<header>)  ->  "Is this header available?"
  4. __has_cpp_attribute(a)   ->  "Is attribute [[a]] available?"
  5. _WIN32 / __linux__       ->  "What platform am I on?"     (last resort)
```

---

## Self-Assessment

### Q1: Write a portability header that selects the best available formatting facility (`std::format`, `fmt::format`, or `snprintf` fallback)

This is a classic portability pattern: detect the best available implementation at compile time and expose a single `pfmt::format` function that the rest of the codebase calls without knowing which backend is active. Notice how each detection step guards against the previous one having already succeeded.

```cpp
// portable_format.hpp
#pragma once

// Step 1: Detect std::format
#if defined(__has_include)
    #if __has_include(<format>)
        #include <format>
        #if defined(__cpp_lib_format) && __cpp_lib_format >= 202110L
            #define PFMT_HAS_STD_FORMAT 1
        #endif
    #endif
#endif

// Step 2: Fall back to fmtlib
#if !defined(PFMT_HAS_STD_FORMAT)
    #if defined(__has_include) && __has_include(<fmt/format.h>)
        #include <fmt/format.h>
        #define PFMT_HAS_FMTLIB 1
    #endif
#endif

// Step 3: snprintf fallback
#if !defined(PFMT_HAS_STD_FORMAT) && !defined(PFMT_HAS_FMTLIB)
    #include <cstdio>
    #include <string>
    #include <array>
    #define PFMT_FALLBACK 1
#endif

namespace pfmt {

#if defined(PFMT_HAS_STD_FORMAT)
    // Best: standard library
    template <typename... Args>
    std::string format(std::format_string<Args...> fmt, Args&&... args) {
        return std::format(fmt, std::forward<Args>(args)...);
    }

    inline constexpr const char* backend_name = "std::format";

#elif defined(PFMT_HAS_FMTLIB)
    // Good: fmtlib (API-compatible with std::format)
    template <typename... Args>
    std::string format(fmt::format_string<Args...> fmt, Args&&... args) {
        return fmt::format(fmt, std::forward<Args>(args)...);
    }

    inline constexpr const char* backend_name = "fmt::format";

#elif defined(PFMT_FALLBACK)
    // Minimal fallback: only supports a single string or integer
    // (Real projects would use a more complete fallback)
    template <typename T>
    std::string format(const char* fmt, T value) {
        std::array<char, 256> buf;
        int n = std::snprintf(buf.data(), buf.size(), fmt, value);
        return std::string(buf.data(), n > 0 ? static_cast<std::size_t>(n) : 0);
    }

    inline constexpr const char* backend_name = "snprintf";
#endif

} // namespace pfmt

// Usage:
// #include "portable_format.hpp"
// auto s = pfmt::format("Hello, {}!", name);
```

The reason the `__has_include(<format>)` check alone is not enough is that a header can exist without the feature being complete - for example, MSVC shipped a partial `<format>` in some versions. The `__cpp_lib_format >= 202110L` check catches that distinction.

### Q2: Create a macro abstraction layer that maps compiler-specific attributes to standard C++ attributes with graceful fallback

A common mistake is writing `#ifdef __has_cpp_attribute` as if `__has_cpp_attribute` might not be defined at all on older compilers. It might not be - so always gate it with `#if defined(__has_cpp_attribute)` first. This example shows the correct idiom for every attribute you are likely to need:

```cpp
// portable_attributes.hpp
#pragma once

// [[nodiscard]] with message (C++20)
#if defined(__has_cpp_attribute)
    #if __has_cpp_attribute(nodiscard) >= 201907L
        #define PAL_NODISCARD(msg) [[nodiscard(msg)]]
    #elif __has_cpp_attribute(nodiscard)
        #define PAL_NODISCARD(msg) [[nodiscard]]
    #else
        #define PAL_NODISCARD(msg)
    #endif
#else
    #define PAL_NODISCARD(msg)
#endif

// [[likely]] / [[unlikely]] (C++20)
#if defined(__has_cpp_attribute) && __has_cpp_attribute(likely) >= 201803L
    #define PAL_LIKELY   [[likely]]
    #define PAL_UNLIKELY [[unlikely]]
#elif defined(__GNUC__)
    // GCC/Clang: use __builtin_expect in if conditions instead
    #define PAL_LIKELY
    #define PAL_UNLIKELY
    #define PAL_EXPECT(expr, val) __builtin_expect(!!(expr), val)
#else
    #define PAL_LIKELY
    #define PAL_UNLIKELY
#endif

#ifndef PAL_EXPECT
    #define PAL_EXPECT(expr, val) (expr)
#endif

// [[no_unique_address]] (C++20) — MSVC uses [[msvc::no_unique_address]]
#if defined(__has_cpp_attribute)
    #if __has_cpp_attribute(no_unique_address) >= 201803L
        #define PAL_NO_UNIQUE_ADDR [[no_unique_address]]
    #elif defined(_MSC_VER) && __has_cpp_attribute(msvc::no_unique_address)
        #define PAL_NO_UNIQUE_ADDR [[msvc::no_unique_address]]
    #else
        #define PAL_NO_UNIQUE_ADDR
    #endif
#else
    #define PAL_NO_UNIQUE_ADDR
#endif

// Force-inline
#if defined(_MSC_VER)
    #define PAL_FORCEINLINE __forceinline
#elif defined(__GNUC__)
    #define PAL_FORCEINLINE inline __attribute__((always_inline))
#else
    #define PAL_FORCEINLINE inline
#endif

// DLL export/import
#if defined(_WIN32)
    #define PAL_EXPORT __declspec(dllexport)
    #define PAL_IMPORT __declspec(dllimport)
#elif defined(__GNUC__)
    #define PAL_EXPORT __attribute__((visibility("default")))
    #define PAL_IMPORT
#else
    #define PAL_EXPORT
    #define PAL_IMPORT
#endif

// Demo usage
#include <iostream>

struct PAL_EXPORT Result {
    PAL_NO_UNIQUE_ADDR struct {} tag;  // Empty base optimization
    int value;
};

PAL_NODISCARD("ignoring error code is a bug")
PAL_FORCEINLINE int compute(int x) {
    if (x > 0) PAL_LIKELY {
        return x * 2;
    } else PAL_UNLIKELY {
        return -1;
    }
}

int main() {
    auto r = compute(42);
    std::cout << "Result: " << r << "\n";
    std::cout << "sizeof(Result): " << sizeof(Result) << "\n";
    return 0;
}
```

The `[[no_unique_address]]` case is a good example of why you cannot just use `__has_cpp_attribute` blindly: MSVC implements the same optimization but under the vendor-namespaced `[[msvc::no_unique_address]]` spelling. The two-level check handles both.

### Q3: Use feature-test macros to write a constexpr-capable function that adapts to the available C++ standard level

The `__cpp_constexpr` macro tracks the evolution of constexpr through the standards. Each version unlocked new capabilities - loops and local variables in C++14, lambdas in C++17, virtual functions and try-catch in C++20. This example selects the best available constexpr form and reports what the compiler actually supports at runtime:

```cpp
#include <type_traits>
#include <cstdint>
#include <iostream>
#include <array>

// Detect constexpr capability level
// C++14: __cpp_constexpr >= 201304 (relaxed constexpr)
// C++17: __cpp_constexpr >= 201603 (constexpr lambdas)
// C++20: __cpp_constexpr >= 201907 (constexpr virtual, try-catch, etc.)
// C++23: __cpp_constexpr >= 202211 (constexpr casts, etc.)

// Detect if constexpr std::vector is available (C++20 constexpr allocator)
#if defined(__cpp_lib_constexpr_vector) && __cpp_lib_constexpr_vector >= 202202L
    #define HAS_CONSTEXPR_VECTOR 1
    #include <vector>
#else
    #define HAS_CONSTEXPR_VECTOR 0
#endif

// Detect if constexpr algorithms are available
#if defined(__cpp_lib_constexpr_algorithms) && __cpp_lib_constexpr_algorithms >= 201806L
    #define HAS_CONSTEXPR_ALGO 1
    #include <algorithm>
#else
    #define HAS_CONSTEXPR_ALGO 0
#endif

// Compile-time string hash with best available features
constexpr std::uint64_t fnv1a_hash(const char* str) noexcept {
    std::uint64_t hash = 0xcbf29ce484222325ULL;

    #if defined(__cpp_constexpr) && __cpp_constexpr >= 201304L
    // C++14: relaxed constexpr with loops
    while (*str) {
        hash ^= static_cast<std::uint64_t>(*str++);
        hash *= 0x100000001b3ULL;
    }
    #else
    // C++11: recursive constexpr
    // (would need different implementation)
    #endif

    return hash;
}

// Constexpr sort: picks best available implementation
template <typename T, std::size_t N>
constexpr std::array<T, N> sorted(std::array<T, N> arr) {
    #if HAS_CONSTEXPR_ALGO
    // C++20: use standard constexpr algorithms
    std::sort(arr.begin(), arr.end());
    #else
    // Fallback: constexpr bubble sort (C++14 relaxed constexpr)
    for (std::size_t i = 0; i < N; ++i)
        for (std::size_t j = i + 1; j < N; ++j)
            if (arr[j] < arr[i]) {
                T tmp = arr[i];
                arr[i] = arr[j];
                arr[j] = tmp;
            }
    #endif
    return arr;
}

// Compile-time feature report
constexpr void print_features() {
    // These are compile-time constants, evaluated in constexpr context
}

int main() {
    // Compile-time hash
    constexpr auto h = fnv1a_hash("hello_world");
    static_assert(h != 0, "Hash should be non-zero");
    std::cout << "Hash: " << h << "\n";

    // Compile-time sorted array
    constexpr auto data = sorted(std::array{5, 3, 1, 4, 2});
    static_assert(data[0] == 1 && data[4] == 5, "Array should be sorted");

    for (auto v : data) std::cout << v << " ";
    std::cout << "\n";

    // Feature report
    std::cout << "constexpr level: " << __cpp_constexpr << "\n";
    #ifdef __cpp_lib_constexpr_algorithms
    std::cout << "constexpr algorithms: " << __cpp_lib_constexpr_algorithms << "\n";
    #else
    std::cout << "constexpr algorithms: not available\n";
    #endif

    return 0;
}
```

Notice the `static_assert` checks: they run at compile time and confirm that the sorted array is actually sorted. This is the real payoff of constexpr - you can validate correctness at compile time, with zero runtime cost, and get a readable error if something is wrong.

---

## Notes

- Feature-test macros are defined in `<version>` (C++20) - include it for library feature macros. Language feature macros are predefined by the compiler without any include.
- `__has_include` only checks preprocessor-level inclusion; it does not guarantee the header's contents are usable (for example, a header may exist but require a specific `-std=` flag to expose its content).
- MSVC defines `_MSC_VER` as a 4-digit number (for example, 1940 for VS 2022 17.10). GCC uses `__GNUC__` (major) plus `__GNUC_MINOR__`. Clang defines `__clang_major__`.
- The `__has_cpp_attribute` test should always be guarded by `#if defined(__has_cpp_attribute)` - older compilers do not recognize the built-in and will produce a preprocessor error.
- Prefer `#if` over `#ifdef` for feature-test macros - `#ifdef __cpp_concepts` is true even if the macro is defined to 0, but `#if __cpp_concepts >= 202002L` tests the actual version number.
- Keep all portability macros in a single `platform.hpp` or `compat.hpp` header to avoid scattering `#ifdef` throughout the codebase.
