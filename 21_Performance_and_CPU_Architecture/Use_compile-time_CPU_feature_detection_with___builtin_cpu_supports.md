# Use compile-time CPU feature detection with __builtin_cpu_supports

**Category:** Performance and CPU Architecture  
**Standard:** C++11  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html>  

---

## Topic Overview

You want to ship a single binary that runs on any x86-64 machine but automatically uses the best available SIMD instructions - SSE2 on older hardware, AVX2 on modern desktops, AVX-512 on servers. Runtime CPU feature detection is how you do that.

There are several ways to achieve this, with different tradeoffs in overhead and control:

| Method | When resolved | Overhead | Portability |
| --- | --- | --- | --- |
| `__builtin_cpu_supports()` | Each call (runtime) | ~1ns (branch) | GCC/Clang |
| `ifunc` attribute | Load time (dynamic linker) | 0 per call | GCC/Clang (Linux) |
| `target_clones` attribute | Load time (automatic) | 0 per call | GCC 6+/Clang 14+ |
| CPUID manually | Runtime | Higher | Any compiler |

The key insight is that `__builtin_cpu_supports` checks a CPU feature flag at runtime, but the overhead (~1ns for a branch) is only paid once per call to the dispatching function, not once per iteration of the inner loop. The inner loop itself runs at full SIMD speed.

---

## Self-Assessment

### Q1: Runtime dispatch with `__builtin_cpu_supports`

The simplest dispatch pattern: write two implementations with different `target` attributes, then check `__builtin_cpu_supports` at call time to pick the right one. The overhead is one branch per call to `add_arrays`, not per element:

```cpp
#include <iostream>
#include <vector>
#include <cstring>

// SSE2 fallback (available on all x86-64)
__attribute__((target("sse2")))
void add_sse2(float* a, const float* b, const float* c, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] + c[i];
    // Compiler uses SSE2 instructions: movups, addps (128-bit, 4 floats)
}

// AVX2 optimized (256-bit, 8 floats per instruction)
__attribute__((target("avx2")))
void add_avx2(float* a, const float* b, const float* c, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] + c[i];
    // Compiler uses AVX2: vmovups, vaddps (256-bit, 8 floats)
}

// Dispatch at runtime:
void add_arrays(float* a, const float* b, const float* c, int n) {
    // __builtin_cpu_supports checks CPUID flags at runtime
    if (__builtin_cpu_supports("avx2")) {
        add_avx2(a, b, c, n);  // faster path
    } else {
        add_sse2(a, b, c, n);  // fallback
    }
    // Cost: one branch per function call (~1ns)
    // The actual SIMD loop runs at full speed.
}

int main() {
    __builtin_cpu_init();  // Initialize CPU feature detection

    std::cout << "CPU features:\n";
    std::cout << "  SSE2:    " << __builtin_cpu_supports("sse2") << '\n';
    std::cout << "  AVX:     " << __builtin_cpu_supports("avx") << '\n';
    std::cout << "  AVX2:    " << __builtin_cpu_supports("avx2") << '\n';
    std::cout << "  AVX512F: " << __builtin_cpu_supports("avx512f") << '\n';
    std::cout << "  FMA:     " << __builtin_cpu_supports("fma") << '\n';

    std::vector<float> a(1000), b(1000, 1.0f), c(1000, 2.0f);
    add_arrays(a.data(), b.data(), c.data(), 1000);
    std::cout << "a[0] = " << a[0] << '\n';  // 3.0
}
```

Notice that each implementation is annotated with `__attribute__((target("avx2")))` or `__attribute__((target("sse2")))`. This tells the compiler to use those instruction sets for that specific function, even if the rest of the translation unit was compiled without those flags. Without these attributes, the compiler would refuse to emit AVX2 instructions in a file compiled for the SSE2 baseline.

### Q2: `ifunc` for load-time dispatch

If `add_arrays` is called very frequently (e.g., inside an inner loop), even a ~1ns branch per call adds up. `ifunc` eliminates that overhead by resolving the function pointer **once** at program load time through the dynamic linker, so subsequent calls go directly to the right implementation with no branching at all.

```cpp
#include <iostream>

// ifunc resolves the function pointer ONCE at load time (dynamic linker).
// No per-call overhead!

__attribute__((target("sse2")))
static void impl_sse2(float* a, const float* b, int n) {
    for (int i = 0; i < n; ++i) a[i] += b[i];
}

__attribute__((target("avx2")))
static void impl_avx2(float* a, const float* b, int n) {
    for (int i = 0; i < n; ++i) a[i] += b[i];
}

// Resolver function: called ONCE by the dynamic linker at load time
static void (*resolve_add(void))(float*, const float*, int) {
    __builtin_cpu_init();
    if (__builtin_cpu_supports("avx2"))
        return impl_avx2;
    return impl_sse2;
}

// The ifunc attribute binds the resolver:
void add_vectors(float* a, const float* b, int n)
    __attribute__((ifunc("resolve_add")));
// Now add_vectors() calls directly into impl_avx2 or impl_sse2
// with ZERO dispatch overhead at call time.

// Note: ifunc is Linux-only (uses glibc IFUNC mechanism).
// On macOS/Windows, use a function pointer initialized at startup.

int main() {
    float a[8] = {1,2,3,4,5,6,7,8};
    float b[8] = {10,20,30,40,50,60,70,80};
    add_vectors(a, b, 8);  // resolved at load time!
    for (int i = 0; i < 8; ++i)
        std::cout << a[i] << ' ';  // 11 22 33 44 55 66 77 88
    std::cout << '\n';
}
```

The mechanism works because the dynamic linker, when it sets up the GOT (Global Offset Table) entry for `add_vectors`, calls `resolve_add` and stores the returned pointer. Every call to `add_vectors` in the program then jumps directly to `impl_avx2` or `impl_sse2` - no conditional branch, no CPU feature check.

### Q3: `target_clones` for automatic multi-versioning

`ifunc` requires you to write both the resolver and the multiple implementations manually. `target_clones` automates this: you write one function body, annotate it with the ISA variants you want, and the compiler generates the versions and the dispatch mechanism automatically.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// target_clones: compiler generates MULTIPLE versions automatically
// and inserts ifunc dispatch.
__attribute__((target_clones("avx512f", "avx2", "sse2", "default")))
void process(float* __restrict a, const float* __restrict b,
             const float* __restrict c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] * c[i] + b[i];
    }
}
// Compiler generates:
//   process.avx512f  (uses 512-bit SIMD, 16 floats/instruction)
//   process.avx2     (uses 256-bit SIMD, 8 floats/instruction)
//   process.sse2     (uses 128-bit SIMD, 4 floats/instruction)
//   process.default  (scalar fallback)
//   process          (ifunc: resolves to best version at load time)
//
// No manual dispatch code needed! The compiler does everything.

// GCC 6+: __attribute__((target_clones(...)))
// Clang 14+: same syntax
// MSVC: no equivalent (use manual dispatch or /arch flags)

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N), b(N, 2.0f), c(N, 3.0f);

    auto t0 = std::chrono::high_resolution_clock::now();
    for (int r = 0; r < 100; ++r)
        process(a.data(), b.data(), c.data(), N);
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);

    std::cout << "Time: " << ms.count() / 100 << " ms/iter\n";
    std::cout << "a[0] = " << a[0] << '\n';  // 8.0 (2*3 + 2)

    // Inspect generated versions:
    //   objdump -d test | grep 'process\.'
    //   000000000040xxxx process.avx512f:
    //   000000000040xxxx process.avx2:
    //   000000000040xxxx process.sse2:
    //   000000000040xxxx process.default:
}
```

`target_clones` is the right choice when you want multi-ISA dispatch with minimal code maintenance overhead. You get the full performance benefit of the best available ISA, automatic fallback, and you only have one copy of the function logic to maintain. The main cost is binary size: each clone adds the full compiled function.

---

## Notes

- Call `__builtin_cpu_init()` before first use of `__builtin_cpu_supports()` (GCC).
- `target_clones` is the easiest approach - just annotate the function, compiler does the rest.
- `ifunc` gives manual control over dispatch logic (useful for complex decisions).
- For portable code: use Highway or `std::experimental::simd` which handle dispatch internally.
- `-march=native` compiles for the build machine only (not portable).
