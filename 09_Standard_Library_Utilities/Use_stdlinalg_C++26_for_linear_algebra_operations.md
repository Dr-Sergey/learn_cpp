# Use std::linalg (C++26) for linear algebra operations

**Category:** Standard Library Utilities  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/linalg>  

---

## Topic Overview

C++26 introduces `<linalg>` (based on proposal P1673), bringing a standardized linear algebra library built on top of `std::mdspan`. Think of it as BLAS-level primitives - dot products, matrix-vector products, matrix-matrix products, triangular solves - expressed as free function templates that never allocate memory and work with any `mdspan`-compatible storage.

### Architecture

The layering is important to understand because it explains why the design works the way it does:

```text
┌──────────────────────────────────────────────┐
│              User code                       │
│   double data[6] = {1,2,3,4,5,6};          │
│   mdspan<double, extents<int,2,3>> A(data); │
├──────────────────────────────────────────────┤
│           std::linalg functions              │
│  matrix_vector_product, dot, scale, add,     │
│  matrix_product, triangular_solve …          │
├──────────────────────────────────────────────┤
│           std::mdspan (C++23)                │
│  Non-owning multi-dimensional view           │
│  Layout policies: row_major, column_major    │
├──────────────────────────────────────────────┤
│        Raw contiguous storage                │
│  std::vector, std::array, C array, new[]     │
└──────────────────────────────────────────────┘
```

### Key Functions in `<linalg>`

| Function | Description |
| --- | --- |
| `std::linalg::dot(x, y)` | Inner (dot) product of two vectors |
| `std::linalg::scale(alpha, x)` | In-place scalar multiplication x *= alpha |
| `std::linalg::add(x, y, z)` | z = x + y element-wise |
| `std::linalg::matrix_vector_product(A, x, y)` | y = A * x |
| `std::linalg::matrix_product(A, B, C)` | C = A * B |
| `std::linalg::triangular_matrix_vector_solve(A, tri, diag, b, x)` | Solve Ax = b for triangular A |
| `std::linalg::transposed(A)` | Returns a transposed view (no copy) |
| `std::linalg::conjugate_transposed(A)` | Conjugate-transpose view |
| `std::linalg::scaled(alpha, x)` | Returns a scaled read-only view |

### Key Design Principles

1. **Zero-copy via mdspan** - functions take `mdspan` parameters, never allocate, never own data
2. **Layout-agnostic** - works with `layout_left` (column-major, Fortran/BLAS), `layout_right` (row-major, C), or custom layouts
3. **Execution policies** - all functions accept an optional first `std::execution::par` (or similar) argument for parallel execution
4. **Accessor-based scaling** - `scaled()` and `conjugated()` return lazy views that avoid copying

### Core Example - Matrix-Vector Product

Here's the simplest possible demonstration: wrap flat arrays in `mdspan` views, then hand them to `matrix_vector_product`. Notice there is no allocation inside the function - all memory is caller-owned.

<!-- compile: needs `linalg` (not in the CI standard library yet) -->
```cpp
// NOTE: C++26 — requires a compiler that supports <linalg>.
// As of 2024, available in experimental form via reference implementations
// (e.g., the mdspan/linalg reference impl on GitHub).

#include <linalg>
#include <mdspan>
#include <vector>
#include <iostream>

int main() {
    // 2×3 matrix A (row-major by default)
    //  | 1 2 3 |
    //  | 4 5 6 |
    double a_data[] = {1, 2, 3, 4, 5, 6};
    std::mdspan<double, std::extents<int, 2, 3>> A(a_data);

    // 3-element vector x = {1, 1, 1}
    double x_data[] = {1.0, 1.0, 1.0};
    std::mdspan<double, std::extents<int, 3>> x(x_data);

    // Output vector y (2 elements)
    double y_data[2] = {};
    std::mdspan<double, std::extents<int, 2>> y(y_data);

    // y = A * x
    std::linalg::matrix_vector_product(A, x, y);

    for (int i = 0; i < 2; ++i)
        std::cout << "y[" << i << "] = " << y[i] << "\n";
    // Output:
    // y[0] = 6     (1+2+3)
    // y[1] = 15    (4+5+6)
}
```

### Dot Product and Scaling

`scaled()` is a good example of the lazy-view design. It returns an accessor-wrapped `mdspan` that applies the scale factor on each read without modifying the underlying data:

<!-- compile: needs `linalg` (not in the CI standard library yet) -->
```cpp
#include <linalg>
#include <mdspan>
#include <iostream>

int main() {
    double a[] = {1.0, 2.0, 3.0};
    double b[] = {4.0, 5.0, 6.0};
    std::mdspan<double, std::extents<int, 3>> va(a), vb(b);

    // dot product: 1*4 + 2*5 + 3*6 = 32
    double d = std::linalg::dot(va, vb);
    std::cout << "dot = " << d << "\n"; // 32

    // in-place scale: a *= 2
    std::linalg::scale(2.0, va);
    // a is now {2, 4, 6}

    // scaled view (lazy, no copy)
    auto half_b = std::linalg::scaled(0.5, vb);
    // half_b[0]==2, half_b[1]==2.5, half_b[2]==3
    std::cout << "scaled b[0] = " << half_b[0] << "\n"; // 2
}
```

---

## Self-Assessment

### Q1: Compute a matrix-vector product using std::linalg::matrix_vector_product

**Answer:**

The caller allocates all storage and wraps it in `mdspan` views. The function signature makes the sizes clear from the types alone - a 3×2 matrix times a 2-element vector produces a 3-element result:

<!-- compile: needs `linalg` (not in the CI standard library yet) -->
```cpp
#include <linalg>
#include <mdspan>
#include <iostream>

int main() {
    // Matrix 3×2 (row-major):
    //  | 1  2 |
    //  | 3  4 |
    //  | 5  6 |
    double m_data[] = {1, 2, 3, 4, 5, 6};
    std::mdspan<double, std::extents<int, 3, 2>> M(m_data);

    // Vector x = {10, 20}
    double x_data[] = {10.0, 20.0};
    std::mdspan<double, std::extents<int, 2>> x(x_data);

    // Result y (3 elements)
    double y_data[3] = {};
    std::mdspan<double, std::extents<int, 3>> y(y_data);

    std::linalg::matrix_vector_product(M, x, y);

    for (int i = 0; i < 3; ++i)
        std::cout << "y[" << i << "] = " << y[i] << "\n";
    // Output:
    // y[0] = 50    (1*10 + 2*20)
    // y[1] = 110   (3*10 + 4*20)
    // y[2] = 170   (5*10 + 6*20)
}
```

**Explanation:** `matrix_vector_product(A, x, y)` computes `y = A · x`. The matrix is represented as a rank-2 `mdspan`, the input and output vectors as rank-1 `mdspan`s. No memory is allocated inside the function - all storage is provided by the caller through `mdspan` views.

### Q2: Explain how std::linalg integrates with std::mdspan for matrix representation

`std::linalg` functions take `std::mdspan` objects as their matrix/vector parameters. This integration works at several levels:

| Aspect | Detail |
| --- | --- |
| **Non-owning** | `mdspan` is a view - `linalg` never allocates or frees memory |
| **Layout policies** | `layout_right` (row-major) or `layout_left` (column-major) are both accepted; custom layouts too |
| **Extents** | Static extents (`extents<int,3,3>`) enable compile-time size checks; dynamic extents (`dextents<int,2>`) allow runtime sizes |
| **Accessor policies** | `scaled()` and `conjugated()` return `mdspan` objects with custom accessor policies that perform arithmetic lazily |
| **Subviews** | `std::submdspan(A, pair{0,2}, full_extent)` creates sub-matrix views; these can be passed directly to `linalg` functions |

The example below shows a dynamic-extent matrix and the `transposed()` view. The transposition produces no copy - it's just a new `mdspan` with swapped layout strides:

<!-- compile: needs `linalg` (not in the CI standard library yet) -->
```cpp
#include <linalg>
#include <mdspan>
#include <vector>
#include <iostream>

int main() {
    std::vector<double> storage(9);
    // Dynamic-extent 3×3 matrix
    std::mdspan<double, std::dextents<int, 2>> A(storage.data(), 3, 3);

    // Fill identity matrix
    for (int i = 0; i < 3; ++i)
        for (int j = 0; j < 3; ++j)
            A[i, j] = (i == j) ? 1.0 : 0.0;

    // transposed() returns a view with swapped layout — no data copy
    auto At = std::linalg::transposed(A);
    // At[i,j] == A[j,i]

    std::cout << "A[0,1]=" << A[0,1] << "  At[0,1]=" << At[0,1] << "\n";
    // Both 0 for identity, but for non-identity they'd differ
}
```

The key insight: `linalg` is a vocabulary built entirely on `mdspan` views. If you can wrap your memory in an `mdspan`, `linalg` can operate on it - whether it's a `std::vector`, a `new[]` buffer, GPU shared memory, or a mmap'd file.

### Q3: Compare std::linalg with BLAS for portability and performance in small matrices

| Aspect | BLAS (C/Fortran) | std::linalg (C++26) |
| --- | --- | --- |
| **Language** | C / Fortran calling conventions | Pure C++ templates |
| **Header** | Vendor-specific headers (MKL, OpenBLAS, ATLAS...) | `#include <linalg>` - standard |
| **Matrix representation** | Raw pointer + leading dimension int | `std::mdspan` - type-safe, layout-aware |
| **Column-major assumption** | Yes (Fortran heritage) | No - works with any layout policy |
| **Small matrix perf** | BLAS call overhead can dominate for <16x16 | Templates inline fully - zero call overhead |
| **Parallel execution** | Library-internal threads (MKL, OpenBLAS) | Explicit `std::execution::par` policy |
| **Complex numbers** | Separate `z`/`c` prefixed routines | Same template, `std::complex<double>` |
| **Compile-time sizes** | None | Static extents enable unrolling/vectorization |
| **Link dependency** | Requires BLAS library at link time | Header-only / standard library |

**When to prefer std::linalg:**

- Small fixed-size matrices (2×2, 3×3, 4×4) - template inlining beats BLAS call overhead
- Portable code that must compile without external libraries
- Integration with other C++ stdlib features (ranges, execution policies)

**When to prefer BLAS:**

- Large matrices (1000×1000+) - vendor-tuned BLAS (MKL, cuBLAS) has years of optimization
- GPU offloading - cuBLAS/rocBLAS are industry standard
- Existing Fortran/C codebases

The API difference is dramatic at a glance:

```cpp
// Conceptual comparison: 3×3 matrix multiply

// BLAS approach (C interface):
// extern "C" void dgemm_(char*, char*, int*, int*, int*,
//                        double*, double*, int*, double*, int*,
//                        double*, double*, int*);
// dgemm_("N", "N", &n, &n, &n, &alpha, A, &lda, B, &ldb, &beta, C, &ldc);
// 13 parameters, column-major assumed, link -lblas

// std::linalg approach:
// std::linalg::matrix_product(A_mdspan, B_mdspan, C_mdspan);
// 3 parameters, type-safe, layout-flexible, no linking dependency
```

---

## Notes

- **Availability (2024):** `<linalg>` is part of C++26 but not yet available in major compilers' standard libraries. The reference implementation at [github.com/kokkos/stdBLAS](https://github.com/kokkos/stdBLAS) is usable today as a header-only library.
- **Conjugated views:** `std::linalg::conjugated(x)` returns an `mdspan` with an accessor that applies `conj()` on read - enables Hermitian operations without copying complex data.
- **In-place vs out-of-place:** Some operations like `scale()` work in-place; others like `matrix_vector_product(A, x, y)` write to a separate output. The standard guarantees no aliasing issues when `x` and `y` are distinct.
- **Overwriting vs updating:** Many functions have `_update` variants (e.g., conceptually `y = alpha*A*x + beta*y` via `matrix_vector_product` with execution policy + scaled views).
- **Integration with `<execution>`:** All `linalg` functions accept an execution policy as the first argument, enabling `std::execution::par` for CPU parallelism or custom policies for GPU dispatch.
- Compile with `-std=c++26` or use the reference implementation with `-std=c++23`.
