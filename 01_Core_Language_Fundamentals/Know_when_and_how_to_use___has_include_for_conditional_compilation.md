# Know when and how to use __has_include for conditional compilation

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/preprocessor/include>  

---

## Topic Overview

`__has_include` is a **preprocessor feature test** introduced in C++17 that checks whether a
header file is available before `#include`-ing it. It lets you write portable code that
gracefully falls back to alternative libraries instead of failing to compile on older toolchains.

### Syntax

Both forms evaluate to `1` (truthy) if the header exists and can be included, `0` otherwise.
The check happens entirely at preprocessing time - it has zero runtime cost.

```cpp
#if __has_include(<header>)       // angle-bracket form (system/standard headers)
#if __has_include("header.h")     // quote form (project-local headers)
```

### Common Use Cases

| Pattern | Purpose |
| --- | --- |
| `__has_include(<optional>)` | Use `std::optional` if available, else `boost::optional` |
| `__has_include(<filesystem>)` | Use `std::filesystem` or fall back to `boost::filesystem` |
| `__has_include(<format>)` | Use `std::format` if supported (C++20) |
| `__has_include("config.h")` | Detect project-specific configuration headers |

### Basic Pattern - Fallback to Boost

Here's the classic use: check whether the standard header exists, and if not, fall through to
a Boost equivalent. The `#error` at the bottom makes sure you get a clear message rather than
cryptic missing-type errors later:

```cpp
#if __has_include(<optional>)
    #include <optional>
    using opt = std::optional<int>;
#elif __has_include(<boost/optional.hpp>)
    #include <boost/optional.hpp>
    using opt = boost::optional<int>;
#else
    #error "No optional implementation available"
#endif
```

### Combining with Feature-Test Macros

Here's the thing: `__has_include` only tells you the header **exists**, not whether a
specific feature is actually implemented. A compiler might ship a stub `<optional>` that
doesn't define everything yet. Combine it with `__cpp_lib_*` macros for a precise check:

```cpp
#if __has_include(<optional>)
    #include <optional>
    #if defined(__cpp_lib_optional) && __cpp_lib_optional >= 201606L
        // Full std::optional is available
        #define HAS_STD_OPTIONAL 1
    #else
        // Header exists but feature is incomplete
        #define HAS_STD_OPTIONAL 0
    #endif
#else
    #define HAS_STD_OPTIONAL 0
#endif
```

Think of this as a two-gate check: gate one confirms the file is there, gate two confirms
the implementation is complete.

### Limitations

1. **No version checking** - `__has_include(<optional>)` cannot distinguish between a partial C++17 `<optional>` and a complete one.
2. **No content inspection** - it only checks if the file exists, not what it defines.
3. **Preprocessor only** - cannot be used in `if constexpr` or runtime expressions.
4. **Compiler-dependent** - older compilers may not support `__has_include` at all; wrap in `#ifdef __has_include`.

### Portability Guard

When you need to support compilers old enough to not know `__has_include` itself, wrap the
whole thing in an `#ifdef`. The pattern below is safe even on a pre-C++17 compiler:

```cpp
#ifdef __has_include                         // guard for pre-C++17 compilers
    #if __has_include(<span>)
        #include <span>
        #define HAS_SPAN 1
    #endif
#endif

#ifndef HAS_SPAN
    #define HAS_SPAN 0
    // provide a polyfill or disable the feature
#endif
```

---

## Self-Assessment

### Q1: Use `__has_include(<optional>)` to provide a fallback to `boost::optional` on older compilers

The goal here is a compatibility header that makes the rest of your code blissfully unaware
of which `optional` it's using. Notice how user code only ever touches `compat::optional`:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// ---- optional_compat.h ----
#pragma once

#ifdef __has_include
    #if __has_include(<optional>)
        #include <optional>
        namespace compat {
            template<typename T>
            using optional = std::optional<T>;
            inline constexpr auto nullopt = std::nullopt;
        }
    #elif __has_include(<boost/optional.hpp>)
        #include <boost/optional.hpp>
        namespace compat {
            template<typename T>
            using optional = boost::optional<T>;
            inline const auto nullopt = boost::none;
        }
    #else
        #error "No optional implementation found"
    #endif
#else
    // Pre-C++17 compiler: assume Boost
    #include <boost/optional.hpp>
    namespace compat {
        template<typename T>
        using optional = boost::optional<T>;
        inline const auto nullopt = boost::none;
    }
#endif

// ---- main.cpp ----
#include "optional_compat.h"
#include <iostream>

int main() {
    compat::optional<int> val = 42;
    if (val) {
        std::cout << "Value: " << *val << "\n";
    }

    compat::optional<int> empty;
    std::cout << "Has value? " << (empty ? "yes" : "no") << "\n";
}
```

**How it works:**

- The preprocessor first checks if `<optional>` exists; if so, aliases `std::optional` into `compat::`.
- If not, it tries `<boost/optional.hpp>` as a fallback.
- User code only uses `compat::optional<T>` - swapping implementations is transparent.

### Q2: Show `__has_include` combined with `__cpp_lib_optional` for a complete feature check

This is the two-step pattern in practice. Step one confirms the header is present; step two
confirms the implementation actually delivers the feature at the required version level:

```cpp
#include <iostream>

// Step 1: Check header availability
#if __has_include(<optional>)
    #include <optional>
#endif

// Step 2: Check feature-test macro for complete implementation
#if defined(__cpp_lib_optional) && __cpp_lib_optional >= 201606L
    #define OPTIONAL_AVAILABLE 1
#else
    #define OPTIONAL_AVAILABLE 0
#endif

// Step 3: Check other useful features the same way
#if __has_include(<any>)
    #include <any>
#endif
#if defined(__cpp_lib_any) && __cpp_lib_any >= 201606L
    #define ANY_AVAILABLE 1
#else
    #define ANY_AVAILABLE 0
#endif

int main() {
    std::cout << "std::optional available: " << OPTIONAL_AVAILABLE << "\n";
    std::cout << "std::any available:      " << ANY_AVAILABLE << "\n";

#if OPTIONAL_AVAILABLE
    std::optional<std::string> greeting = "Hello, C++17!";
    std::cout << greeting.value_or("(empty)") << "\n";
#endif

#if ANY_AVAILABLE
    std::any a = 3.14;
    std::cout << "any holds double: " << std::any_cast<double>(a) << "\n";
#endif
}
```

**How it works:**

- `__has_include(<optional>)` confirms the header file exists.
- `__cpp_lib_optional >= 201606L` confirms the implementation is feature-complete.
- This two-step check prevents using a header that exists but is only a stub or partial implementation.

### Q3: Explain the limitations: `__has_include` cannot check for specific versions of a header

**Answer:**

1. **No version granularity:** `__has_include(<optional>)` returns 1 if the file exists at all - even if the compiler ships a broken or incomplete `<optional>`. You cannot write `__has_include(<optional> >= 201606L)`.

2. **No content introspection:** The preprocessor does not parse the included file; it only checks the filesystem. A header could define none of the expected types and `__has_include` would still succeed.

3. **Solution - feature-test macros:** The standard defines `__cpp_lib_*` macros (e.g., `__cpp_lib_optional`, `__cpp_lib_ranges`) with version numbers. Always combine them with `__has_include`:

```cpp
#if __has_include(<ranges>)
    #include <ranges>
    #if defined(__cpp_lib_ranges) && __cpp_lib_ranges >= 202110L
        // C++23 ranges with fold, zip, etc.
    #elif defined(__cpp_lib_ranges)
        // Basic C++20 ranges only
    #endif
#endif
```

4. **Cannot check non-header resources:** Only works on files that would be found by `#include`. Cannot test for compiler builtins, command-line defines, or linked libraries.

5. **Preprocessor-time only:** Cannot be used inside `if constexpr` or template metaprogramming - it is purely a preprocessing directive.

---

## Notes

- Always guard `__has_include` with `#ifdef __has_include` for portability with pre-C++17 compilers.
- The full list of standard library feature-test macros is in `<version>` (C++20) or the SD-6 document.
- `__has_include` is useful in library code that must support multiple platforms - application code that targets a fixed C++ standard usually doesn't need it.
- Both GCC, Clang, and MSVC support `__has_include` as an extension even in C++14 mode.
