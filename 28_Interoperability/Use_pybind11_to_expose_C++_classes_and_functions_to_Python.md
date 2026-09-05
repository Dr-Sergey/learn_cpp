# Use pybind11 to expose C++ classes and functions to Python

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://pybind11.readthedocs.io/>  

---

## Topic Overview

This topic covers **advanced pybind11 patterns**: `py::class_<T>` with inheritance and virtual functions, `py::register_exception` for custom exception types, and `py::buffer_protocol` to expose C++ containers as NumPy arrays without copying data.

These are the patterns you reach for once the basics are not enough - when you need Python to subclass a C++ abstract base, when your custom exception hierarchy needs to survive the language boundary, or when a large C++ data structure needs to look like a NumPy array without any copying at all.

### Key Binding Patterns

If the full pybind11 API feels overwhelming, this table gives you a quick map to the most common patterns you will actually use.

| Pattern | API | Use Case |
| --- | --- | --- |
| Constructor | `.def(py::init<args>())` | Create from Python |
| Method | `.def("name", &T::method)` | Regular methods |
| Static method | `.def_static("name", &T::func)` | Class methods |
| Property (R/W) | `.def_property("n", get, set)` | Getter + setter |
| Property (RO) | `.def_property_readonly("n", get)` | Read-only access |
| Operator | `.def(py::self + py::self)` | `__add__`, `__mul__`, etc. |
| Inheritance | `py::class_<D, B>(m, "D")` | Base class bindings |
| Virtual | `class PyB : B { ... }` | Override from Python |
| Buffer | `.def_buffer(...)` | NumPy zero-copy |

---

## Self-Assessment

### Q1: Expose a C++ class with py::class_ including constructors, methods, and properties

**Answer:**

There is one concept here that deserves extra attention before you look at the code: the **trampoline class**. When a C++ class has virtual methods that Python code should be allowed to override, pybind11 needs a helper class - the trampoline - that sits between Python and C++. It intercepts virtual calls and routes them to Python if the Python subclass overrides the method. Without it, calling `area()` on a Python-derived class would silently call the C++ base version instead.

```cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <pybind11/operators.h>
#include <string>
#include <vector>
#include <sstream>
#include <cmath>
#include <numbers>

namespace py = pybind11;

// C++ class hierarchy
class Shape {
public:
    std::string name;
    Shape(const std::string& n) : name(n) {}
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual std::string describe() const {
        return name + " (area=" + std::to_string(area()) + ")";
    }
};

class Circle : public Shape {
    double radius_;
public:
    Circle(double r) : Shape("Circle"), radius_(r) {
        if (r < 0) throw std::invalid_argument("Negative radius");
    }
    double area() const override { return std::numbers::pi * radius_ * radius_; }
    double radius() const { return radius_; }
    void set_radius(double r) { radius_ = r; }
};

class Rectangle : public Shape {
public:
    double width, height;
    Rectangle(double w, double h) : Shape("Rectangle"), width(w), height(h) {}
    double area() const override { return width * height; }
    double perimeter() const { return 2 * (width + height); }

    // Operator support
    bool operator==(const Rectangle& other) const {
        return width == other.width && height == other.height;
    }
};

// Trampoline class for virtual overrides in Python
class PyShape : public Shape {
public:
    using Shape::Shape;  // Inherit constructors

    double area() const override {
        PYBIND11_OVERRIDE_PURE(double, Shape, area);
    }

    std::string describe() const override {
        PYBIND11_OVERRIDE(std::string, Shape, describe);
    }
};

// Module
PYBIND11_MODULE(shapes, m) {
    // Base class with trampoline - note both Shape and PyShape in the template
    py::class_<Shape, PyShape>(m, "Shape")
        .def(py::init<const std::string&>())
        .def("area", &Shape::area)
        .def("describe", &Shape::describe)
        .def_readwrite("name", &Shape::name);

    // Derived: Circle
    py::class_<Circle, Shape>(m, "Circle")
        .def(py::init<double>(), py::arg("radius"))
        .def("area", &Circle::area)
        .def_property("radius", &Circle::radius, &Circle::set_radius);

    // Derived: Rectangle with operators
    py::class_<Rectangle, Shape>(m, "Rectangle")
        .def(py::init<double, double>(), py::arg("width"), py::arg("height"))
        .def("area", &Rectangle::area)
        .def("perimeter", &Rectangle::perimeter)
        .def_readwrite("width", &Rectangle::width)
        .def_readwrite("height", &Rectangle::height)
        .def(py::self == py::self)  // __eq__
        .def("__repr__", [](const Rectangle& r) {
            return "Rectangle(" + std::to_string(r.width) + ", " +
                   std::to_string(r.height) + ")";
        });

    // Free function that accepts base class
    m.def("total_area", [](const std::vector<Shape*>& shapes) {
        double total = 0;
        for (auto* s : shapes) total += s->area();
        return total;
    });
}
```

The key line is `py::class_<Shape, PyShape>` - not `py::class_<Shape>`. That second template argument is what connects the Python override mechanism. `PYBIND11_OVERRIDE_PURE` handles the case where the C++ method is pure virtual; `PYBIND11_OVERRIDE` handles the case where there is a default C++ implementation that Python can optionally override.

```python
# Python usage - including overriding virtual from Python:
from shapes import Shape, Circle, Rectangle

c = Circle(5.0)
print(c.area())      # 78.539...
c.radius = 10.0
print(c.area())      # 314.159...

r = Rectangle(3, 4)
print(r.perimeter())  # 14.0
print(r == Rectangle(3, 4))  # True

# Override virtual method from Python
class Triangle(Shape):
    def __init__(self, base, height):
        super().__init__("Triangle")
        self.base = base
        self.height = height

    def area(self):
        return 0.5 * self.base * self.height

t = Triangle(6, 8)
print(t.area())       # 24.0
print(t.describe())   # Triangle (area=24.0)
```

Notice that `t.describe()` works even though `describe` is defined in C++ - it calls `area()` via virtual dispatch, which correctly routes to Python's `Triangle.area()`.

### Q2: Handle C++ exceptions by translating them to Python exceptions using py::register_exception

**Answer:**

When your C++ code has a custom exception hierarchy, you want that hierarchy to survive the language boundary intact. pybind11's `py::register_exception` creates a real Python exception class for each C++ exception, and you can wire them up to inherit from each other on the Python side - so `except AppError` will also catch `ValidationError` in Python, mirroring the C++ inheritance.

The tricky part is registration order and the translator. Register from most-derived to least-derived, because the translator runs through all registered translators in reverse order (last-registered, first-tried).

```cpp
#include <pybind11/pybind11.h>
#include <stdexcept>

namespace py = pybind11;

// Custom exception hierarchy
class AppError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class ValidationError : public AppError {
public:
    std::string field;
    ValidationError(const std::string& f, const std::string& msg)
        : AppError(msg), field(f) {}
};

class NotFoundError : public AppError {
public:
    int code;
    NotFoundError(const std::string& msg, int c)
        : AppError(msg), code(c) {}
};

void validate_email(const std::string& email) {
    if (email.find('@') == std::string::npos)
        throw ValidationError("email", "Invalid email: " + email);
}

void find_user(int id) {
    throw NotFoundError("User not found", 404);
}

PYBIND11_MODULE(exceptions_demo, m) {
    // Register exception hierarchy
    // Base exception: maps to Python's RuntimeError
    static auto pyAppError = py::register_exception<AppError>(
        m, "AppError", PyExc_RuntimeError);

    // Derived: maps under AppError
    static auto pyValidation = py::register_exception<ValidationError>(
        m, "ValidationError", pyAppError.ptr());

    static auto pyNotFound = py::register_exception<NotFoundError>(
        m, "NotFoundError", pyAppError.ptr());

    // Custom translator with attributes
    py::register_exception_translator([](std::exception_ptr p) {
        try {
            if (p) std::rethrow_exception(p);
        } catch (const NotFoundError& e) {
            py::object exc = pyNotFound(e.what());
            exc.attr("code") = e.code;
            PyErr_SetObject(pyNotFound.ptr(), exc.ptr());
        } catch (const ValidationError& e) {
            py::object exc = pyValidation(e.what());
            exc.attr("field") = e.field;
            PyErr_SetObject(pyValidation.ptr(), exc.ptr());
        }
    });

    m.def("validate_email", &validate_email);
    m.def("find_user", &find_user);
}
```

After registration, the Python exception hierarchy mirrors the C++ one, so you can catch by base class on the Python side.

```python
from exceptions_demo import validate_email, find_user, \
    AppError, ValidationError, NotFoundError

try:
    validate_email("bad-email")
except ValidationError as e:
    print(f"Field: {e.field}, Error: {e}")
    # Field: email, Error: Invalid email: bad-email

try:
    find_user(42)
except NotFoundError as e:
    print(f"Code: {e.code}, Error: {e}")
    # Code: 404, Error: User not found

# Hierarchy works:
try:
    find_user(42)
except AppError:  # catches NotFoundError too
    print("Caught AppError (base)")
```

### Q3: Use py::buffer_protocol to expose a C++ matrix as a NumPy array without copying data

**Answer:**

The buffer protocol is how pybind11 enables zero-copy sharing between C++ memory and NumPy. The key idea is that you describe your C++ buffer to Python using `py::buffer_info` - giving it the pointer, element size, format descriptor, number of dimensions, shape, and strides. NumPy then treats your C++ memory as its own array, with no data ever copied.

The tricky part is getting the strides right. Strides describe how many bytes to skip to advance one step in each dimension. For a row-major 2D matrix, the row stride is `sizeof(double) * num_cols` and the column stride is `sizeof(double)`.

```cpp
#include <pybind11/pybind11.h>
#include <pybind11/numpy.h>
#include <vector>
#include <stdexcept>
#include <sstream>

namespace py = pybind11;

// C++ Matrix class
class Matrix {
    std::vector<double> data_;
    size_t rows_, cols_;

public:
    Matrix(size_t rows, size_t cols)
        : data_(rows * cols, 0.0), rows_(rows), cols_(cols) {}

    double& operator()(size_t r, size_t c) {
        return data_[r * cols_ + c];
    }

    double operator()(size_t r, size_t c) const {
        return data_[r * cols_ + c];
    }

    size_t rows() const { return rows_; }
    size_t cols() const { return cols_; }
    double* data() { return data_.data(); }
    const double* data() const { return data_.data(); }
    size_t size() const { return data_.size(); }

    void fill(double value) {
        std::fill(data_.begin(), data_.end(), value);
    }
};

PYBIND11_MODULE(matrix_mod, m) {
    py::class_<Matrix>(m, "Matrix", py::buffer_protocol())
        .def(py::init<size_t, size_t>())
        .def("fill", &Matrix::fill)
        .def_property_readonly("rows", &Matrix::rows)
        .def_property_readonly("cols", &Matrix::cols)

        // Buffer protocol: expose as NumPy array
        .def_buffer([](Matrix& mat) -> py::buffer_info {
            return py::buffer_info(
                mat.data(),                          // Pointer to data
                sizeof(double),                      // Size of one element
                py::format_descriptor<double>::format(), // Python struct format
                2,                                    // Number of dimensions
                {mat.rows(), mat.cols()},            // Shape
                {sizeof(double) * mat.cols(),        // Strides (row-major)
                 sizeof(double)}
            );
        })

        // Construct from NumPy array
        .def(py::init([](py::array_t<double> arr) {
            auto buf = arr.unchecked<2>();
            Matrix mat(buf.shape(0), buf.shape(1));
            for (py::ssize_t i = 0; i < buf.shape(0); ++i)
                for (py::ssize_t j = 0; j < buf.shape(1); ++j)
                    mat(i, j) = buf(i, j);
            return mat;
        }))

        .def("__repr__", [](const Matrix& mat) {
            return "<Matrix " + std::to_string(mat.rows()) + "x" +
                   std::to_string(mat.cols()) + ">";
        });
}
```

Because `py::buffer_protocol()` is listed in the class template, NumPy's `np.array(mat, copy=False)` calls your `def_buffer` lambda and gets back a direct pointer into the C++ vector's storage. Any write through the NumPy array modifies the C++ object, and vice versa.

```python
import numpy as np
from matrix_mod import Matrix

# Create C++ matrix and get NumPy view (zero-copy!)
mat = Matrix(3, 4)
mat.fill(42.0)

# Convert to NumPy array - SHARES memory (no copy)
arr = np.array(mat, copy=False)
print(arr.shape)  # (3, 4)
print(arr[0, 0])  # 42.0

# Modify through NumPy - C++ matrix also changes
arr[1, 2] = 99.0
a2 = np.array(mat, copy=False)
print(a2[1, 2])    # 99.0  <- modified through NumPy

# Construct Matrix from NumPy
np_mat = np.array([[1, 2], [3, 4]], dtype=np.float64)
cpp_mat = Matrix(np_mat)
print(cpp_mat.rows, cpp_mat.cols)  # 2 2
```

---

## Notes

- `py::buffer_protocol()` must appear inside the `py::class_<>` definition as a tag argument - it cannot be added after the fact.
- Buffer protocol gives true zero-copy sharing: NumPy and C++ look at the same physical memory. If the C++ object is destroyed while a NumPy view is alive, you get a dangling pointer - manage the lifetime carefully.
- `PYBIND11_OVERRIDE_PURE` handles pure virtual methods; `PYBIND11_OVERRIDE` handles virtual methods that have a default C++ implementation.
- A trampoline class (like `PyShape` above) is required for any C++ class whose virtual methods you want Python to be able to override.
- Exception translators are run in LIFO order - register the most specific exception type last so it gets tried first.
- `py::register_exception` creates a real Python exception class that inherits from whatever base you specify, which means the normal Python `except` hierarchy works correctly.
