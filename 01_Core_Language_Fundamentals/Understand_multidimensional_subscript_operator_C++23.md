# Understand multidimensional subscript operator[] (C++23)

**Category:** Core Language Fundamentals  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/operator_member_access>  

---

## Topic Overview

C++23 allows `operator[]` to accept **multiple arguments**, enabling natural multidimensional indexing syntax like `m[i, j]` instead of workarounds like `m(i, j)` or `m[i][j]`.

### Before C++23: Workarounds

Before C++23, there was no clean way to write `m[i, j]`. You had to pick one of these options - or accidentally write something that looked right but wasn't:

```cpp
// Approach 1: operator() - works but [] is more natural for indexing
double& Matrix::operator()(size_t r, size_t c);
m(2, 3) = 5.0;

// Approach 2: proxy row object - verbose and error-prone
auto row = m[2];    // returns a proxy/row view
row[3] = 5.0;       // or m[2][3]

// Approach 3: m[i,j] before C++23 - WRONG! Comma operator!
m[i, j];  // Evaluates i (discards), then indexes with j only!
```

### C++23: Multidimensional operator[]

C++23 finally makes `m[i, j]` mean what everyone always wanted it to mean - a call to `operator[]` with two arguments:

```cpp
class Matrix {
    std::vector<double> data_;
    size_t cols_;
public:
    Matrix(size_t rows, size_t cols) : data_(rows * cols), cols_(cols) {}

    // C++23: multiple parameters in operator[]
    double& operator[](size_t row, size_t col) {
        return data_[row * cols_ + col];
    }
    const double& operator[](size_t row, size_t col) const {
        return data_[row * cols_ + col];
    }
};

Matrix m(3, 4);
m[1, 2] = 42.0;   // Clean, natural syntax!
```

### Deprecation of Comma Expressions in []

The transition from comma operator to multi-argument indexing was deliberate and staged:

| Standard | `m[i, j]` means |
| --- | --- |
| C++20 and earlier | Comma operator: evaluate `i`, discard, index with `j` |
| C++20 | Deprecated the comma-expression usage |
| C++23 | Multi-argument `operator[]` - `i` and `j` are both arguments |

### Integration with std::mdspan

`std::mdspan` (C++23) is the standard library's multidimensional view type, and it uses this operator directly. Here's how that feels to use:

```cpp
#include <mdspan>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6};
    // 2x3 matrix view over the flat data
    std::mdspan m(data.data(), 2, 3);

    // Multidimensional subscript!
    std::cout << m[0, 0] << "\n";  // 1
    std::cout << m[1, 2] << "\n";  // 6

    // 3D mdspan example
    std::vector<int> cube(24);
    std::mdspan c(cube.data(), 2, 3, 4);
    c[1, 2, 3] = 99;  // natural 3D indexing
}
```

---

## Self-Assessment

### Q1: Implement operator[](size_t row, size_t col) for a Matrix class and verify it replaces (i,j) call syntax

Both `operator[]` and `operator()` are provided here so you can see how the C++23 subscript syntax maps onto the same underlying index calculation.

```cpp
#include <iostream>
#include <vector>
#include <cassert>

class Matrix {
    std::vector<double> data_;
    size_t rows_, cols_;
public:
    Matrix(size_t rows, size_t cols)
        : data_(rows * cols, 0.0), rows_(rows), cols_(cols) {}

    // C++23 multidimensional operator[]
    double& operator[](size_t r, size_t c) {
        return data_[r * cols_ + c];
    }
    const double& operator[](size_t r, size_t c) const {
        return data_[r * cols_ + c];
    }

    // Old-style operator() for comparison
    double& operator()(size_t r, size_t c) {
        return data_[r * cols_ + c];
    }

    size_t rows() const { return rows_; }
    size_t cols() const { return cols_; }
};

int main() {
    Matrix m(3, 4);

    // C++23 syntax - natural and clean
    m[0, 0] = 1.0;
    m[1, 2] = 42.0;
    m[2, 3] = 99.0;

    // Verify via operator()
    assert(m(0, 0) == 1.0);
    assert(m(1, 2) == 42.0);
    assert(m(2, 3) == 99.0);

    // Print matrix
    for (size_t r = 0; r < m.rows(); ++r) {
        for (size_t c = 0; c < m.cols(); ++c)
            std::cout << m[r, c] << "\t";
        std::cout << "\n";
    }
    // operator[] now fully replaces operator() for indexing
}
```

**How this works:**

- `operator[](size_t r, size_t c)` takes multiple arguments - this is the C++23 feature.
- `m[1, 2]` is syntactic sugar that calls `m.operator[](1, 2)`.
- This replaces the old `m(1, 2)` pattern - `[]` is the conventional indexing operator.

### Q2: Explain why C++23 deprecated comma expressions inside [] and how the new syntax differs

**Answer:**

Before C++23, writing `m[i, j]` invoked the **comma operator**: `i` was evaluated and discarded, then `m` was indexed with `j` alone. This was almost always a bug - the programmer intended multi-dimensional indexing.

**Timeline:**

1. **C++20:** The committee **deprecated** using the comma operator inside `operator[]`, producing a warning.
2. **C++23:** The comma syntax was **repurposed** - `m[i, j]` now calls `operator[](i, j)` with both arguments.

**Key differences:**

| Aspect | Old (comma operator) | New (multi-arg) |
| --- | --- | --- |
| Number of args to `operator[]` | 1 (only `j`) | 2 (`i` and `j`) |
| `i` is used? | No, discarded | Yes, passed as 1st arg |
| Intent match | Almost never correct | Matches programmer intent |
| Backward compat | Code relying on comma in `[]` breaks | Intentional - old use was almost always a bug |

### Q3: Show how multidimensional subscript integrates with std::mdspan's custom accessor

`std::mdspan` handles the index-to-offset math for you, and the layout policy determines whether that math is row-major or column-major - the call syntax stays the same either way.

```cpp
#include <mdspan>
#include <vector>
#include <iostream>
#include <cassert>

int main() {
    // Flat storage
    std::vector<double> storage(12, 0.0);

    // Create a 3x4 mdspan (2D view over flat data)
    std::mdspan mat(storage.data(), 3, 4);

    // Use multidimensional operator[] to write
    mat[0, 0] = 1.0;
    mat[1, 2] = 5.5;
    mat[2, 3] = 9.9;

    // Verify
    assert(mat[0, 0] == 1.0);
    assert(mat[1, 2] == 5.5);
    assert(mat[2, 3] == 9.9);

    // mdspan provides extent information
    std::cout << "Rows: " << mat.extent(0) << "\n";  // 3
    std::cout << "Cols: " << mat.extent(1) << "\n";  // 4

    // 3D mdspan
    std::vector<int> cube(2 * 3 * 4, 0);
    std::mdspan vol(cube.data(), 2, 3, 4);
    vol[1, 2, 3] = 42;
    std::cout << "vol[1,2,3] = " << vol[1, 2, 3] << "\n";  // 42

    // mdspan with custom layout (column-major)
    std::mdspan<double, std::dextents<size_t, 2>, std::layout_left> col_mat(
        storage.data(), 3, 4);
    col_mat[1, 0] = 7.7;  // same operator[] syntax regardless of layout
    std::cout << "col_mat[1,0] = " << col_mat[1, 0] << "\n";
}
```

**How this works:**

- `std::mdspan` is a non-owning multidimensional view over contiguous data.
- It uses C++23 `operator[]` with multiple indices for natural element access.
- The layout policy (`layout_right` default = row-major, `layout_left` = column-major) changes the mapping from indices to flat offset, but the `operator[]` syntax stays the same.
- Custom accessors can transform element access (e.g., atomic access, bounds checking) transparently.

---

## Notes

- Compile with `-std=c++23` or `-std=c++2b` (GCC 13+, Clang 16+, MSVC 19.34+) to use multidimensional `operator[]`.
- You can combine single-dimension and multi-dimension overloads of `operator[]` in the same class.
- `std::mdspan` is a zero-overhead abstraction - no runtime cost vs manual index math.
- For bounds-checked access, write your own `operator[]` wrapper or wait for `std::mdarray` (proposed owning multi-dimensional array).
