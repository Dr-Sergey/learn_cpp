# Use the Highway library for portable SIMD without platform ifdefs

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://github.com/google/highway>  

---

## Topic Overview

Highway is Google's C++ SIMD library that provides a single API across x86 (SSE/AVX/AVX-512), ARM (NEON/SVE), WASM, RISC-V, and POWER. What makes it stand out from other portable SIMD approaches is its built-in runtime dispatch: you write the code once, it gets compiled into multiple target-specific versions in a single binary, and at startup it automatically picks the best version for the current CPU.

```cpp
Highway architecture:
  Your code (hwy::HWY_NAMESPACE) -> compiled N times
                                    for each target:
    +---------+  +---------+  +---------+  +---------+
    |  SSE4   |  |  AVX2   |  | AVX-512 |  |  NEON   |
    +---------+  +---------+  +---------+  +---------+
    HWY_DYNAMIC_DISPATCH selects the best at runtime.
    Single binary, zero per-call dispatch overhead (resolved at load time).
```

| Feature | Value |
| --- | --- |
| Targets | SSE4, AVX2, AVX-512, NEON, SVE, WASM SIMD, RISC-V V |
| Dispatch | Compile-time or runtime (dynamic) |
| API style | Free functions: `Add(d, a, b)` with tag dispatch |
| Install | Header-only or CMake `FetchContent` |
| Users | JPEG XL, Chrome, Chromium, WebRTC |

---

## Self-Assessment

### Q1: Element-wise multiply with Highway

Highway's structure is different from anything you'll have seen before. Your implementation lives in a header file (conventionally named `*-inl.h`) that gets re-included multiple times - once per target - via `foreach_target.h`. Each inclusion compiles the code for a different instruction set. The dispatch entry point then selects the best version at runtime. The pattern is unusual but it's the key to getting a single binary with multiple optimized paths.

```cpp
// Highway uses a unique pattern:
// 1. Code is written ONCE inside HWY_NAMESPACE
// 2. The .cc file is compiled multiple times via #include "impl-inl.h"
// 3. HWY_DYNAMIC_DISPATCH selects the best version at runtime

// --- multiply-inl.h (the implementation, compiled for each target) ---
#undef HWY_TARGET_INCLUDE
#define HWY_TARGET_INCLUDE "multiply-inl.h"
#include "hwy/foreach_target.h"  // re-includes this file for each target
#include "hwy/highway.h"

HWY_BEFORE_NAMESPACE();
namespace project {
namespace HWY_NAMESPACE {  // changes for each target: N_SSE4, N_AVX2, etc.

using namespace hwy::HWY_NAMESPACE;

void MulArrayImpl(const float* a, const float* b, float* c, size_t n) {
    const ScalableTag<float> d;  // descriptor: knows current SIMD width
    const size_t N = Lanes(d);   // 4 (SSE), 8 (AVX2), 16 (AVX-512)

    size_t i = 0;
    for (; i + N <= n; i += N) {
        auto va = Load(d, a + i);     // SIMD load
        auto vb = Load(d, b + i);
        auto vc = Mul(d, va, vb);     // SIMD multiply (not operator*!)
        Store(vc, d, c + i);          // SIMD store
    }
    // Scalar tail
    for (; i < n; ++i) c[i] = a[i] * b[i];
}

}  // namespace HWY_NAMESPACE
}  // namespace project
HWY_AFTER_NAMESPACE();

// --- multiply.cc (dispatch entry point) ---
// #include "multiply-inl.h"  (already included above for illustration)
// HWY_EXPORT(MulArrayImpl);  // creates dispatch table
// void MulArray(const float* a, const float* b, float* c, size_t n) {
//     HWY_DYNAMIC_DISPATCH(MulArrayImpl)(a, b, c, n);
// }

// Note: This is a conceptual example. Highway's actual build system
// uses foreach_target.h to recompile the -inl.h for each target.
```

Notice that inside the implementation, you use `Mul(d, va, vb)` rather than `va * vb`. Highway uses free functions with a tag-dispatch descriptor `d` rather than operator overloads. The descriptor carries the type and width information at compile time, which is what allows the same source line to compile to `vmulps ymm` on AVX2 and `vmul.4s` on NEON.

### Q2: Highway's dynamic dispatch mechanism

The dispatch mechanism is one of Highway's most practically useful features. It is similar to `ifunc` (GNU indirect functions) but portable. The resolution happens once at program startup, so every subsequent call goes directly to a function pointer with no branching overhead.

```cpp
#include <iostream>

// Highway dynamic dispatch explained:
//
// Step 1: HWY_EXPORT(FuncName)
//   Creates a static table of function pointers:
//     table[0] = FuncName compiled for SSE4
//     table[1] = FuncName compiled for AVX2
//     table[2] = FuncName compiled for AVX-512
//   (only targets enabled at compile time)
//
// Step 2: HWY_DYNAMIC_DISPATCH(FuncName)
//   First call: detects CPU with CPUID, selects best target
//   All calls: returns function pointer (resolved, no branch)
//   Equivalent to ifunc (see #725) but portable!
//
// Build flags:
//   g++ -O2 -mavx2      -> enables SSE4 + AVX2 targets
//   g++ -O2 -mavx512f   -> enables SSE4 + AVX2 + AVX-512
//   g++ -O2             -> enables SSE4 only
//
// The SAME source code compiles for all targets.
// Highway's foreach_target.h re-includes the -inl.h file
// once per target with different #defines.

int main() {
    std::cout << "Highway dispatch flow:\n";
    std::cout << "  1. Compile: -inl.h compiled N times (once per target)\n";
    std::cout << "  2. Link: all versions in one binary\n";
    std::cout << "  3. Runtime: CPUID selects best version\n";
    std::cout << "  4. Call: direct function pointer (zero overhead)\n";
    std::cout << "\nVs manual ifdef approach:\n";
    std::cout << "  #ifdef __AVX2__  -> one version per binary\n";
    std::cout << "  Highway          -> all versions in one binary\n";
    std::cout << "\nVs __builtin_cpu_supports:\n";
    std::cout << "  builtin_cpu_supports -> branch per call\n";
    std::cout << "  Highway              -> resolved once at startup\n";
}
```

The "resolved once at startup" point is important for hot paths. If you used `__builtin_cpu_supports` inside a loop, you'd have a branch on every iteration even though the answer is always the same. Highway's approach eliminates that by resolving the function pointer once and caching it.

### Q3: Highway `HWY_DYNAMIC_DISPATCH` vs `#ifdef __AVX2__` guards

The maintainability argument for Highway is compelling on its own, even before you factor in performance. The JPEG XL example in the code is real: what started as thousands of lines of ifdefs was refactored to a fraction of the size while maintaining or improving performance and gaining ARM and WASM support for free.

```cpp
#include <iostream>

int main() {
    std::cout << "Maintainability comparison:\n\n";
    std::cout << "+------------------------+-----------------------------+-----------------------------+\n";
    std::cout << "| Aspect                 | #ifdef __AVX2__             | Highway DYNAMIC_DISPATCH    |\n";
    std::cout << "+------------------------+-----------------------------+-----------------------------+\n";
    std::cout << "| Code duplication        | Full (copy per arch)        | None (one implementation)   |\n";
    std::cout << "| Bug fixes               | Fix in every #ifdef branch  | Fix once                    |\n";
    std::cout << "| Testing                 | Build & test N binaries     | One binary, one test        |\n";
    std::cout << "| New architecture        | Add new #ifdef + full impl  | Recompile (automatic)       |\n";
    std::cout << "| Lines of code           | N * base_code               | 1 * base_code + boilerplate |\n";
    std::cout << "| Runtime dispatch        | Manual if/else              | Automatic                   |\n";
    std::cout << "| Fallback handling       | Manual                      | Automatic (lowest target)   |\n";
    std::cout << "+------------------------+-----------------------------+-----------------------------+\n";

    // Real example: JPEG XL image codec
    //   Before Highway: 5000 lines of #ifdef'd SIMD code
    //   After Highway:  1500 lines of portable code
    //   Same performance, 3x less code, supports ARM+x86+WASM

    // CMake integration:
    //   include(FetchContent)
    //   FetchContent_Declare(highway
    //     GIT_REPOSITORY https://github.com/google/highway.git
    //     GIT_TAG master)
    //   FetchContent_MakeAvailable(highway)
    //   target_link_libraries(myapp PRIVATE hwy)
}
```

The "Fix once" row is the biggest practical win. When you have a bug in vectorized code, fixing it in every `#ifdef` branch is error-prone - it's easy to forget one, and each branch has to be tested separately. With Highway, one fix propagates to all targets automatically.

---

## Notes

- Highway uses `ScalableTag<T>` (tag dispatch) instead of fixed types like `__m256`.
- `Lanes(d)` returns the current target's SIMD width (not known at compile time for scalable).
- Highway API: `Add(d, a, b)` not `a + b` - the tag `d` selects the right instruction set.
- Install: `FetchContent` in CMake or `vcpkg install highway`.
- For new projects needing SIMD: Highway is the recommended choice for multi-platform support.
