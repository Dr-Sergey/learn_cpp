# Use std::mdspan (C++23) for multi-dimensional array views

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/container/mdspan>  

---

## Topic Overview

`std::mdspan` (C++23) is a non-owning, multi-dimensional view over a contiguous range of elements - think of it as an N-dimensional generalization of `std::span`. You have a flat array (or `vector`, or any contiguous buffer), and `mdspan` gives it a shape, a layout policy, and a safe element-access API without touching the data itself. The big idea is that storage and shape are completely decoupled: the same buffer can be viewed as a 2×3 matrix, a 6-element vector, or a column-major 3×2 matrix, all through different `mdspan` objects pointing at identical memory.

### Template Parameters

```cpp
std::mdspan<ElementType, Extents, LayoutPolicy, AccessorPolicy>
```

| Parameter | Purpose | Default |
| --- | --- | --- |
| `ElementType` | Type of elements (e.g., `double`) | - (required) |
| `Extents` | Logical dimensions - mix of static and dynamic | - (required) |
| `LayoutPolicy` | Memory ordering | `layout_right` (row-major, C-style) |
| `AccessorPolicy` | How elements are read/written | `default_accessor<ElementType>` |

### Layout Policies

Understanding the layout policies is crucial, especially if you interface with BLAS or NumPy. Row-major and column-major both describe valid ways to flatten a 2D grid into a 1D array - they just differ in which axis is contiguous in memory:

```text
Row-major (layout_right, C/C++ default):
  Logical [i][j] -> data[i * cols + j]

  Memory: [0,0] [0,1] [0,2] | [1,0] [1,1] [1,2]
          --- row 0 -------   --- row 1 -------

Column-major (layout_left, Fortran/BLAS):
  Logical [i][j] -> data[j * rows + i]

  Memory: [0,0] [1,0] | [0,1] [1,1] | [0,2] [1,2]
          -- col 0 --   -- col 1 --   -- col 2 --
```

### Core Syntax - Creating and Using mdspan

The `[i, j]` multi-dimensional subscript is new in C++23. On C++20 with a reference implementation, use `(i, j)` instead:

```cpp
#include <mdspan>
#include <vector>
#include <iostream>

int main() {
    // === 2D mdspan over a flat vector ===
    std::vector<int> data = {1, 2, 3, 4, 5, 6};

    // 2 rows × 3 columns, row-major (default)
    std::mdspan<int, std::extents<int, 2, 3>> m(data.data());

    // Access with multidimensional subscript (C++23)
    std::cout << m[0, 0] << "\n"; // 1
    std::cout << m[1, 2] << "\n"; // 6

    // === Dynamic extents ===
    int rows = 2, cols = 3;
    std::mdspan<int, std::dextents<int, 2>> dyn(data.data(), rows, cols);
    std::cout << dyn[1, 1] << "\n"; // 5

    // === Mixed static/dynamic extents ===
    // 'dynamic_extent' rows, fixed 3 columns
    std::mdspan<int, std::extents<int, std::dynamic_extent, 3>> mixed(data.data(), 2);
    std::cout << mixed[0, 2] << "\n"; // 3

    // === 3D mdspan ===
    double cube_data[24]; // 2×3×4
    std::mdspan<double, std::extents<int, 2, 3, 4>> cube(cube_data);
    cube[1, 2, 3] = 42.0;
}
```

### Column-Major Layout

Here the same six integers are viewed through a `layout_left` (column-major) `mdspan`. The logical coordinates are the same, but the mapping to flat memory is different - so the printed values differ:

```cpp
#include <mdspan>
#include <iostream>

int main() {
    int data[] = {1, 2, 3, 4, 5, 6};

    // Column-major: memory reads down columns first
    std::mdspan<int, std::extents<int, 2, 3>, std::layout_left> m(data);

    // layout_left: data[j * rows + i]
    // m[0,0]=1, m[1,0]=2 | m[0,1]=3, m[1,1]=4 | m[0,2]=5, m[1,2]=6
    for (int i = 0; i < 2; ++i) {
        for (int j = 0; j < 3; ++j)
            std::cout << m[i, j] << " ";
        std::cout << "\n";
    }
    // Output:
    // 1 3 5
    // 2 4 6
}
```

---

## Self-Assessment

### Q1: Create a 2D mdspan view over a flat std::vector and access elements with [i,j]

**Answer:**

The key thing to notice is that `flat` never moves. The `mdspan` wraps a raw pointer and adds the row/column arithmetic on top - it's purely a view:

```cpp
#include <mdspan>
#include <vector>
#include <iostream>

int main() {
    // Flat storage for a 3×4 matrix
    std::vector<int> flat(12);
    for (int i = 0; i < 12; ++i) flat[i] = (i + 1) * 10;
    // flat = {10, 20, 30, 40, 50, 60, 70, 80, 90, 100, 110, 120}

    // Create 2D view: 3 rows × 4 columns, row-major
    std::mdspan<int, std::extents<int, 3, 4>> mat(flat.data());

    // Access elements using multidimensional subscript
    std::cout << "mat[0,0] = " << mat[0, 0] << "\n"; // 10
    std::cout << "mat[1,2] = " << mat[1, 2] << "\n"; // 70 (row 1, col 2 -> index 1*4+2=6 -> 70)
    std::cout << "mat[2,3] = " << mat[2, 3] << "\n"; // 120

    // Print entire matrix
    for (int i = 0; i < 3; ++i) {
        for (int j = 0; j < 4; ++j)
            std::cout << mat[i, j] << "\t";
        std::cout << "\n";
    }
    // Output:
    // 10  20  30  40
    // 50  60  70  80
    // 90  100 110 120
}
```

**Explanation:** `mdspan` wraps a raw pointer (`flat.data()`) with extents (3×4). The `[i, j]` syntax uses C++23 multidimensional subscript. With `layout_right` (default), `mat[i, j]` maps to `flat[i * 4 + j]`. The vector's data is never copied - `mdspan` is purely a view.

### Q2: Explain the difference between row-major (C) and column-major (Fortran) layout policies

The layout affects which traversal order is cache-friendly. Accessing memory out of sequential order forces the CPU to fetch cache lines that aren't needed yet, which can cause a dramatic slowdown on large matrices.

| Property | Row-major (`layout_right`) | Column-major (`layout_left`) |
| --- | --- | --- |
| Convention | C, C++, Python (NumPy default) | Fortran, MATLAB, BLAS |
| Contiguous along | Last index (columns within a row) | First index (rows within a column) |
| Index formula (2D) | `data[i * cols + j]` | `data[j * rows + i]` |
| Cache-friendly traversal | Inner loop over columns: `for(j)` | Inner loop over rows: `for(i)` |
| `mdspan` layout | `std::layout_right` (default) | `std::layout_left` |

The same six-element array viewed two different ways:

```cpp
#include <mdspan>
#include <iostream>

int main() {
    int data[] = {1, 2, 3, 4, 5, 6};

    // Same raw data, different interpretations:
    std::mdspan<int, std::extents<int, 2, 3>, std::layout_right> row_major(data);
    std::mdspan<int, std::extents<int, 2, 3>, std::layout_left>  col_major(data);

    std::cout << "Row-major [0,1] = " << row_major[0, 1] << "\n"; // 2
    std::cout << "Col-major [0,1] = " << col_major[0, 1] << "\n"; // 3

    // Row-major mapping: [0,1] -> 0*3+1 = index 1 -> data[1] = 2
    // Col-major mapping: [0,1] -> 1*2+0 = index 2 -> data[2] = 3

    // Cache-friendly iteration for row-major:
    for (int i = 0; i < 2; ++i)         // outer: rows
        for (int j = 0; j < 3; ++j)     // inner: columns (contiguous)
            std::cout << row_major[i, j] << " ";
    // 1 2 3 4 5 6 — sequential memory access

    std::cout << "\n";

    // Cache-friendly iteration for column-major:
    for (int j = 0; j < 3; ++j)         // outer: columns
        for (int i = 0; i < 2; ++i)     // inner: rows (contiguous)
            std::cout << col_major[i, j] << " ";
    // 1 2 3 4 5 6 — sequential memory access
}
```

**Key insight:** The layout determines which traversal order gives sequential memory access. Mismatching loop order with layout can cause cache thrashing in large matrices - potentially orders-of-magnitude slowdown.

### Q3: Show how mdspan enables zero-copy interoperability with BLAS/LAPACK style APIs

`mdspan::data_handle()` returns the raw pointer that C/Fortran APIs need. `extent(n)` and `stride(n)` give you the dimension metadata. Because you're using the same pointer the `mdspan` already holds, no data is copied at any point:

```cpp
#include <mdspan>
#include <vector>
#include <iostream>
#include <cstddef>

// Simulated BLAS-style function that expects:
//   raw pointer, rows, columns, leading dimension
void legacy_blas_gemv(const double* A, int rows, int cols, int lda,
                      const double* x, double* y) {
    // y = A * x  (column-major assumed by BLAS)
    for (int i = 0; i < rows; ++i) {
        y[i] = 0.0;
        for (int j = 0; j < cols; ++j)
            y[i] += A[j * lda + i];  // column-major access
        y[i] *= x[0]; // simplified
    }
}

// Modern C++ wrapper using mdspan
template <typename T, typename Extents>
void print_matrix(std::mdspan<T, Extents> m) {
    for (std::size_t i = 0; i < m.extent(0); ++i) {
        for (std::size_t j = 0; j < m.extent(1); ++j)
            std::cout << m[i, j] << "\t";
        std::cout << "\n";
    }
}

int main() {
    // Same underlying data used by both mdspan and BLAS
    std::vector<double> storage = {1, 2, 3, 4, 5, 6};

    // mdspan view: column-major to match BLAS convention
    std::mdspan<double, std::extents<int, 2, 3>, std::layout_left> A(storage.data());

    // Pass raw pointer to BLAS — zero copy
    // A.data_handle() returns the raw pointer
    // A.extent(0) = rows, A.extent(1) = cols
    // A.stride(1) = leading dimension for column-major
    double x[] = {1.0};
    double y[2] = {};
    legacy_blas_gemv(A.data_handle(),
                     static_cast<int>(A.extent(0)),
                     static_cast<int>(A.extent(1)),
                     static_cast<int>(A.stride(1)),
                     x, y);

    // mdspan and BLAS operate on the SAME memory
    std::cout << "Matrix (column-major via mdspan):\n";
    print_matrix(A);
    // Output:
    // 1  3  5
    // 2  4  6

    std::cout << "y = " << y[0] << ", " << y[1] << "\n";
}
```

**Explanation:** `mdspan::data_handle()` returns the raw pointer BLAS expects. `extent(n)` and `stride(n)` provide the dimension metadata. By using `layout_left` (column-major), the mdspan's logical layout already matches BLAS conventions - no transposition or copying needed. This is the core value proposition: the same memory block has a safe, typed, bounds-checkable C++ API (`mdspan`) and can be passed directly to legacy C/Fortran BLAS routines through its raw pointer.

---

## Notes

- **Compiler support (2024):** `mdspan` is available in GCC 14+, Clang 18+, MSVC 17.9+. Kokkos provides a standalone reference implementation at [github.com/kokkos/mdspan](https://github.com/kokkos/mdspan).
- **`submdspan()` (C++26):** Creates sub-views of mdspan - e.g., extract a row, column, or sub-matrix. Available in the reference implementation today.
- **`mdarray` (future):** An owning counterpart to `mdspan` - like `std::vector` is to `std::span`. Expected in C++26.
- **Performance:** `mdspan` has zero overhead over raw pointer arithmetic when extents are static. Dynamic extents add one integer per dynamic dimension.
- **Multidimensional subscript:** `m[i, j]` requires C++23. On C++20, use `m(i, j)` with the reference implementation (operator() overload).
- Compile with `-std=c++23 -Wall -Wextra`.
