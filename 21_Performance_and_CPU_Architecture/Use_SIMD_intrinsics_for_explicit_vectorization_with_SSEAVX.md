# Use SIMD intrinsics for explicit vectorization with SSE/AVX

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html>  

---

## Topic Overview

SIMD (Single Instruction, Multiple Data) intrinsics give you direct access to the CPU's vector instructions. The key selling point over auto-vectorization is that intrinsics *guarantee* the instructions you ask for will be emitted - the compiler cannot decide the loop is too complex and fall back to scalar code. The tradeoff is portability: intrinsic code is tied to a specific ISA family (SSE/AVX on x86).

The fundamental idea is that instead of operating on one float at a time, you operate on a register that holds 4, 8, or 16 floats simultaneously, and the CPU performs the same operation on all of them in a single instruction:

```cpp
Scalar:                  AVX2 (256-bit):
  float a, b, c;           __m256 a, b, c;
  c = a * b + c;           c = _mm256_fmadd_ps(a, b, c);
  // 1 FMA                 // 8 FMAs in 1 instruction!
  // 4 cycles              // 4 cycles (same latency, 8x throughput)
```

The latency is the same - the CPU still needs 4 cycles to produce the result. But instead of doing that work for 1 float, you're doing it for 8. That is the throughput win.

| Register type | Width | Floats | Doubles | Ints (32-bit) |
| --- | --- | --- | --- | --- |
| `__m128` | 128-bit (SSE) | 4 | 2 | 4 |
| `__m256` | 256-bit (AVX) | 8 | 4 | 8 |
| `__m512` | 512-bit (AVX-512) | 16 | 8 | 16 |

---

## Self-Assessment

### Q1: Dot product with `_mm256_fmadd_ps` vs scalar

A dot product is the classic SIMD example because it has a simple memory access pattern (sequential), pure arithmetic work, and a well-understood scalar baseline. Here both versions compute the same result; the benchmark shows you the actual speedup on your hardware.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <immintrin.h>
#include <cstdlib>

// Scalar dot product
float dot_scalar(const float* a, const float* b, int n) {
    float sum = 0;
    for (int i = 0; i < n; ++i)
        sum += a[i] * b[i];
    return sum;
}

// AVX2 + FMA dot product
float dot_avx2(const float* a, const float* b, int n) {
    __m256 acc = _mm256_setzero_ps();  // 8 accumulators = 0

    int i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);   // load 8 floats from a
        __m256 vb = _mm256_loadu_ps(b + i);   // load 8 floats from b
        acc = _mm256_fmadd_ps(va, vb, acc);   // acc += va * vb (fused)
        // vfmadd231ps ymm0, ymm1, ymm2  (single instruction)
    }

    // Horizontal sum: reduce 8 accumulators to 1
    __m128 hi = _mm256_extractf128_ps(acc, 1);  // upper 4
    __m128 lo = _mm256_castps256_ps128(acc);     // lower 4
    __m128 sum128 = _mm_add_ps(lo, hi);          // 4 values
    sum128 = _mm_hadd_ps(sum128, sum128);        // 2 values
    sum128 = _mm_hadd_ps(sum128, sum128);        // 1 value
    float result = _mm_cvtss_f32(sum128);

    // Scalar tail for remaining elements
    for (; i < n; ++i)
        result += a[i] * b[i];
    return result;
}

int main() {
    constexpr int N = 10'000'000;
    float* a = static_cast<float*>(std::aligned_alloc(32, N * sizeof(float)));
    float* b = static_cast<float*>(std::aligned_alloc(32, N * sizeof(float)));
    for (int i = 0; i < N; ++i) { a[i] = 1.0f; b[i] = 2.0f; }

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile float r = 0;
        for (int rep = 0; rep < 100; ++rep) r = fn(a, b, N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms (result=" << r << ")\n";
    };

    bench(dot_scalar, "Scalar");
    bench(dot_avx2,   "AVX2  ");
    // Scalar: ~800ms, AVX2: ~120ms (6-7x speedup)
    // Not full 8x due to memory bandwidth limit

    std::free(a); std::free(b);
}
// Compile: g++ -O2 -mavx2 -mfma -o dot dot.cpp
```

The speedup is close to but not quite 8x. The reason is memory bandwidth: at 10 million floats, the arrays don't fit comfortably in L2 cache, so the loop is partially limited by how fast the CPU can pull data from L3 or DRAM, not just by arithmetic throughput. For data that fits in L1/L2 cache, the speedup would be closer to the theoretical maximum.

### Q2: Intrinsic naming convention

The intrinsic naming convention looks intimidating at first but it follows a consistent pattern. Once you know the pattern, you can look at any intrinsic name and know exactly what it does without consulting the manual.

```cpp
#include <iostream>
#include <immintrin.h>

// Naming: _mm[width]_[operation]_[type]
//
// Width:  (none)=128, 256=256, 512=512
// Type:   ps=packed single, pd=packed double,
//         epi32=packed int32, si128=128-bit integer
//
// Examples:
//   _mm_add_ps       -> 128-bit, add, packed single (4 floats)
//   _mm256_mul_pd    -> 256-bit, multiply, packed double (4 doubles)
//   _mm256_fmadd_ps  -> 256-bit, fused multiply-add, packed single (8 floats)
//   _mm512_load_epi32-> 512-bit, load, packed int32 (16 ints)
//   _mm_set1_ps      -> 128-bit, broadcast scalar to all lanes
//   _mm256_setzero_ps-> 256-bit, all zeros

int main() {
    // Register types and their capacity:
    __m128  f4  = _mm_set1_ps(1.0f);      // 4 floats
    __m128d d2  = _mm_set1_pd(2.0);       // 2 doubles
    __m128i i4  = _mm_set1_epi32(3);      // 4 int32s
    __m256  f8  = _mm256_set1_ps(4.0f);   // 8 floats
    __m256d d4  = _mm256_set1_pd(5.0);    // 4 doubles
    __m256i i8  = _mm256_set1_epi32(6);   // 8 int32s

    // Common operations:
    __m256 a = _mm256_set_ps(8,7,6,5,4,3,2,1);  // explicit (reverse order!)
    __m256 b = _mm256_set1_ps(10.0f);            // broadcast
    __m256 c = _mm256_add_ps(a, b);              // add
    __m256 d = _mm256_mul_ps(a, b);              // multiply
    __m256 e = _mm256_fmadd_ps(a, b, c);         // a*b + c (fused)
    __m256 f = _mm256_min_ps(a, b);              // element-wise min
    __m256 g = _mm256_cmp_ps(a, b, _CMP_LT_OQ); // a < b (mask)

    // Extract result:
    float result[8];
    _mm256_storeu_ps(result, c);
    for (int i = 0; i < 8; ++i)
        std::cout << result[i] << ' ';  // 11 12 13 14 15 16 17 18
    std::cout << '\n';
}
```

One trap worth highlighting: `_mm256_set_ps` takes its arguments in **reverse lane order**. The first argument goes into lane 7 (the highest), not lane 0. If you need lane 0 to hold value 1, you write `_mm256_set_ps(8,7,6,5,4,3,2,1)` - the 1 is at the end. This surprises almost everyone the first time.

### Q3: Handle non-multiple-of-8 lengths with scalar tail

AVX2 processes 8 floats per iteration. Real arrays almost never have lengths that are exact multiples of 8. There are three standard ways to handle the remainder, each with its own tradeoffs.

```cpp
#include <iostream>
#include <immintrin.h>
#include <cstdlib>

// SIMD processes 8 elements at a time. What about the remainder?

// Method 1: Scalar tail loop (simplest, always correct)
void add_tail(float* c, const float* a, const float* b, int n) {
    int i = 0;
    // SIMD main loop: processes 8 elements per iteration
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        _mm256_storeu_ps(c + i, _mm256_add_ps(va, vb));
    }
    // Scalar tail: handles remaining 0-7 elements
    for (; i < n; ++i) {
        c[i] = a[i] + b[i];
    }
}

// Method 2: Masked load/store (AVX2, cleaner but slower)
void add_masked(float* c, const float* a, const float* b, int n) {
    int i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);
        __m256 vb = _mm256_loadu_ps(b + i);
        _mm256_storeu_ps(c + i, _mm256_add_ps(va, vb));
    }
    // Masked tail: handles remainder with a single SIMD instruction
    if (i < n) {
        int remaining = n - i;
        // Create mask: bits set for valid elements
        __m256i mask = _mm256_set_epi32(
            remaining > 7 ? -1 : 0, remaining > 6 ? -1 : 0,
            remaining > 5 ? -1 : 0, remaining > 4 ? -1 : 0,
            remaining > 3 ? -1 : 0, remaining > 2 ? -1 : 0,
            remaining > 1 ? -1 : 0, remaining > 0 ? -1 : 0);
        __m256 va = _mm256_maskload_ps(a + i, mask);
        __m256 vb = _mm256_maskload_ps(b + i, mask);
        _mm256_maskstore_ps(c + i, mask, _mm256_add_ps(va, vb));
    }
}

// Method 3: Pad arrays to multiple of 8 (fastest, wastes memory)
void add_padded(float* c, const float* a, const float* b, int n_padded) {
    // Caller ensures n_padded is multiple of 8, extra elements = 0
    for (int i = 0; i < n_padded; i += 8) {
        __m256 va = _mm256_load_ps(a + i);   // aligned!
        __m256 vb = _mm256_load_ps(b + i);
        _mm256_store_ps(c + i, _mm256_add_ps(va, vb));
    }
}

int main() {
    constexpr int N = 13;  // Not multiple of 8!
    float a[16] = {}, b[16] = {}, c[16] = {};  // padded to 16
    for (int i = 0; i < N; ++i) { a[i] = float(i); b[i] = 10.0f; }

    add_tail(c, a, b, N);
    for (int i = 0; i < N; ++i)
        std::cout << c[i] << ' ';  // 10 11 12 13 14 15 16 17 18 19 20 21 22
    std::cout << '\n';
}
```

Method 1 (scalar tail) is the go-to for most code: it's simple, readable, and always correct. Method 3 (padding) is worth the extra memory when you own the allocation and need maximum performance on the main loop. Method 2 (masked loads) is useful when you can't pad and want to keep the tail in SIMD - but masked loads are slower than regular loads, so the benefit only shows up when the tail accounts for a significant fraction of total work.

---

## Notes

- Compile with `-mavx2 -mfma` for AVX2+FMA intrinsics.
- Intel Intrinsics Guide: search by operation name to find the right intrinsic.
- `_loadu_ps` (unaligned) is nearly as fast as `_load_ps` (aligned) on modern CPUs.
- FMA (`_mm256_fmadd_ps`) is more accurate than separate mul+add (single rounding).
- For portability, consider Highway or `std::experimental::simd` instead of raw intrinsics.
