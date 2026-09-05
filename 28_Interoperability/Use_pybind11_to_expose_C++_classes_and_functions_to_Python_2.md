# Set up a pybind11 build with CMake and scikit-build

**Category:** Interoperability  
**Standard:** C++11  
**Reference:** <https://pybind11.readthedocs.io/>  

---

## Topic Overview

This second pybind11 topic focuses on **practical binding patterns**: class binding with constructors, methods, and properties; automatic C++ to Python exception translation; and the buffer protocol for zero-copy array sharing. The emphasis is on common pitfalls and the patterns you will actually use in real-world code.

### pybind11 Build Setup

Before you can use pybind11, you need to build your extension. Here are the two most common approaches: CMake (for larger projects) and a direct compiler invocation (for quick experiments or CI scripts).

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(mymodule)

find_package(pybind11 CONFIG REQUIRED)
# or: add_subdirectory(pybind11) if vendored

pybind11_add_module(mymodule src/bindings.cpp)
target_compile_features(mymodule PRIVATE cxx_std_17)
```

```bash
# Alternative: build with pip
pip install pybind11
c++ -O3 -shared -std=c++17 -fPIC \
    $(python3 -m pybind11 --includes) \
    bindings.cpp -o mymodule$(python3-config --extension-suffix)
```

---

## Self-Assessment

### Q1: Bind a C++ class with py::class_ including constructor, methods, and properties

**Answer:**

This example binds a `Histogram` class. Pay attention to how named arguments (`py::arg`) with default values make the Python API feel native - users get meaningful parameter names and can call `Histogram()` with no arguments to get sensible defaults. The `__repr__` and `__len__` bindings are just regular `.def()` calls using Python's dunder names, so `len(h)` and `repr(h)` work naturally.

```cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <string>
#include <vector>
#include <numeric>
#include <algorithm>

namespace py = pybind11;

// C++ class: Histogram
class Histogram {
    std::vector<int> bins_;
    double min_, max_;
    int num_bins_;

public:
    Histogram(double min_val, double max_val, int num_bins)
        : bins_(num_bins, 0), min_(min_val), max_(max_val), num_bins_(num_bins)
    {
        if (min_val >= max_val)
            throw std::invalid_argument("min must be less than max");
        if (num_bins <= 0)
            throw std::invalid_argument("num_bins must be positive");
    }

    void add(double value) {
        if (value < min_ || value >= max_) return;
        int idx = static_cast<int>((value - min_) / (max_ - min_) * num_bins_);
        idx = std::min(idx, num_bins_ - 1);
        bins_[idx]++;
    }

    void add_many(const std::vector<double>& values) {
        for (double v : values) add(v);
    }

    int total() const { return std::accumulate(bins_.begin(), bins_.end(), 0); }
    int max_count() const { return *std::max_element(bins_.begin(), bins_.end()); }
    const std::vector<int>& bins() const { return bins_; }
    double min_val() const { return min_; }
    double max_val() const { return max_; }

    void clear() { std::fill(bins_.begin(), bins_.end(), 0); }
};

PYBIND11_MODULE(histogram_mod, m) {
    py::class_<Histogram>(m, "Histogram",
        "A simple histogram for binning numerical data")
        // Constructor with named arguments and defaults
        .def(py::init<double, double, int>(),
             py::arg("min_val") = 0.0,
             py::arg("max_val") = 1.0,
             py::arg("num_bins") = 10)

        // Methods
        .def("add", &Histogram::add, py::arg("value"),
             "Add a single value to the histogram")
        .def("add_many", &Histogram::add_many, py::arg("values"),
             "Add multiple values at once")
        .def("clear", &Histogram::clear)

        // Read-only properties
        .def_property_readonly("total", &Histogram::total)
        .def_property_readonly("max_count", &Histogram::max_count)
        .def_property_readonly("bins", &Histogram::bins)
        .def_property_readonly("min_val", &Histogram::min_val)
        .def_property_readonly("max_val", &Histogram::max_val)

        // Special methods
        .def("__len__", &Histogram::total)
        .def("__repr__", [](const Histogram& h) {
            return "<Histogram [" + std::to_string(h.min_val()) + ", " +
                   std::to_string(h.max_val()) + ") " +
                   std::to_string(h.total()) + " values>";
        });
}
```

From the Python side, this feels like a properly designed Python class with a full docstring, default arguments visible in `help()`, and idiomatic property access.

```python
from histogram_mod import Histogram
import random

h = Histogram(0.0, 100.0, 10)
h.add_many([random.uniform(0, 100) for _ in range(1000)])
print(h)           # <Histogram [0.0, 100.0) 1000 values>
print(h.total)     # 1000
print(h.bins)      # [98, 102, 105, ...]
print(len(h))      # 1000
```

### Q2: Handle exceptions: translate a C++ exception to a Python RuntimeError automatically

**Answer:**

This is one of pybind11's most convenient features: you get exception translation for free. When a standard C++ exception propagates out of a bound function, pybind11 catches it at the boundary and raises the corresponding Python exception - no registration, no boilerplate on your part.

The table below shows exactly what maps to what. Notice that `std::invalid_argument` becomes `ValueError` (not `RuntimeError`), which is the Pythonically correct mapping since it signals a bad input.

```cpp
#include <pybind11/pybind11.h>
#include <stdexcept>
#include <fstream>
#include <string>

namespace py = pybind11;

// C++ functions that may throw
std::string read_config(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open())
        throw std::runtime_error("Cannot open file: " + path);

    std::string content((std::istreambuf_iterator<char>(file)),
                         std::istreambuf_iterator<char>());
    if (content.empty())
        throw std::runtime_error("File is empty: " + path);
    return content;
}

int parse_int(const std::string& s) {
    try {
        size_t pos;
        int result = std::stoi(s, &pos);
        if (pos != s.size())
            throw std::invalid_argument("Trailing characters");
        return result;
    } catch (const std::out_of_range&) {
        throw std::overflow_error("Integer too large: " + s);
    }
}

PYBIND11_MODULE(config_mod, m) {
    // Automatic translation table:
    // C++ exception              -> Python exception
    // std::runtime_error         -> RuntimeError
    // std::invalid_argument      -> ValueError
    // std::out_of_range          -> IndexError
    // std::overflow_error        -> OverflowError
    // std::domain_error          -> ValueError
    // std::length_error          -> ValueError
    // std::bad_alloc             -> MemoryError
    // std::bad_cast              -> RuntimeError
    // Any unhandled std::exception -> RuntimeError

    m.def("read_config", &read_config, py::arg("path"));
    m.def("parse_int", &parse_int, py::arg("s"));

    // No special registration needed - pybind11 does it automatically!
}
```

The Python code below receives the right exception types without any Python-side try/except gymnastics needed in the binding itself.

```python
from config_mod import read_config, parse_int

try:
    read_config("/nonexistent")
except RuntimeError as e:
    print(e)  # Cannot open file: /nonexistent

try:
    parse_int("123abc")
except ValueError as e:
    print(e)  # Trailing characters

try:
    parse_int("99999999999999999")
except OverflowError as e:
    print(e)  # Integer too large: 99999999999999999
```

### Q3: Use py::buffer_protocol to expose a C++ array as a NumPy array without copying

**Answer:**

The buffer protocol is the mechanism that lets a Python object claim to own a contiguous block of memory and expose it to NumPy. When you tag your class with `py::buffer_protocol()` and supply a `def_buffer` lambda, `np.asarray(obj)` reads the buffer descriptor you return - getting the data pointer, element type, dimensions, and strides - and builds a NumPy array that points directly at your C++ memory.

Getting the strides right is the most common mistake. Strides are in bytes, not elements. For a 3D image buffer with layout `[rows][cols][channels]`, the strides are `width * channels * sizeof(uint8_t)` for rows, `channels * sizeof(uint8_t)` for columns, and `sizeof(uint8_t)` for the channel dimension.

```cpp
#include <pybind11/pybind11.h>
#include <pybind11/numpy.h>
#include <vector>
#include <cmath>

namespace py = pybind11;

// Image class with buffer protocol
class Image {
    std::vector<uint8_t> pixels_;
    size_t width_, height_, channels_;

public:
    Image(size_t w, size_t h, size_t channels = 3)
        : pixels_(w * h * channels, 0), width_(w), height_(h), channels_(channels) {}

    uint8_t& pixel(size_t r, size_t c, size_t ch) {
        return pixels_[(r * width_ + c) * channels_ + ch];
    }

    void fill_gradient() {
        for (size_t r = 0; r < height_; ++r)
            for (size_t c = 0; c < width_; ++c) {
                pixel(r, c, 0) = static_cast<uint8_t>(255.0 * c / width_);   // R
                pixel(r, c, 1) = static_cast<uint8_t>(255.0 * r / height_);  // G
                pixel(r, c, 2) = 128;                                         // B
            }
    }

    size_t width() const { return width_; }
    size_t height() const { return height_; }
    size_t channels() const { return channels_; }
    uint8_t* data() { return pixels_.data(); }
};

PYBIND11_MODULE(image_mod, m) {
    py::class_<Image>(m, "Image", py::buffer_protocol())
        .def(py::init<size_t, size_t, size_t>(),
             py::arg("width"), py::arg("height"), py::arg("channels") = 3)
        .def("fill_gradient", &Image::fill_gradient)
        .def_property_readonly("width", &Image::width)
        .def_property_readonly("height", &Image::height)

        // Buffer protocol
        .def_buffer([](Image& img) -> py::buffer_info {
            return py::buffer_info(
                img.data(),                    // pointer to buffer
                sizeof(uint8_t),               // element size
                py::format_descriptor<uint8_t>::format(),
                3,                             // ndim
                {img.height(), img.width(), img.channels()},  // shape
                {img.width() * img.channels() * sizeof(uint8_t),  // row stride
                 img.channels() * sizeof(uint8_t),                 // col stride
                 sizeof(uint8_t)}                                  // channel stride
            );
        })

        // Construct from NumPy array
        .def_static("from_numpy", [](py::array_t<uint8_t> arr) {
            auto buf = arr.unchecked<3>();  // H x W x C
            Image img(buf.shape(1), buf.shape(0), buf.shape(2));
            for (py::ssize_t r = 0; r < buf.shape(0); ++r)
                for (py::ssize_t c = 0; c < buf.shape(1); ++c)
                    for (py::ssize_t ch = 0; ch < buf.shape(2); ++ch)
                        img.pixel(r, c, ch) = buf(r, c, ch);
            return img;
        });
}
```

Because both the C++ `Image` and the NumPy array point at the same memory, writes through either side are immediately visible on the other.

```python
import numpy as np
from image_mod import Image

# Create C++ Image and get NumPy view
img = Image(640, 480)
img.fill_gradient()

# Zero-copy: NumPy array shares Image's memory
arr = np.asarray(img)  # uses buffer protocol
print(arr.shape)   # (480, 640, 3)
print(arr.dtype)   # uint8
print(arr[0, 320, 0])  # ~127 (gradient midpoint)

# Modify via NumPy, C++ Image also changes
arr[0, 0, :] = [255, 0, 0]  # Set top-left pixel to red
```

---

## Notes

- `py::buffer_protocol()` must be listed inside the `py::class_<>` definition as a tag argument - it is not something you add separately.
- Strides must exactly match the actual memory layout of your C++ object. Getting them wrong makes NumPy silently read garbage data - there is no runtime check.
- For large data, always prefer the buffer protocol over copying. `py::array_t` constructors copy by default; the buffer protocol does not.
- pybind11 auto-translates all standard C++ exceptions to the appropriate Python exception type - no explicit registration needed for the common types.
- Default argument values in `py::init` are visible in Python's `help()` output, which makes your binding much friendlier to use interactively.
- Use `py::array::c_style | py::array::forcecast` when you need to guarantee a C-contiguous layout before your function processes the array.
