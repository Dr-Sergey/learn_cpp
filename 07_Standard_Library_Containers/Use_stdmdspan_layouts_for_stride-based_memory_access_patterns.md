# Use std::mdspan layouts for stride-based memory access patterns

**Category:** Standard Library Containers  
**Standard:** C++23, C++26 (mdspan)  
**Reference:** <https://en.cppreference.com/w/cpp/container/mdspan>  

---

## Topic Overview

`std::mdspan` (C++23) is a non-owning multidimensional view over contiguous memory. It generalizes `std::span` to multiple dimensions. **Layout policies** control how multi-dimensional indices map to linear memory offsets - enabling row-major, column-major, and custom stride-based access. Like `std::span`, it borrows memory from somewhere else; it never owns or allocates anything.

### What mdspan Does

The same one-dimensional block of memory can be interpreted as a matrix in many different ways. mdspan makes this explicit through layout policies:

```cpp
1D memory:  [0][1][2][3][4][5][6][7][8][9][10][11]

mdspan<int, extents<3,4>> with layout_right (row-major, C-style):
  [0][1][2][3]     Row 0
  [4][5][6][7]     Row 1
  [8][9][10][11]   Row 2

  m[i, j] -> memory[i * 4 + j]

mdspan<int, extents<3,4>> with layout_left (column-major, Fortran-style):
  [0][3][6][9]     (reading columns)
  [1][4][7][10]
  [2][5][8][11]

  m[i, j] -> memory[j * 3 + i]
```

### Layout Policies

| Layout | Index mapping | Use case |
| --- | --- | --- |
| `layout_right` | Row-major (last index varies fastest) | C/C++ arrays, image processing |
| `layout_left` | Column-major (first index varies fastest) | Fortran, MATLAB, BLAS/LAPACK |
| `layout_stride` | Custom strides per dimension | Submatrices, padded layouts, transpositions |

### Core API

Here is the minimal setup - wrap a flat buffer in an mdspan and access it with row/column indices:

```cpp
#include <mdspan>
#include <vector>
#include <iostream>
#include <numeric>

int main() {
    std::vector<int> data(12);
    std::iota(data.begin(), data.end(), 0);

    // 3x4 matrix, row-major (default)
    std::mdspan<int, std::extents<size_t, 3, 4>> mat(data.data());

    // Access with multi-dimensional indices
    std::cout << mat[0, 0] << "\n";  // 0
    std::cout << mat[1, 2] << "\n";  // 6  (1*4 + 2)
    std::cout << mat[2, 3] << "\n";  // 11 (2*4 + 3)

    // Query dimensions
    std::cout << "rows: " << mat.extent(0) << "\n";  // 3
    std::cout << "cols: " << mat.extent(1) << "\n";  // 4

    // Modify through the view
    mat[1, 1] = 99;
    std::cout << data[5] << "\n";  // 99

    return 0;
}
```

Assigning through `mat[1, 1]` directly modifies the underlying `data` vector - the mdspan is just a view, not a copy.

### Extents

```cpp
// Fixed extents (compile-time dimensions):
std::extents<size_t, 3, 4>

// Dynamic extents (runtime dimensions):
std::dextents<size_t, 2>  // 2D with runtime rows/cols

// Mixed:
std::extents<size_t, std::dynamic_extent, 4>  // dynamic rows, 4 cols
```

---

## Self-Assessment

### Q1: Create a column-major mdspan view over a row-major buffer and access elements correctly

The important thing to understand here is that the same buffer produces completely different element access when you change the layout policy. Row-major and column-major are not different orderings of the data - they are different interpretations of the same linear memory:

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <mdspan>

int main() {
    // === 12 elements: we'll view them as 3x4 matrix ===
    std::vector<int> data(12);
    std::iota(data.begin(), data.end(), 1);  // 1, 2, 3, ..., 12

    // === Row-major view (layout_right, default) ===
    std::mdspan<int, std::extents<size_t, 3, 4>, std::layout_right> row_major(data.data());

    std::cout << "Row-major (layout_right):\n";
    for (size_t i = 0; i < row_major.extent(0); ++i) {
        for (size_t j = 0; j < row_major.extent(1); ++j) {
            std::cout << row_major(i, j) << "\t";
        }
        std::cout << "\n";
    }
    // Output:
    // 1   2   3   4
    // 5   6   7   8
    // 9   10  11  12
    // m[i, j] = data[i*4 + j]

    // === Column-major view (layout_left) over SAME buffer ===
    std::mdspan<int, std::extents<size_t, 3, 4>, std::layout_left> col_major(data.data());

    std::cout << "\nColumn-major (layout_left) - same underlying data:\n";
    for (size_t i = 0; i < col_major.extent(0); ++i) {
        for (size_t j = 0; j < col_major.extent(1); ++j) {
            std::cout << col_major(i, j) << "\t";
        }
        std::cout << "\n";
    }
    // Output:
    // 1   4   7   10
    // 2   5   8   11
    // 3   6   9   12
    // m[i, j] = data[j*3 + i]

    // === Verify element access ===
    std::cout << "\nElement (1,2):\n";
    std::cout << "  row_major(1,2) = " << row_major(1, 2) << "\n";  // 7 (1*4+2+1=7)
    std::cout << "  col_major(1,2) = " << col_major(1, 2) << "\n";  // 8 (2*3+1+1=8)
    // Different values! Same buffer, different layout interpretation.

    // === Practical use: BLAS/Fortran interop ===
    // Fortran libraries expect column-major. C++ uses row-major.
    // With mdspan, you choose the layout that matches your data source:
    //
    // void call_blas_routine(float* data, int rows, int cols);
    // auto blas_view = mdspan<float, dextents<size_t,2>, layout_left>(data, rows, cols);

    return 0;
}
```

`layout_right` (default) is row-major: the last index varies fastest in memory. `m[i, j]` maps to `data[i * cols + j]`. `layout_left` is column-major: the first index varies fastest. `m[i, j]` maps to `data[j * rows + i]`. This is critical for interop with Fortran/BLAS/LAPACK (column-major) or image libraries (often row-major). Without mdspan you would need to manually compute the right offset every time - a common source of bugs.

### Q2: Implement a custom layout policy for tiled (blocked) matrix storage

Standard row-major and column-major layouts are optimal for row-wise or column-wise traversal, but neither is ideal for algorithms that access small square blocks (like matrix multiplication). A tiled layout groups each block contiguously in memory so that working on a tile generates no cache misses within the block:

```cpp
#include <iostream>
#include <vector>
#include <cassert>
#include <cstddef>

// === Tiled (blocked) layout ===
// Divides a matrix into TILE x TILE blocks stored contiguously.
// Better cache behavior for matrix operations that work on blocks.
//
// For a 4x4 matrix with TILE=2:
//
// Logical:        Memory layout (tiled):
// [ 0  1  2  3]   Block(0,0): [0, 1, 4, 5]
// [ 4  5  6  7]   Block(0,1): [2, 3, 6, 7]
// [ 8  9 10 11]   Block(1,0): [8, 9, 12, 13]
// [12 13 14 15]   Block(1,1): [10, 11, 14, 15]

template <size_t TILE>
class TiledLayout {
    size_t rows_, cols_;
public:
    TiledLayout(size_t rows, size_t cols) : rows_(rows), cols_(cols) {
        assert(rows % TILE == 0 && cols % TILE == 0);
    }

    // Map (i, j) to linear offset in tiled storage
    size_t operator()(size_t i, size_t j) const {
        size_t block_row = i / TILE;
        size_t block_col = j / TILE;
        size_t local_row = i % TILE;
        size_t local_col = j % TILE;

        size_t blocks_per_row = cols_ / TILE;
        size_t block_id = block_row * blocks_per_row + block_col;
        size_t local_offset = local_row * TILE + local_col;

        return block_id * (TILE * TILE) + local_offset;
    }

    size_t required_size() const { return rows_ * cols_; }
};

// Generic matrix using custom layout
template <typename T, typename Layout>
class MatrixView {
    T* data_;
    Layout layout_;
    size_t rows_, cols_;
public:
    MatrixView(T* data, size_t rows, size_t cols, Layout layout)
        : data_(data), layout_(layout), rows_(rows), cols_(cols) {}

    T& operator()(size_t i, size_t j) { return data_[layout_(i, j)]; }
    const T& operator()(size_t i, size_t j) const { return data_[layout_(i, j)]; }
    size_t rows() const { return rows_; }
    size_t cols() const { return cols_; }
};

int main() {
    constexpr size_t ROWS = 4, COLS = 4, TILE = 2;

    // Allocate flat storage
    TiledLayout<TILE> layout(ROWS, COLS);
    std::vector<int> storage(layout.required_size(), 0);

    // Create tiled view
    MatrixView mat(storage.data(), ROWS, COLS, layout);

    // Fill with logical (row-major) values
    int val = 0;
    for (size_t i = 0; i < ROWS; ++i)
        for (size_t j = 0; j < COLS; ++j)
            mat[i, j] = val++;

    // Print logical view
    std::cout << "Logical view:\n";
    for (size_t i = 0; i < ROWS; ++i) {
        for (size_t j = 0; j < COLS; ++j)
            std::cout << mat[i, j] << "\t";
        std::cout << "\n";
    }

    // Print physical memory layout
    std::cout << "\nPhysical memory (tiled):\n";
    size_t blocks = (ROWS / TILE) * (COLS / TILE);
    for (size_t b = 0; b < blocks; ++b) {
        std::cout << "Block " << b << ": ";
        for (size_t e = 0; e < TILE * TILE; ++e)
            std::cout << storage[b * TILE * TILE + e] << " ";
        std::cout << "\n";
    }
    // Output:
    // Block 0: 0 1 4 5     (top-left 2x2 block)
    // Block 1: 2 3 6 7     (top-right 2x2 block)
    // Block 2: 8 9 12 13   (bottom-left 2x2 block)
    // Block 3: 10 11 14 15 (bottom-right 2x2 block)

    // === Cache benefit ===
    // When processing block (0,0), elements [0,1,4,5] are contiguous in memory
    // -> fits in one cache line -> no cache misses within a block.
    // Row-major would scatter block elements across different cache lines.

    return 0;
}
```

The mapping formula for a tiled element is `block_id * TILE^2 + local_row * TILE + local_col`. A real `std::mdspan` custom layout would implement the `layout_mapping` concept with `operator()`, `required_span_size()`, `is_unique()`, and so on - this example shows the core idea without the boilerplate.

### Q3: Explain how strides work in mdspan and how they differ from padding

Strides are the general mechanism behind all mdspan layouts. Once you understand strides, row-major and column-major are just special cases. The reason padding trips people up is that it looks like a stride choice but has a different semantic meaning:

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <mdspan>
#include <array>

int main() {
    // === What are strides? ===
    // A stride for dimension d is the number of elements to skip in memory
    // when incrementing index d by 1.
    //
    // For a 3x4 row-major matrix (layout_right):
    //   stride[0] = 4  (skip 4 elements to go to next row)
    //   stride[1] = 1  (skip 1 element to go to next column)
    //   m[i, j] = data[i*4 + j*1] = data[i*stride[0] + j*stride[1]]

    std::vector<int> data(12);
    std::iota(data.begin(), data.end(), 0);

    // === layout_stride: custom strides ===
    // View as 3x4 with explicit strides
    std::array<size_t, 2> strides = {4, 1};  // row-major strides
    std::mdspan<int, std::extents<size_t, 3, 4>, std::layout_stride>
        mat(data.data(), std::layout_stride::mapping(
            std::extents<size_t, 3, 4>{}, strides));

    std::cout << "3x4 with strides {4,1} (row-major):\n";
    for (size_t i = 0; i < 3; ++i) {
        for (size_t j = 0; j < 4; ++j)
            std::cout << mat[i, j] << "\t";
        std::cout << "\n";
    }
    // Same as layout_right

    // === Stride for submatrix view ===
    // Take every other column from a 3x6 buffer:
    std::vector<int> big(18);
    std::iota(big.begin(), big.end(), 0);

    // View 3x3 submatrix: columns 0, 2, 4 of a 3x6 matrix
    std::array<size_t, 2> sub_strides = {6, 2};  // row stride=6 (full row width), col stride=2 (skip every other)
    std::mdspan<int, std::extents<size_t, 3, 3>, std::layout_stride>
        sub(big.data(), std::layout_stride::mapping(
            std::extents<size_t, 3, 3>{}, sub_strides));

    std::cout << "\nSubmatrix (every other column):\n";
    for (size_t i = 0; i < 3; ++i) {
        for (size_t j = 0; j < 3; ++j)
            std::cout << sub(i, j) << "\t";
        std::cout << "\n";
    }
    // Output:
    // 0   2   4
    // 6   8   10
    // 12  14  16

    // === Strides vs Padding ===
    std::cout << "\n=== Strides vs Padding ===\n";
    std::cout << "STRIDES:\n";
    std::cout << "  Define how many elements to skip per index increment.\n";
    std::cout << "  Strides change the MAPPING from indices to memory.\n";
    std::cout << "  Can create sparse views (e.g., every Nth element).\n";
    std::cout << "  Examples: submatrix, transposed view, column selection.\n\n";

    std::cout << "PADDING:\n";
    std::cout << "  Extra unused memory at the end of each row.\n";
    std::cout << "  Used for alignment (e.g., GPU requires 256-byte rows).\n";
    std::cout << "  Stride > logical_cols means there's padding.\n";
    std::cout << "  The 'padded' elements are not part of the logical matrix.\n\n";

    // Example: 3x4 matrix with padding to 8-element rows (for alignment)
    std::vector<int> padded_data(3 * 8, -1);  // 3 rows x 8 stride
    for (int i = 0; i < 3; ++i)
        for (int j = 0; j < 4; ++j)
            padded_data[i * 8 + j] = i * 4 + j;

    std::array<size_t, 2> padded_strides = {8, 1};  // row stride = 8 (not 4!)
    std::mdspan<int, std::extents<size_t, 3, 4>, std::layout_stride>
        padded_mat(padded_data.data(), std::layout_stride::mapping(
            std::extents<size_t, 3, 4>{}, padded_strides));

    std::cout << "Padded matrix (stride=8 for 4-wide rows):\n";
    for (size_t i = 0; i < 3; ++i) {
        for (size_t j = 0; j < 4; ++j)
            std::cout << padded_mat(i, j) << "\t";
        std::cout << "  [padding: ";
        for (int p = 4; p < 8; ++p)
            std::cout << padded_data[i * 8 + p] << " ";
        std::cout << "]\n";
    }

    // === summary ===
    // stride > extent -> padding (unused gap between rows)
    // stride < extent -> overlapping elements (unusual)
    // stride = extent -> standard dense layout

    return 0;
}
```

Strides and padding are related but different concepts. A stride defines the step size when you increment an index. Padding is what you get when the stride for a row is larger than the number of logical columns - the extra elements exist in memory but are outside the logical view. This situation is common with GPU and SIMD code that requires rows to be aligned to power-of-two byte boundaries. The key insight is that `layout_stride` handles both cases with the same mechanism: you just set your strides to match your physical memory layout, and mdspan takes care of the index arithmetic.

---

## Notes

- **`std::mdspan` is C++23**, not C++20 (despite early proposals). Check compiler support.
- **Zero-overhead:** mdspan is designed to compile to the same code as manual pointer arithmetic. No runtime cost.
- **`submdspan` (C++26):** Creates a subview of an mdspan - like array slicing in NumPy. Uses `std::submdspan(mat, pair{1,3}, pair{0,4})` for rows 1-2, all columns.
- **Not owning:** Like `span`, `mdspan` doesn't own memory. Pair with `vector`, `array`, or custom allocators.
- **GPU interop:** mdspan's layout policies match GPU memory layouts. NVIDIA's `std::experimental::mdspan` is widely used in CUDA code.
- **`std::mdarray` (proposed):** An owning version of mdspan - like `vector` is to `span`.
