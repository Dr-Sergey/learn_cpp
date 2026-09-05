# Integrate C++ testing with Python using pybind11 + pytest

**Category:** Testing and Verification  
**Standard:** C++17  
**Reference:** <https://pybind11.readthedocs.io>  

---

## Topic Overview

pybind11 lets you expose C++ functions and classes to Python with minimal boilerplate. Combined with **pytest**, this creates a powerful integration-testing workflow: write C++ logic, bind it, then test from Python where the testing ecosystem (parametrize, fixtures, assertions) is extremely expressive.

The nice part about this approach is that Python's testing tools are significantly more flexible than most C++ frameworks for certain kinds of tests. Things like parametrized test tables with hundreds of entries, property-based testing with Hypothesis, and comparison against reference implementations (NumPy, SciPy) all become trivial from the Python side. You get to keep your C++ logic at native speed while using Python's testing expressiveness to drive it.

### Workflow

The data flow goes in one direction: C++ source compiles to an extension module, which Python imports just like any other module.

```cpp
+---------------------------------------------+
|  C++ Library (.cpp / .hpp)                  |
|  - Core logic, algorithms, classes          |
+---------------+-----------------------------+
                | pybind11 bindings (PYBIND11_MODULE)
                v
+---------------------------------------------+
|  Python extension module (.pyd / .so)       |
|  import my_module                           |
+---------------+-----------------------------+
                | pytest discovers & runs tests
                v
+---------------------------------------------+
|  test_my_module.py                          |
|  - pytest assertions, parametrize, fixtures |
+---------------------------------------------+
```

### C++ -> Python Exception Mapping

One thing you want to be aware of before writing tests is how pybind11 maps C++ exceptions to Python ones. You need to know this when using `pytest.raises(...)` to test error conditions. The table below covers the standard mappings - custom exceptions can also be registered, as shown in Q2.

| C++ Exception            | Python Exception |
| --- | --- |
| `std::runtime_error`     | `RuntimeError`   |
| `std::invalid_argument`  | `ValueError`     |
| `std::out_of_range`      | `IndexError`     |
| `std::domain_error`      | `ValueError`     |
| `std::overflow_error`    | `OverflowError`  |
| `std::bad_alloc`         | `MemoryError`    |
| `py::error_already_set`  | (re-raised)      |

---

## Self-Assessment

### Q1: Write pybind11 bindings for a C++ library and test them from pytest

**Answer:**

The example spans three files: the C++ header with the actual logic, the bindings file that exposes it to Python, and the pytest test file. Pay attention to the `#include <pybind11/stl.h>` line in the bindings - that single include is what makes `std::vector<double>` automatically convert to and from Python lists.

```cpp
// math_lib.hpp
#pragma once
#include <cmath>
#include <stdexcept>
#include <vector>
#include <numeric>

namespace mathlib {

double safe_sqrt(double x) {
    if (x < 0.0) throw std::domain_error("sqrt of negative number");
    return std::sqrt(x);
}

double mean(const std::vector<double>& v) {
    if (v.empty()) throw std::invalid_argument("empty vector");
    return std::accumulate(v.begin(), v.end(), 0.0) / v.size();
}

double dot_product(const std::vector<double>& a,
                   const std::vector<double>& b) {
    if (a.size() != b.size())
        throw std::invalid_argument("vectors must be same length");
    return std::inner_product(a.begin(), a.end(), b.begin(), 0.0);
}

} // namespace mathlib
```

```cpp
// bindings.cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>        // auto-converts std::vector <-> list
#include "math_lib.hpp"

namespace py = pybind11;

PYBIND11_MODULE(math_ext, m) {
    m.doc() = "C++ math library exposed to Python";

    m.def("safe_sqrt", &mathlib::safe_sqrt,
          py::arg("x"),
          "Compute sqrt, throws on negative input");

    m.def("mean", &mathlib::mean,
          py::arg("values"),
          "Compute arithmetic mean of a vector");

    m.def("dot_product", &mathlib::dot_product,
          py::arg("a"), py::arg("b"),
          "Compute dot product of two vectors");
}
```

```python
# test_math_ext.py
import math_ext
import pytest
import math

def test_safe_sqrt_positive():
    assert math_ext.safe_sqrt(25.0) == pytest.approx(5.0)
    assert math_ext.safe_sqrt(0.0) == pytest.approx(0.0)

def test_safe_sqrt_negative_raises():
    with pytest.raises(ValueError):  # domain_error -> ValueError
        math_ext.safe_sqrt(-1.0)

def test_mean_basic():
    assert math_ext.mean([1.0, 2.0, 3.0]) == pytest.approx(2.0)

def test_mean_empty_raises():
    with pytest.raises(ValueError):  # invalid_argument -> ValueError
        math_ext.mean([])

def test_dot_product():
    assert math_ext.dot_product([1, 2, 3], [4, 5, 6]) == pytest.approx(32.0)

def test_dot_product_mismatched_raises():
    with pytest.raises(ValueError):
        math_ext.dot_product([1, 2], [3])
```

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(math_ext)

find_package(pybind11 REQUIRED)
pybind11_add_module(math_ext bindings.cpp)
```

The exception tests use `pytest.raises(ValueError)` because `std::domain_error` and `std::invalid_argument` both map to Python's `ValueError`. Check the mapping table above whenever you're testing exception behavior.

### Q2: Show that pybind11 propagates C++ exceptions as Python exceptions with correct messages

**Answer:**

Beyond the standard exception mappings, pybind11 also lets you register custom exception translators so that your own exception types become first-class Python exception classes. The example shows a custom `ApiError` that appears as `err_demo.ApiError` on the Python side.

```cpp
// err_demo.cpp - binding that throws various C++ exceptions
#include <pybind11/pybind11.h>
#include <stdexcept>

namespace py = pybind11;

void throw_runtime(const std::string& msg) {
    throw std::runtime_error(msg);
}

void throw_out_of_range(int idx, int size) {
    if (idx >= size)
        throw std::out_of_range(
            "index " + std::to_string(idx) +
            " out of range [0, " + std::to_string(size) + ")");
}

void throw_overflow() {
    throw std::overflow_error("integer overflow detected");
}

// Custom exception with pybind11 registration
struct ApiError : std::runtime_error {
    int code;
    ApiError(int c, const std::string& msg)
        : std::runtime_error(msg), code(c) {}
};

PYBIND11_MODULE(err_demo, m) {
    m.def("throw_runtime", &throw_runtime);
    m.def("throw_out_of_range", &throw_out_of_range);
    m.def("throw_overflow", &throw_overflow);

    // Register custom exception -> custom Python exception class
    static py::exception<ApiError> exc(m, "ApiError");
    py::register_exception_translator([](std::exception_ptr p) {
        try {
            if (p) std::rethrow_exception(p);
        } catch (const ApiError& e) {
            exc(e.what());
        }
    });

    m.def("throw_api_error", [](int code, const std::string& msg) {
        throw ApiError(code, msg);
    });
}
```

```python
# test_exception_propagation.py
import err_demo
import pytest

def test_runtime_error_message():
    with pytest.raises(RuntimeError, match="disk full"):
        err_demo.throw_runtime("disk full")

def test_out_of_range_becomes_index_error():
    with pytest.raises(IndexError, match=r"index 10 out of range \[0, 5\)"):
        err_demo.throw_out_of_range(10, 5)

def test_overflow_becomes_overflow_error():
    with pytest.raises(OverflowError, match="integer overflow"):
        err_demo.throw_overflow()

def test_custom_api_error():
    with pytest.raises(err_demo.ApiError, match="not found"):
        err_demo.throw_api_error(404, "not found")

def test_exception_message_preserved():
    """The exact C++ what() string arrives in Python."""
    try:
        err_demo.throw_runtime("exact message 💡")
    except RuntimeError as e:
        assert str(e) == "exact message 💡"
```

The last test is particularly useful to know: the full `what()` string from the C++ exception arrives intact in Python. That means you can use `pytest.raises(match=...)` with exact substrings from your C++ error messages, which makes tests much more expressive.

### Q3: Use pytest parametrize to run the same C++ function with many test inputs from Python

**Answer:**

This is one of the biggest advantages of testing from Python. What would be a large, verbose parametrized test table in C++ (or worse, a loop) becomes clean and readable with `@pytest.mark.parametrize`. The fixture-based approach at the end is especially useful when you want to test with inputs of varying size.

```python
# test_parametrized.py
import math_ext
import pytest
import math

# Basic parametrize: one argument
@pytest.mark.parametrize("x, expected", [
    (0.0,  0.0),
    (1.0,  1.0),
    (4.0,  2.0),
    (9.0,  3.0),
    (2.0,  math.sqrt(2)),
    (100.0, 10.0),
    (1e-10, math.sqrt(1e-10)),
])
def test_safe_sqrt_values(x, expected):
    assert math_ext.safe_sqrt(x) == pytest.approx(expected, rel=1e-12)


# Parametrize with IDs for readable output
@pytest.mark.parametrize("values, expected_mean", [
    pytest.param([1.0],         1.0,   id="single"),
    pytest.param([1.0, 3.0],    2.0,   id="two-elem"),
    pytest.param([0.0, 0.0],    0.0,   id="zeros"),
    pytest.param([-1.0, 1.0],   0.0,   id="cancellation"),
    pytest.param(list(range(1, 101)), 50.5, id="1-to-100"),
], ids=str)
def test_mean_parametrized(values, expected_mean):
    assert math_ext.mean([float(v) for v in values]) == pytest.approx(expected_mean)


# Parametrize error cases
@pytest.mark.parametrize("x", [-1.0, -100.0, -0.001, float('-inf')])
def test_safe_sqrt_negative_parametrized(x):
    with pytest.raises(ValueError):
        math_ext.safe_sqrt(x)


# Parametrize with multiple fixtures (cartesian product)
@pytest.mark.parametrize("a", [[1, 0, 0], [0, 1, 0], [0, 0, 1]])
@pytest.mark.parametrize("b", [[1, 0, 0], [0, 1, 0], [0, 0, 1]])
def test_dot_product_unit_vectors(a, b):
    """Dot product of unit vectors: 1 if same, 0 if different."""
    result = math_ext.dot_product(
        [float(x) for x in a],
        [float(x) for x in b]
    )
    expected = 1.0 if a == b else 0.0
    assert result == pytest.approx(expected)


# Fixture-based parametrize for complex setup
@pytest.fixture(params=[10, 100, 1000, 10000])
def random_vector(request):
    import random
    random.seed(42)
    n = request.param
    return [random.gauss(0, 1) for _ in range(n)]

def test_mean_of_random_is_near_zero(random_vector):
    """Law of large numbers: mean -> 0 for N(0,1) samples."""
    result = math_ext.mean(random_vector)
    # Tolerance scales with 1/sqrt(n)
    n = len(random_vector)
    assert abs(result) < 5.0 / math.sqrt(n)
```

The double `@pytest.mark.parametrize` on `test_dot_product_unit_vectors` automatically generates the Cartesian product - all 9 combinations of unit vectors, with one line of setup. Doing that in C++ with Google Test would require either a triple-nested loop or a hand-built test table with 9 rows.

---

## Notes

- Build with `pip install .` (using a `setup.py` with `pybind11.setup_helpers`) or CMake + `pybind11_add_module`
- `pybind11/stl.h` auto-converts `std::vector`, `std::map`, `std::optional`, etc.
- `pytest.approx` is essential for floating-point comparisons - never use `==` directly on floats
- Run: `pytest -v test_math_ext.py` shows each parametrized case as a separate line
- CI pattern: build extension -> `pip install .` -> `pytest --tb=short`
