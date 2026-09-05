# Use SWIG to generate bindings for multiple languages from one interface file

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://www.swig.org/>  

---

## Topic Overview

**SWIG (Simplified Wrapper and Interface Generator)** reads C/C++ header files and a `.i` interface file, then auto-generates binding code for Python, Java, C#, Ruby, Go, Lua, and 20+ other languages. One interface file, multiple language targets.

The key selling point is the "write once, wrap everywhere" story. Instead of hand-coding a Python extension module and a Java JNI layer separately, you describe your API once in a `.i` file and let SWIG generate all the wrapper code. This makes SWIG especially attractive when you need to expose a large C API to multiple languages, or when you're working on a legacy codebase where manually written bindings would be a maintenance nightmare.

### SWIG Workflow

Here is the high-level picture. Notice that the same `.i` file drives wrappers for every target language - SWIG runs once per target, producing a C++ wrapper file plus the target-language proxy code.

```cpp
                        ┌──────────────┐
mathlib.h ──────────────│              │──► mathlib_wrap.cxx (Python)
                        │    SWIG      │──► mathlib_wrap.cxx (Java)
mathlib.i ──────────────│  generator   │──► mathlib_wrap.cxx (C#)
(interface file)        │              │──► mathlib.py / MathLib.java / etc.
                        └──────────────┘
                               │
                        One .i file generates wrappers
                        for ALL target languages
```

### SWIG vs pybind11 vs nanobind

If you're trying to decide which tool to use, this table summarises the tradeoffs. The short version: SWIG when you need multiple languages or have a large C API; pybind11/nanobind when you're targeting Python only and want a more Pythonic result.

| Feature | SWIG | pybind11 | nanobind |
| --- | :---: | :---: | :---: |
| Multiple languages | **20+** | Python only | Python only |
| C++ class support | Good | **Excellent** | **Excellent** |
| Template support | Limited (%template) | **Full** | **Full** |
| Python-ness | Basic | **Pythonic** | **Pythonic** |
| C code support | **Excellent** | Good | Good |
| Maintenance effort | Low (auto-gen) | Medium | Medium |
| Binary size | Medium | Large | **Small** |
| Learning curve | Moderate | Easy | Easy |

---

## Self-Assessment

### Q1: Write a SWIG .i interface file for a C++ class and generate Python bindings

**Answer:**

The example starts with a normal C++ header and implementation - SWIG doesn't require you to modify your existing code at all. The magic is entirely in the `.i` file.

```cpp
// ═══════════ mathlib.h — C++ header ═══════════
#ifndef MATHLIB_H
#define MATHLIB_H

#include <vector>
#include <string>
#include <cmath>
#include <stdexcept>

class Matrix {
    std::vector<double> data_;
    int rows_, cols_;

public:
    Matrix(int rows, int cols);
    ~Matrix() = default;

    double get(int r, int c) const;
    void set(int r, int c, double val);
    int rows() const { return rows_; }
    int cols() const { return cols_; }

    Matrix multiply(const Matrix& other) const;
    double determinant() const;  // for 2x2

    std::string to_string() const;
};

// Free functions
double dot_product(const std::vector<double>& a, const std::vector<double>& b);
std::vector<double> linspace(double start, double end, int n);

#endif
```

```cpp
// ═══════════ mathlib.cpp — implementation ═══════════
#include "mathlib.h"
#include <sstream>
#include <numeric>

Matrix::Matrix(int rows, int cols) : data_(rows * cols, 0), rows_(rows), cols_(cols) {}

double Matrix::get(int r, int c) const {
    if (r < 0 || r >= rows_ || c < 0 || c >= cols_)
        throw std::out_of_range("Index out of bounds");
    return data_[r * cols_ + c];
}

void Matrix::set(int r, int c, double val) {
    if (r < 0 || r >= rows_ || c < 0 || c >= cols_)
        throw std::out_of_range("Index out of bounds");
    data_[r * cols_ + c] = val;
}

Matrix Matrix::multiply(const Matrix& other) const {
    if (cols_ != other.rows_) throw std::invalid_argument("Shape mismatch");
    Matrix result(rows_, other.cols_);
    for (int i = 0; i < rows_; ++i)
        for (int j = 0; j < other.cols_; ++j) {
            double sum = 0;
            for (int k = 0; k < cols_; ++k)
                sum += get(i, k) * other.get(k, j);
            result.set(i, j, sum);
        }
    return result;
}

double Matrix::determinant() const {
    if (rows_ != 2 || cols_ != 2) throw std::runtime_error("Only 2x2 supported");
    return get(0,0)*get(1,1) - get(0,1)*get(1,0);
}

std::string Matrix::to_string() const {
    std::ostringstream oss;
    for (int i = 0; i < rows_; ++i) {
        for (int j = 0; j < cols_; ++j)
            oss << get(i, j) << " ";
        oss << "\n";
    }
    return oss.str();
}

double dot_product(const std::vector<double>& a, const std::vector<double>& b) {
    return std::inner_product(a.begin(), a.end(), b.begin(), 0.0);
}

std::vector<double> linspace(double start, double end, int n) {
    std::vector<double> result(n);
    for (int i = 0; i < n; ++i)
        result[i] = start + (end - start) * i / (n - 1);
    return result;
}
```

The `.i` interface file is the heart of SWIG. The `%{...%}` block is code that gets copied verbatim into the generated wrapper (for includes). The `%include` directives bring in SWIG's STL support so that `std::vector` and `std::string` get translated correctly. The `%exception` block translates C++ exceptions into the target language's exception type. The `%extend` block adds Python-specific methods (`__str__`, `__repr__`) without touching the C++ source.

```swig
// ═══════════ mathlib.i — SWIG interface file ═══════════
%module mathlib

%{
// This block is copied verbatim into the generated wrapper
#include "mathlib.h"
%}

// Enable STL support
%include "std_string.i"
%include "std_vector.i"

// Instantiate templates SWIG needs to wrap
%template(DoubleVector) std::vector<double>;

// Enable exception translation
%include "exception.i"
%exception {
    try {
        $action
    } catch (const std::out_of_range& e) {
        SWIG_exception(SWIG_IndexError, e.what());
    } catch (const std::invalid_argument& e) {
        SWIG_exception(SWIG_ValueError, e.what());
    } catch (const std::exception& e) {
        SWIG_exception(SWIG_RuntimeError, e.what());
    }
}

// Add Python __str__ method
%extend Matrix {
    std::string __str__() {
        return $self->to_string();
    }
    std::string __repr__() {
        return "<Matrix " + std::to_string($self->rows()) + "x" +
               std::to_string($self->cols()) + ">";
    }
}

// Parse the header — SWIG generates wrappers for everything in it
%include "mathlib.h"
```

Running SWIG is a two-step process: first generate the wrapper code, then compile it. Notice that switching to Java bindings is just changing one flag - the same `.i` file works.

```bash
# Generate Python bindings:
swig -c++ -python mathlib.i
# Produces: mathlib_wrap.cxx, mathlib.py

# Compile:
g++ -shared -fPIC -o _mathlib.so \
    mathlib.cpp mathlib_wrap.cxx \
    -I/usr/include/python3.11 \
    $(python3-config --ldflags)

# Generate Java bindings (same .i file!):
swig -c++ -java mathlib.i
# Produces: mathlib_wrap.cxx, Matrix.java, mathlib.java, etc
```

The Python usage looks just like a native Python class. SWIG generates proxy objects that delegate to the C++ implementation transparently.

```python
# Python usage:
import mathlib

m = mathlib.Matrix(2, 2)
m.set(0, 0, 1); m.set(0, 1, 2)
m.set(1, 0, 3); m.set(1, 1, 4)
print(m)  # 1 2 \n 3 4
print(f"Det: {m.determinant()}")  # Det: -2.0

v = mathlib.linspace(0, 1, 5)
print(list(v))  # [0.0, 0.25, 0.5, 0.75, 1.0]
```

### Q2: Explain when SWIG is preferable to pybind11: multiple target languages, legacy codebase

**Answer:**

The decision usually comes down to two things: how many target languages you need, and how C++-heavy your API is. Here is the breakdown:

**Use SWIG when:**

| Scenario | Why SWIG wins |
| --- | --- |
| Need Python + Java + C# | One `.i` file -> all targets |
| Large C API (100+ functions) | Auto-wraps entire headers with `%include` |
| Legacy C codebase | SWIG handles pure C naturally |
| Minimal maintenance budget | Auto-generated - update `.h`, re-run SWIG |
| Embedded scripting (Lua, Tcl) | SWIG supports niche languages |

**Use pybind11/nanobind when:**

| Scenario | Why pybind11 wins |
| --- | --- |
| Python-only target | More Pythonic API |
| Heavy template use | SWIG templates need explicit `%template` |
| Virtual method override in Python | Trampoline classes are natural |
| NumPy integration | First-class `py::array_t` support |
| Modern C++ (concepts, ranges) | Better C++17/20 support |

**SWIG limitations** worth knowing up front:

- Every template instantiation you want to expose needs an explicit `%template` declaration in the `.i` file.
- STL containers need explicit includes (`%include "std_vector.i"`, etc.) - they're not automatic.
- Generated Python code looks "C-like" rather than Pythonic. No properties by default, for example.
- No direct NumPy buffer protocol support - you need custom typemaps for that.
- Complex C++ features (SFINAE, `constexpr`, concepts) may confuse the SWIG parser.

### Q3: Compare SWIG-generated bindings with manual pybind11 bindings for type safety

**Answer:**

The type safety difference between SWIG and pybind11 is significant and worth understanding before you choose between them. The core issue is that SWIG's type checking happens at runtime through proxy objects, while pybind11's type checking is backed by the C++ template system and happens at the Python call site.

| Aspect | SWIG | pybind11 |
| --- | --- | --- |
| **Type checking** | Runtime (proxy objects) | **Compile-time** (templates) |
| **Wrong types** | Silent conversion or crash | **TypeError with message** |
| **Overload resolution** | First-match (order-dependent) | **Best-match** |
| **Default args** | Supported | Supported + visible in help() |
| **Return by value** | Copy (usually) | Copy, move, or reference (configurable) |
| **Smart pointers** | `%shared_ptr` directive | **Automatic** (shared_ptr holder) |
| **Const correctness** | Mostly ignored | **Enforced** |

The concrete Python-side difference looks like this:

```python
# SWIG: type safety is weaker
import mathlib_swig
m = mathlib_swig.Matrix(2, 2)
m.set(0, 0, "hello")  # May silently pass or crash — no compile-time check
# SWIG relies on C-style conversions
```

```python
# pybind11: strict type checking
import mathlib_pb11
m = mathlib_pb11.Matrix(2, 2)
m.set(0, 0, "hello")  # TypeError: incompatible function arguments
# pybind11 checks types at call time using C++ template matching
```

Here is the underlying reason for the difference. SWIG generates C code and embeds type information in runtime proxy classes. pybind11 compiles the binding code as C++ and inherits the C++ type system directly, so type errors surface at the Python call site with clear messages rather than deep inside C code.

```cpp
SWIG approach:
  C++ header -> SWIG parser -> generated C code -> compiled
  Type info: embedded in proxy classes at runtime
  Safety: depends on SWIG typemaps (customizable but manual)

pybind11 approach:
  C++ binding code -> C++ compiler -> compiled
  Type info: C++ template type system (checked at compile time)
  Safety: inherits C++ type system directly
```

---

## Notes

- SWIG is best for "wrap an entire C library quickly" scenarios where you need multiple target languages.
- Use `%rename` to give Python-friendly names: `%rename(__getitem__) Matrix::get;`
- `%extend` adds methods that don't exist in C++ - useful for `__str__`, `__len__`, and other Python protocol methods.
- SWIG 4.1+ supports C++17; C++20 support is partial and some newer constructs may confuse the parser.
- For new projects targeting Python only, pybind11 or nanobind is strongly preferred over SWIG.
- SWIG-generated code can be large - the `-O` flag enables optimizations that reduce the wrapper size.
