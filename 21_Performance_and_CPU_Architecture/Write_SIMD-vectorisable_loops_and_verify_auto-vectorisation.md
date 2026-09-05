# Write SIMD-vectorisable loops and verify auto-vectorisation

**Category:** Performance and CPU Architecture  
**Standard:** C++11  
**Reference:** <https://llvm.org/docs/Vectorizers.html>  

---

## Topic Overview

Auto-vectorisation is the compiler's ability to transform a plain scalar loop into one that uses SIMD instructions - processing multiple elements per clock instead of one. You don't need intrinsics for this; the compiler can do it automatically, but only when it can prove the loop is safe and structured in a way it recognizes. Your job is to write loops that meet those conditions.

The checklist and table below capture the most common requirements and blockers:

```cpp
Vectorisable loop checklist:
  - Simple loop structure (for i = 0..N)
  - No pointer aliasing (or use __restrict__)
  - No loop-carried dependencies
  - No function calls (or use inline)
  - No complex control flow (or use [[likely]])
  - Contiguous memory access
```

| Blocker | Example | Fix |
| --- | --- | --- |
| Pointer aliasing | `a[i] = b[i] + c[i]` where a==b possible | `__restrict__` |
| Loop-carried dep | `a[i] = a[i-1] + 1` | Reformulate algorithm |
| Function call | `a[i] = sqrt(b[i])` without `-ffast-math` | Use `-ffast-math` or manual |
| FP associativity | `sum += a[i]` | `-ffast-math` or manual acc |
| Non-contiguous | `a[i*stride]` with unknown stride | Ensure stride=1 |

---

## Self-Assessment

### Q1: Write a loop that auto-vectorises, verify with `-fopt-info-vec`

The simplest way to write a vectorisable loop is to keep it as straightforward as possible: count up from zero, access arrays at `[i]`, no conditionals, no cross-iteration dependencies, and `__restrict__` on any pointer parameters the compiler can't prove are non-aliasing. Here are two loops that vectorise cleanly, plus the classic FP-sum problem and its fix:

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// This loop WILL auto-vectorise: simple, no deps, contiguous access
void add_arrays(float* __restrict__ c,
                const float* __restrict__ a,
                const float* __restrict__ b, int n) {
    for (int i = 0; i < n; ++i)
        c[i] = a[i] + b[i];
    // GCC -O2 -mavx2 -fopt-info-vec:
    //   "note: LOOP VECTORIZED"
    // Assembly: vaddps ymm0, ymm1, ymm2 (8 floats per instruction)
}

// This loop also vectorises: multiply-accumulate
void scale_and_add(float* __restrict__ c,
                   const float* __restrict__ a,
                   const float* __restrict__ b,
                   float alpha, int n) {
    for (int i = 0; i < n; ++i)
        c[i] = alpha * a[i] + b[i];
    // vfmadd231ps ymm (FMA: fused multiply-add, 8 floats)
}

// This loop DOES NOT vectorise without -ffast-math
float sum_bad(const float* a, int n) {
    float sum = 0;
    for (int i = 0; i < n; ++i)
        sum += a[i];  // FP not associative: must be sequential
    return sum;
    // -fopt-info-vec-missed: "not vectorized: reduction: unsafe fp math"
}

// Fix: use -ffast-math OR manual reduction
float sum_good(const float* a, int n) {
    float s0 = 0, s1 = 0, s2 = 0, s3 = 0;
    int i = 0;
    for (; i + 3 < n; i += 4) {
        s0 += a[i]; s1 += a[i+1]; s2 += a[i+2]; s3 += a[i+3];
    }
    for (; i < n; ++i) s0 += a[i];
    return s0 + s1 + s2 + s3;
    // 4 independent chains -> vectorisable!
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N, 1.0f), b(N, 2.0f), c(N);

    auto t0 = std::chrono::high_resolution_clock::now();
    for (int r = 0; r < 100; ++r)
        add_arrays(c.data(), a.data(), b.data(), N);
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    std::cout << "add_arrays: " << ms.count() << " ms\n";
    std::cout << "c[0] = " << c[0] << '\n';  // 3.0
}
// Compile: g++ -O2 -mavx2 -fopt-info-vec test.cpp
// Output:  test.cpp:X: note: LOOP VECTORIZED
```

The reason `sum_bad` doesn't vectorise is subtle: floating-point addition is not mathematically associative, so the compiler can't reorder the additions without potentially changing the result. The `sum_good` version sidesteps this by using four independent accumulators - each chain computes a partial sum independently, so the compiler can execute them in parallel without reordering anything.

### Q2: Conditions that prevent auto-vectorisation

Each of these five blockers is worth understanding on its own. The function-call blocker is particularly surprising - `std::log` is a single function call but the compiler can't vectorise it because there's no SIMD version available by default. The indirect-access blocker shows up frequently in performance-critical lookup tables.

```cpp
#include <iostream>
#include <cmath>

// BLOCKER 1: Pointer aliasing
void alias_problem(float* a, float* b, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] + 1.0f;
    // Compiler doesn't know if a and b overlap!
    // May generate BOTH vectorized + scalar versions with runtime check.
    // Fix: add __restrict__
}

// BLOCKER 2: Function calls
void call_problem(float* a, const float* b, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = std::log(b[i]);  // log() is not vectorizable by default
    // Fix 1: -ffast-math (allows approximate log)
    // Fix 2: use SVML (Intel Short Vector Math Library)
    // Fix 3: manual vectorization with _mm256_log_ps (if available)
}

// BLOCKER 3: Complex control flow
void branch_problem(float* a, const float* b, int n) {
    for (int i = 0; i < n; ++i) {
        if (b[i] > 0)
            a[i] = b[i] * 2;
        else
            a[i] = -b[i];
    }
    // May or may not vectorize. If it does, uses masked operations.
    // Simpler: a[i] = b[i] > 0 ? b[i]*2 : -b[i]; (ternary helps)
}

// BLOCKER 4: Non-unit stride
void stride_problem(float* a, const float* b, int n, int stride) {
    for (int i = 0; i < n; ++i)
        a[i * stride] = b[i * stride] + 1.0f;
    // Unknown stride -> compiler can't prove stride=1 -> no vectorization
    // Fix: ensure stride is a compile-time constant
}

// BLOCKER 5: Indirect access
void indirect_problem(float* a, const float* b, const int* idx, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[idx[i]];  // gather: b[idx[i]] is random
    // AVX2: can use _mm256_i32gather_ps (slower than sequential)
    // Scalar may be better if indices are truly random
}

int main() {
    // Check vectorization:
    //   GCC:   g++ -O2 -mavx2 -fopt-info-vec-all test.cpp 2>&1 | grep -E 'VECTORIZED|not vectorized'
    //   Clang: clang++ -O2 -mavx2 -Rpass=loop-vectorize -Rpass-missed=loop-vectorize test.cpp
    std::cout << "Compile with -fopt-info-vec-all to see vectorization decisions\n";
}
```

The indirect-access case deserves special attention. AVX2 does have a gather instruction (`_mm256_i32gather_ps`), but it is often slower than scalar code for truly random indices because the hardware still has to perform the random memory accesses - it just batches the *request* slightly, not the actual memory fetches. Whether it helps depends on how random the indices are.

### Q3: `__restrict__` to enable vectorisation

The `__restrict__` annotation is your way of telling the compiler "I promise these pointers don't point to the same memory." In return, the compiler can skip the runtime alias check it would otherwise generate - resulting in smaller, simpler assembly that is guaranteed to take the vectorised path.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// WITHOUT __restrict__: compiler adds alias check
void process_no_restrict(float* a, const float* b, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] * 2.0f + 1.0f;
    // Assembly (GCC -O2 -mavx2):
    //   cmp rdi, rsi       ; do a and b overlap?
    //   ja .vectorized     ; if not, use SIMD
    //   .scalar_loop:      ; if overlap, scalar fallback
    //     movss xmm0, [rsi+rcx*4]
    //     mulss xmm0, xmm1
    //     ...
    //   .vectorized:       ; SIMD version (50% of binary size is overlap check!)
    //     vmovups ymm0, [rsi+rcx*4]
    //     vmulps  ymm0, ymm0, ymm1
    //     vaddps  ymm0, ymm0, ymm2
    //     vmovups [rdi+rcx*4], ymm0
}

// WITH __restrict__: no alias check needed
void process_restrict(float* __restrict__ a,
                      const float* __restrict__ b, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] * 2.0f + 1.0f;
    // Assembly: ONLY the vectorized version
    //   vmovups ymm0, [rsi+rcx*4]
    //   vmulps  ymm0, ymm0, ymm1
    //   vaddps  ymm0, ymm0, ymm2
    //   vmovups [rdi+rcx*4], ymm0
    // No overlap check, smaller binary, slightly faster
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N), b(N, 3.14f);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 100; ++r) fn(a.data(), b.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_no_restrict, "No restrict");
    bench(process_restrict,    "Restrict   ");
    // Performance difference: 0-5% (alias check overhead is small)
    // Code size difference: ~2x (two versions vs one)
    // Main benefit: guarantees vectorization, simpler assembly

    // __restrict__ is NOT standard C++, but supported by all major compilers:
    //   GCC/Clang: __restrict__ or __restrict
    //   MSVC: __restrict
    //   C99: restrict (standard in C, not C++)
}
```

The raw runtime improvement from `__restrict__` is modest (0-5%), but the code size benefit is real - the binary no longer needs to carry both a scalar fallback and a vectorised version with a branch between them. More importantly, `__restrict__` *guarantees* you'll get the vectorised path rather than leaving it to the compiler's judgment.

---

## Notes

- GCC: `-fopt-info-vec-all` shows all vectorization decisions with reasons.
- Clang: `-Rpass=loop-vectorize` + `-Rpass-missed=loop-vectorize`.
- `__restrict__` is the single most impactful annotation for auto-vectorisation.
- `-ffast-math` enables FP reduction vectorisation but changes semantics (be careful).
- Complementary to #626 (vectorization hints/pragmas) and #722 (vectorization-friendly code).
