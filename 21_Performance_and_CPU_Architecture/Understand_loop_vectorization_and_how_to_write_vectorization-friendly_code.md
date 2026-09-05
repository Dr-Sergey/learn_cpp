# Understand loop vectorization and how to write vectorization-friendly code

**Category:** Performance and CPU Architecture  
**Standard:** C++11  
**Reference:** <https://gcc.gnu.org/projects/tree-ssa/vectorization.html>  

---

## Topic Overview

Auto-vectorization is the compiler's ability to transform a scalar loop into one that uses SIMD instructions - SSE, AVX, or AVX-512 - processing 4, 8, or 16 elements per instruction instead of one. When it works, you get a 4-16x speedup essentially for free, without changing any of your C++ logic.

The compiler needs certain guarantees before it will vectorize a loop. If any of these conditions are violated, the compiler either generates a slower scalar version or adds runtime alias checks that eat into the performance gain:

| Vectorization requirement | Description |
| --- | --- |
| No loop-carried dependency | Iteration `i` must not depend on `i-1` result |
| No pointer aliasing | Compiler must prove `a[i]` and `b[i]` don't overlap |
| Simple control flow | No branches inside loop (or easily masked) |
| Countable loop | Trip count known or bounded |
| Aligned data | Helps performance (not strictly required) |

Here is what the transformation looks like at the instruction level. Instead of one addition per cycle, the vectorized version does eight at once using AVX 256-bit registers:

```cpp
Scalar loop:                     Vectorized (AVX, 8 floats):
  for i in 0..N:                   for i in 0..N step 8:
    a[i] = b[i] + c[i]              va = _mm256_load_ps(&b[i])
  1 addition per cycle               vc = _mm256_load_ps(&c[i])
                                     vr = _mm256_add_ps(va, vc)
                                     _mm256_store_ps(&a[i], vr)
                                   8 additions per cycle!
```

---

## Self-Assessment

### Q1: Vectorizable loop and verify with `-fopt-info-vec`

The simplest vectorizable loop is an elementwise operation where each output element depends only on the corresponding input elements at the same index. There are no dependencies between iterations, no aliasing concerns (with `__restrict`), and no branches - the compiler can vectorize it immediately.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// This loop has NO dependencies between iterations -> vectorizable!
void add_arrays(float* __restrict a, const float* __restrict b,
                const float* __restrict c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + c[i];
    }
    // Each iteration is independent: a[0] doesn't depend on a[1], etc.
}

// Compile and verify:
//   g++ -O2 -ftree-vectorize -march=native -fopt-info-vec test.cpp
//   Output: "test.cpp:6:5: optimized: loop vectorized using 32 byte vectors"
//
// Or with Clang:
//   clang++ -O2 -Rpass=loop-vectorize test.cpp
//   Output: "vectorized loop (vectorization width: 8, interleaved count: 4)"
//
// Without vectorization:
//   g++ -O2 -fno-tree-vectorize -fopt-info-vec-missed test.cpp
//   Output: no vectorization, uses scalar addss

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N), b(N, 1.0f), c(N, 2.0f);

    auto t0 = std::chrono::high_resolution_clock::now();
    for (int r = 0; r < 100; ++r)
        add_arrays(a.data(), b.data(), c.data(), N);
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);

    std::cout << "Time: " << ms.count() / 100 << " ms/iteration\n";
    std::cout << "a[0]=" << a[0] << '\n';  // 3.0
}
```

Use `-fopt-info-vec-optimized` to confirm the loop was vectorized, and `-fopt-info-vec-missed` to diagnose why it was not. These flags save a lot of guessing.

### Q2: Remove loop-carried dependency to enable vectorization

A loop-carried dependency means that iteration `i` reads something that iteration `i-1` wrote. The compiler cannot start iteration `i` until `i-1` finishes, which prevents vectorization. The prefix sum is the classic example - `a[i]` must be `a[i-1] + a[i]`, so you literally cannot compute them in parallel.

The manual parallel reduction sidesteps this by using four independent accumulators. There are no dependencies between the four chains, so the compiler can vectorize them. Even without `-ffast-math`, the chains are provably independent:

```cpp
#include <iostream>
#include <vector>

// BAD: loop-carried dependency prevents vectorization
void prefix_sum(float* a, int n) {
    for (int i = 1; i < n; ++i) {
        a[i] = a[i] + a[i - 1];  // a[i] depends on a[i-1]!
        // Compiler CANNOT vectorize: each element needs the previous result.
        // -fopt-info-vec-missed: "not vectorized: dependence prevents vectorization"
    }
}

// GOOD: independent reduction (no dependency between iterations)
void scale_array(float* a, float factor, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = a[i] * factor;  // each iteration independent!
        // Compiler vectorizes: 8 multiplications per SIMD instruction
    }
}

// TRICK: Convert reduction to parallel form
void sum_array_bad(const float* data, int n, float& result) {
    result = 0;
    for (int i = 0; i < n; ++i) {
        result += data[i];  // loop-carried: result depends on previous
        // Still vectorizable with -ffast-math! Compiler reorders FP ops.
        // Without -ffast-math: NOT vectorized (FP not associative)
    }
}

void sum_array_good(const float* data, int n, float& result) {
    // Manual parallel reduction: 4 independent accumulators
    float s0=0, s1=0, s2=0, s3=0;
    int i = 0;
    for (; i+3 < n; i+=4) {
        s0 += data[i];    // independent!
        s1 += data[i+1];  // independent!
        s2 += data[i+2];  // independent!
        s3 += data[i+3];  // independent!
    }
    for (; i < n; ++i) s0 += data[i];
    result = (s0+s1) + (s2+s3);
    // Vectorizable even without -ffast-math
}

int main() {
    std::vector<float> data(1000, 1.0f);
    float r;
    sum_array_good(data.data(), 1000, r);
    std::cout << "Sum: " << r << '\n';  // 1000
}
```

Notice that `sum_array_bad` can be vectorized if you add `-ffast-math`, which allows the compiler to reorder floating-point operations. Without it, FP addition is not associative (rounding errors change with order), so the compiler respects program order and produces a scalar loop. The `sum_array_good` version avoids this issue entirely.

### Q3: `__restrict__` to enable vectorization with pointer aliasing

This is where most people get caught out. When you write `void f(float* a, const float* b, ...)`, the compiler has to assume that `a` and `b` might point to the same memory (or overlapping memory). If `a[i] = b[i] + ...` modified `b[i+1]`, the loop would not be independent. So the compiler either skips vectorization or emits a runtime alias check that runs both a vectorized and a scalar path, with a branch to pick between them.

`__restrict__` is your promise to the compiler that the pointers do not alias. With that guarantee, it can vectorize without the runtime check:

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// WITHOUT restrict: compiler cannot prove a != b
void add_may_alias(float* a, const float* b, const float* c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + c[i];
    }
    // Problem: what if a and b point to the same array?
    //   a[0] = b[0] + c[0]  might modify b[1] if a == b-1!
    // Compiler must assume aliasing -> generates scalar code OR
    //   adds runtime alias check (slower startup)
    // -fopt-info-vec-missed: "can't determine dependence" or
    //   "versioned for alias checks"
}

// WITH __restrict__: promise no aliasing
void add_no_alias(float* __restrict__ a,
                  const float* __restrict__ b,
                  const float* __restrict__ c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + c[i];
    }
    // __restrict__ tells compiler: a, b, c don't overlap
    // -> vectorizes immediately without runtime checks
    // -fopt-info-vec: "loop vectorized"
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N), b(N, 1.0f), c(N, 2.0f);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 100; ++r) fn(a.data(), b.data(), c.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
        std::cout << label << ": " << us.count()/100 << " us\n";
    };

    bench(add_may_alias, "May alias   ");
    bench(add_no_alias,  "No alias    ");
    // No-alias version may be faster due to no runtime checks
    // and better vectorization strategy.
}
```

The performance difference depends on the compiler's heuristics. Sometimes it will vectorize both (with a version check for the aliased case). But `__restrict__` always produces cleaner, simpler code and gives the compiler the best shot at the fastest output.

---

## Notes

- GCC: `-ftree-vectorize` (included in `-O2` since GCC 12, `-O3` for older versions).
- Diagnostics: `-fopt-info-vec-optimized` (successes), `-fopt-info-vec-missed` (failures). Read the missed report - it usually tells you exactly what is blocking vectorization.
- Clang: `-Rpass=loop-vectorize`, `-Rpass-missed=loop-vectorize`.
- `-ffast-math` allows FP reordering, enabling vectorization of reductions that would otherwise require the manual accumulator trick.
- `__restrict__` is a compiler extension (GCC/Clang/MSVC); C99 has `restrict` natively.
- Align data to 32/64 bytes for AVX/AVX-512 to enable aligned load/store instructions: `alignas(32) float data[N]`.
