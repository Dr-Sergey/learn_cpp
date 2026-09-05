# Choose between header-only, compiled, and module-based library distribution

**Category:** API and Library Design  
**Standard:** C++20  
**Reference:** <https://isocpp.org/wiki/faq/cpp20-library-design>  

---

## Topic Overview

When you ship a C++ library, one of the first decisions is how users actually consume it. Do they `#include` a header and get everything inline? Do they link against a compiled `.a` or `.lib`? Or, if you can require C++20, do you give them a module they `import`? Each model has real tradeoffs, and the right choice depends on your library's size, its template-heaviness, and your users' build constraints.

### Comparison

The table below captures the high-level tradeoffs. If it feels like a lot, the core tension is this: header-only is the most convenient to distribute but the most expensive to compile; compiled is fast to build against but loses template flexibility; modules aim to give you both.

| Aspect | Header-Only | Compiled (.a/.lib) | C++20 Modules |
| --- | --- | --- | --- |
| Integration | `#include` only | Link step required | `import` |
| Build speed | Slow (recompiled per TU) | Fast (compiled once) | Fast |
| Template support | Full | Explicit instantiation needed | Full |
| ABI coupling | High (inline everything) | Low (stable .so) | Medium |
| Ease of use | Simplest | Moderate | Moderate |
| Example | nlohmann/json, Catch2 | Boost.Filesystem, OpenSSL | std module |

### When to Choose Each

The right model usually follows directly from what your library is. Here is a plain-language guide for each option, including the warning signs that tell you you have picked the wrong one.

```cpp
Header-only:
  // GOOD: Mostly templates / constexpr
  // GOOD: Small library (< 10K LOC)
  // GOOD: Users want zero build config
  // BAD: Slow compile times for large libraries

Compiled library:
  // GOOD: Non-template code
  // GOOD: ABI stability needed (shared library)
  // GOOD: Large codebase
  // BAD: Users must link correctly

C++20 Modules:
  // GOOD: Best of both worlds (templates + fast build)
  // GOOD: New projects with modern compiler requirements
  // BAD: Toolchain support still maturing (2026)
```

### Header-Only Library Pattern

The key rule for header-only libraries is that every non-template function must be marked `inline`, or you will violate the One Definition Rule the moment two translation units include the header. Templates are naturally exempt because the compiler handles them differently. Here is what a minimal, correct header-only library looks like:

```cpp
// mylib.hpp - entire library in one header
#ifndef MYLIB_HPP
#define MYLIB_HPP

#include <string>
#include <vector>

namespace mylib {

// Templates are naturally header-only
template<typename T>
T clamp(T val, T lo, T hi) {
    return val < lo ? lo : val > hi ? hi : val;
}

// Non-template functions: use inline to avoid ODR violations
inline std::string trim(std::string_view sv) {
    auto start = sv.find_first_not_of(" \t\n\r");
    auto end = sv.find_last_not_of(" \t\n\r");
    if (start == std::string_view::npos) return {};
    return std::string(sv.substr(start, end - start + 1));
}

} // namespace mylib
#endif
```

This works perfectly for small, template-heavy libraries. The downside shows up at scale - if fifty source files include this header, `trim` gets compiled fifty times.

### Compiled Library Pattern

A compiled library splits the public interface (the header) from the implementation (the `.cpp` file). The `MYLIB_API` macro handles the platform difference between Windows `__declspec` and GCC/Clang `__attribute__` visibility. Notice how the implementation is completely hidden - users only ever see `api.hpp`.

```cpp
// mylib/api.hpp - public header
#pragma once
#include <string>

#ifdef MYLIB_BUILDING
  #define MYLIB_API __declspec(dllexport)  // or __attribute__((visibility("default")))
#else
  #define MYLIB_API __declspec(dllimport)
#endif

namespace mylib {
    MYLIB_API std::string process(const std::string& input);
    MYLIB_API int version();
}

// mylib/api.cpp - compiled once, linked
#define MYLIB_BUILDING
#include "api.hpp"

namespace mylib {
    std::string process(const std::string& input) {
        // Implementation hidden from users
        return "processed: " + input;
    }
    int version() { return 2; }
}
```

The user compiles their code and links against your `.a` or `.dll`. They never recompile your implementation. This is the right model whenever you need ABI stability or have a large non-template codebase.

---

## Self-Assessment

### Q1: What are the ODR risks with header-only libraries

Every translation unit that includes the header gets its own copy of all functions. Non-inline, non-template functions in headers violate the One Definition Rule because the linker sees multiple identical definitions. The fix is to mark all non-template functions `inline`, or use anonymous namespaces for internal helpers (though the latter creates separate copies per translation unit rather than truly sharing one definition).

### Q2: How do modules solve the header-only build time problem

Modules are compiled once into a Binary Module Interface (BMI). When 100 source files `import mylib;`, the module is parsed and type-checked once, not 100 times. This eliminates the redundant parsing, macro expansion, and template instantiation that `#include` causes in every translation unit that uses the header.

### Q3: Show a hybrid approach: header-only with optional compiled mode

Many popular libraries (fmt, spdlog) let users choose at compile time. A preprocessor macro controls whether the functions are `inline` (header-only mode) or not (compiled mode). Here is the pattern:

```cpp
// config.hpp
#ifdef MYLIB_HEADER_ONLY
  #define MYLIB_INLINE inline
#else
  #define MYLIB_INLINE
#endif

// api.hpp
#pragma once
#include "config.hpp"

namespace mylib {
    MYLIB_INLINE std::string process(const std::string& input);
}

#ifdef MYLIB_HEADER_ONLY
  #include "api_impl.hpp" // Include implementation
#endif

// Users choose: -DMYLIB_HEADER_ONLY or link libmylib.a
```

This lets users who want zero build config use the header-only mode, while users building large projects can switch to the compiled mode to keep their builds fast.

---

## Notes

- Most modern C++ libraries (fmt, spdlog, Eigen) offer both header-only and compiled modes for exactly this reason.
- C++20 modules are the future but require CMake 3.28+ and recent compilers - tooling is still catching up as of 2026.
- Header-only libraries are easier for package managers (vcpkg, Conan) to distribute because there is no build step involved.
