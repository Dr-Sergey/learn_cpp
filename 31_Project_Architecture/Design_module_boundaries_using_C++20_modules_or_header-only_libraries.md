# Design module boundaries using C++20 modules or header-only libraries

**Category:** Project Architecture  
**Standard:** C++20  

---

## Topic Overview

**Module boundaries** define what a component exposes and what it hides. C++20 modules replace the header/source split with explicit `export` declarations, giving precise control over the public API. Header-only libraries remain relevant for template-heavy code and maximum portability. Choosing the right boundary mechanism affects build times, encapsulation, and API clarity.

Here's the thing that trips people up about traditional headers: everything in a header is visible to every translation unit that includes it, including all the internal helper functions and implementation details you never intended to make public. C++20 modules fix this by requiring you to explicitly `export` what you want the outside world to see - everything else is module-private and genuinely invisible to consumers.

### Comparison

| Aspect | C++20 Modules | Header-Only | Traditional Header/Source |
| --- | --- | --- | --- |
| Build time | Fast (compiled once) | Slow (re-parsed each TU) | Medium |
| API control | Explicit `export` | Everything visible | Convention-based |
| Template support | Full | Full | Requires header |
| Compiler support | GCC 14+, Clang 17+, MSVC 19.28+ | Universal | Universal |
| Macro leakage | None | Full | Full |

---

## Self-Assessment

### Q1: Design module boundaries with C++20 modules

With modules, you write a primary module interface file (`.cppm`) that declares what's exported, and an implementation unit (`.cpp`) that provides the definitions. The consumer just writes `import math;` and gets access to everything in the `export namespace math` block - and nothing else. The `detail` namespace here is completely invisible to the consumer; it doesn't even appear in the module's compiled representation.

Watch how the template function `lerp` lives entirely in the interface file - that's fine with modules. Unlike traditional headers, you don't pay a re-parse cost for it in every translation unit.

**Answer:**

```cpp
// === math.cppm (primary module interface) ===
export module math;

// Internal implementation (NOT exported)
namespace detail {
    constexpr double PI = 3.14159265358979323846;
    double normalize_angle(double radians) {
        // ... reduce to [0, 2*PI)
        return radians;
    }
}

// Exported: public API
export namespace math {
    struct Vec2 {
        double x, y;
        double length() const;
        Vec2 normalized() const;
    };

    struct Vec3 {
        double x, y, z;
        double length() const;
        Vec3 cross(const Vec3& other) const;
        double dot(const Vec3& other) const;
    };

    // Free functions
    double degrees_to_radians(double deg);
    double radians_to_degrees(double rad);

    // Template function (works in modules)
    template<typename T>
    T lerp(const T& a, const T& b, double t) {
        return a + (b - a) * t;
    }
}

// === math.cpp (module implementation unit) ===
module math;

namespace math {
    double Vec2::length() const {
        return std::sqrt(x * x + y * y);
    }
    Vec2 Vec2::normalized() const {
        double len = length();
        return {x / len, y / len};
    }
    // ... other implementations
}

// === Consumer ===
import math;

void example() {
    math::Vec2 v{3.0, 4.0};
    double len = v.length();  // OK: exported

    // detail::normalize_angle(1.0);  // ERROR: not exported
    // detail::PI;                    // ERROR: not exported

    auto mid = math::lerp(v, math::Vec2{6.0, 8.0}, 0.5);
}
```

This is a much cleaner encapsulation story than `#pragma once` headers where everything leaks through. The consumer sees `math::Vec2`, `math::lerp`, etc., and the compiler enforces that `detail::` truly stays private.

### Q2: Use module partitions for large modules

When a module grows big, you split it into **partitions**. Each partition is its own file with a name like `graphics:types`, and they're assembled into a unified interface by the primary module. A consumer just writes `import graphics;` and gets everything - they never need to know about the partition structure.

The key syntax to notice: within a partition, `import :types;` (with the leading colon) imports a sibling partition by its short name. The primary module file uses `export import :types;` to re-export the partition's contents to consumers.

**Answer:**

```cpp
// === Large module split into partitions ===

// --- graphics:types (partition) ---
export module graphics:types;

export struct Color { float r, g, b, a; };
export struct Rect { int x, y, int w, h; };
export struct Vertex { float pos[3]; float uv[2]; Color color; };

// --- graphics:renderer (partition) ---
export module graphics:renderer;
import :types;  // Import sibling partition

export class IRenderer {
public:
    virtual ~IRenderer() = default;
    virtual void clear(Color bg) = 0;
    virtual void draw_rect(Rect r, Color c) = 0;
    virtual void submit(const Vertex* verts, size_t count) = 0;
};

// --- graphics:texture (partition) ---
export module graphics:texture;
import :types;

export class Texture {
public:
    Texture(int width, int height);
    void set_pixel(int x, int y, Color c);
    Color get_pixel(int x, int y) const;
private:
    std::vector<Color> data_;  // Not exported
    int width_, height_;
};

// --- graphics (primary interface - re-exports partitions) ---
export module graphics;
export import :types;
export import :renderer;
export import :texture;

// === Consumer sees unified module ===
import graphics;

void draw(IRenderer& r) {
    r.clear({0, 0, 0, 1});
    r.draw_rect({10, 10, 100, 100}, {1, 0, 0, 1});
}
```

From the consumer's perspective this looks just like one big `graphics` module. The partition structure is an internal organizational detail. This lets large teams split a module across files without creating a complex import graph for consumers to deal with.

### Q3: Design a header-only library with clean boundaries

Header-only libraries remain the most portable choice, especially for template-heavy code where definitions must be visible to the compiler at every call site. The main discipline is using a `detail/` directory (and `detail` namespace) as a convention to mark internal helpers that users shouldn't depend on.

Notice the CMakeLists.txt snippet at the bottom - a header-only library uses `INTERFACE` linkage, meaning it has no compiled sources. The `$<BUILD_INTERFACE:...>` / `$<INSTALL_INTERFACE:...>` generator expressions ensure the right include paths are set whether the library is used from the build tree or after installation.

**Answer:**

```cpp
// === Header-only library structure ===
// include/mylib/mylib.hpp          <-- Single-include entry point
// include/mylib/core/types.hpp
// include/mylib/core/algorithm.hpp
// include/mylib/detail/impl.hpp    <-- Internal, not part of API

// --- include/mylib/mylib.hpp ---
#pragma once
// Public API headers only
#include "mylib/core/types.hpp"
#include "mylib/core/algorithm.hpp"
// NOTE: detail/ is NOT included here

// --- include/mylib/core/types.hpp ---
#pragma once

namespace mylib {

template<typename T, std::size_t N>
class SmallVector {
public:
    void push_back(const T& val);
    T& operator[](std::size_t i);
    std::size_t size() const { return size_; }
    static constexpr std::size_t capacity() { return N; }

private:
    std::array<T, N> storage_;
    std::size_t size_ = 0;
};

// Template implementation in same header (required for header-only)
template<typename T, std::size_t N>
void SmallVector<T, N>::push_back(const T& val) {
    if (size_ >= N) throw std::out_of_range("SmallVector full");
    storage_[size_++] = val;
}

}  // namespace mylib

// --- include/mylib/detail/impl.hpp ---
// INTERNAL: users should not include this directly
#pragma once
namespace mylib::detail {
    // Implementation helpers, not part of public API
    template<typename T>
    constexpr bool is_trivially_relocatable_v =
        std::is_trivially_copyable_v<T> &&
        std::is_trivially_destructible_v<T>;
}

// === CMakeLists.txt for header-only library ===
// add_library(mylib INTERFACE)
// target_include_directories(mylib INTERFACE
//     $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
//     $<INSTALL_INTERFACE:include>
// )
// target_compile_features(mylib INTERFACE cxx_std_20)
```

The `detail/` directory convention is just that - a convention - so users *can* include internal headers if they want. That's the downside vs. true modules. The discipline is enforced by code review and documentation, not the compiler.

---

## Notes

- **C++20 modules** give true encapsulation: non-exported names are invisible, no macro leakage.
- Module **partitions** let you split large modules while presenting a unified API.
- Header-only libraries: use `detail/` namespace and directory convention to mark internals.
- For template-heavy code, header-only is still the pragmatic choice (maximum compatibility).
- CMake module support: use `CXX_SCAN_FOR_MODULES` property (CMake 3.28+).
- Never `#include` inside a module - use `import` or `import <header>` for standard library.
- Module build order matters: BMI (Binary Module Interface) files must be built before consumers.
