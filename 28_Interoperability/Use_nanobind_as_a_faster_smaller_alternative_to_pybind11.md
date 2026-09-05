# Use nanobind as a faster, smaller alternative to pybind11

**Category:** Interoperability  
**Standard:** C++17  
**Reference:** <https://nanobind.readthedocs.io>  

---

## Topic Overview

**nanobind** is a next-generation C++ -> Python binding library created by the same author as pybind11 (Wenzel Jakob). It produces **smaller binaries** (roughly 2-5x), **compiles faster** (roughly 2-3x), and has **lower runtime overhead** than pybind11, while offering a nearly identical API. If you're starting a new Python extension project that targets Python only and you can use C++17, nanobind is worth choosing over pybind11 from the start.

The reason it's faster and smaller is architectural: instead of embedding its own copy of the binding runtime inside every module, nanobind puts the runtime in a single shared library that all modules load on demand. This is the same idea as a shared C++ runtime versus static linking.

### nanobind vs pybind11 Comparison

| Metric | pybind11 | nanobind |
| --- | :---: | :---: |
| Binary size (typical) | 200-500 KB | **50-150 KB** |
| Compile time | Baseline | **2-3x faster** |
| Import time | ~10 ms | **~2 ms** |
| Min C++ standard | C++11 | **C++17** |
| Object model | Holder-based | **Intrusive refcount** |
| GIL handling | Manual | **Automatic** |
| NumPy support | `py::array` | `nb::ndarray` (zero-copy) |
| Ownership model | Shared holder | **Move-first** |

### Architecture

Here is why nanobind modules end up smaller. Each pybind11 module is self-contained - the binding runtime is compiled in. nanobind splits that runtime into a shared library that is loaded once and shared by all nanobind modules in the same process.

```cpp
nanobind module (.so / .pyd)
┌──────────────────────────────────────┐
│ NB_MODULE(mymod, m) {               │
│   m.def("add", &add);              │  <- Binding code (~50-150 KB)
│   nb::class_<Vec>(m, "Vec")...     │
│ }                                    │
├──────────────────────────────────────┤
│ nanobind core (shared library)       │  <- Shared across all modules
│ libnanobind.so (~400 KB)             │     (loaded once)
└──────────────────────────────────────┘
vs pybind11: each module embeds its own copy of the runtime
```

---

## Self-Assessment

### Q1: Rewrite a pybind11 module as nanobind and compare binary size and import time

**Answer:**

The API is almost identical, so migration is mostly mechanical search-and-replace. The examples below show the same `Vec3` module in both versions so you can see the differences side by side. The main changes are the macro name, the `nb::` namespace, and a couple of method name shortenings (`def_rw` instead of `def_readwrite`).

```cpp
// ═══════════ pybind11 version: math_pb11.cpp ═══════════
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <cmath>
#include <vector>
#include <numeric>

namespace py = pybind11;

double dot_product(const std::vector<double>& a, const std::vector<double>& b) {
    return std::inner_product(a.begin(), a.end(), b.begin(), 0.0);
}

struct Vec3 {
    double x, y, z;
    double length() const { return std::sqrt(x*x + y*y + z*z); }
    Vec3 normalized() const {
        double len = length();
        return {x/len, y/len, z/len};
    }
};

PYBIND11_MODULE(math_pb11, m) {
    m.def("dot_product", &dot_product);
    py::class_<Vec3>(m, "Vec3")
        .def(py::init<double, double, double>())
        .def_readwrite("x", &Vec3::x)
        .def_readwrite("y", &Vec3::y)
        .def_readwrite("z", &Vec3::z)
        .def("length", &Vec3::length)
        .def("normalized", &Vec3::normalized);
}
```

The nanobind version below is structurally identical. Spot the differences: `NB_MODULE`, `nb::init`, `nb::class_`, and `def_rw` instead of `def_readwrite`. Also note that the STL include path changes from `<pybind11/stl.h>` to `<nanobind/stl/vector.h>` - nanobind uses per-type includes rather than a catch-all header, which is part of why it compiles faster.

```cpp
// ═══════════ nanobind version: math_nb.cpp ═══════════
#include <nanobind/nanobind.h>
#include <nanobind/stl/vector.h>  // std::vector conversion
#include <cmath>
#include <vector>
#include <numeric>

namespace nb = nanobind;

double dot_product(const std::vector<double>& a, const std::vector<double>& b) {
    return std::inner_product(a.begin(), a.end(), b.begin(), 0.0);
}

struct Vec3 {
    double x, y, z;
    double length() const { return std::sqrt(x*x + y*y + z*z); }
    Vec3 normalized() const {
        double len = length();
        return {x/len, y/len, z/len};
    }
};

NB_MODULE(math_nb, m) {   // NB_MODULE instead of PYBIND11_MODULE
    m.def("dot_product", &dot_product);
    nb::class_<Vec3>(m, "Vec3")
        .def(nb::init<double, double, double>())
        .def_rw("x", &Vec3::x)        // def_rw, not def_readwrite
        .def_rw("y", &Vec3::y)
        .def_rw("z", &Vec3::z)
        .def("length", &Vec3::length)
        .def("normalized", &Vec3::normalized);
}
```

After compiling both, you can measure the actual difference. These are representative numbers - your results will vary based on compiler flags and the size of your module, but the ratios are consistent.

```python
# ═══════════ Benchmark comparison ═══════════
import time, os

# Binary size comparison
pb11_size = os.path.getsize("math_pb11.cpython-312-x86_64-linux-gnu.so")
nb_size = os.path.getsize("math_nb.cpython-312-x86_64-linux-gnu.so")
print(f"pybind11: {pb11_size / 1024:.0f} KB")   # ~280 KB
print(f"nanobind: {nb_size / 1024:.0f} KB")      # ~70 KB
print(f"Ratio: {pb11_size / nb_size:.1f}x")       # ~4x smaller

# Import time
t0 = time.perf_counter()
import math_pb11
t1 = time.perf_counter()
print(f"pybind11 import: {(t1-t0)*1000:.1f} ms")  # ~12 ms

t0 = time.perf_counter()
import math_nb
t1 = time.perf_counter()
print(f"nanobind import: {(t1-t0)*1000:.1f} ms")  # ~3 ms
```

### Q2: Explain nanobind's ownership model: it uses reference counting more aggressively than pybind11

**Answer:**

This is the biggest conceptual difference between nanobind and pybind11, and it's worth understanding before you write any binding code.

In pybind11, every C++ object that Python holds is wrapped in a "holder" - typically a `shared_ptr`. This means you always have two reference counts: the Python object's ref count managed by the GC, and the shared_ptr's ref count. There's overhead for the extra indirection and the atomic operations on the ref count.

nanobind takes a different approach: by default, Python directly owns the C++ object. When the Python GC collects the Python wrapper, the C++ destructor runs immediately. There's no intermediate holder. If you need shared ownership between Python and C++, you opt in explicitly using `nb::intrusive_base`.

```cpp
pybind11 ownership model:
┌──────────────────────┐    ┌──────────────────┐
│ Python object (PyObj)│───►│ Holder (shared_ptr│
│ ref_count = 1        │    │ or unique_ptr)    │───► C++ Object
└──────────────────────┘    └──────────────────┘
  ↑ Python GC manages          ↑ C++ ref count = 1

Every C++ object gets wrapped in a holder -> overhead

nanobind ownership model:
┌──────────────────────┐
│ Python object (PyObj)│───► C++ Object (directly embedded)
│ ref_count = 1        │     or pointer with nb_ref attached
└──────────────────────┘
  ↑ Single ref count for both Python and C++
```

Here are the three ownership options you'll use in practice. The default (direct ownership) is fast and safe for simple objects. `intrusive_base` is for objects that need to be shared between Python and C++. Explicit `shared_ptr` is for objects you're already managing with `shared_ptr` throughout your C++ codebase.

```cpp
#include <nanobind/nanobind.h>

namespace nb = nanobind;

// ═══════════ Default: nanobind owns the C++ object ═══════════
struct Particle {
    double x, y, z;
    double mass;
};

// ═══════════ Intrusive ref counting (for shared ownership) ═══════════
struct SharedResource : nb::intrusive_base {
    // nb::intrusive_base adds:
    //   - inc_ref() / dec_ref()
    //   - ref count managed by both Python and C++
    int data;
    SharedResource(int d) : data(d) {}
};

NB_MODULE(ownership_demo, m) {
    // Default: nanobind takes ownership via move
    nb::class_<Particle>(m, "Particle")
        .def(nb::init<double, double, double, double>())
        .def_rw("x", &Particle::x)
        .def_rw("mass", &Particle::mass);
    // When Python GC collects Particle Python object -> C++ object destroyed

    // ═══════════ Return value policies ═══════════
    // nanobind is stricter than pybind11:
    //   - reference: Python gets non-owning view (dangling risk!)
    //   - move: C++ object moved into Python ownership (DEFAULT)
    //   - copy: C++ object copied into Python ownership
    //   - reference_internal: tied to parent object lifetime

    struct Container {
        Particle particle{0,0,0,1};
        Particle& get() { return particle; }
    };

    nb::class_<Container>(m, "Container")
        .def(nb::init<>())
        // reference_internal: returned ref lives as long as Container
        .def("get", &Container::get, nb::rv_policy::reference_internal);

    // ═══════════ Shared ownership with intrusive refcount ═══════════
    nb::class_<SharedResource>(m, "SharedResource")
        .def(nb::init<int>())
        .def_rw("data", &SharedResource::data);
    // Both C++ and Python can hold refs; destroyed when BOTH sides release
}
```

The `reference_internal` policy is the one you'll use when a method returns a reference to a member of the object. It keeps the parent object alive as long as the returned reference is alive - otherwise you'd get a dangling reference if Python GC'd the container while you still held the reference.

### Q3: Show how nanobind's nb::type_caster enables custom type conversions

**Answer:**

nanobind's type caster system lets you teach the binding layer how to convert between a C++ type and a Python type automatically, so any function that uses that C++ type gets the conversion for free. This is more powerful than it sounds - you write the conversion code once, and every function binding that accepts or returns that type just works.

The example below converts between a C++ `Color` struct and a Python tuple. After the type caster is defined, you can call the `blend` function from Python by passing plain tuples - nanobind converts them to `Color` objects transparently.

```cpp
#include <nanobind/nanobind.h>
#include <nanobind/stl/string.h>

namespace nb = nanobind;

// ═══════════ Custom C++ type ═══════════
struct Color {
    uint8_t r, g, b, a;

    Color() : r(0), g(0), b(0), a(255) {}
    Color(uint8_t r, uint8_t g, uint8_t b, uint8_t a = 255)
        : r(r), g(g), b(b), a(a) {}

    std::string hex() const {
        char buf[10];
        snprintf(buf, sizeof(buf), "#%02x%02x%02x%02x", r, g, b, a);
        return buf;
    }
};

// ═══════════ Custom type caster: Python tuple <-> Color ═══════════
namespace nanobind::detail {

template <>
struct type_caster<Color> {
    // Declare the Python type name (shown in docstrings)
    NB_TYPE_CASTER(Color, const_name("Color"));

    // Python -> C++
    bool from_python(handle src, uint8_t flags, cleanup_list* cleanup) noexcept {
        // Accept tuple of (r, g, b) or (r, g, b, a)
        if (!isinstance<tuple>(src)) return false;

        tuple t = borrow<tuple>(src);
        size_t len = t.size();
        if (len != 3 && len != 4) return false;

        try {
            value.r = cast<uint8_t>(t[0]);
            value.g = cast<uint8_t>(t[1]);
            value.b = cast<uint8_t>(t[2]);
            value.a = len == 4 ? cast<uint8_t>(t[3]) : 255;
            return true;
        } catch (...) {
            return false;
        }
    }

    // C++ -> Python
    static handle from_cpp(const Color& c, rv_policy policy,
                           cleanup_list* cleanup) noexcept {
        return make_tuple(c.r, c.g, c.b, c.a).release();
    }
};

}  // namespace nanobind::detail

// ═══════════ Now Color auto-converts in any binding ═══════════
Color blend(const Color& a, const Color& b, float t) {
    auto mix = [t](uint8_t x, uint8_t y) -> uint8_t {
        return static_cast<uint8_t>(x * (1 - t) + y * t);
    };
    return Color(mix(a.r, b.r), mix(a.g, b.g), mix(a.b, b.b), mix(a.a, b.a));
}

NB_MODULE(color_mod, m) {
    m.def("blend", &blend);
    // Python usage:
    //   blend((255, 0, 0), (0, 0, 255), 0.5)  -> (127, 0, 127, 255)
    // Tuples auto-converted to Color objects!
}
```

The `from_python` function returns `false` if conversion fails (rather than throwing), and nanobind reports a clean `TypeError` to Python. The `from_cpp` function does the reverse - converts a `Color` back to a Python tuple when a binding function returns one. Once you have this type caster in scope, every other binding that accepts or returns `Color` picks it up automatically.

---

## Notes

- nanobind requires **C++17** minimum - if you're stuck on C++11/14, pybind11 is your option.
- The API is roughly 95% compatible with pybind11, so porting an existing module is usually straightforward.
- `nb::ndarray` provides zero-copy tensor interop with NumPy, PyTorch, TensorFlow, and JAX - all with the same C++ code.
- Use `nb::rv_policy::reference_internal` when binding methods that return references to members - it keeps the parent object alive.
- nanobind's core is a shared library, so all nanobind modules in one process share the runtime - this is where most of the binary size savings come from.
- Install via `pip install nanobind`; CMake integration uses `find_package(nanobind)` after installing.
