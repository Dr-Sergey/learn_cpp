# Use std::linalg for linear algebra without external dependencies

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/linalg>  

---

## Topic Overview

This file focuses on **replacing Eigen/BLAS dependencies with `std::linalg`** and practical numerical patterns. (See also file #582 for core API overview and BLAS mapping.)

### Why std::linalg Over External Libraries

If you've ever wrestled with CMake's `find_package(Eigen3)` failing on a fresh machine, or hit an ABI mismatch between library versions, you already understand the pain this is solving. `std::linalg` ships with the standard library, which means no build system integration, no external download, and no version negotiation.

Here is the contrast at a glance:

```cpp
Before C++26:                           After C++26:
─────────────                           ───────────
#include <Eigen/Dense>                  #include <linalg>
// Requires: find_package(Eigen3)       // Requires: nothing (standard library)
// Build deps, ABI issues, version      // Portable, stable ABI, always available
// mismatches across platforms           // on conforming compilers
```

| Comparison                | Eigen            | std::linalg         |
| --- | --- | --- |
| **Header-only?**          | Yes              | Implementation-defined |
| **Build dependency**      | External package | None (standard)     |
| **BLAS backend**          | Optional         | Implementation may use |
| **Expression templates**  | Yes (complex)    | No (explicit calls) |
| **mdspan integration**    | No               | Native              |
| **Compile time**          | Very slow        | Moderate            |

The most important row for day-to-day work is "Build dependency." The trade-off is that Eigen's expression templates can fuse operations automatically; `std::linalg` requires explicit intermediate steps. For most code outside of high-performance hot paths, the standard library version is the right default.

---

## Self-Assessment

### Q1: Compute a matrix-vector product using std::linalg::matrix_vector_product with mdspan inputs

**Answer:**

All `std::linalg` functions work on `mdspan` views, not on new matrix types. You create a raw array, wrap it in an `mdspan` to give it shape, and pass that to the algorithm. Here's a concrete 3D rotation example to make the mechanics tangible:

<!-- compile: needs `linalg` (not in the CI standard library yet) -->
```cpp
#include <linalg>
#include <mdspan>
#include <array>
#include <iostream>

// Practical example: 3D transformation (translate a point)
int main() {
    // ═══════════ 3D rotation matrix (90 degrees around Z-axis) ═══════════
    // | 0 -1  0 |       | x |       | -y |
    // | 1  0  0 |   x   | y |   =   |  x |
    // | 0  0  1 |       | z |       |  z |

    std::array<double, 9> rot_data = {
        0, -1,  0,   // row 0
        1,  0,  0,   // row 1
        0,  0,  1    // row 2
    };
    std::mdspan rot(rot_data.data(), 3, 3);

    std::array<double, 3> point = {3.0, 4.0, 5.0};
    std::mdspan p(point.data(), 3);

    std::array<double, 3> result = {};
    std::mdspan r(result.data(), 3);

    // Compute: result = rot * point (no Eigen required!)
    std::linalg::matrix_vector_product(rot, p, r);

    std::cout << "Rotated point: ("
              << result[0] << ", "   // -4
              << result[1] << ", "   //  3
              << result[2] << ")\n"; //  5

    // ═══════════ Dot product (vector · vector) ═══════════
    std::array<double, 3> a = {1, 2, 3};
    std::array<double, 3> b = {4, 5, 6};
    std::mdspan va(a.data(), 3);
    std::mdspan vb(b.data(), 3);

    double dot = std::linalg::dot(va, vb);
    std::cout << "Dot product: " << dot << '\n';  // 1*4 + 2*5 + 3*6 = 32

    // ═══════════ Vector norm ═══════════
    double norm = std::linalg::vector_norm2(va);
    std::cout << "||a||2 = " << norm << '\n';  // sqrt(1+4+9) = 3.7416...
}
```

Notice that `matrix_vector_product`, `dot`, and `vector_norm2` all follow the same pattern: pass `mdspan` views in, get results back. No matrix class to construct, no temporary objects, just raw storage given a shape.

---

### Q2: Use std::linalg::scaled to apply a scalar factor without creating a temporary matrix

**Answer:**

`scaled()` is a lazy view - it does not allocate any storage. It wraps an existing `mdspan` and applies the factor element-by-element only when an algorithm actually reads from it. The same is true for `transposed()` and `conjugated()`. You can compose them freely and no temporaries appear until a consuming algorithm runs.

<!-- compile: needs `linalg` (not in the CI standard library yet) -->
```cpp
#include <linalg>
#include <mdspan>
#include <array>
#include <iostream>

// Practical example: physics simulation with timestep scaling
int main() {
    // Forces matrix (3 objects x 2 components: fx, fy)
    std::array<double, 6> forces = {
        10.0,  0.0,    // Object 0: 10N right
         0.0, -9.8,    // Object 1: gravity only
         5.0,  5.0     // Object 2: diagonal force
    };
    std::mdspan F(forces.data(), 3, 2);

    // Masses
    std::array<double, 3> mass = {1.0, 2.0, 0.5};

    // Compute acceleration for object i: a = F/m
    // Then compute velocity update: v += a * dt
    double dt = 0.016;  // 60 FPS timestep

    std::array<double, 2> velocity = {0.0, 0.0};  // Object 1's velocity
    std::mdspan v(velocity.data(), 2);

    // acceleration = force / mass = row 1 of F / mass[1]
    // velocity += acceleration * dt
    // = velocity += (1/mass * dt) * force_row

    // Use scaled() to avoid temporary: computes (dt/mass) * F[row] lazily
    double scale_factor = dt / mass[1];  // 0.016 / 2.0 = 0.008

    // Extract row 1 as a subspan (using submdspan)
    auto force_row = std::submdspan(F, 1, std::full_extent);
    // Apply: v += scaled(scale_factor, force_row)
    std::linalg::add(v, std::linalg::scaled(scale_factor, force_row), v);

    std::cout << "Object 1 velocity after 1 frame:\n";
    std::cout << "  vx = " << velocity[0] << '\n';  // 0 + 0.008 * 0 = 0
    std::cout << "  vy = " << velocity[1] << '\n';  // 0 + 0.008 * (-9.8) = -0.0784

    // ═══════════ scaled() + transposed() composition ═══════════
    // These are both lazy views - combining them still creates NO temporaries
    std::array<double, 4> mat = {1, 2, 3, 4};
    std::mdspan M(mat.data(), 2, 2);

    // 2 * M^T - zero allocation, computed on the fly
    auto view = std::linalg::scaled(2.0, std::linalg::transposed(M));
    // view[0,0] = 2*M[0,0] = 2, view[0,1] = 2*M[1,0] = 6
    // view[1,0] = 2*M[0,1] = 4, view[1,1] = 2*M[1,1] = 8
    std::cout << "2*M^T[0,1] = " << view[0, 1] << '\n';  // 6
}
```

The reason this matters in performance-sensitive code is that creating intermediate matrix copies is often the dominant cost. By composing lazy views and only materializing the result in the final write, you save both allocation and memory bandwidth.

---

### Q3: Explain how std::linalg maps to BLAS Level 1/2/3 routines for optimized execution

**Answer:**

The most important thing to understand here is that `std::linalg` specifies *behavior*, not *implementation*. A standard-library vendor is free to implement every function as a direct call to a native BLAS library like OpenBLAS or MKL. The same `matrix_product` call can be 100-500x faster on one implementation vs. another, depending entirely on the backend. Here's the mapping and why it matters:

```cpp
// std::linalg is designed as a portable BLAS interface.
// The standard specifies BEHAVIOR, not implementation - vendors are free
// to map these to highly optimized native BLAS.

// ═══════════ BLAS Level Classification ═══════════

// Level 1: Vector-Vector operations - O(n)
// ────────────────────────────────────────
// std::linalg::dot(x, y)          -> BLAS: ddot    (dot product)
// std::linalg::vector_norm2(x)    -> BLAS: dnrm2   (Euclidean norm)
// std::linalg::scale(alpha, x)    -> BLAS: dscal   (scale vector)
// std::linalg::add(x, y, z)       -> BLAS: daxpy   (z = x + y)
// std::linalg::copy(x, y)         -> BLAS: dcopy   (y = x)

// Level 2: Matrix-Vector operations - O(n^2)
// ────────────────────────────────────────────
// std::linalg::matrix_vector_product(A, x, y) -> BLAS: dgemv  (y = A*x)
// std::linalg::triangular_matrix_vector_solve  -> BLAS: dtrsv  (solve Ax=b)

// Level 3: Matrix-Matrix operations - O(n^3)
// ────────────────────────────────────────────
// std::linalg::matrix_product(A, B, C)         -> BLAS: dgemm  (C = A*B)
// std::linalg::triangular_matrix_product        -> BLAS: dtrmm

// ═══════════ Why native BLAS is faster ═══════════

/*
Generic C++:
  for i in rows:
    for j in cols:
      sum += A[i][k] * B[k][j]    -> cache-unfriendly, scalar

Optimized BLAS (e.g., OpenBLAS dgemm):

  - Block the matrices into tiles that fit in L1/L2 cache
  - Use SIMD (AVX-512): process 8 doubles per instruction
  - Unroll loops and prefetch memory
  - Multi-thread across cores

  -> 100-500x faster for large matrices

Benchmark (1000x1000 matrix multiply):
  Loop:      ~3000 ms
  OpenBLAS:  ~15 ms
  MKL:       ~8 ms
  Same std::linalg::matrix_product() call - backend determines speed
*/

// ═══════════ Execution policies for parallelism ═══════════

#include <execution>
#include <linalg>
#include <mdspan>
#include <array>

void parallel_example() {
    std::array<double, 100> a_data{}, b_data{}, c_data{};
    std::mdspan A(a_data.data(), 10, 10);
    std::mdspan B(b_data.data(), 10, 10);
    std::mdspan C(c_data.data(), 10, 10);

    // Sequential (default):
    std::linalg::matrix_product(A, B, C);

    // Parallel execution policy:
    std::linalg::matrix_product(std::execution::par, A, B, C);
    // Implementation may use:
    // - Multi-threaded BLAS backend
    // - OpenMP parallel loops
    // - std::execution::par threadpool
}
```

The reason to care about BLAS levels is performance characterization. Level 1 operations are memory-bandwidth bound; Level 2 and 3 operations can be compute-bound for large matrices. Knowing the level tells you whether throwing parallelism at the problem will help.

---

## Notes

- `std::linalg` eliminates Eigen/BLAS as build dependencies for standard linear algebra.
- All functions operate on `mdspan` views - no new matrix types or ownership semantics to learn.
- `scaled()`, `transposed()`, and `conjugated()` are lazy views - zero allocation, freely composable.
- For production numerics with dense matrices larger than roughly 100x100, check that your stdlib implementation maps to a real BLAS backend; a naive loop implementation will be much slower.
- For tiny matrices like 3x3 or 4x4 graphics transforms, hand-written or `constexpr` code may still beat the general-purpose routines.
