# Set up integration testing of C++ code with Python using pybind11

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://pybind11.readthedocs.io/>  

---

## Topic Overview

This file focuses on **embedding Python inside a C++ test harness** and deciding **when Python integration tests complement C++ unit tests**. (See also file #767 for writing pybind11 bindings and testing from the pytest side.)

The key thing to understand upfront is that there are two directions you can go. You can call C++ from Python (build an extension module, import it in pytest), or you can call Python from C++ (embed the Python interpreter inside your C++ test executable). Each direction has different use cases, and knowing which one fits your situation saves a lot of confusion.

### Testing Architecture: Two Directions

```cpp
Direction A: C++ -> Python (embedded interpreter)
+----------------------+
|  C++ test main()     |
|  py::scoped_interp.  |---> Runs Python test scripts
|  py::exec("...")     |     from inside C++ executable
+----------------------+

Direction B: Python -> C++ (extension module)
+----------------------+
|  pytest               |
|  import my_module    |---> Calls compiled C++ code
|  assert ...          |     through pybind11 bindings
+----------------------+
```

### When to Use Each Test Strategy

If you're not sure which direction to pick for a given scenario, this table should help. The general principle is: use C++ tests for things that need the C++ toolchain (memory safety, template behavior, RAII), and use Python tests for things that benefit from Python's ecosystem (data pipelines, NumPy interop, expressive test tables).

| Scenario                                | C++ Unit Tests | Python Integration Tests |
| --- | :---: | :---: |
| Algorithm correctness                   | Yes | |
| Memory safety / UB detection            | Yes | |
| End-to-end workflow validation          | | Yes |
| Data pipeline with NumPy/Pandas        | | Yes |
| Cross-language serialization            | | Yes |
| Performance-critical hot path           | Yes | |
| Rapid prototyping of test scenarios     | | Yes |
| CI smoke test of Python bindings        | | Yes |

---

## Self-Assessment

### Q1: Bind a C++ class with pybind11 and write a pytest test that calls it from Python

**Answer:**

This example walks through all three files needed to test a C++ class from Python: the class definition, the pybind11 bindings, and the pytest test. The binding layer uses lambda wrappers for `get` and `set` because those methods need to adapt the C++ reference-return style to a value-based Python interface.

```cpp
// matrix.hpp
#pragma once
#include <vector>
#include <stdexcept>
#include <sstream>

class Matrix {
    std::vector<double> data_;
    size_t rows_, cols_;
public:
    Matrix(size_t r, size_t c)
        : data_(r * c, 0.0), rows_(r), cols_(c) {}

    size_t rows() const { return rows_; }
    size_t cols() const { return cols_; }

    double& at(size_t r, size_t c) {
        if (r >= rows_ || c >= cols_)
            throw std::out_of_range("index out of bounds");
        return data_[r * cols_ + c];
    }

    double at(size_t r, size_t c) const {
        if (r >= rows_ || c >= cols_)
            throw std::out_of_range("index out of bounds");
        return data_[r * cols_ + c];
    }

    Matrix multiply(const Matrix& other) const {
        if (cols_ != other.rows_)
            throw std::invalid_argument("dimension mismatch");
        Matrix result(rows_, other.cols_);
        for (size_t i = 0; i < rows_; ++i)
            for (size_t j = 0; j < other.cols_; ++j)
                for (size_t k = 0; k < cols_; ++k)
                    result.at(i, j) += at(i, k) * other.at(k, j);
        return result;
    }

    std::string repr() const {
        std::ostringstream os;
        os << "Matrix(" << rows_ << "x" << cols_ << ")";
        return os.str();
    }
};
```

```cpp
// bindings.cpp
#include <pybind11/pybind11.h>
#include "matrix.hpp"

namespace py = pybind11;

PYBIND11_MODULE(matrix_ext, m) {
    py::class_<Matrix>(m, "Matrix")
        .def(py::init<size_t, size_t>(),
             py::arg("rows"), py::arg("cols"))
        .def("rows", &Matrix::rows)
        .def("cols", &Matrix::cols)
        .def("get", [](const Matrix& m, size_t r, size_t c) {
            return m.at(r, c);
        })
        .def("set", [](Matrix& m, size_t r, size_t c, double v) {
            m.at(r, c) = v;
        })
        .def("multiply", &Matrix::multiply)
        .def("__repr__", &Matrix::repr);
}
```

```python
# test_matrix.py - pytest tests
import matrix_ext
import pytest

class TestMatrix:
    def test_create(self):
        m = matrix_ext.Matrix(3, 4)
        assert m.rows() == 3
        assert m.cols() == 4

    def test_set_and_get(self):
        m = matrix_ext.Matrix(2, 2)
        m.set(0, 0, 1.0)
        m.set(0, 1, 2.0)
        m.set(1, 0, 3.0)
        m.set(1, 1, 4.0)
        assert m.get(0, 0) == pytest.approx(1.0)
        assert m.get(1, 1) == pytest.approx(4.0)

    def test_out_of_range(self):
        m = matrix_ext.Matrix(2, 2)
        with pytest.raises(IndexError):  # out_of_range -> IndexError
            m.get(5, 0)

    def test_multiply_identity(self):
        # 2x2 identity * any 2x2 = same matrix
        eye = matrix_ext.Matrix(2, 2)
        eye.set(0, 0, 1.0); eye.set(1, 1, 1.0)

        a = matrix_ext.Matrix(2, 2)
        a.set(0, 0, 5.0); a.set(0, 1, 6.0)
        a.set(1, 0, 7.0); a.set(1, 1, 8.0)

        result = eye.multiply(a)
        assert result.get(0, 0) == pytest.approx(5.0)
        assert result.get(1, 1) == pytest.approx(8.0)

    def test_dimension_mismatch(self):
        a = matrix_ext.Matrix(2, 3)
        b = matrix_ext.Matrix(2, 3)  # 3 cols * 2 rows: mismatch
        with pytest.raises(ValueError):
            a.multiply(b)

    def test_repr(self):
        m = matrix_ext.Matrix(3, 5)
        assert repr(m) == "Matrix(3x5)"
```

The `__repr__` binding makes the class work naturally with Python's `repr()` - a small touch that makes debugging much easier when a test fails and you need to see what the object contains.

### Q2: Use pybind11's embedded interpreter to run Python test scripts from a C++ test main

**Answer:**

This is Direction A from the overview: the C++ executable itself starts a Python interpreter and runs test code inside it. This is useful when you want to keep everything in a single binary - for example, when you can't easily install a Python package but need to test the Python-facing behavior.

The `PYBIND11_EMBEDDED_MODULE` macro defines a module that exists only inside the C++ binary - no `.so` file on disk. Python code running inside the embedded interpreter can import it with a regular `import calculator`.

<!-- compile: needs third-party library header `pybind11/embed.h` -->
```cpp
// embedded_test_runner.cpp
// Runs Python test scripts FROM WITHIN a C++ executable
// Useful when you can't easily pip-install the extension

#include <pybind11/embed.h>  // Must come before any other py includes
#include <pybind11/pybind11.h>
#include <iostream>
#include <string>
#include <vector>

namespace py = pybind11;

// Embed a module directly (no separate .so needed)
PYBIND11_EMBEDDED_MODULE(calculator, m) {
    m.def("add", [](int a, int b) { return a + b; });
    m.def("divide", [](double a, double b) -> double {
        if (b == 0.0)
            throw std::invalid_argument("division by zero");
        return a / b;
    });
    m.def("fibonacci", [](int n) -> std::vector<int> {
        if (n <= 0) return {};
        std::vector<int> fib = {0};
        if (n == 1) return fib;
        fib.push_back(1);
        for (int i = 2; i < n; ++i)
            fib.push_back(fib[i-1] + fib[i-2]);
        return fib;
    });
}

int main() {
    py::scoped_interpreter guard{};  // Start the Python interpreter

    try {
        // Method 1: Inline Python test
        py::exec(R"(
import calculator

# Test addition
assert calculator.add(2, 3) == 5
assert calculator.add(-1, 1) == 0
print("add tests passed")

# Test division
assert abs(calculator.divide(10, 3) - 3.3333) < 0.001
print("divide tests passed")

# Test division by zero
try:
    calculator.divide(1, 0)
    assert False, "should have raised"
except ValueError as e:
    assert "division by zero" in str(e)
print("exception propagation test passed")

# Test fibonacci
assert calculator.fibonacci(0) == []
assert calculator.fibonacci(1) == [0]
assert calculator.fibonacci(7) == [0, 1, 1, 2, 3, 5, 8]
print("fibonacci tests passed")
        )");

        // Method 2: Run an external pytest file
        py::module_ pytest = py::module_::import("pytest");
        int exit_code = pytest.attr("main")(
            py::make_tuple("-v", "--tb=short", "tests/")
        ).cast<int>();

        if (exit_code != 0) {
            std::cerr << "pytest failed with exit code " << exit_code << "\n";
            return 1;
        }

        std::cout << "All embedded tests passed!\n";
        return 0;

    } catch (const py::error_already_set& e) {
        std::cerr << "Python error: " << e.what() << "\n";
        return 1;
    }
}
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(embedded_test_runner)

find_package(pybind11 REQUIRED)

add_executable(run_tests embedded_test_runner.cpp)
target_link_libraries(run_tests PRIVATE pybind11::embed)
```

Method 2 (importing pytest and calling `pytest.attr("main")`) is particularly powerful: you can write your test files as ordinary pytest scripts but run them from your C++ test harness. This means you get pytest's full output formatting and `--tb=short` tracebacks even when launching from C++.

### Q3: Explain when Python-based integration tests complement unit tests in C++ projects

**Answer:**

Python integration tests are most valuable when C++ unit tests would require too much infrastructure to set up, or when you want to validate behavior against a trusted reference implementation. Here are the four main scenarios.

**1. End-to-end workflow validation:**

```python
# Test the entire pipeline: load -> process -> export
def test_full_pipeline():
    import data_processor as dp

    raw = dp.load_csv("test_data.csv")     # C++ CSV parser
    cleaned = dp.remove_outliers(raw, 3.0)  # C++ statistics
    result = dp.export_json(cleaned)         # C++ serialization

    import json
    parsed = json.loads(result)
    assert len(parsed["rows"]) > 0
    assert all(abs(r["z_score"]) < 3.0 for r in parsed["rows"])
```

**2. Testing interop with Python ecosystems (NumPy, Pandas):**

```python
import numpy as np

def test_numpy_interop():
    import linalg_ext as la

    a = np.array([[1, 2], [3, 4]], dtype=np.float64)
    b = np.array([[5, 6], [7, 8]], dtype=np.float64)
    result = la.matrix_multiply(a, b)

    expected = a @ b  # NumPy reference
    np.testing.assert_allclose(result, expected)
```

**3. Rapid test-case prototyping:**

```python
# Python makes it trivial to generate edge cases
import itertools

@pytest.mark.parametrize("args", itertools.product(
    [0, 1, -1, 2**31-1, -2**31],  # int32 boundaries
    repeat=2
))
def test_add_boundary(args):
    # Quick exploration - would be verbose in C++
    a, b = args
    result = calculator.add(a, b)
    assert result == a + b
```

**4. When to stay with C++ unit tests:**

- **Performance-sensitive paths** - Python overhead masks real timings
- **Memory safety** - Python can't detect UB; use ASan/MSan in C++
- **Template instantiation testing** - must be in C++ to test compile-time behavior
- **Destructor/RAII correctness** - Python GC hides lifetime bugs

**Decision rule:** Use C++ tests for *correctness of individual components*. Use Python tests for *integration across boundaries*, *data-driven exploration*, and *validation against reference implementations (NumPy, SciPy, etc.)*.

---

## Notes

- `pybind11::embed` links the Python interpreter into your C++ executable - no separate `.so` needed
- `PYBIND11_EMBEDDED_MODULE` defines an importable module within the same binary
- Embedded interpreter cost: ~50ms startup on modern hardware - acceptable for test suites
- Combine both directions: C++ unit tests (GTest) + Python integration tests (pytest) in the same CI pipeline
- Watch for GIL: multi-threaded C++ code tested from Python needs `py::gil_scoped_release`
