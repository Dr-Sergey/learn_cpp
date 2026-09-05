# Use -O2 vs -O3 vs -Oz: understand what each optimization level enables

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html>  

---

## Topic Overview

Optimization levels are a dial you turn between "easy to debug" and "as fast as possible." The compiler uses the level you choose to decide which transformations are worth doing - more aggressive levels spend more compile time looking for ways to make the binary smaller or faster. Here is the full spectrum:

| Level | Goal | Key Features |
| --- | --- | --- |
| `-O0` | Debug | No optimization, fast compile, 1:1 with source |
| `-O1` | Basic | Simple optimizations, moderate compile time |
| `-O2` | Standard | Most optimizations without size/speed tradeoff |
| `-O3` | Aggressive | Auto-vectorization, loop unrolling, function cloning |
| `-Os` | Size | Like `-O2` but prefers smaller code |
| `-Oz` | Min size | Clang: aggressive size reduction (no inlining) |

For most projects, `-O2` is the right default. You choose `-O3` when you have benchmarked and confirmed the extra optimizations help your specific workload. You reach for `-Oz` when binary size is a hard constraint, such as firmware or WebAssembly targets.

---

## Self-Assessment

### Q1: Five optimizations enabled at -O3 but not -O2

These are the headline features that `-O3` unlocks. Each one can dramatically help certain workloads and do little or nothing for others - which is exactly why they are not on by default:

| Optimization | GCC Flag | Effect |
| --- | --- | --- |
| Auto-vectorization | `-ftree-vectorize` | Uses SIMD (SSE/AVX) for loops |
| Loop unrolling | `-funroll-loops` | Duplicates loop body to reduce branch overhead |
| Function cloning | `-fipa-cp-clone` | Creates specialized copies of functions |
| Loop interchange | `-floop-interchange` | Swaps nested loop order for cache locality |
| Predictive commoning | `-fpredictive-commoning` | Reuses values from previous iterations |

Auto-vectorization is the one you feel most often. When you have a tight loop over an array, `-O3` will try to rewrite it to process multiple elements at once using SIMD instructions. The following loop is a perfect candidate:

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <chrono>

// This loop benefits from auto-vectorization at -O3
void sum_arrays(const float* a, const float* b, float* c, int n) {
    for (int i = 0; i < n; ++i)
        c[i] = a[i] + b[i];
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N, 1.0f), b(N, 2.0f), c(N);

    auto start = std::chrono::high_resolution_clock::now();
    sum_arrays(a.data(), b.data(), c.data(), N);
    auto end = std::chrono::high_resolution_clock::now();

    auto ms = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    std::cout << "Sum: " << c[0] << ", Time: " << ms << " us\n";
}
// Compile and compare:
//   g++ -O2 -o prog_O2 prog.cpp && ./prog_O2  -> ~15ms
//   g++ -O3 -o prog_O3 prog.cpp && ./prog_O3  -> ~4ms (vectorized!)
//
// Check vectorization: g++ -O3 -ftree-vectorizer-verbose=1 prog.cpp
// Or: g++ -O3 -fopt-info-vec-optimized prog.cpp
//   -> "loop vectorized using 128 bit vectors"
```

The speedup here is real - roughly 4x for a workload that maps cleanly onto SIMD. Use `-fopt-info-vec-optimized` to confirm the compiler actually vectorized the loop; sometimes data alignment or aliasing concerns prevent it.

### Q2: `-O3` and floating-point reordering

Here is a subtlety that trips people up: `-O3` by itself does **not** reorder floating-point operations. The dangerous flag is `-ffast-math`, which is bundled into `-Ofast`. The distinction matters because floating-point arithmetic is not associative - `(a + b) + c` and `a + (b + c)` can produce different results when the magnitudes differ by many orders of magnitude.

```cpp
#include <iostream>
#include <iomanip>
#include <cmath>

// Kahan summation: relies on precise FP order
double kahan_sum(const double* data, int n) {
    double sum = 0.0;
    double compensation = 0.0;

    for (int i = 0; i < n; ++i) {
        double y = data[i] - compensation;
        double t = sum + y;           // sum is big, y is small
        compensation = (t - sum) - y; // recovers the lost bits
        sum = t;
    }
    return sum;
}

int main() {
    // Summing many small values with one large value
    constexpr int N = 1000000;
    double data[10];

    // Manual test with known precision sensitivity
    double a = 1.0;
    double b = 1e-16;
    double c = -1.0;

    // Mathematically: a + b + c = 1e-16
    // With FP reordering: (b + c) + a might lose b entirely
    double result1 = (a + b) + c;  // preserves b: 1e-16
    double result2 = a + (b + c);  // b + c = -1.0 (b lost!), then + a = 0.0

    std::cout << std::setprecision(20);
    std::cout << "(a + b) + c = " << result1 << '\n';
    std::cout << "a + (b + c) = " << result2 << '\n';
    std::cout << "Difference:   " << std::abs(result1 - result2) << '\n';
}
// Expected output:
// (a + b) + c = 1e-16
// a + (b + c) = 0
// Difference:   1e-16
//
// NOTE: -O3 alone does NOT reorder FP by default.
// The DANGEROUS flag is -ffast-math (or -Ofast):
//   g++ -Ofast           -> enables -ffast-math
//   g++ -O3 -ffast-math  -> allows FP reassociation
// Use -fno-fast-math to disable, or -ffp-contract=off for specific control.
```

If you ever need maximum performance on floating-point heavy code and you are considering `-ffast-math`, benchmark both with and without it and run your numerical validation suite. Some algorithms like Kahan summation are specifically designed to fight floating-point rounding loss - `-ffast-math` can silently break them.

### Q3: `-Oz` for minimum binary size

`-Oz` is Clang's most aggressive size-reduction mode. The main lever it pulls is disabling inlining almost entirely - instead of duplicating a function's body at each call site, it emits a real call instruction. That saves code size at the cost of one extra function call per site. For IoT firmware or WebAssembly targets where every kilobyte matters, this trade-off is usually acceptable:

```cpp
#include <iostream>

// At -Oz, this function will NOT be inlined (saves code size)
inline int compute(int x) {
    return x * x + 2 * x + 1;
}

int main() {
    int sum = 0;
    for (int i = 0; i < 100; ++i)
        sum += compute(i);  // -O3: inlined, -Oz: call instruction

    std::cout << "Sum: " << sum << '\n';
    return 0;
}
// Expected output:
// Sum: 338350
```

You can measure the size impact directly:

```bash
# Compare binary sizes:
clang++ -O2 -o prog_O2 prog.cpp && ls -la prog_O2  # ~17KB
clang++ -O3 -o prog_O3 prog.cpp && ls -la prog_O3  # ~18KB (larger: unrolled/inlined)
clang++ -Os -o prog_Os prog.cpp && ls -la prog_Os  # ~16KB (O2 minus size-increasing opts)
clang++ -Oz -o prog_Oz prog.cpp && ls -la prog_Oz  # ~15KB (aggressive: no inlining)

# Strip symbols for production:
strip prog_Oz && ls -la prog_Oz  # ~10KB
```

The summary for choosing an optimization level in practice:

| Level | Binary Size | Performance | Use Case |
| --- | --- | --- | --- |
| `-O2` | Medium | Good | Default for most projects |
| `-O3` | Large | Best | HPC, number crunching |
| `-Os` | Small | Good | Embedded, mobile |
| `-Oz` | Smallest | Slower | IoT, firmware, WASM |

---

## Notes

- `-O2` is the **recommended default** for production code.
- `-O3` can occasionally be **slower** than `-O2` due to cache pressure from code bloat.
- `-Oz` is Clang-only; GCC uses `-Os` as the minimum-size option.
- `-Ofast` = `-O3` + `-ffast-math` - dangerous for numeric code.
- Always benchmark with your actual workload; micro-benchmarks can be misleading.
- Use `-march=native` with `-O3` for maximum vectorization on your specific CPU.
