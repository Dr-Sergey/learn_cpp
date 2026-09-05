# Use SIMD intrinsics (SSE/AVX) for explicit vectorisation

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html>  

---

## Topic Overview

This topic focuses on aligned vs unaligned loads and building portable SIMD abstractions - complementing #632 which covers dot product, naming convention, and tail loops.

One of the first things you'll wonder about when writing SIMD code is whether you need to worry about alignment. Here's the historical context: before the Haswell generation (roughly 2013), using an unaligned load instruction on a misaligned pointer could be 3x slower than the aligned version. On modern CPUs that gap has essentially disappeared for typical cases, though a few corner cases still bite you.

```cpp
Aligned load:              Unaligned load:
  _mm_load_ps(ptr)           _mm_loadu_ps(ptr)
  movaps xmm0, [ptr]        movups xmm0, [ptr]
  Requires: ptr % 16 == 0   Any alignment OK
  Penalty: crash if unaligned  Penalty: ~0 on modern CPUs

Historical (pre-2012): unaligned was ~3x slower
Modern (Haswell+): unaligned is nearly free (0-1 cycle penalty)
```

| Load intrinsic | Alignment required | Use when |
| --- | --- | --- |
| `_mm_load_ps` / `_mm256_load_ps` | 16/32-byte | You control allocation |
| `_mm_loadu_ps` / `_mm256_loadu_ps` | None | General purpose |
| `_mm_load_ss` | None | Load single scalar |
| `_mm256_broadcast_ss` | None | Splat scalar to all lanes |

---

## Self-Assessment

### Q1: Dot product with FMA (complementary implementation using 4 accumulators)

Using a single accumulator for a dot product leaves performance on the table. FMA has a latency of 4 cycles, meaning the CPU has to wait 4 cycles before the result of one FMA is ready to feed into the next one. With four independent accumulators, you can keep four FMAs in flight at once, hiding that latency and fully utilizing both FMA execution ports.

```cpp
#include <iostream>
#include <immintrin.h>
#include <chrono>
#include <cstdlib>

// 4 independent accumulators to maximize FMA throughput.
// FMA latency = 4 cycles, throughput = 1/cycle on port 0+1.
// With 4 accumulators: 4 FMAs in flight, fully utilizing both FMA ports.
float dot_4acc(const float* a, const float* b, int n) {
    __m256 acc0 = _mm256_setzero_ps();
    __m256 acc1 = _mm256_setzero_ps();
    __m256 acc2 = _mm256_setzero_ps();
    __m256 acc3 = _mm256_setzero_ps();

    int i = 0;
    for (; i + 32 <= n; i += 32) {
        acc0 = _mm256_fmadd_ps(_mm256_loadu_ps(a+i),    _mm256_loadu_ps(b+i),    acc0);
        acc1 = _mm256_fmadd_ps(_mm256_loadu_ps(a+i+8),  _mm256_loadu_ps(b+i+8),  acc1);
        acc2 = _mm256_fmadd_ps(_mm256_loadu_ps(a+i+16), _mm256_loadu_ps(b+i+16), acc2);
        acc3 = _mm256_fmadd_ps(_mm256_loadu_ps(a+i+24), _mm256_loadu_ps(b+i+24), acc3);
    }
    // Merge accumulators
    acc0 = _mm256_add_ps(acc0, acc1);
    acc2 = _mm256_add_ps(acc2, acc3);
    acc0 = _mm256_add_ps(acc0, acc2);

    // Horizontal sum
    __m128 hi = _mm256_extractf128_ps(acc0, 1);
    __m128 lo = _mm256_castps256_ps128(acc0);
    __m128 s = _mm_add_ps(lo, hi);
    s = _mm_hadd_ps(s, s);
    s = _mm_hadd_ps(s, s);
    float result = _mm_cvtss_f32(s);

    for (; i < n; ++i) result += a[i] * b[i];
    return result;
}

int main() {
    constexpr int N = 10'000'000;
    float* a = static_cast<float*>(std::aligned_alloc(32, N * sizeof(float)));
    float* b = static_cast<float*>(std::aligned_alloc(32, N * sizeof(float)));
    for (int i = 0; i < N; ++i) { a[i] = 0.1f; b[i] = 0.2f; }

    auto t0 = std::chrono::high_resolution_clock::now();
    volatile float r = 0;
    for (int rep = 0; rep < 100; ++rep) r = dot_4acc(a, b, N);
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    std::cout << "4-acc AVX2: " << ms.count() << " ms\n";  // ~100ms
    std::cout << "Result: " << r << '\n';  // ~200000
    std::free(a); std::free(b);
}
```

After the main loop, the four accumulators are merged with a tree of adds, then a horizontal reduction collapses the 8-lane vector down to a single scalar. The scalar tail loop handles any remaining elements that didn't make it into a full 32-element chunk.

### Q2: Aligned vs unaligned loads

This example lets you measure the difference empirically on your own machine. The reason to still prefer aligned allocations is not mainly the load speed - it's correctness: `_mm256_load_ps` will crash at runtime on a misaligned pointer, so if you're using aligned loads you can catch alignment bugs immediately rather than silently getting slightly wrong behavior.

```cpp
#include <iostream>
#include <immintrin.h>
#include <chrono>
#include <cstdlib>
#include <vector>

void add_aligned(float* __restrict c, const float* __restrict a,
                 const float* __restrict b, int n) {
    for (int i = 0; i < n; i += 8) {
        __m256 va = _mm256_load_ps(a + i);   // movaps: REQUIRES 32-byte alignment
        __m256 vb = _mm256_load_ps(b + i);   // Segfault if not aligned!
        _mm256_store_ps(c + i, _mm256_add_ps(va, vb));
    }
}

void add_unaligned(float* __restrict c, const float* __restrict a,
                   const float* __restrict b, int n) {
    for (int i = 0; i < n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);  // movups: any alignment OK
        __m256 vb = _mm256_loadu_ps(b + i);
        _mm256_storeu_ps(c + i, _mm256_add_ps(va, vb));
    }
}

int main() {
    constexpr int N = 1024 * 1024 * 8;  // 32 MB
    // 32-byte aligned allocation
    float* a = static_cast<float*>(std::aligned_alloc(32, N * sizeof(float)));
    float* b = static_cast<float*>(std::aligned_alloc(32, N * sizeof(float)));
    float* c = static_cast<float*>(std::aligned_alloc(32, N * sizeof(float)));

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 100; ++r) fn(c, a, b, N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(add_aligned,   "Aligned  ");
    bench(add_unaligned, "Unaligned");
    // Modern CPUs (Haswell+): nearly identical performance!
    // Aligned: ~650ms, Unaligned: ~660ms (1-2% difference)
    //
    // When alignment matters:
    //   - Cache line splits (address crosses 64-byte boundary)
    //   - Page splits (address crosses 4KB boundary) -> ~100 cycle penalty
    //   - AVX-512: aligned access avoids micro-op split

    std::free(a); std::free(b); std::free(c);
}
```

The comment about page splits is worth remembering. If a 32-byte AVX load straddles a 4KB page boundary, the hardware has to do two TLB lookups instead of one, which can cost around 100 cycles. This is rare in practice but it is why aligned allocation is still the right default even if unaligned loads are "fast."

### Q3: Portable SIMD abstraction with scalar fallback

Writing raw intrinsics directly into application code has a portability problem: the code simply won't compile on a machine without AVX2. A common solution is to wrap the intrinsics in a thin abstraction layer and provide a scalar fallback behind a preprocessor guard. The user code then writes against the abstraction and gets the right backend automatically.

```cpp
#include <iostream>
#include <cmath>

// Portable SIMD wrapper: uses AVX2 if available, scalar otherwise
namespace simd {

#ifdef __AVX2__
#include <immintrin.h>

struct Vec8f {
    __m256 v;
    Vec8f() : v(_mm256_setzero_ps()) {}
    explicit Vec8f(float x) : v(_mm256_set1_ps(x)) {}
    Vec8f(__m256 x) : v(x) {}
    static Vec8f load(const float* p)  { return _mm256_loadu_ps(p); }
    void store(float* p) const        { _mm256_storeu_ps(p, v); }
    static constexpr int size = 8;
};

Vec8f operator+(Vec8f a, Vec8f b) { return _mm256_add_ps(a.v, b.v); }
Vec8f operator*(Vec8f a, Vec8f b) { return _mm256_mul_ps(a.v, b.v); }
Vec8f fmadd(Vec8f a, Vec8f b, Vec8f c) { return _mm256_fmadd_ps(a.v, b.v, c.v); }

#else  // Scalar fallback

struct Vec8f {
    float v[8];
    Vec8f() : v{} {}
    explicit Vec8f(float x) { for (auto& e : v) e = x; }
    static Vec8f load(const float* p) {
        Vec8f r; for (int i=0; i<8; ++i) r.v[i] = p[i]; return r;
    }
    void store(float* p) const { for (int i=0; i<8; ++i) p[i] = v[i]; }
    static constexpr int size = 8;
};

Vec8f operator+(Vec8f a, Vec8f b) {
    Vec8f r; for (int i=0; i<8; ++i) r.v[i] = a.v[i] + b.v[i]; return r;
}
Vec8f operator*(Vec8f a, Vec8f b) {
    Vec8f r; for (int i=0; i<8; ++i) r.v[i] = a.v[i] * b.v[i]; return r;
}
Vec8f fmadd(Vec8f a, Vec8f b, Vec8f c) {
    Vec8f r; for (int i=0; i<8; ++i) r.v[i] = a.v[i]*b.v[i]+c.v[i]; return r;
}

#endif

} // namespace simd

// User code is IDENTICAL regardless of backend:
void saxpy(float* y, float a, const float* x, int n) {
    simd::Vec8f va(a);
    int i = 0;
    for (; i + simd::Vec8f::size <= n; i += simd::Vec8f::size) {
        auto vy = simd::Vec8f::load(y + i);
        auto vx = simd::Vec8f::load(x + i);
        simd::fmadd(vx, va, vy).store(y + i);  // y = a*x + y
    }
    for (; i < n; ++i) y[i] += a * x[i]; // scalar tail
}

int main() {
    float x[16], y[16];
    for (int i = 0; i < 16; ++i) { x[i] = float(i); y[i] = 100.0f; }
    saxpy(y, 2.0f, x, 16);
    for (int i = 0; i < 16; ++i)
        std::cout << y[i] << ' '; // 100 102 104 106 ... 130
    std::cout << '\n';
}
// With AVX2: saxpy uses vfmadd231ps
// Without:   saxpy uses scalar multiply+add (correct but slow)
```

The scalar fallback lets you develop and test on any machine, then compile with `-mavx2 -mfma` when you deploy to hardware that supports it. For production code, consider using Google Highway or `std::experimental::simd`, which implement this abstraction more completely and handle more ISA variants.

---

## Notes

- For production portable SIMD: use Highway library or `std::experimental::simd` instead of raw `#ifdef`.
- Multiple accumulators (4+) are essential to saturate FMA throughput (latency hiding).
- `std::aligned_alloc(32, size)` for AVX2, `std::aligned_alloc(64, size)` for AVX-512.
- `_mm256_stream_ps` for write-only buffers (non-temporal, bypasses cache).
- Check CPU support at runtime with `__builtin_cpu_supports("avx2")` (see #725).
