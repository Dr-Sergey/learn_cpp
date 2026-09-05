# Understand cache-oblivious algorithms and recursive tiling (Part 2: Transpose)

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Cache-oblivious_algorithm>  

---

## Topic Overview

Matrix transpose is a clean case study for cache-oblivious thinking because the performance problem is so visible. The naive approach writes to `B[j][i]` while reading from `A[i][j]`, which means the writes jump by a full row width between consecutive iterations of the inner loop. For a 2048-column matrix of doubles, each write jumps 16 KB - well past L1 cache. The result is a cache miss on nearly every store.

The recursive approach fixes this by splitting the matrix into quadrants and recursing. Eventually the submatrix is small enough that both the source and destination quadrants fit in L1 simultaneously, so reads and writes are cheap. No tuning, no parameters - the recursion finds the right granularity automatically:

```cpp
Naive transpose (stride N):           Recursive transpose:
for i: 0..N                           transpose(A, B, r, c, n) {
  for j: 0..N                           if n <= BASE:
    B[j][i] = A[i][j]                     naive_transpose(r, c, n)
     ^--- column access,                 else:
     stride = N * sizeof(T)                split into 4 quadrants
     -> cache miss per access!             recurse on each
                                        }
```

Here is how the three approaches stack up:

| Method | Cache misses (NxN) | Tuning needed? |
| --- | --- | --- |
| Naive | O(N²) | No |
| Cache-aware blocked | O(N²/B) | Yes (block size) |
| Cache-oblivious | O(N²/B) | No |

The cache-oblivious version matches the asymptotic cache complexity of the tuned blocked version, but without requiring you to know anything about the machine.

---

## Self-Assessment

### Q1: Cache-oblivious matrix transpose

The function signature looks busy because it passes the top-left corner of each submatrix by (row, col) offsets. The logic itself is simple: if the submatrix is at or below the base-case size, do it naively; otherwise split into four quadrants and recurse.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

using Matrix = std::vector<double>;
constexpr int BASE = 32;  // Base case: 32x32 fits in L1

// Naive transpose: O(N^2) cache misses
void naive_transpose(const Matrix& A, Matrix& B, int N) {
    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            B[j * N + i] = A[i * N + j];  // B[j][i] = A[i][j]
    // B column write: stride = N doubles = N*8 bytes
    // For N=2048: stride = 16KB -> misses L1 cache on every write!
}

// Cache-oblivious recursive transpose
void recursive_transpose(const Matrix& A, Matrix& B, int N,
                          int ar, int ac, int br, int bc, int n) {
    if (n <= BASE) {
        // Base case: submatrix fits in L1 cache
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                B[(bc + j) * N + (br + i)] = A[(ar + i) * N + (ac + j)];
        return;
    }
    int h = n / 2;
    // Recurse on 4 quadrants:
    recursive_transpose(A, B, N, ar,   ac,   br,   bc,   h);  // top-left
    recursive_transpose(A, B, N, ar,   ac+h, br+h, bc,   h);  // top-right -> bottom-left
    recursive_transpose(A, B, N, ar+h, ac,   br,   bc+h, h);  // bottom-left -> top-right
    recursive_transpose(A, B, N, ar+h, ac+h, br+h, bc+h, h);  // bottom-right
}

int main() {
    const int N = 2048;
    Matrix A(N * N), B1(N * N, 0), B2(N * N, 0);
    for (int i = 0; i < N * N; ++i) A[i] = i;

    auto t0 = std::chrono::high_resolution_clock::now();
    naive_transpose(A, B1, N);
    auto t1 = std::chrono::high_resolution_clock::now();
    recursive_transpose(A, B2, N, 0, 0, 0, 0, N);
    auto t2 = std::chrono::high_resolution_clock::now();

    auto ms1 = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    auto ms2 = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1);

    std::cout << "Naive:     " << ms1.count() << " ms\n";
    std::cout << "Recursive: " << ms2.count() << " ms\n";
    std::cout << "Correct:   " << (B1 == B2) << '\n';
    // N=2048: Naive ~25ms, Recursive ~8ms (3x faster)
}
```

That 3x speedup at N=2048 is typical. The bigger the matrix, the larger the naive cache miss penalty, and the bigger the gap.

### Q2: Why cache-oblivious works across all cache levels

The reason this trips people up is that it sounds like the algorithm must "know" when a subproblem fits in L1. It doesn't - and that is exactly the point. Because the recursion creates subproblems of every size from N down to BASE, there will always be a recursion level where the subproblem fits in L1, another where it fits in L2, and another where it fits in L3. The CPU's memory hierarchy automatically takes care of the rest.

```cpp
#include <iostream>

// Cache-oblivious algorithms don't need cache-size parameters because:
//
// 1. Recursive subdivision creates subproblems of ALL sizes:
//    N -> N/2 -> N/4 -> N/8 -> ... -> BASE
//
// 2. At some recursion level, subproblems fit in L1 (32KB).
//    At a higher level, they fit in L2 (256KB).
//    At an even higher level, they fit in L3 (8MB).
//
// 3. Once a subproblem fits in a cache level, ALL subsequent
//    operations on it hit that cache.
//
// Example for 4096x4096 matrix transpose (double, 8 bytes each):
//
// Recursion level    Subproblem size     Fits in?
// 0                  4096x4096 = 128MB    (nothing)
// 1                  2048x2048 = 32MB     (nothing)
// 2                  1024x1024 = 8MB      L3 (if 8MB+)
// 3                  512x512   = 2MB      L3
// 4                  256x256   = 512KB    L2 (if 1MB)
// 5                  128x128   = 128KB    L2
// 6                  64x64     = 32KB     L1!
// 7 (BASE=32)        32x32     = 8KB      L1
//
// Key: at level 6, the 64x64 block (32KB) fits entirely in L1.
// All 64*64 = 4096 reads AND writes hit L1 cache.
// No tuning needed - it automatically adapts.

// Cache-aware alternative: you must specify block size
void cache_aware_transpose(const double* A, double* B, int N, int block) {
    // block size must be tuned to L1 cache: block*block*8 <= 32KB
    // If L1 changes (different CPU), block must change too!
    for (int i = 0; i < N; i += block)
        for (int j = 0; j < N; j += block)
            for (int ii = i; ii < i + block && ii < N; ++ii)
                for (int jj = j; jj < j + block && jj < N; ++jj)
                    B[jj * N + ii] = A[ii * N + jj];
}

int main() {
    std::cout << "Cache hierarchy (typical Intel):\n";
    std::cout << "L1: 32KB  (4 cycles latency)\n";
    std::cout << "L2: 256KB (12 cycles latency)\n";
    std::cout << "L3: 8MB   (40 cycles latency)\n";
    std::cout << "RAM:       (200+ cycles latency)\n";
    std::cout << "\nCache-oblivious: no tuning needed.\n";
    std::cout << "Cache-aware blocked: must tune block size per CPU.\n";
}
```

The table in the comments is worth working through once. After level 6, every base-case operation in a 4096x4096 transpose fits entirely in L1. The algorithm "tuned itself" without knowing anything about the hardware.

### Q3: Cache-oblivious vs cache-aware blocked transpose

This benchmark shows the practical tradeoff. A cache-aware blocked transpose with the right block size is slightly faster on its target machine. But pick the wrong block size and it becomes slower than cache-oblivious. The cache-oblivious version just works everywhere.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

using Matrix = std::vector<double>;

void cache_aware_transpose(const Matrix& A, Matrix& B, int N, int block) {
    for (int i = 0; i < N; i += block)
        for (int j = 0; j < N; j += block)
            for (int ii = i; ii < std::min(i + block, N); ++ii)
                for (int jj = j; jj < std::min(j + block, N); ++jj)
                    B[jj * N + ii] = A[ii * N + jj];
}

void cache_oblivious_transpose(const Matrix& A, Matrix& B, int N,
                                int ar, int ac, int br, int bc, int n) {
    if (n <= 32) {
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                B[(bc + j) * N + (br + i)] = A[(ar + i) * N + (ac + j)];
        return;
    }
    int h = n / 2;
    cache_oblivious_transpose(A, B, N, ar,   ac,   br,   bc,   h);
    cache_oblivious_transpose(A, B, N, ar,   ac+h, br+h, bc,   h);
    cache_oblivious_transpose(A, B, N, ar+h, ac,   br,   bc+h, h);
    cache_oblivious_transpose(A, B, N, ar+h, ac+h, br+h, bc+h, h);
}

int main() {
    const int N = 2048;
    Matrix A(N * N), B(N * N);
    for (int i = 0; i < N * N; ++i) A[i] = i;

    auto bench = [&](auto fn, const char* label) {
        std::fill(B.begin(), B.end(), 0);
        auto t0 = std::chrono::high_resolution_clock::now();
        fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench([&]{ cache_aware_transpose(A, B, N, 32); },  "Cache-aware (32) ");
    bench([&]{ cache_aware_transpose(A, B, N, 64); },  "Cache-aware (64) ");
    bench([&]{ cache_oblivious_transpose(A, B, N, 0,0,0,0, N); }, "Cache-oblivious  ");

    // Typical results (N=2048):
    // Cache-aware (32):  ~9 ms   (good if 32 matches L1)
    // Cache-aware (64):  ~8 ms   (slightly better: 64*64*8=32KB = L1)
    // Cache-oblivious:   ~8 ms   (competitive, no tuning!)
    //
    // Cache-aware with WRONG block size (e.g., 512):
    // Cache-aware (512): ~20 ms  (512*512*8=2MB > L1, bad!)
    //
    // Verdict: cache-oblivious is robust; cache-aware can be
    // slightly faster with perfect tuning but fragile.
}
```

Notice how cache-aware with a 512-element block degrades to 20 ms because the block no longer fits in L1. The cache-oblivious version sits comfortably in between and never requires that kind of babysitting.

---

## Notes

- Cache-oblivious is "set and forget" - no tuning for different CPUs.
- Cache-aware can be ~10% faster with the perfect block size, but requires profiling per platform.
- For production: use cache-oblivious for portability, cache-aware for peak performance on a known target.
- Both are better than naive by 3-10x for large matrices.
- Libraries like BLAS use cache-aware blocking with runtime CPU detection to get the best of both worlds.
