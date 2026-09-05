# Understand multidimensional subscript operator[] (C++23) - Part 2

**Category:** Core Language Fundamentals  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/operator_member_access>  

---

## Topic Overview

This is a continuation of the multidimensional `operator[]` topic, focusing on practical implementation patterns, the pre-C++23 comma operator trap, and `std::mdspan` usage.

### Defining Multi-Index operator[] for a 2D Matrix

Here's a clean, reusable matrix class that shows the C++23 subscript in a real context, including a bounds-checked `.at()` alternative:

```cpp
#include <vector>
#include <stdexcept>

class Matrix {
    std::vector<double> data_;
    size_t rows_, cols_;
public:
    Matrix(size_t rows, size_t cols, double init = 0.0)
        : data_(rows * cols, init), rows_(rows), cols_(cols) {}

    // C++23 multidimensional operator[]
    double& operator[](size_t i, size_t j) {
        return data_[i * cols_ + j];
    }
    const double& operator[](size_t i, size_t j) const {
        return data_[i * cols_ + j];
    }

    // Bounds-checked version
    double& at(size_t i, size_t j) {
        if (i >= rows_ || j >= cols_)
            throw std::out_of_range("Matrix index out of range");
        return data_[i * cols_ + j];
    }

    size_t rows() const { return rows_; }
    size_t cols() const { return cols_; }
};
```

### The Pre-C++23 Comma Operator Trap

This is the trap that the feature was designed to close. Watch what the pre-C++23 compiler actually does with `arr[2, 5]`:

```cpp
int arr[10] = {0,1,2,3,4,5,6,7,8,9};

// Before C++23: comma operator!
int x = arr[2, 5];
// Equivalent to: int x = arr[(2, 5)] -> arr[5]
// The 2 is evaluated and DISCARDED; only 5 is used as the index.

// This was almost always a bug. C++20 deprecated it, C++23 repurposed it.
```

### std::mdspan: The Standard Multidimensional View

`std::mdspan` is the standard library type that benefits most from this feature. Notice how the multi-index syntax makes the loop body read exactly like math notation:

```cpp
#include <mdspan>
#include <array>
#include <iostream>

int main() {
    std::array<int, 12> flat = {1,2,3,4,5,6,7,8,9,10,11,12};

    // 3x4 view (row-major by default)
    std::mdspan m(flat.data(), 3, 4);

    // Natural multidimensional access
    for (size_t i = 0; i < m.extent(0); ++i) {
        for (size_t j = 0; j < m.extent(1); ++j)
            std::cout << m[i, j] << " ";
        std::cout << "\n";
    }
    // Output:
    // 1 2 3 4
    // 5 6 7 8
    // 9 10 11 12
}
```

---

## Self-Assessment

### Q1: Define operator[](size_t i, size_t j) for a 2D matrix class and use it as m[i,j]

The `Grid` class below provides both mutable and const overloads so you can use the same `m[i,j]` syntax whether you're reading or writing.

```cpp
#include <iostream>
#include <vector>

class Grid {
    std::vector<int> data_;
    size_t width_;
public:
    Grid(size_t h, size_t w, int fill = 0)
        : data_(h * w, fill), width_(w) {}

    int& operator[](size_t row, size_t col) {
        return data_[row * width_ + col];
    }
    int operator[](size_t row, size_t col) const {
        return data_[row * width_ + col];
    }
};

int main() {
    Grid g(3, 4);

    // Set values using m[i,j] syntax
    g[0, 0] = 1;
    g[1, 1] = 5;
    g[2, 3] = 9;

    // Read using m[i,j] syntax
    std::cout << "g[0,0] = " << g[0, 0] << "\n";  // 1
    std::cout << "g[1,1] = " << g[1, 1] << "\n";  // 5
    std::cout << "g[2,3] = " << g[2, 3] << "\n";  // 9

    // Also works with const reference
    const Grid& cg = g;
    std::cout << "const g[2,3] = " << cg[2, 3] << "\n";  // 9
}
```

**How this works:**

- `operator[](size_t row, size_t col)` is a C++23 feature - the compiler passes both `row` and `col` as separate arguments.
- `g[2, 3]` calls `g.operator[](2, 3)` - no comma operator involved.
- Both mutable and const overloads are provided for read/write and read-only access.

### Q2: Explain that before C++23 the comma in m[i,j] was the comma operator, not multi-indexing

**Answer:**

Before C++23, `operator[]` could only take a **single argument**. Writing `m[i, j]` invoked the **comma operator** inside the brackets:

1. Evaluate `i` (for side effects only)
2. Discard the result of `i`
3. Evaluate `j`
4. Pass `j` as the sole argument to `operator[]`

Step through this concrete example to see how far off the result lands from what you'd expect:

```cpp
// Pre-C++23 behavior:
int matrix[3][4] = {};
int val = matrix[0][2, 3];
// Step 1: Evaluate subscript of matrix[0] -> gets int*
// Step 2: Comma operator: (2, 3) -> evaluates 2, discards, result is 3
// Step 3: matrix[0][3] - index 3, NOT 2*cols+3
```

**Why this was deprecated:**

- Nearly 100% of `m[i, j]` usage intended multi-dimensional access, not the comma operator.
- The comma operator in `[]` was a constant source of subtle bugs.
- Deprecating it in C++20 and repurposing in C++23 was a safe, backward-compatible change because the old behavior was almost always unintentional.

### Q3: Show how std::mdspan uses multidimensional operator[] for multi-dimensional element access

`std::mdspan` supports 2D, 3D, and higher-dimensional views over the same flat buffer, and the layout policy is separate from the access syntax.

```cpp
#include <mdspan>
#include <vector>
#include <iostream>
#include <numeric>

int main() {
    // Create flat storage with values 0-23
    std::vector<int> storage(24);
    std::iota(storage.begin(), storage.end(), 0);

    // 2D view: 4x6 matrix
    std::mdspan mat2d(storage.data(), 4, 6);
    std::cout << "mat2d[2, 3] = " << mat2d[2, 3] << "\n";  // 2*6+3 = 15

    // 3D view: 2x3x4 tensor
    std::mdspan tensor(storage.data(), 2, 3, 4);
    std::cout << "tensor[1, 2, 3] = " << tensor[1, 2, 3] << "\n";  // 1*12+2*4+3 = 23

    // mdspan properties
    std::cout << "rank = " << mat2d.rank() << "\n";       // 2
    std::cout << "extent(0) = " << mat2d.extent(0) << "\n"; // 4
    std::cout << "extent(1) = " << mat2d.extent(1) << "\n"; // 6
    std::cout << "size = " << mat2d.size() << "\n";         // 24

    // mdspan is non-owning - changes to the view modify the underlying data
    mat2d[0, 0] = 999;
    std::cout << "storage[0] = " << storage[0] << "\n";  // 999

    // Column-major layout
    std::mdspan<int, std::dextents<size_t, 2>, std::layout_left> col(
        storage.data(), 4, 6);
    // col[2, 3] maps to offset 3*4+2 = 14 (column-major)
    std::cout << "col-major col[2,3] = " << col[2, 3] << "\n";
}
```

**How this works:**

- `std::mdspan` is a lightweight, non-owning, multidimensional view over contiguous memory.
- It uses C++23 multidimensional `operator[]` for element access.
- The layout policy determines how indices map to flat offsets:
  - `layout_right` (default): row-major (C-style)
  - `layout_left`: column-major (Fortran-style)
- `std::mdspan` has zero overhead - all index calculations are inline.

---

## Notes

- `std::mdspan` is C++23's answer to scientific computing's need for efficient multi-dimensional arrays.
- The `operator[]` overload can accept any number of arguments - not just 2. Works for 3D, 4D, etc.
- You can mix single-argument and multi-argument `operator[]` overloads in the same class using overload resolution.
- For an owning multi-dimensional container, `std::mdarray` is proposed for a future standard.
