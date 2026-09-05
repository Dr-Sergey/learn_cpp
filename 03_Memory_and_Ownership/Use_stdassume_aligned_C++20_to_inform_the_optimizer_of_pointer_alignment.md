# Use std::assume_aligned (C++20) to Inform the Optimizer of Pointer Alignment

**Category:** Memory and Ownership  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/memory/assume_aligned>  

---

## Topic Overview

### What Is `std::assume_aligned`

`std::assume_aligned<N>(ptr)` returns `ptr` with a promise to the compiler that it is aligned to `N` bytes. The function itself generates zero code - it is a pure hint. The payoff is that the optimizer can then emit SIMD instructions that require alignment guarantees, without inserting the usual runtime alignment checks.

```cpp
#include <memory>

void process(float* data, size_t n) {
    // Tell the compiler: data is 32-byte aligned
    float* aligned = std::assume_aligned<32>(data);
    for (size_t i = 0; i < n; ++i)
        aligned[i] *= 2.0f;
    // Compiler can now use AVX (256-bit) instructions without
    // generating alignment check prologues
}
```

Without the hint, the compiler would need to generate a scalar prologue to handle any leading unaligned elements before it can safely enter the vectorized loop. With the hint, it can jump straight into the vectorized body.

### Why Alignment Matters for SIMD

| Alignment | SIMD Width | Instructions |
| --- | --- | --- |
| 16 bytes | SSE (128-bit) | `movaps` (aligned, faster) vs `movups` |
| 32 bytes | AVX (256-bit) | `vmovaps` - requires 32-byte alignment |
| 64 bytes | AVX-512 | `vmovaps zmm` - requires 64-byte alignment |

Without alignment info, the compiler must generate:

1. A scalar prologue to reach alignment boundary
2. The vectorized body
3. A scalar epilogue for remaining elements

With `assume_aligned`, steps 1 and 3 can be eliminated.

---

## Self-Assessment

### Q1: Apply `std::assume_aligned<32>(ptr)` to a SIMD buffer and inspect the vectorized assembly

The real value of this example is the compile command at the bottom. If you paste the two scale functions into Compiler Explorer with `-O2 -mavx2`, you will see `vmovaps` (aligned) versus `vmovups` (unaligned) in the assembly, plus the presence or absence of the alignment-checking prologue.

```cpp
#include <iostream>
#include <memory>
#include <cstddef>
#include <cstdlib>
#include <new>
#include <numeric>

// Allocate 32-byte aligned memory
float* alloc_aligned(size_t count) {
#ifdef _MSC_VER
    return static_cast<float*>(_aligned_malloc(count * sizeof(float), 32));
#else
    void* p = nullptr;
    posix_memalign(&p, 32, count * sizeof(float));
    return static_cast<float*>(p);
#endif
}

void free_aligned(float* p) {
#ifdef _MSC_VER
    _aligned_free(p);
#else
    free(p);
#endif
}

// Without alignment hint — compiler generates alignment checks
void scale_unaligned(float* data, size_t n, float factor) {
    for (size_t i = 0; i < n; ++i)
        data[i] *= factor;
}

// With alignment hint — compiler can use aligned AVX instructions
void scale_aligned(float* raw, size_t n, float factor) {
    float* data = std::assume_aligned<32>(raw);
    for (size_t i = 0; i < n; ++i)
        data[i] *= factor;
    // On Godbolt with -O2 -mavx2:
    // Generates vmovaps (aligned store) instead of vmovups
    // No alignment check prologue needed
}

// Sum with alignment hint
float sum_aligned(const float* raw, size_t n) {
    const float* data = std::assume_aligned<32>(raw);
    float total = 0.0f;
    for (size_t i = 0; i < n; ++i)
        total += data[i];
    return total;
}

int main() {
    constexpr size_t N = 1024;
    float* buf = alloc_aligned(N);

    // Fill with data
    for (size_t i = 0; i < N; ++i) buf[i] = static_cast<float>(i);

    // Verify alignment
    std::cout << "Buffer address: " << buf << "\n";
    std::cout << "Aligned to 32: "
              << (reinterpret_cast<uintptr_t>(buf) % 32 == 0 ? "YES" : "NO") << "\n\n";

    scale_aligned(buf, N, 2.0f);
    std::cout << "buf[0]=" << buf[0] << " buf[1]=" << buf[1]
              << " buf[100]=" << buf[100] << "\n";

    float s = sum_aligned(buf, N);
    std::cout << "Sum: " << s << "\n";

    free_aligned(buf);
    return 0;
}
// To see the difference, compile with:
//   g++ -O2 -mavx2 -std=c++20 -S assume_aligned.cpp
// Compare scale_unaligned vs scale_aligned assembly
```

### Q2: Explain the UB if the pointer is not actually aligned to the stated boundary

This is the dangerous side of the feature. `assume_aligned` is a promise, not a check. If the pointer is not actually aligned to the stated boundary, the compiler may generate an instruction like `MOVAPS` that causes a hardware fault on misaligned addresses, or it may fold the alignment assumption into address calculations and silently produce wrong results. There is no runtime error - just undefined behavior.

```cpp
#include <iostream>
#include <memory>
#include <cstdint>

void demonstrate_misalignment_ub() {
    // std::assume_aligned is a PROMISE to the compiler.
    // If the pointer is NOT actually aligned, behavior is UNDEFINED.
    //
    // What can go wrong:
    // 1. Compiler generates MOVAPS (aligned move) -> SIGBUS/SIGSEGV on misaligned addr
    // 2. Compiler folds alignment assumptions into address calculations -> wrong offsets
    // 3. On some architectures: hardware trap, program crash
    // 4. Silently wrong results (misread data from wrong offset)

    alignas(32) float aligned_buf[8] = {1, 2, 3, 4, 5, 6, 7, 8};

    // CORRECT: pointer IS 32-byte aligned
    float* good = std::assume_aligned<32>(aligned_buf);
    std::cout << "Aligned access: " << good[0] << "\n";  // OK

    // WRONG: offset by 4 bytes — NOT 32-byte aligned anymore
    float* misaligned = aligned_buf + 1;  // Offset by sizeof(float)
    // float* bad = std::assume_aligned<32>(misaligned);  // UB!
    // bad[0];  // May crash (MOVAPS on misaligned address)
    //          // Or silently read wrong data
    //          // Or "work" today, crash tomorrow

    // How to verify alignment at runtime (debug builds):
    auto check = [](const void* ptr, size_t alignment) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(ptr);
        bool ok = (addr % alignment == 0);
        std::cout << "Address " << ptr << " aligned to " << alignment
                  << ": " << (ok ? "YES" : "NO") << "\n";
        return ok;
    };

    check(aligned_buf, 32);      // YES
    check(aligned_buf + 1, 32);  // NO
    check(aligned_buf + 8, 32);  // Depends on sizeof(float)*8 vs 32

    std::cout << "\nRule: ALWAYS ensure actual alignment matches assume_aligned<N>\n";
    std::cout << "Use alignas(N), aligned_alloc, or _aligned_malloc\n";
}

int main() {
    demonstrate_misalignment_ub();
    return 0;
}
```

### Q3: Compare `std::assume_aligned` with `__builtin_assume_aligned` for portability

Before C++20, GCC and Clang both had a compiler-specific builtin for this. MSVC used a different approach. The portable wrapper below is a practical pattern you will see in performance-sensitive codebases that need to support multiple compilers.

```cpp
#include <iostream>
#include <memory>
#include <cstddef>

// ================================================================
// Comparison: std::assume_aligned vs __builtin_assume_aligned
// ================================================================
//
// | Feature                 | std::assume_aligned<N> | __builtin_assume_aligned |
// |-------------------------|------------------------|--------------------------|
// | Standard                | C++20                  | GCC/Clang extension      |
// | MSVC support            | Yes (VS 2019 16.8+)   | No                       |
// | constexpr               | Yes                    | No                       |
// | Template parameter      | Yes (compile-time N)   | No (runtime argument)    |
// | Type safety             | Returns same type      | Returns void*            |
// | Header                  | <memory>               | Built-in                 |

// Portable wrapper that works everywhere:
template<size_t N, typename T>
T* portable_assume_aligned(T* ptr) {
#if __cplusplus >= 202002L
    return std::assume_aligned<N>(ptr);
#elif defined(__GNUC__) || defined(__clang__)
    return static_cast<T*>(__builtin_assume_aligned(ptr, N));
#elif defined(_MSC_VER)
    // MSVC: no builtin equivalent pre-C++20, but __assume helps
    __assume(reinterpret_cast<uintptr_t>(ptr) % N == 0);
    return ptr;
#else
    return ptr;  // No alignment hint available
#endif
}

void process(float* raw, size_t n) {
    float* data = portable_assume_aligned<32>(raw);
    for (size_t i = 0; i < n; ++i)
        data[i] *= 2.0f;
}

int main() {
    alignas(32) float buf[64];
    for (int i = 0; i < 64; ++i) buf[i] = static_cast<float>(i);

    process(buf, 64);

    std::cout << "buf[0]=" << buf[0] << " buf[1]=" << buf[1] << "\n";

    std::cout << "\n=== Recommendation ===\n";
    std::cout << "C++20 available:  use std::assume_aligned<N>(ptr)\n";
    std::cout << "GCC/Clang only:   __builtin_assume_aligned(ptr, N)\n";
    std::cout << "Portable:         write a wrapper with #ifdef\n";
    std::cout << "MSVC pre-C++20:   __assume() or __declspec(align(N))\n";

    return 0;
}
```

---

## Notes

- `std::assume_aligned<N>(ptr)` is a zero-cost hint - it generates no code, only informs the optimizer.
- **UB if the pointer is not actually aligned** - always use `alignas`, `aligned_alloc`, or platform APIs.
- Enables vectorized SIMD code without runtime alignment checks or scalar prologues.
- Combine with `alignas(N)` for stack buffers and `std::aligned_alloc` for heap buffers.
- Check alignment in debug builds with `assert(reinterpret_cast<uintptr_t>(ptr) % N == 0)`.
