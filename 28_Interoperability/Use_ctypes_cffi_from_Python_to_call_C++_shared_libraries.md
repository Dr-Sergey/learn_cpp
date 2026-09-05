# Use ctypes / cffi from Python to call C++ shared libraries

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://docs.python.org/3/library/ctypes.html>  

---

## Topic Overview

`ctypes` and `cffi` let Python call functions in compiled C/C++ shared libraries (`.so` / `.dll`) without any compilation step on the Python side. The C++ library must export functions with `extern "C"` linkage - that is the key requirement that makes the whole thing work.

The reason `extern "C"` is mandatory is that Python (and ctypes) uses the raw symbol name from the shared library to find functions. C++ name mangling would turn `add` into something like `_Z3addii`, which ctypes cannot predict. With `extern "C"`, the symbol stays as `add` and ctypes can find it directly.

### Call Flow

Here is what happens when Python calls into a C++ library via ctypes. The `dlopen`/`LoadLibrary` step loads the shared library into the Python process's memory space, and from that point on, calls go directly to native code with no interpreter overhead.

```cpp
Python Process
┌─────────────────────────────────────┐
│ import ctypes                       │
│ lib = ctypes.CDLL("./libmath.so")   │──► dlopen / LoadLibrary
│ lib.add.restype = ctypes.c_int      │
│ result = lib.add(3, 4)              │──► FFI call to C symbol "add"
│ print(result)  # 7                  │
└─────────────────────────────────────┘
         │
         ▼
C++ Shared Library (libmath.so)
┌─────────────────────────────────────┐
│ extern "C" int add(int a, int b) {  │
│     return a + b;                   │
│ }                                   │
└─────────────────────────────────────┘
```

### ctypes vs cffi vs pybind11

If you're trying to decide which tool fits your situation, here is the breakdown. The key axis is whether you need to wrap C++ classes and whether you can tolerate a compilation step on the Python side.

| Feature | ctypes | cffi | pybind11/nanobind |
| --- | :---: | :---: | :---: |
| Install needed | No (stdlib) | Yes (pip) | Yes (pip + compiler) |
| C++ class support | No | No | **Yes** |
| ABI mode (no compile) | Yes | Yes | No |
| API mode (compile) | No | Yes | Yes |
| Performance | Good | Better | **Best** |
| Type checking | Manual | Auto from headers | **Auto** |
| Exception handling | No | No | **Yes** |

---

## Self-Assessment

### Q1: Export a C-linkage function from a C++ shared library and call it from Python using ctypes

**Answer:**

The C++ side just needs `extern "C"` around the functions you want to export. You can still use the full C++ standard library inside the function bodies - `extern "C"` only affects the symbol name, not what code is allowed.

```cpp
// ═══════════ mathlib.cpp — C++ shared library ═══════════
#include <cmath>
#include <cstring>

extern "C" {

// Simple arithmetic
int add(int a, int b) {
    return a + b;
}

double compute_distance(double x1, double y1, double x2, double y2) {
    double dx = x2 - x1;
    double dy = y2 - y1;
    return std::sqrt(dx * dx + dy * dy);
}

// String processing — return value via output parameter
int string_reverse(const char* input, char* output, int max_len) {
    if (!input || !output) return -1;
    int len = static_cast<int>(std::strlen(input));
    if (len >= max_len) return -2;

    for (int i = 0; i < len; ++i) {
        output[i] = input[len - 1 - i];
    }
    output[len] = '\0';
    return 0;
}

// Array processing
double array_sum(const double* arr, int n) {
    double sum = 0.0;
    for (int i = 0; i < n; ++i) sum += arr[i];
    return sum;
}

}  // extern "C"
```

Build the shared library with `-shared -fPIC` on Linux, or the equivalent on Windows/macOS:

```bash
# Build shared library
# Linux:
g++ -shared -fPIC -o libmath.so mathlib.cpp
# Windows:
# cl /LD mathlib.cpp /Fe:mathlib.dll
# macOS:
# g++ -shared -o libmath.dylib mathlib.cpp
```

On the Python side, the most important habit is always setting `argtypes` and `restype` before calling a function. Without them, ctypes assumes every argument and return value is a C `int`, which will silently produce wrong answers or even crash when you pass floats or pointers.

```python
# ═══════════ use_mathlib.py — Python consumer ═══════════
import ctypes
import os

# Load the shared library
if os.name == 'nt':
    lib = ctypes.CDLL("./mathlib.dll")
else:
    lib = ctypes.CDLL("./libmath.so")

# ─── Set argument and return types (critical for correctness!) ───
lib.add.argtypes = [ctypes.c_int, ctypes.c_int]
lib.add.restype = ctypes.c_int

lib.compute_distance.argtypes = [
    ctypes.c_double, ctypes.c_double,
    ctypes.c_double, ctypes.c_double
]
lib.compute_distance.restype = ctypes.c_double

lib.array_sum.argtypes = [ctypes.POINTER(ctypes.c_double), ctypes.c_int]
lib.array_sum.restype = ctypes.c_double

# ─── Call functions ───
print(lib.add(3, 4))                    # Output: 7
print(lib.compute_distance(0, 0, 3, 4)) # Output: 5.0

# ─── Pass an array ───
data = [1.0, 2.0, 3.0, 4.0, 5.0]
arr = (ctypes.c_double * len(data))(*data)  # Create C array
print(lib.array_sum(arr, len(data)))         # Output: 15.0

# ─── String function ───
lib.string_reverse.argtypes = [ctypes.c_char_p, ctypes.c_char_p, ctypes.c_int]
lib.string_reverse.restype = ctypes.c_int

output = ctypes.create_string_buffer(256)
lib.string_reverse(b"Hello", output, 256)
print(output.value)  # Output: b'olleH'
```

The `create_string_buffer(256)` call allocates a mutable buffer that the C function can write into - you can't pass a Python `bytes` literal as an output buffer since Python bytes objects are immutable.

### Q2: Show how to pass structs between Python and C++ using ctypes Structure

**Answer:**

Passing structs is where ctypes gets more involved. You need to define a Python `ctypes.Structure` subclass whose field layout exactly matches the C/C++ struct. If the layouts don't match - for example due to padding differences - you'll get subtly wrong values that are hard to diagnose.

```cpp
// ═══════════ geometry.cpp — C++ struct-based API ═══════════
#include <cmath>

struct Point {
    double x;
    double y;
};

struct Rect {
    Point top_left;
    Point bottom_right;
};

struct Stats {
    double min_val;
    double max_val;
    double mean;
    int count;
};

extern "C" {

double point_distance(const Point* a, const Point* b) {
    double dx = b->x - a->x;
    double dy = b->y - a->y;
    return std::sqrt(dx * dx + dy * dy);
}

double rect_area(const Rect* r) {
    double w = std::abs(r->bottom_right.x - r->top_left.x);
    double h = std::abs(r->bottom_right.y - r->top_left.y);
    return w * h;
}

// Return struct by filling output parameter
void compute_stats(const double* data, int n, Stats* out) {
    if (n <= 0) return;
    out->min_val = data[0];
    out->max_val = data[0];
    double sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] < out->min_val) out->min_val = data[i];
        if (data[i] > out->max_val) out->max_val = data[i];
        sum += data[i];
    }
    out->mean = sum / n;
    out->count = n;
}

}  // extern "C"
```

The Python `_fields_` list must list the fields in the same order and with the same types as the C struct. Nested structs work by using the nested Python class type directly as a field type - ctypes handles the layout automatically.

```python
# ═══════════ use_geometry.py ═══════════
import ctypes

lib = ctypes.CDLL("./libgeometry.so")

# ─── Define matching Python structures ───
class Point(ctypes.Structure):
    _fields_ = [
        ("x", ctypes.c_double),
        ("y", ctypes.c_double),
    ]

class Rect(ctypes.Structure):
    _fields_ = [
        ("top_left", Point),        # Nested struct
        ("bottom_right", Point),
    ]

class Stats(ctypes.Structure):
    _fields_ = [
        ("min_val", ctypes.c_double),
        ("max_val", ctypes.c_double),
        ("mean", ctypes.c_double),
        ("count", ctypes.c_int),
    ]

# ─── Set types ───
lib.point_distance.argtypes = [ctypes.POINTER(Point), ctypes.POINTER(Point)]
lib.point_distance.restype = ctypes.c_double

lib.rect_area.argtypes = [ctypes.POINTER(Rect)]
lib.rect_area.restype = ctypes.c_double

lib.compute_stats.argtypes = [
    ctypes.POINTER(ctypes.c_double), ctypes.c_int, ctypes.POINTER(Stats)
]
lib.compute_stats.restype = None

# ─── Use structs ───
a = Point(0.0, 0.0)
b = Point(3.0, 4.0)
print(f"Distance: {lib.point_distance(a, b)}")  # Output: Distance: 5.0

r = Rect(Point(0, 0), Point(10, 5))
print(f"Area: {lib.rect_area(r)}")  # Output: Area: 50.0

# ─── Output parameter pattern ───
data = [3.0, 1.0, 4.0, 1.0, 5.0, 9.0]
arr = (ctypes.c_double * len(data))(*data)
stats = Stats()
lib.compute_stats(arr, len(data), ctypes.byref(stats))
print(f"Min: {stats.min_val}, Max: {stats.max_val}, "
      f"Mean: {stats.mean:.2f}, Count: {stats.count}")
# Output: Min: 1.0, Max: 9.0, Mean: 3.83, Count: 6
```

The `ctypes.byref(stats)` call passes a pointer to the `Stats` struct - it's slightly more efficient than `ctypes.pointer(stats)` because it doesn't create a separate Python object for the pointer itself.

### Q3: Explain when ctypes is sufficient vs when pybind11 or nanobind is needed

**Answer:**

The right mental model is this: ctypes is a low-level FFI for calling C-style functions from an existing binary you don't control. pybind11/nanobind is for building a Python extension module from C++ source code you do control. Here is when each wins:

| Scenario | ctypes/cffi | pybind11/nanobind |
| --- | :---: | :---: |
| Call a few C functions | **Yes** | Overkill |
| Use existing `.so`/`.dll` | **Yes** | Needs rebuild |
| Wrap C++ classes | Painful | **Yes** |
| Handle C++ exceptions | No | **Yes** |
| Template functions | No | **Yes** |
| NumPy array interop | Manual | **Yes** (buffer protocol) |
| Callbacks Python -> C++ | Fragile | **Yes** |
| Performance-critical tight loop | Good | **Better** (less overhead) |
| No build step needed | **Yes** | No |

**Use ctypes when:**

- You need to call a small number of C-style functions from an existing library.
- The library already has a C API (for example SQLite, OpenSSL, or system libraries).
- You want zero compile dependencies on the Python side.

**Use pybind11/nanobind when:**

- You're wrapping C++ classes with methods, inheritance, and virtual functions.
- You need automatic type conversion (`std::string` to `str`, `std::vector` to `list`, etc.).
- You want exceptions to propagate properly from C++ into Python.
- You're building a Python package that ships C++ source code.

The code below shows why ctypes without `argtypes` is dangerous - it silently defaults to `c_int` everywhere, which means wrong results for floats or a segfault for pointers:

```python
# ctypes: manual, error-prone, but zero dependencies
lib = ctypes.CDLL("./libstats.so")
lib.mean.argtypes = [ctypes.POINTER(ctypes.c_double), ctypes.c_int]
lib.mean.restype = ctypes.c_double
arr = (ctypes.c_double * 3)(1.0, 2.0, 3.0)
result = lib.mean(arr, 3)
# Forget to set argtypes? Silent wrong answer or segfault
```

pybind11's approach is fundamentally safer because the type information is baked in at compile time:

```python
# pybind11: automatic, safe, but requires compilation
import stats  # compiled C++ extension module
result = stats.mean([1.0, 2.0, 3.0])  # accepts Python list directly
# Wrong type? Raises TypeError with clear message
```

---

## Notes

- Always set `argtypes` and `restype` before calling a function - without them, ctypes assumes `c_int` for everything, which silently corrupts floats and pointers.
- `ctypes.byref(x)` passes a pointer to `x` and is faster than `ctypes.pointer(x)` because it avoids creating a Python wrapper object for the pointer.
- cffi's API mode precompiles the wrapper, which gives faster startup than ctypes on first import.
- Struct padding differences between C++ and Python definitions can cause subtle wrong-value bugs - use `#pragma pack` or `__attribute__((packed))` on the C++ side if you need to control layout precisely.
- On Windows, use `CDLL` for `__cdecl` functions and `WinDLL` for `__stdcall` (Win32 API) - using the wrong one causes stack corruption on 32-bit.
- Memory allocated in C++ must be freed in C++ - never call Python's garbage collector to free memory that was allocated with `malloc` or `new` in C++.
