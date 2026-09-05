# Use std::experimental::simd (Parallelism TS 2) for portable SIMD

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://github.com/VcDevel/std-simd>  

---

## Topic Overview

This topic covers the ABI types, portable dot product, and comparison with Highway - complementing #723 which covers element-wise ops and `simd_mask`.

The `simd<T, Abi>` template is more flexible than it first appears. The `Abi` parameter isn't just about performance - it also controls portability and ABI stability. Understanding the four options lets you choose the right one for each situation:

```cpp
simd<T, Abi> template:
  T   = float, double, int, etc.
  Abi = how the SIMD register is mapped

  native_simd<float>       -> best for current CPU (8 on AVX2, 4 on SSE)
  fixed_size_simd<float,4> -> exactly 4 lanes (portable width)
  simd<float, compatible>  -> ABI-compatible across TU boundaries
```

| ABI | Width | Best for |
| --- | --- | --- |
| `simd_abi::native` | Hardware natural | Maximum throughput |
| `simd_abi::fixed_size<N>` | Exactly N | Algorithm needs specific width |
| `simd_abi::compatible` | >=1 | ABI stability across shared libraries |
| `simd_abi::scalar` | 1 | Debugging, reference implementation |

---

## Self-Assessment

### Q1: Portable SIMD dot product (compiles to AVX2 or NEON)

This is the most compelling demonstration of `stdx::simd`: a single function that compiles to `vfmadd231ps ymm` on AVX2, `fmla v.4s` on ARM NEON, and a plain loop on any other target. You write it once, and the library does the platform work.

<!-- compile: needs third-party library header `experimental/simd` -->
```cpp
#include <experimental/simd>
#include <iostream>
#include <vector>
#include <chrono>

namespace stdx = std::experimental;

// This SINGLE function compiles to:
//   x86 AVX2: vfmadd231ps ymm (8 floats/iteration)
//   ARM NEON: fmla v.4s (4 floats/iteration)
//   Scalar:   multiply + add (1 float/iteration)
float dot_simd(const float* a, const float* b, int n) {
    using V = stdx::native_simd<float>;
    constexpr int W = V::size();

    V acc(0.0f);  // SIMD accumulator
    int i = 0;
    for (; i + W <= n; i += W) {
        V va(a + i, stdx::element_aligned);
        V vb(b + i, stdx::element_aligned);
        acc += va * vb;  // fused multiply-add (if available)
        // acc = _mm256_fmadd_ps(va, vb, acc)  on AVX2+FMA
    }

    // Horizontal reduction: sum all lanes
    float result = stdx::reduce(acc);  // built-in horizontal sum!
    // On AVX2: vextractf128 + vaddps + vhaddps + vhaddps

    // Scalar tail
    for (; i < n; ++i)
        result += a[i] * b[i];
    return result;
}

float dot_scalar(const float* a, const float* b, int n) {
    float sum = 0;
    for (int i = 0; i < n; ++i) sum += a[i] * b[i];
    return sum;
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N, 1.0f), b(N, 2.0f);

    std::cout << "native_simd width: " << stdx::native_simd<float>::size() << '\n';

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile float r = 0;
        for (int rep = 0; rep < 100; ++rep) r = fn(a.data(), b.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(dot_scalar, "Scalar");
    bench(dot_simd,   "SIMD  ");
    // Scalar: ~800ms, SIMD: ~120ms on AVX2 (6.7x)
}
// Compile: g++ -O2 -std=c++17 -mavx2 -mfma test.cpp
```

The `stdx::reduce(acc)` call at the end is doing a horizontal sum - adding all the lanes of the SIMD register together to produce a single scalar result. This is the step where you "collapse" the SIMD accumulator back to a regular float.

### Q2: ABI template parameters explained

Choosing the wrong ABI type is a common source of subtle bugs. The most dangerous one is passing a `native_simd` value across a shared library boundary if the two sides were compiled with different `-march` flags - the register width might not match. That's exactly what `compatible` is designed to prevent.

<!-- compile: needs third-party library header `experimental/simd` -->
```cpp
#include <experimental/simd>
#include <iostream>

namespace stdx = std::experimental;

int main() {
    // 1. native_simd: uses hardware-natural width
    using Native = stdx::native_simd<float>;
    std::cout << "native_simd<float> width: " << Native::size() << '\n';
    // SSE: 4, AVX2: 8, AVX-512: 16, NEON: 4

    // 2. fixed_size_simd: exactly N lanes
    using Fixed4 = stdx::fixed_size_simd<float, 4>;
    using Fixed16 = stdx::fixed_size_simd<float, 16>;
    std::cout << "fixed_size_simd<float,4>: " << Fixed4::size() << '\n';   // always 4
    std::cout << "fixed_size_simd<float,16>: " << Fixed16::size() << '\n'; // always 16
    // On SSE: Fixed16 uses 4 xmm registers internally
    // On AVX2: Fixed16 uses 2 ymm registers
    // On AVX-512: Fixed16 uses 1 zmm register

    // 3. compatible: guaranteed ABI-compatible across shared libraries
    using Compat = stdx::simd<float, stdx::simd_abi::compatible>;
    std::cout << "compatible width: " << Compat::size() << '\n';
    // Always 4 (SSE-width) for maximum compatibility

    // 4. scalar: one element (for debugging/testing)
    using Scalar = stdx::simd<float, stdx::simd_abi::scalar>;
    std::cout << "scalar width: " << Scalar::size() << '\n';  // 1

    // Usage guidelines:
    //   - Use native_simd for maximum performance
    //   - Use fixed_size_simd when algorithm needs exact width
    //     (e.g., 3D vector = fixed_size<3>, RGBA pixel = fixed_size<4>)
    //   - Use compatible across shared library boundaries
    //   - Use scalar for unit testing SIMD code paths

    // Type deduction helpers:
    using V = stdx::native_simd<float>;
    V a(1.0f), b(2.0f);
    auto c = a + b;          // type: V (same as native_simd<float>)
    auto mask = a < b;       // type: V::mask_type
    float sum = stdx::reduce(c);  // horizontal sum
    std::cout << "reduce result: " << sum << '\n'; // 3.0 * V::size()
}
```

The `scalar` ABI is particularly useful during development. Because it processes one element at a time, it behaves exactly like a normal scalar loop - you can use `std::cout` or a debugger to inspect the values, which is impossible with actual SIMD registers. Write the algorithm with `scalar`, verify correctness, then switch to `native` for production.

### Q3: `stdx::simd` vs Highway library

Both libraries let you write portable SIMD code. The difference is maturity, compiler support, and runtime dispatch. If you need to ship today across multiple platforms, Highway is the more pragmatic choice. If you're writing code for the long term and want to use a future C++ standard feature, `stdx::simd` is the right investment.

```cpp
#include <iostream>

int main() {
    std::cout << "stdx::simd vs Highway comparison:\n\n";
    std::cout << "+---------------------+---------------------------+---------------------------+\n";
    std::cout << "| Feature             | stdx::simd (TS 2)         | Highway (google/highway)  |\n";
    std::cout << "+---------------------+---------------------------+---------------------------+\n";
    std::cout << "| Standard            | C++ proposed (TS 2)       | Third-party library       |\n";
    std::cout << "| Compiler support    | GCC 11+ only (practical)  | GCC, Clang, MSVC          |\n";
    std::cout << "| API style           | Operator overloads (+,*)  | Free functions (Add, Mul) |\n";
    std::cout << "| Dynamic dispatch    | Not built-in              | HWY_DYNAMIC_DISPATCH      |\n";
    std::cout << "| Runtime CPU detect  | Manual                    | Automatic                 |\n";
    std::cout << "| Target coverage     | x86, ARM                  | x86,ARM,WASM,RISC-V,PPC  |\n";
    std::cout << "| Production usage    | Rare (experimental)       | JPEG XL, Chrome, others   |\n";
    std::cout << "| API stability       | May change before C++26   | Stable (1.0+)             |\n";
    std::cout << "| Performance         | Optimal                   | Optimal                   |\n";
    std::cout << "| Masking             | where(mask, v) = x        | IfThenElse(mask, a, b)    |\n";
    std::cout << "+---------------------+---------------------------+---------------------------+\n";

    // Recommendation:
    //   - stdx::simd: best for future-proof code, will become std::simd in C++26
    //   - Highway: best for production NOW (broader compiler/platform support)
    //   - Both produce the same assembly quality
    //   - For new projects: Highway if shipping soon, stdx::simd if long-term
}
```

The performance row is the same for both - this is genuinely not a trade-off between speed and convenience. The choice is entirely about compiler coverage, runtime dispatch features, and API stability preferences.

---

## Notes

- Install standalone `std-simd`: `git clone https://github.com/VcDevel/std-simd`.
- `stdx::reduce(v)` is the portable horizontal sum (replaces manual hadd chain).
- `native_simd` width changes with compile flags (`-mavx2` -> 8, `-msse4.2` -> 4).
- On ARM: compile with `-march=armv8-a+simd` for NEON.
- Both `stdx::simd` and Highway produce identical assembly to hand-written intrinsics.
