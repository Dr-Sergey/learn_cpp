# Use cache-oblivious algorithms and tiling for memory-bound code

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/performance>  

---

## Topic Overview

Most performance-critical algorithms become memory-bound once the data stops fitting in L1 or L2 cache. The standard fix is tiling: instead of iterating over the full dataset in an order that causes cache thrashing, you restructure the loop nests so that a small "tile" of data is loaded into cache and reused as many times as possible before you move on.

Cache-aware tiling works well but requires you to know your cache sizes and tune the block size. Cache-oblivious algorithms take a different approach: recursive subdivision divides the problem in half until subproblems naturally fit in whatever cache level is available, without ever knowing its size. The insight is that recursive halving automatically hits all cache levels on the way down.

```cpp
Cache-oblivious approach:

  1. Divide problem in half (recursively)
  2. At some depth, subproblem fits in L1 -> all accesses hit L1
  3. At a higher depth, fits in L2 -> all accesses hit L2
  4. No tuning parameters needed!

  Problem size:  N -> N/2 -> N/4 -> ... -> BASE
  Fits in:       RAM   L3    L2         L1
```

| Approach | Cache parameters needed | Portability |
| --- | --- | --- |
| Naive loops | None (but terrible cache behavior) | Portable |
| Cache-aware tiling | Block size tuned to L1/L2 | Tuned per CPU |
| Cache-oblivious | None (recursive division) | Portable + efficient |

---

## Self-Assessment

### Q1: The cache-oblivious model explained

The formal basis for why recursive halving works without knowing cache sizes is called the "tall-cache assumption" - a reasonable assumption that the cache can hold at least as many cache lines as there are elements in one row of the working set. Under that assumption, optimal cache behavior emerges automatically from recursive subdivision.

```cpp
#include <iostream>

// Cache-oblivious algorithms work WITHOUT knowing cache sizes because:
//
// Principle: tall-cache assumption (M >= B^2, where M=cache, B=block size)
// Under this assumption, recursive halving hits the optimal cache behavior.
//
// Example: matrix multiply C += A * B (NxN)
//
// Cache-AWARE: needs block size parameter
//   for (ii=0; ii<N; ii+=BLOCK)        // BLOCK must match L1 size!
//     for (jj=0; jj<N; jj+=BLOCK)
//       for (kk=0; kk<N; kk+=BLOCK)
//         for (i=ii; i<ii+BLOCK; i++)
//           for (j=jj; j<jj+BLOCK; j++)
//             for (k=kk; k<kk+BLOCK; k++)
//               C[i][j] += A[i][k] * B[k][j];
//
// Cache-OBLIVIOUS: no block size parameter
//   multiply(A, B, C, n) {
//     if (n <= THRESHOLD) { base_case(); return; }
//     split into 8 sub-problems of size n/2
//     recursively multiply each
//   }
//   // At recursion depth d: subproblem size = N/2^d
//   // When N/2^d fits in L1 -> all accesses hit cache
//   // This happens automatically at the right depth!

int main() {
    std::cout << "Cache-oblivious model properties:\n";
    std::cout << "  1. No cache-size parameters needed\n";
    std::cout << "  2. Works across ALL cache levels simultaneously\n";
    std::cout << "  3. Recursive halving is the key technique\n";
    std::cout << "  4. Optimal for matrix operations: O(N^3 / (B*sqrt(M)))\n";
    std::cout << "  5. Portable: same code works on any CPU\n";
}
```

The cache-aware approach requires you to choose a block size that fits in L1 cache. If you tune it for your development machine and deploy to a different CPU with different cache sizes, the tuned block size may no longer be optimal. The cache-oblivious approach avoids this problem entirely.

### Q2: Cache-oblivious matrix multiplication

Matrix multiplication makes the cache behavior difference concrete because the naive triple loop has a famous access pattern: `A` is read row-by-row (cache-friendly), but `B` is read column-by-column (cache-hostile - each access to `B[k][j]` is N elements apart, causing a cache miss for large N). The recursive version reorders the computation so both matrices are accessed in smaller, cache-friendly blocks at each recursion depth.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

using Mat = std::vector<double>;
constexpr int BASE = 32; // Base case: 32x32 fits in L1

// Naive triple loop: terrible cache behavior for B
void naive_matmul(const Mat& A, const Mat& B, Mat& C, int N) {
    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            for (int k = 0; k < N; ++k)
                C[i*N+j] += A[i*N+k] * B[k*N+j]; // B stride = N
}

// Cache-oblivious recursive multiply
void co_matmul(const Mat& A, const Mat& B, Mat& C, int N,
               int ar, int ac, int br, int bc, int cr, int cc, int n) {
    if (n <= BASE) {
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                for (int k = 0; k < n; ++k)
                    C[(cr+i)*N+(cc+j)] += A[(ar+i)*N+(ac+k)] * B[(br+k)*N+(bc+j)];
        return;
    }
    int h = n / 2;
    // 8 recursive sub-multiplications (Strassen-like decomposition):
    // C_tl += A_tl * B_tl + A_tr * B_bl
    co_matmul(A,B,C,N, ar,ac,    br,bc,    cr,cc,    h);
    co_matmul(A,B,C,N, ar,ac+h,  br+h,bc,  cr,cc,    h);
    // C_tr += A_tl * B_tr + A_tr * B_br
    co_matmul(A,B,C,N, ar,ac,    br,bc+h,  cr,cc+h,  h);
    co_matmul(A,B,C,N, ar,ac+h,  br+h,bc+h,cr,cc+h,  h);
    // C_bl += A_bl * B_tl + A_br * B_bl
    co_matmul(A,B,C,N, ar+h,ac,  br,bc,    cr+h,cc,  h);
    co_matmul(A,B,C,N, ar+h,ac+h,br+h,bc,  cr+h,cc,  h);
    // C_br += A_bl * B_tr + A_br * B_br
    co_matmul(A,B,C,N, ar+h,ac,  br,bc+h,  cr+h,cc+h,h);
    co_matmul(A,B,C,N, ar+h,ac+h,br+h,bc+h,cr+h,cc+h,h);
}

int main() {
    const int N = 512;
    Mat A(N*N, 1.0), B(N*N, 1.0), C1(N*N, 0), C2(N*N, 0);

    auto bench = [](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench([&]{ naive_matmul(A, B, C1, N); },                        "Naive    ");
    bench([&]{ co_matmul(A, B, C2, N, 0,0,0,0,0,0, N); }, "Recursive");
    std::cout << "Match: " << (C1 == C2) << '\n';
    // N=512: Naive ~350ms, Recursive ~120ms (3x speedup)
}
```

A 3x speedup without any hardware-specific tuning is the appeal of the cache-oblivious approach. The `BASE = 32` constant is the only thing you might tune, and it is not sensitive - anything in the range 16-64 works well across typical CPUs.

### Q3: Measure cache miss rates with `perf`

Numbers from a benchmark only tell you that one version is faster. Hardware counters tell you *why* - how many cache misses each version generates. This example shows the code you'd build and measure with `perf stat`:

```cpp
#include <iostream>
#include <vector>

// This program is designed to be measured with perf:
//   perf stat -e L1-dcache-load-misses,LLC-load-misses ./test

using Mat = std::vector<double>;

void naive(const Mat& A, const Mat& B, Mat& C, int N) {
    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            for (int k = 0; k < N; ++k)
                C[i*N+j] += A[i*N+k] * B[k*N+j];
}

void tiled(const Mat& A, const Mat& B, Mat& C, int N, int tile) {
    for (int ii = 0; ii < N; ii += tile)
        for (int jj = 0; jj < N; jj += tile)
            for (int kk = 0; kk < N; kk += tile)
                for (int i = ii; i < ii+tile && i < N; ++i)
                    for (int j = jj; j < jj+tile && j < N; ++j)
                        for (int k = kk; k < kk+tile && k < N; ++k)
                            C[i*N+j] += A[i*N+k] * B[k*N+j];
}

int main() {
    const int N = 1024;
    Mat A(N*N, 1.0), B(N*N, 1.0), C(N*N, 0);

    // Build two versions:
    //   g++ -O2 -DNAIVE -o naive test.cpp
    //   g++ -O2 -DTILED -o tiled test.cpp
    //
    // Measure:
    //   perf stat -e L1-dcache-load-misses,LLC-load-misses ./naive
    //   perf stat -e L1-dcache-load-misses,LLC-load-misses ./tiled
    //
    // Expected results (N=1024):
    //   NAIVE:
    //     L1-dcache-load-misses:  2,000,000,000  (B column access misses L1)
    //     LLC-load-misses:          500,000,000
    //     Time: ~5 seconds
    //
    //   TILED (tile=64):
    //     L1-dcache-load-misses:    200,000,000  (10x fewer!)
    //     LLC-load-misses:            5,000,000  (100x fewer!)
    //     Time: ~1 second

#ifdef NAIVE
    naive(A, B, C, N);
#else
    tiled(A, B, C, N, 64);
#endif
    std::cout << "C[0][0] = " << C[0] << '\n';  // N (=1024)
}
```

A 100x reduction in LLC misses is what makes a 5x wall-time speedup possible. This is a useful mental model to carry: every LLC miss costs roughly 100 cycles; reducing their count is often worth more than any arithmetic optimization you could make.

---

## Notes

- Cache-oblivious = portable performance without tuning. Cache-aware = peak performance with tuning.
- Key cache parameters: L1 = 32KB, L2 = 256KB, L3 = 8-32MB, cache line = 64B.
- `perf stat -e L1-dcache-load-misses,LLC-load-misses` is the key measurement command.
- Real BLAS libraries (OpenBLAS, MKL) use multi-level cache-aware tiling with runtime CPU detection.
- For simple transformations (map, filter): sequential access is already cache-friendly; tiling only helps for random/strided access.
