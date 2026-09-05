# Write Portable SIMD Code with Highway and std::simd

**Category:** Cross Platform Development  
**Standard:** C++23, C++26 (Parallelism TS v2)  
**Reference:** <https://en.cppreference.com/w/cpp/experimental/simd>  

---

## Topic Overview

SIMD (Single Instruction, Multiple Data) is essential for high-performance numeric code, but writing portable SIMD is notoriously difficult. The three major ISAs - x86 SSE/AVX, ARM NEON/SVE, and RISC-V V - have incompatible intrinsics, different register widths, and distinct semantic models (fixed-width vs. scalable vectors). Portable SIMD libraries abstract these differences behind a unified API with minimal overhead. If you have ever tried to maintain separate SSE2, AVX2, and NEON code paths for the same algorithm, you already know why this matters.

| Library | Status | Width Model | Runtime Dispatch | Platforms |
| --- | --- | --- | --- | --- |
| `std::experimental::simd` (Parallelism TS v2) | TS, partial impl. | Fixed (ABI-width) | No | GCC libstdc++ |
| `std::simd` (C++26 target) | In progress | Fixed | No | Future |
| Google Highway | Production, active | Scalable (tag-based) | Yes (`HWY_DYNAMIC_DISPATCH`) | x86, ARM, WASM, RISC-V |
| xsimd | Production | Fixed | Optional (batch selection) | x86, ARM |
| Agner Fog VCL | Mature | Fixed (explicit) | Manual | x86 only |
| Eve | Production | Fixed | Yes | x86, ARM |

Highway uses a tag-based dispatch model where the same source code is compiled for multiple targets, and a runtime dispatcher selects the best one. This is the "write once, run on any SIMD width" approach. Key design: functions are templates parameterized by a `D` (descriptor) tag that encodes the lane type and the current target width, so the same kernel body works for 128-bit SSE2 and 512-bit AVX-512 without any changes.

Here is a picture of where the abstraction layer sits:

```cpp
SIMD abstraction layers:

  Application  --------------------------------
       |
  +----v----------------------------------------+
  |  Portable SIMD API (Highway / std::simd)    |
  +---------+----------+------------------------+
  | SSE/AVX | NEON/SVE |  WASM SIMD / RVV       |
  +---------+----------+------------------------+
       |
  Hardware registers (128/256/512+ bits)
```

The `std::simd` proposal (`std::simd<T, Abi>`) follows a different philosophy: the `Abi` tag selects a fixed width at compile time (for example, `simd<float, simd_abi::fixed_size<8>>`), and there is no built-in runtime dispatch. Libraries like Highway complement this by adding the dispatch layer on top.

---

## Self-Assessment

### Q1: Write a portable SIMD vector addition using Google Highway with runtime dispatch

The unusual-looking include structure at the top is intentional: Highway's multi-target compilation model requires the file to include itself multiple times, once per SIMD target. `HWY_BEFORE_NAMESPACE` and `HWY_AFTER_NAMESPACE` create a distinct namespace for each compiled version so they can all coexist in the same binary.

<!-- compile: needs third-party library header `hwy/foreach_target.h` -->
```cpp
// Requires: google/highway library (CMake: FetchContent or find_package)
// Build with: -march=native (or Highway handles multi-target compilation)

#undef HWY_TARGET_INCLUDE
#define HWY_TARGET_INCLUDE "simd_add.cpp"  // This file
#include "hwy/foreach_target.h"           // Generates code for each target
#include "hwy/highway.h"

#include <cstddef>
#include <iostream>
#include <vector>

HWY_BEFORE_NAMESPACE();
namespace project {
namespace HWY_NAMESPACE {  // Unique per target

namespace hn = hwy::HWY_NAMESPACE;

// Core kernel — compiled once per SIMD target
void AddArraysImpl(const float* HWY_RESTRICT a,
                   const float* HWY_RESTRICT b,
                   float* HWY_RESTRICT out,
                   std::size_t count) {
    const hn::ScalableTag<float> d;          // Descriptor: adapts to lane width
    const std::size_t N = hn::Lanes(d);      // Lanes at runtime (4, 8, 16...)

    std::size_t i = 0;
    for (; i + N <= count; i += N) {
        auto va = hn::Load(d, a + i);        // Load N floats
        auto vb = hn::Load(d, b + i);
        auto vr = hn::Add(va, vb);           // SIMD add
        hn::Store(vr, d, out + i);
    }

    // Scalar tail
    for (; i < count; ++i) {
        out[i] = a[i] + b[i];
    }
}

}  // namespace HWY_NAMESPACE
}  // namespace project
HWY_AFTER_NAMESPACE();

// Runtime dispatch table
#if HWY_ONCE
namespace project {

HWY_EXPORT(AddArraysImpl);  // Creates dispatch function

void AddArrays(const float* a, const float* b, float* out, std::size_t n) {
    // Automatically selects best available target at runtime
    HWY_DYNAMIC_DISPATCH(AddArraysImpl)(a, b, out, n);
}

}  // namespace project

int main() {
    constexpr std::size_t N = 1024;
    std::vector<float> a(N, 1.0f), b(N, 2.0f), out(N);

    project::AddArrays(a.data(), b.data(), out.data(), N);

    // Verify
    bool ok = true;
    for (std::size_t i = 0; i < N; ++i)
        if (out[i] != 3.0f) { ok = false; break; }

    std::cout << "Result: " << (ok ? "PASS" : "FAIL") << "\n";
    std::cout << "Best target: " << hwy::TargetName(hwy::DispatchedTarget()) << "\n";
    return 0;
}
#endif  // HWY_ONCE
```

The `HWY_RESTRICT` annotation tells the compiler that the pointers do not alias, which is critical for the optimizer to generate efficient SIMD loads and stores. Without it, the compiler has to assume `a`, `b`, and `out` could point to overlapping memory.

### Q2: Implement horizontal sum and dot product using `std::experimental::simd` (Parallelism TS v2 / GCC)

`std::experimental::simd` takes a different approach from Highway: instead of a runtime dispatch mechanism, you use `native_simd<T>` which selects the best fixed width for the current compilation target. The `where()` function for predicated operations is one of its more elegant features - it expresses "set these lanes, leave others unchanged" without any explicit branch.

<!-- compile: needs third-party library header `experimental/simd` -->
```cpp
// Requires: GCC with libstdc++ (has <experimental/simd> support)
// Compile: g++ -std=c++20 -O2 -march=native

#include <experimental/simd>
#include <iostream>
#include <numeric>
#include <array>
#include <cassert>

namespace stdx = std::experimental;

// Horizontal sum: reduce all lanes to a single scalar
template <typename T, typename Abi>
T horizontal_sum(const stdx::simd<T, Abi>& v) {
    return stdx::reduce(v, std::plus<>{});
}

// Dot product using simd
template <typename T>
T simd_dot_product(const T* a, const T* b, std::size_t n) {
    using V = stdx::native_simd<T>;                // Best width for this platform
    constexpr std::size_t lanes = V::size();

    V accumulator(0);  // Zero-initialized SIMD register

    std::size_t i = 0;
    for (; i + lanes <= n; i += lanes) {
        V va(a + i, stdx::element_aligned);         // Aligned load
        V vb(b + i, stdx::element_aligned);
        accumulator += va * vb;                      // Fused multiply-add where available
    }

    T result = horizontal_sum(accumulator);

    // Scalar remainder
    for (; i < n; ++i) {
        result += a[i] * b[i];
    }

    return result;
}

// Conditional operation with where()
template <typename T>
void clamp_simd(T* data, std::size_t n, T low, T high) {
    using V = stdx::native_simd<T>;
    constexpr std::size_t lanes = V::size();

    std::size_t i = 0;
    for (; i + lanes <= n; i += lanes) {
        V v(data + i, stdx::element_aligned);

        // SIMD predicated operations (no branches)
        where(v < low, v) = low;
        where(v > high, v) = high;

        v.copy_to(data + i, stdx::element_aligned);
    }

    for (; i < n; ++i) {
        data[i] = std::clamp(data[i], low, high);
    }
}

int main() {
    using simd_f = stdx::native_simd<float>;
    std::cout << "Native float SIMD width: " << simd_f::size() << " lanes\n";

    constexpr std::size_t N = 256;
    alignas(64) std::array<float, N> a{}, b{};
    std::iota(a.begin(), a.end(), 1.0f);
    std::iota(b.begin(), b.end(), 1.0f);

    float dot = simd_dot_product(a.data(), b.data(), N);
    std::cout << "Dot product: " << dot << "\n";

    // Clamp test
    alignas(64) std::array<float, 8> vals = {-5, -1, 0, 3, 7, 10, 15, 20};
    clamp_simd(vals.data(), vals.size(), 0.0f, 10.0f);

    std::cout << "Clamped: ";
    for (auto v : vals) std::cout << v << " ";
    std::cout << "\n";

    return 0;
}
```

The scalar tail loop after the main SIMD loop is not optional - array lengths are rarely a multiple of the SIMD width. Forgetting it produces wrong results for the last few elements, which is the kind of bug that only shows up with real data.

### Q3: Show compile-time SIMD target selection and a fallback strategy when no SIMD is available

Sometimes you want the compiler to select the right code path at build time rather than runtime. This avoids the runtime dispatch overhead and is the right approach when you are compiling a dedicated binary for a known target (like an embedded system or a container image for a specific server generation).

```cpp
#include <cstddef>
#include <cstdint>
#include <iostream>
#include <vector>
#include <chrono>

// Detect SIMD capability at compile time
enum class SimdLevel { None, SSE2, AVX2, AVX512, NEON };

consteval SimdLevel detect_simd() {
    #if defined(__AVX512F__)
        return SimdLevel::AVX512;
    #elif defined(__AVX2__)
        return SimdLevel::AVX2;
    #elif defined(__SSE2__) || defined(_M_X64)
        return SimdLevel::SSE2;
    #elif defined(__ARM_NEON) || defined(__aarch64__)
        return SimdLevel::NEON;
    #else
        return SimdLevel::None;
    #endif
}

constexpr const char* simd_name(SimdLevel l) {
    switch (l) {
        case SimdLevel::AVX512: return "AVX-512";
        case SimdLevel::AVX2:   return "AVX2";
        case SimdLevel::SSE2:   return "SSE2";
        case SimdLevel::NEON:   return "NEON";
        case SimdLevel::None:   return "Scalar";
    }
    return "Unknown";
}

// Scale array by constant — multi-target implementation
void scale_array(float* data, std::size_t n, float factor) {
    constexpr auto level = detect_simd();

    if constexpr (level == SimdLevel::AVX2 || level == SimdLevel::AVX512) {
        #if defined(__AVX2__)
        #include <immintrin.h>
        __m256 vfactor = _mm256_set1_ps(factor);
        std::size_t i = 0;
        for (; i + 8 <= n; i += 8) {
            __m256 v = _mm256_loadu_ps(data + i);
            v = _mm256_mul_ps(v, vfactor);
            _mm256_storeu_ps(data + i, v);
        }
        for (; i < n; ++i) data[i] *= factor;
        #endif
    } else if constexpr (level == SimdLevel::SSE2) {
        #if defined(__SSE2__)
        #include <xmmintrin.h>
        __m128 vfactor = _mm_set1_ps(factor);
        std::size_t i = 0;
        for (; i + 4 <= n; i += 4) {
            __m128 v = _mm_loadu_ps(data + i);
            v = _mm_mul_ps(v, vfactor);
            _mm_storeu_ps(data + i, v);
        }
        for (; i < n; ++i) data[i] *= factor;
        #endif
    } else if constexpr (level == SimdLevel::NEON) {
        #if defined(__ARM_NEON)
        #include <arm_neon.h>
        float32x4_t vfactor = vdupq_n_f32(factor);
        std::size_t i = 0;
        for (; i + 4 <= n; i += 4) {
            float32x4_t v = vld1q_f32(data + i);
            v = vmulq_f32(v, vfactor);
            vst1q_f32(data + i, v);
        }
        for (; i < n; ++i) data[i] *= factor;
        #endif
    } else {
        // Scalar fallback — compiler may auto-vectorize
        for (std::size_t i = 0; i < n; ++i)
            data[i] *= factor;
    }
}

int main() {
    constexpr auto level = detect_simd();
    std::cout << "Compiled SIMD target: " << simd_name(level) << "\n";

    constexpr std::size_t N = 1'000'000;
    std::vector<float> data(N, 1.0f);

    auto start = std::chrono::high_resolution_clock::now();
    scale_array(data.data(), N, 2.5f);
    auto end = std::chrono::high_resolution_clock::now();

    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    std::cout << "Scale " << N << " floats: " << us << " us\n";
    std::cout << "Sample: " << data[0] << ", " << data[N/2] << ", " << data[N-1] << "\n";

    return 0;
}
```

The `if constexpr` branches are dead-code-eliminated at compile time, so the binary only contains the code for the selected path. This also means the compiler cannot accidentally try to compile an AVX intrinsic when the target is NEON - the other branches simply do not exist in the output.

---

## Notes

- Google Highway is the recommended portable SIMD library for production code today - it supports x86 (SSE2 through AVX-512), ARM (NEON, SVE, SVE2), WASM SIMD, and RISC-V V.
- `std::experimental::simd` is available in GCC's libstdc++ but not in MSVC or libc++; it is not yet standardized as `std::simd`.
- Runtime dispatch (compiling for multiple targets, selecting at startup) is crucial for distributing binaries that run on varied hardware - Highway's `HWY_DYNAMIC_DISPATCH` automates this.
- Prefer `_loadu_ps` (unaligned loads) over `_load_ps` (aligned) - modern CPUs have negligible penalty for unaligned access, and it avoids hard-to-diagnose alignment bugs.
- Auto-vectorization by the compiler often produces good results for simple loops; use explicit SIMD only when the compiler demonstrably fails (check with `-fopt-info-vec` on GCC or `/Qvec-report:2` on MSVC).
- Always benchmark with realistic data sizes and access patterns - SIMD gains are often bottlenecked by memory bandwidth, not compute, so the speedup may be smaller than expected for memory-bound workloads.
