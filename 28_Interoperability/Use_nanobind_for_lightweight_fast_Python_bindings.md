# Use nanobind for lightweight, fast Python bindings

**Category:** Interoperability  
**Standard:** C++17  
**Reference:** <https://nanobind.readthedocs.io/>  

---

## Topic Overview

This topic covers **defining nanobind modules and binding C++ types to Python** with the `NB_MODULE` macro and `nb::class_<T>`. The focus is on the fundamental binding patterns and how nanobind's unique ownership model - unique_ptr by default instead of pybind11's shared_ptr - affects object lifetime.

Nanobind is the spiritual successor to pybind11, designed to be dramatically leaner. If you have ever waited painfully long for a pybind11 binding to compile or winced at the size of the resulting `.so` file, nanobind exists to fix exactly those pain points.

### nanobind Module Structure

Here is the overall layout of a nanobind module. Think of this as a map for everything that can go inside `NB_MODULE` - free functions, classes, enums, and module-level attributes all have their place.

```cpp
NB_MODULE(module_name, m) {
    │
    ├── m.def("func", &func)           // Free functions
    │
    ├── nb::class_<MyClass>(m, "MyClass")
    │   ├── .def(nb::init<args>())      // Constructor
    │   ├── .def("method", &MyClass::method)  // Methods
    │   ├── .def_rw("field", &MyClass::field) // Read-write property
    │   └── .def_ro("field", &MyClass::field) // Read-only property
    │
    ├── nb::enum_<MyEnum>(m, "MyEnum")  // Enums
    │   └── .value("Item", MyEnum::Item)
    │
    └── m.attr("VERSION") = "1.0"      // Module attributes
}
```

---

## Self-Assessment

### Q1: Compare nanobind with pybind11 for compilation time and binary size on a simple binding

**Answer:**

The numbers here are striking. On a binding that is not even particularly large - just a matrix class with five methods, three properties, and one constructor - nanobind is roughly three to four times better across every metric.

```cpp
Benchmark: Matrix class with 5 methods, 3 properties, 1 constructor
─────────────────────────────────────────────────────────────────────
                   pybind11        nanobind        Ratio
Compile time:      8.2 s           2.9 s           2.8x faster
Binary size:       312 KB          78 KB           4.0x smaller
Import time:       11.3 ms         2.8 ms          4.0x faster
Memory (RSS):      4.2 MB          1.1 MB          3.8x less
─────────────────────────────────────────────────────────────────────
```

The reason for the difference comes down to where the runtime code lives. pybind11 is fully header-only, which means every binding file gets its own private copy of the entire pybind11 machinery compiled into it. Nanobind takes a different approach: a small shared runtime (`libnanobind.so`) is loaded once for all nanobind modules, and each individual module only contains its thin wrapper code.

```cpp
pybind11:
┌─────────────────────────────────┐
│ Your module                      │
│ ┌───────────────────────────┐   │
│ │ pybind11 runtime (copy)   │   │  // Each module includes full runtime
│ │ ~200 KB of template code  │   │  // (header-only, all compiled in)
│ └───────────────────────────┘   │
└─────────────────────────────────┘

nanobind:
┌─────────────────────────────────┐  ┌──────────────────────┐
│ Your module (thin wrapper)       │  │ libnanobind.so       │
│ ~50 KB bindings only             │──│ ~400 KB (shared)     │
└─────────────────────────────────┘  │ loaded once for ALL  │
                                     │ nanobind modules     │
                                     └──────────────────────┘
```

So if you build five nanobind extension modules, they all share the same runtime. With pybind11, each one duplicates it.

### Q2: Define a nanobind module with NB_MODULE and bind a struct with nb::class_

**Answer:**

Here is a full working example that binds an enum, two classes with varying property types, and a free function. Pay attention to `def_rw` vs `def_ro` vs `def_prop_ro` - they cover different scenarios depending on whether you need a simple field reference or a computed property backed by a method.

```cpp
#include <nanobind/nanobind.h>
#include <nanobind/stl/string.h>
#include <nanobind/stl/vector.h>
#include <cmath>
#include <string>
#include <vector>

namespace nb = nanobind;

// C++ types to expose
enum class Shape { Circle, Square, Triangle };

struct Point {
    double x, y;
    Point(double x = 0, double y = 0) : x(x), y(y) {}
    double distance_to(const Point& other) const {
        double dx = x - other.x, dy = y - other.y;
        return std::sqrt(dx*dx + dy*dy);
    }
    std::string repr() const {
        return "Point(" + std::to_string(x) + ", " + std::to_string(y) + ")";
    }
};

struct Polygon {
    std::string name;
    std::vector<Point> vertices;

    Polygon(const std::string& name) : name(name) {}

    void add_vertex(double x, double y) {
        vertices.emplace_back(x, y);
    }

    double perimeter() const {
        if (vertices.size() < 2) return 0;
        double total = 0;
        for (size_t i = 0; i < vertices.size(); ++i) {
            size_t j = (i + 1) % vertices.size();
            total += vertices[i].distance_to(vertices[j]);
        }
        return total;
    }

    size_t num_vertices() const { return vertices.size(); }
};

// Module definition
NB_MODULE(geometry, m) {
    m.doc() = "Geometry module - demonstrates nanobind bindings";

    // Enum binding
    nb::enum_<Shape>(m, "Shape")
        .value("Circle", Shape::Circle)
        .value("Square", Shape::Square)
        .value("Triangle", Shape::Triangle);

    // Point class
    nb::class_<Point>(m, "Point")
        .def(nb::init<double, double>(),
             nb::arg("x") = 0.0, nb::arg("y") = 0.0)
        .def_rw("x", &Point::x)
        .def_rw("y", &Point::y)
        .def("distance_to", &Point::distance_to)
        .def("__repr__", &Point::repr);

    // Polygon class
    nb::class_<Polygon>(m, "Polygon")
        .def(nb::init<const std::string&>(), nb::arg("name"))
        .def_ro("name", &Polygon::name)
        .def("add_vertex", &Polygon::add_vertex,
             nb::arg("x"), nb::arg("y"))
        .def("perimeter", &Polygon::perimeter)
        .def_prop_ro("num_vertices",   // read-only property
            [](const Polygon& p) { return p.num_vertices(); });

    // Free function
    m.def("midpoint", [](const Point& a, const Point& b) {
        return Point((a.x + b.x) / 2, (a.y + b.y) / 2);
    }, nb::arg("a"), nb::arg("b"));
}
```

Notice how `def_prop_ro` wraps a lambda rather than pointing directly at a member - that is how you turn a method into a Python property. From Python's perspective, `polygon.num_vertices` looks like an attribute, not a function call.

```python
# Python usage:
from geometry import Point, Polygon, Shape, midpoint

p1 = Point(0, 0)
p2 = Point(3, 4)
print(p1.distance_to(p2))  # 5.0
print(midpoint(p1, p2))    # Point(1.5, 2.0)

tri = Polygon("triangle")
tri.add_vertex(0, 0)
tri.add_vertex(3, 0)
tri.add_vertex(0, 4)
print(tri.perimeter)        # Error - it's a method, not property
print(tri.num_vertices)     # 3 (def_prop_ro makes it a property)
print(tri.perimeter())      # 12.0
```

### Q3: Explain the ownership model difference: nanobind uses unique_ptr by default, pybind11 shared_ptr

**Answer:**

This is the part of nanobind that trips people up the most, because the default behavior is quite different from what pybind11 users expect. The reason nanobind chose `unique_ptr` semantics is that `shared_ptr` has a real cost: every object needs a control block with atomic reference counts, which adds memory overhead and cache pressure. Most Python-owned objects never need shared C++ ownership, so nanobind makes the cheap thing the default.

```cpp
pybind11 default:
┌───────────────┐    ┌──────────────┐    ┌──────────────┐
│ Python obj    │───►│ shared_ptr   │───►│ C++ object   │
│ ref_count     │    │ control block│    │              │
└───────────────┘    │ ref_count    │    └──────────────┘
                     └──────────────┘
    Every object has shared_ptr overhead (control block + atomics)
    C++ code can also hold shared_ptr -> shared ownership

nanobind default:
┌───────────────┐    ┌──────────────┐
│ Python obj    │───►│ C++ object   │  // direct pointer, no control block
│ ref_count     │    │ (owned)      │
└───────────────┘    └──────────────┘
    Python exclusively owns the C++ object
    C++ code cannot hold long-lived references (dangling risk!)
```

The danger with the nanobind default is this: if a C++ method returns a reference to a member, and Python later garbage-collects the parent object, that reference becomes dangling. The fix is `rv_policy::reference_internal`, which tells nanobind to keep the parent alive as long as the child reference exists.

```cpp
#include <nanobind/nanobind.h>
#include <memory>

namespace nb = nanobind;

struct Engine {
    int horsepower;
    Engine(int hp) : horsepower(hp) {}
};

struct Car {
    std::string model;
    Engine engine;
    Car(const std::string& m, int hp) : model(m), engine(hp) {}
    Engine& get_engine() { return engine; }
};

NB_MODULE(ownership_demo, m) {
    nb::class_<Engine>(m, "Engine")
        .def(nb::init<int>())
        .def_rw("horsepower", &Engine::horsepower);

    nb::class_<Car>(m, "Car")
        .def(nb::init<const std::string&, int>())
        .def_ro("model", &Car::model)
        // CRITICAL: reference_internal ties engine's lifetime to car
        .def("get_engine", &Car::get_engine,
             nb::rv_policy::reference_internal);
        // Without reference_internal:
        //   e = car.get_engine()
        //   del car  -> engine's C++ object freed
        //   e.horsepower  -> CRASH (dangling pointer!)
        //
        // With reference_internal:
        //   e holds reference to car -> car stays alive
}
```

When you genuinely do need C++ code to share ownership with Python - for example, when passing objects into C++ containers that outlive the Python side - you can opt back into `shared_ptr` explicitly.

**When to use shared_ptr with nanobind:**

```cpp
// Opt-in when C++ code needs shared ownership
struct SharedWidget {
    int id;
    SharedWidget(int i) : id(i) {}
};

// Explicitly declare shared_ptr holder
nb::class_<SharedWidget>(m, "SharedWidget",
                         nb::holder<std::shared_ptr<SharedWidget>>())
    .def(nb::init<int>());

// Now C++ functions can accept/return shared_ptr<SharedWidget>
```

The key insight is that nanobind gives you the fast default and lets you opt into the expensive one when you actually need it, rather than always paying for `shared_ptr` whether you need it or not.

---

## Notes

- nanobind requires C++17; pybind11 works with C++11 - check your project constraints before switching.
- The default unique ownership model means less overhead, but you must be careful with return value policies whenever a method returns a reference or pointer into a member.
- `nb::rv_policy::reference_internal` is critical for returning references to sub-objects, so the parent stays alive.
- `NB_MODULE` compiles faster than `PYBIND11_MODULE` because nanobind does far less template instantiation.
- `def_rw` / `def_ro` are nanobind's shorter names for `def_readwrite` / `def_readonly` - same concept, less typing.
- Use `nb::holder<std::shared_ptr<T>>()` only when C++ APIs genuinely require shared ownership; don't reach for it by habit.
