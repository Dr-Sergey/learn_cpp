# Understand memory bandwidth limits and arithmetic intensity (roofline model)

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Roofline_model>  

---

## Topic Overview

When a piece of code is running slower than you expect, the roofline model gives you a framework to diagnose why. The idea is simple: every kernel is limited by one of two things - either the CPU cannot compute fast enough (compute-bound), or memory cannot deliver data fast enough (memory-bound). The roofline model shows you which ceiling you are hitting.

The key metric is **arithmetic intensity**: the ratio of floating-point operations to bytes of memory traffic (FLOP/byte). A kernel with high arithmetic intensity does a lot of computation per byte of data moved - it is likely compute-bound. A kernel with low arithmetic intensity spends most of its time waiting for data - it is memory-bound.

```cpp
  Performance (GFLOP/s)
       ^
       |         _______________
       |        /               <- compute ceiling (peak FLOP/s)
       |       /
       |      /  <- memory bandwidth ceiling
       |     /
       |    /
       |   /
       +--/------------------------------> Arithmetic Intensity (FLOP/byte)
          ^                      ^
      memory-bound          compute-bound

  Ridge point = peak_GFLOPS / peak_bandwidth_GB_s
```

The **ridge point** is the arithmetic intensity where the memory bandwidth ceiling meets the compute ceiling. If your kernel's arithmetic intensity is below the ridge point, you are memory-bound; above it, you are compute-bound.

Here is how common kernels fall on that scale:

| Kernel | FLOP/byte | Bound |
| --- | --- | --- |
| Stream copy (`a[i]=b[i]`) | 0 | Memory |
| DAXPY (`a[i]=a[i]+s*b[i]`) | 2 FLOP / 16 bytes = 0.125 | Memory |
| Dot product | 2 FLOP / 8 bytes = 0.25 | Memory |
| Matrix multiply (NxN) | 2N³ / 3N²×8 = N/12 FLOP/byte | Compute (for large N) |

---

## Self-Assessment

### Q1: Arithmetic intensity of DGEMM vs stream copy

Working out the arithmetic intensity by hand is the most important skill here. You count the FLOPs the kernel performs and the bytes it reads and writes, then take the ratio. The examples below cover several common kernels and then classify them against a representative machine's ridge point:

```cpp
#include <iostream>
#include <cmath>

int main() {
    // STREAM COPY: a[i] = b[i]
    //   FLOPs:  0 (just a copy)
    //   Bytes:  read 8 (double b) + write 8 (double a) = 16 bytes/element
    //   AI:     0 FLOP / 16 bytes = 0.0 FLOP/byte
    //   => ALWAYS memory-bound. Stream bandwidth is the limit.

    // DAXPY: a[i] = a[i] + s * b[i]
    //   FLOPs:  2 (multiply + add)
    //   Bytes:  read a (8) + read b (8) + write a (8) = 24 bytes
    //           (or 16 if write-allocate eliminated)
    //   AI:     2/24 = 0.083 or 2/16 = 0.125 FLOP/byte
    //   => Memory-bound on all modern CPUs

    // DGEMM (NxN): C = A * B
    //   FLOPs:  2*N^3 (N^3 multiply + N^3 add)
    //   Bytes:  3 * N^2 * 8 (read A, B, write C, each N^2 doubles)
    //   AI:     2*N^3 / (24*N^2) = N/12 FLOP/byte
    //   For N=1024: AI = 1024/12 = 85 FLOP/byte
    //   => Compute-bound for N > ~100!

    double peak_gflops = 200.0;    // typical Skylake AVX-512
    double bandwidth   = 50.0;     // GB/s DDR4 dual-channel
    double ridge_point = peak_gflops / bandwidth;  // FLOP/byte

    std::cout << "Peak compute: " << peak_gflops << " GFLOP/s\n";
    std::cout << "Peak bandwidth: " << bandwidth << " GB/s\n";
    std::cout << "Ridge point: " << ridge_point << " FLOP/byte\n";
    std::cout << '\n';

    auto classify = [&](const char* name, double ai) {
        double attainable = std::min(peak_gflops, bandwidth * ai);
        std::cout << name << ": AI=" << ai << " FLOP/byte -> "
                  << attainable << " GFLOP/s ("
                  << (ai < ridge_point ? "MEMORY-bound" : "COMPUTE-bound")
                  << ")\n";
    };

    classify("Stream copy   ", 0.0);
    classify("DAXPY         ", 0.125);
    classify("Dot product   ", 0.25);
    classify("SpMV          ", 0.5);
    classify("DGEMM (N=1024)", 85.0);
}
```

The DGEMM result is striking. A 1024x1024 matrix multiply has an arithmetic intensity of 85 FLOP/byte - well above the ridge point of 4.0 (200/50). Every byte fetched from memory is used for 85 arithmetic operations. That is why matrix multiply benefits so dramatically from SIMD and cache blocking, and why it is the canonical compute-bound kernel.

### Q2: Roofline model to classify compute-bound vs memory-bound

You can verify the roofline classification experimentally by measuring actual GFLOP/s and comparing to the theoretical limits. A memory-bound kernel will plateau at `bandwidth * AI` regardless of how much you optimize the compute path; a compute-bound kernel will plateau at `peak_gflops`:

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// Memory-bound kernel: stream triad  a[i] = b[i] + s * c[i]
void stream_triad(double* __restrict a, const double* __restrict b,
                  const double* __restrict c, double s, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + s * c[i];
    }
    // AI = 2 FLOP / 24 bytes = 0.083
}

// Compute-bound kernel: many operations per element
void compute_heavy(double* __restrict a, const double* __restrict b, int n) {
    for (int i = 0; i < n; ++i) {
        double x = b[i];
        // 20 FLOPs per element:
        x = x*x + x; x = x*x + x; x = x*x + x; x = x*x + x;
        x = x*x + x; x = x*x + x; x = x*x + x; x = x*x + x;
        x = x*x + x; x = x*x + x;
        a[i] = x;
    }
    // AI = 20 FLOP / 16 bytes = 1.25
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<double> a(N), b(N, 1.1), c(N, 2.2);

    auto bench = [&](auto fn, const char* label, double flops_per_elem) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 10; ++r) fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        double sec = std::chrono::duration<double>(t1 - t0).count() / 10;
        double gflops = N * flops_per_elem / sec / 1e9;
        std::cout << label << ": " << gflops << " GFLOP/s (" << sec*1000 << " ms)\n";
    };

    bench([&]{ stream_triad(a.data(), b.data(), c.data(), 3.0, N); },
          "Stream triad (mem-bound) ", 2);
    bench([&]{ compute_heavy(a.data(), b.data(), N); },
          "Compute heavy (cpu-bound)", 20);

    // Stream triad: ~10 GFLOP/s (bounded by memory bandwidth)
    // Compute heavy: ~100 GFLOP/s (bounded by CPU FLOP throughput)
}
```

If you add more arithmetic to the stream triad without increasing data movement, its measured GFLOP/s will not improve - that is the signature of a memory-bound kernel. If you add more arithmetic to the compute-heavy kernel, it slows down proportionally - that is the signature of a compute-bound kernel.

### Q3: Loop fusion to increase arithmetic intensity

One of the most practical roofline-improving techniques is loop fusion. If you have two separate loops that process the same array, fusing them into one loop means the data gets loaded from memory once instead of twice, effectively doubling the arithmetic intensity:

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <cmath>

// UNFUSED: two separate loops, each memory-bound
void unfused(double* a, double* b, const double* c, int n) {
    // Loop 1: reads c, writes a (24 bytes/element)
    for (int i = 0; i < n; ++i)
        a[i] = c[i] * 2.0;

    // Loop 2: reads a, writes b (24 bytes/element)
    for (int i = 0; i < n; ++i)
        b[i] = std::sqrt(a[i]) + a[i];

    // Total: 3 FLOP / 48 bytes = 0.0625 FLOP/byte -> deeply memory-bound
    // Data traversed twice: c->a then a->b
    // a[] evicted from cache between loops if N is large!
}

// FUSED: single loop, better arithmetic intensity
void fused(double* a, double* b, const double* c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = c[i] * 2.0;
        b[i] = std::sqrt(a[i]) + a[i];
    }
    // Total: 3 FLOP / 32 bytes = 0.094 FLOP/byte
    // But more importantly: a[i] is in register, NOT re-read from memory!
    // Effective: 3 FLOP / 24 bytes = 0.125 (c read once, a+b written once)
    // Data traversed once: much better cache behavior
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<double> a(N), b(N), c(N, 4.0);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 10; ++r) fn(a.data(), b.data(), c.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() / 10 << " ms\n";
    };

    bench(unfused, "Unfused (2 passes)");
    bench(fused,   "Fused (1 pass)    ");
    // Typical: fused is 30-50% faster (one memory pass vs two)
}
```

The key insight in the fused version is that `a[i]` lives in a register between the two computations. It is never written to memory and read back - the intermediate result flows directly from the first operation to the second. For large N where `a[]` has been evicted from cache by the time the second loop starts, the unfused version is essentially paying for two full memory passes.

---

## Notes

- **Ridge point** = Peak GFLOP/s divided by Peak bandwidth (GB/s). Kernels with arithmetic intensity below this point are memory-bound.
- Most real-world code is memory-bound (AI < 1). This means the most impactful optimizations are those that reduce data movement, not those that speed up arithmetic.
- Loop fusion, tiling, and AoS-to-SoA transforms all increase arithmetic intensity by reusing data that is already in cache.
- Intel tools: `likwid-perfctr` and `Intel Advisor` can generate roofline plots automatically from hardware counters.
- The STREAM benchmark (`STREAM Triad`) measures practical memory bandwidth - use it to calibrate your roofline model for a specific machine.
