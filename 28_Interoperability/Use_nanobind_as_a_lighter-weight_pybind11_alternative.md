# Use nanobind as a lighter-weight pybind11 alternative

**Category:** Interoperability  
**Standard:** C++17  
**Reference:** <https://nanobind.readthedocs.io/>  

---

## Topic Overview

This topic focuses on **nanobind's ndarray support for zero-copy tensor interop**, its smart-pointer ownership semantics, and practical porting from pybind11. nanobind is designed for performance-critical scientific computing where binary size and import speed matter.

The zero-copy aspect is particularly significant for numerical code. When you pass a NumPy array to a C++ function through pybind11, there's often a copy involved. With `nb::ndarray`, nanobind gives your C++ code direct access to the memory buffer that NumPy owns - no copy, no overhead, no synchronization needed. This is exactly what you want for image processing, signal processing, or machine learning pipelines where arrays can be megabytes or gigabytes in size.

### CMake Setup

Setting up nanobind with CMake is straightforward. The `nanobind_add_module` function handles all the details of building a Python extension with the right flags.

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.18)
project(mymodule)

find_package(Python 3.8 COMPONENTS Interpreter Development.Module REQUIRED)
find_package(nanobind CONFIG REQUIRED)

nanobind_add_module(mymodule src/bindings.cpp)
# Produces: mymodule.cpython-312-x86_64-linux-gnu.so
```

### Key API Differences from pybind11

If you're porting an existing pybind11 module, here is the complete mapping. Most of it is just a namespace change and a few method renames.

| pybind11 | nanobind | Notes |
| --- | --- | --- |
| `PYBIND11_MODULE(name, m)` | `NB_MODULE(name, m)` | Same structure |
| `py::class_<T>` | `nb::class_<T>` | Same API |
| `.def_readwrite(...)` | `.def_rw(...)` | Shorter name |
| `.def_readonly(...)` | `.def_ro(...)` | Shorter name |
| `py::array_t<T>` | `nb::ndarray<T>` | Zero-copy, multi-framework |
| `py::arg("name")` | `nb::arg("name")` | Same |
| `py::return_value_policy` | `nb::rv_policy` | Shorter name |

---

## Self-Assessment

### Q1: Port a simple pybind11 module to nanobind and compare compile time and binary size

**Answer:**

The example ports a `Signal` class that represents audio samples. The pybind11 original is shown first, then the nanobind port below it. Read them side by side - the structure is identical, only the names change.

```cpp
// ═══════════ pybind11 original: signal_pb11.cpp ═══════════
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <vector>
#include <cmath>
#include <numbers>

namespace py = pybind11;

struct Signal {
    std::vector<float> samples;
    int sample_rate;

    Signal(int rate, int duration_ms)
        : sample_rate(rate),
          samples(rate * duration_ms / 1000, 0.0f) {}

    void generate_sine(float freq) {
        for (size_t i = 0; i < samples.size(); ++i) {
            float t = static_cast<float>(i) / sample_rate;
            samples[i] = std::sin(2.0f * std::numbers::pi * freq * t);
        }
    }

    float rms() const {
        float sum = 0;
        for (float s : samples) sum += s * s;
        return std::sqrt(sum / samples.size());
    }
};

PYBIND11_MODULE(signal_pb11, m) {
    py::class_<Signal>(m, "Signal")
        .def(py::init<int, int>(), py::arg("rate"), py::arg("duration_ms"))
        .def("generate_sine", &Signal::generate_sine, py::arg("freq"))
        .def("rms", &Signal::rms)
        .def_readwrite("samples", &Signal::samples)
        .def_readonly("sample_rate", &Signal::sample_rate);
}
```

The nanobind port is almost a word-for-word translation. The C++ business logic (`Signal`, `generate_sine`, `rms`) is completely unchanged. Only the binding code at the bottom differs, and even that is mostly just substituting `nb::` for `py::` and `def_rw`/`def_ro` for `def_readwrite`/`def_readonly`. Also notice the STL include: nanobind uses `<nanobind/stl/vector.h>` rather than a blanket `<pybind11/stl.h>`, which is more explicit and faster to compile.

```cpp
// ═══════════ nanobind port: signal_nb.cpp ═══════════
#include <nanobind/nanobind.h>
#include <nanobind/stl/vector.h>
#include <vector>
#include <cmath>
#include <numbers>

namespace nb = nanobind;

struct Signal {
    std::vector<float> samples;
    int sample_rate;

    Signal(int rate, int duration_ms)
        : sample_rate(rate),
          samples(rate * duration_ms / 1000, 0.0f) {}

    void generate_sine(float freq) {
        for (size_t i = 0; i < samples.size(); ++i) {
            float t = static_cast<float>(i) / sample_rate;
            samples[i] = std::sin(2.0f * std::numbers::pi * freq * t);
        }
    }

    float rms() const {
        float sum = 0;
        for (float s : samples) sum += s * s;
        return std::sqrt(sum / samples.size());
    }
};

NB_MODULE(signal_nb, m) {
    nb::class_<Signal>(m, "Signal")
        .def(nb::init<int, int>(), nb::arg("rate"), nb::arg("duration_ms"))
        .def("generate_sine", &Signal::generate_sine, nb::arg("freq"))
        .def("rms", &Signal::rms)
        .def_rw("samples", &Signal::samples)           // def_rw not def_readwrite
        .def_ro("sample_rate", &Signal::sample_rate);   // def_ro not def_readonly
}

// Compile time: ~3s (pybind11: ~8s)
// Binary size:  ~60 KB (pybind11: ~250 KB)
```

The comment at the bottom is representative of what you can expect. For a module of this size, the compile time roughly triples in your favour, and the binary is about 4x smaller.

### Q2: Explain how nanobind's ownership model differs from pybind11 for smart pointer types

**Answer:**

This is the most important conceptual difference when migrating from pybind11, and it's also where nanobind gives you the most performance headroom. The mental model to carry around is: pybind11 defaults to shared ownership everywhere; nanobind defaults to unique ownership and makes you opt in to sharing.

```cpp
#include <nanobind/nanobind.h>
#include <memory>

namespace nb = nanobind;

// ═══════════ pybind11: shared_ptr by default ═══════════
// py::class_<Foo, std::shared_ptr<Foo>>(m, "Foo")
//   - Every Python wrapper holds a shared_ptr
//   - C++ and Python share ownership
//   - Overhead: control block + atomic ref count
//   - Works naturally with shared_ptr APIs

// ═══════════ nanobind: unique ownership by default ═══════════
// nb::class_<Foo>(m, "Foo")
//   - Python OWNS the C++ object directly
//   - No shared_ptr overhead
//   - C++ code that needs shared_ptr must explicitly opt in

struct Texture {
    int width, height;
    std::vector<uint8_t> data;
    Texture(int w, int h) : width(w), height(h), data(w * h * 4) {}
};

// ═══════════ Using shared_ptr with nanobind (opt-in) ═══════════
struct SharedTexture : nb::intrusive_base {
    // intrusive_base provides ref counting shared between Python and C++
    int width, height;
    SharedTexture(int w, int h) : width(w), height(h) {}
};

// Or use std::shared_ptr explicitly:
struct ManagedTexture {
    int width, height;
    ManagedTexture(int w, int h) : width(w), height(h) {}
};

NB_MODULE(texture_mod, m) {
    // Default: nanobind owns via unique_ptr-like semantics
    nb::class_<Texture>(m, "Texture")
        .def(nb::init<int, int>())
        .def_ro("width", &Texture::width);
    // Python del -> C++ destructor runs immediately

    // Intrusive refcount: shared between Python and C++
    nb::class_<SharedTexture>(m, "SharedTexture")
        .def(nb::init<int, int>());

    // Explicit shared_ptr holder
    nb::class_<ManagedTexture>(m, "ManagedTexture",
        nb::holder<std::shared_ptr<ManagedTexture>>())
        .def(nb::init<int, int>());
}

// Summary:
// pybind11: shared_ptr everywhere (safe but overhead)
// nanobind: unique ownership (fast, explicit sharing when needed)
```

The practical consequence: if you have C++ code that stores a `shared_ptr<Texture>` and you want Python to be able to hold the same object, you need to use either `nb::intrusive_base` or the explicit `nb::holder<std::shared_ptr<...>>()` option. The default unique-ownership path won't work for that case.

### Q3: Show nanobind's nb::ndarray for zero-copy NumPy interop with explicit memory ownership

**Answer:**

`nb::ndarray` is where nanobind really shines for scientific computing. The template parameters encode constraints that nanobind checks at binding time - if Python passes a non-contiguous array to a function that requires contiguous layout, nanobind raises a clean `TypeError` rather than silently misbehaving or segfaulting.

The in-place function below is the most powerful use case: Python passes a NumPy array, and C++ modifies it directly in the NumPy buffer. No copy, no synchronization - the NumPy array is updated when the function returns.

```cpp
#include <nanobind/nanobind.h>
#include <nanobind/ndarray.h>
#include <vector>
#include <cmath>

namespace nb = nanobind;

// ═══════════ Accept NumPy array, process in-place (zero-copy) ═══════════
void normalize_inplace(
    nb::ndarray<float, nb::shape<nb::any>, nb::c_contig, nb::device::cpu> arr)
{
    // Zero-copy: arr.data() points directly to NumPy's buffer
    float* data = arr.data();
    size_t n = arr.shape(0);

    // Find max
    float max_val = 0;
    for (size_t i = 0; i < n; ++i)
        max_val = std::max(max_val, std::abs(data[i]));

    // Normalize in-place (modifies NumPy array directly!)
    if (max_val > 0) {
        for (size_t i = 0; i < n; ++i)
            data[i] /= max_val;
    }
}

// ═══════════ Return a new NumPy array (with ownership transfer) ═══════════
nb::ndarray<nb::numpy, float, nb::shape<nb::any, nb::any>>
create_matrix(size_t rows, size_t cols) {
    // Allocate C++ buffer
    float* data = new float[rows * cols];
    for (size_t i = 0; i < rows * cols; ++i)
        data[i] = static_cast<float>(i);

    // Create ndarray that OWNS the data
    // When Python GC collects the array, the deleter runs
    size_t shape[2] = {rows, cols};
    nb::capsule deleter(data, [](void* p) noexcept {
        delete[] static_cast<float*>(p);
    });

    return nb::ndarray<nb::numpy, float, nb::shape<nb::any, nb::any>>(
        data, 2, shape, deleter);
}

// ═══════════ 2D array with bounds checking ═══════════
float sum_2d(
    nb::ndarray<float, nb::shape<nb::any, nb::any>, nb::c_contig> matrix)
{
    size_t rows = matrix.shape(0);
    size_t cols = matrix.shape(1);
    const float* data = matrix.data();

    float sum = 0;
    for (size_t i = 0; i < rows * cols; ++i)
        sum += data[i];
    return sum;
}

NB_MODULE(ndarray_demo, m) {
    m.def("normalize_inplace", &normalize_inplace,
          "Normalize array values to [-1, 1] range (modifies in-place)");
    m.def("create_matrix", &create_matrix,
          nb::arg("rows"), nb::arg("cols"),
          "Create a rows x cols matrix with sequential values");
    m.def("sum_2d", &sum_2d, "Sum all elements of a 2D array");
}
```

The `nb::capsule` in `create_matrix` is the ownership transfer mechanism. When nanobind returns the array to Python, it attaches the capsule as a "base" object. When Python's GC eventually collects the NumPy array, it also collects the capsule, which runs the lambda deleter - freeing the C++-allocated buffer. This is how you safely hand off C++-allocated memory to Python without leaking it.

```python
# Usage from Python:
import numpy as np
import ndarray_demo

# Zero-copy in-place operation
arr = np.array([1.0, -3.0, 2.0, -0.5], dtype=np.float32)
ndarray_demo.normalize_inplace(arr)
print(arr)  # [0.333, -1.0, 0.666, -0.166]  (modified in-place!)

# C++ creates and returns array
mat = ndarray_demo.create_matrix(3, 4)
print(mat.shape)  # (3, 4)
print(mat)        # [[0, 1, 2, 3], [4, 5, 6, 7], [8, 9, 10, 11]]

# 2D sum
print(ndarray_demo.sum_2d(mat))  # 66.0
```

Notice that `arr` is printed after `normalize_inplace` and the values have changed - the function modified the NumPy buffer in place. From Python's side, it looks like the function mutated the array, which is exactly the expected behavior.

---

## Notes

- `nb::ndarray` supports NumPy, PyTorch, TensorFlow, and JAX tensors with the same C++ code - the framework is detected at runtime from the array's type.
- Shape constraints like `nb::shape<3, nb::any>` are verified at binding time and raise a clean `TypeError` on mismatch rather than crashing inside native code.
- `nb::c_contig` requires C-contiguous (row-major) layout; use `nb::f_contig` for Fortran (column-major) order when needed.
- The `nb::capsule` deleter pattern ensures C++-allocated memory is freed when Python GCs the array - it's the safe alternative to assuming Python will call `free` or `delete`.
- nanobind's ndarray is more powerful than pybind11's `py::array_t` because it supports multiple tensor frameworks and gives stricter shape/layout guarantees.
- For GPU tensors, use `nb::device::cuda` instead of `nb::device::cpu` in the ndarray template parameters.
