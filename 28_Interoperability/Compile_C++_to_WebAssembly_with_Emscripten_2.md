# Compile C++ to WebAssembly with Emscripten (Part 2: Bindings and Memory)

**Category:** Interoperability  
**Standard:** C++11  
**Reference:** <https://emscripten.org/docs/>  

---

## Topic Overview

This part focuses on **EMSCRIPTEN_BINDINGS**, automatic type marshaling between C++ and JavaScript, and the **linear memory model** of WebAssembly. Understanding how types cross the C++/JS boundary - and how Wasm's single flat memory works - will save you a lot of debugging time when things don't behave the way you expect.

### Type Marshaling: C++ -> JavaScript

Not all C++ types can cross the boundary transparently. Primitives pass through directly as Wasm native values, but anything more complex requires marshaling - sometimes by copy, sometimes by handle. Here's how the common types map:

| C++ Type | JS Type | Marshaling |
| --- | --- | --- |
| `int`, `float`, `double` | `number` | Direct (Wasm native) |
| `bool` | `boolean` | Direct |
| `std::string` | `string` | Copy (UTF-8 encode/decode) |
| `std::vector<T>` | `Array` (via `register_vector`) | Copy |
| `std::map<K,V>` | `Object` (via `register_map`) | Copy |
| `T*` / `T&` | Embind wrapper | Handle-based |
| `enum` | `number` or `string` (via `enum_`) | Mapped |
| `val` | Any JS value | Pass-through |

### WebAssembly Memory Model

The reason this feels unusual at first is that WebAssembly doesn't have a GC-managed heap like JavaScript. It has one flat `ArrayBuffer` - a single contiguous chunk of bytes - and your C++ stack and heap both live inside it. JavaScript can read and write this buffer directly through typed array views like `HEAP32` and `HEAPF64`:

```cpp
JavaScript                          WebAssembly
┌─────────────┐                    ┌─────────────────────┐
│ JS Heap      │                    │ Linear Memory        │
│ (GC managed) │  <- separate ->   │ (one ArrayBuffer)    │
│              │                    │ ┌─────────────────┐ │
│ Module.HEAP8 │──────refers to──->│ │ Stack  ↓         │ │
│ Module.HEAPU8│                    │ │ ...              │ │
│ Module.HEAPF64                    │ │ Heap   ↑         │ │
│              │                    │ │ (malloc/free)    │ │
│              │                    │ └─────────────────┘ │
└─────────────┘                    └─────────────────────┘
```

---

## Self-Assessment

### Q1: Compile a C++ function to WASM with emcc and call it from JavaScript

**Answer:**

A great practical use case for Wasm is image processing - pixel math is the kind of tight loop that benefits from near-native speed. This example allocates a pixel buffer inside Wasm memory and lets the C++ side crunch through it:

```cpp
// image_process.cpp - image processing in C++, called from JS
#include <emscripten.h>
#include <cstdint>
#include <cmath>

extern "C" {

// Grayscale conversion: process raw RGBA pixel buffer
EMSCRIPTEN_KEEPALIVE
void grayscale(uint8_t* pixels, int width, int height) {
    for (int i = 0; i < width * height * 4; i += 4) {
        uint8_t r = pixels[i];
        uint8_t g = pixels[i + 1];
        uint8_t b = pixels[i + 2];
        // Luminance formula
        uint8_t gray = static_cast<uint8_t>(0.299 * r + 0.587 * g + 0.114 * b);
        pixels[i] = pixels[i + 1] = pixels[i + 2] = gray;
        // Alpha (pixels[i+3]) unchanged
    }
}

// Gaussian blur (simplified 3x3 kernel)
EMSCRIPTEN_KEEPALIVE
void blur(uint8_t* src, uint8_t* dst, int width, int height) {
    for (int y = 1; y < height - 1; ++y) {
        for (int x = 1; x < width - 1; ++x) {
            for (int c = 0; c < 3; ++c) {
                int sum = 0;
                for (int dy = -1; dy <= 1; ++dy)
                    for (int dx = -1; dx <= 1; ++dx)
                        sum += src[((y + dy) * width + (x + dx)) * 4 + c];
                dst[(y * width + x) * 4 + c] = sum / 9;
            }
            dst[(y * width + x) * 4 + 3] = 255;  // Alpha
        }
    }
}

// Allocate/free buffer accessible from JS
EMSCRIPTEN_KEEPALIVE
uint8_t* create_buffer(int size) { return new uint8_t[size]; }

EMSCRIPTEN_KEEPALIVE
void destroy_buffer(uint8_t* buf) { delete[] buf; }

}
```

```bash
emcc image_process.cpp -o image.js \
    -sEXPORTED_FUNCTIONS='["_grayscale","_blur","_create_buffer","_destroy_buffer","_malloc","_free"]' \
    -sEXPORTED_RUNTIME_METHODS='["cwrap"]' \
    -sALLOW_MEMORY_GROWTH -O2
```

On the JavaScript side, notice the data-shuttle pattern: you allocate a buffer inside Wasm memory, copy the canvas pixel data in, call the C++ function, then copy the results back out. The key to this is `Module.HEAPU8`, which is a view into the Wasm linear memory:

```javascript
// JavaScript: apply grayscale to canvas image data
const grayscale = Module.cwrap('grayscale', null, ['number', 'number', 'number']);
const createBuf = Module.cwrap('create_buffer', 'number', ['number']);
const destroyBuf = Module.cwrap('destroy_buffer', null, ['number']);

function applyGrayscale(canvas) {
    const ctx = canvas.getContext('2d');
    const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);

    // Allocate Wasm memory + copy pixels in
    const buf = createBuf(imgData.data.length);
    Module.HEAPU8.set(imgData.data, buf);

    // Process in Wasm (FAST!)
    grayscale(buf, canvas.width, canvas.height);

    // Copy pixels back to JS
    imgData.data.set(Module.HEAPU8.subarray(buf, buf + imgData.data.length));
    ctx.putImageData(imgData, 0, 0);

    destroyBuf(buf);
}
```

### Q2: Use EMSCRIPTEN_BINDINGS to expose a class to JavaScript with automatic type marshaling

**Answer:**

For a more complex object like a `Matrix`, Embind lets you expose the full class API to JavaScript without writing any manual glue. C++ exceptions automatically surface as JavaScript `Error` objects on the JS side, which is a nice touch:

```cpp
// matrix.cpp
#include <emscripten/bind.h>
#include <vector>
#include <stdexcept>
#include <string>
#include <sstream>

class Matrix {
    std::vector<double> data_;
    int rows_, cols_;
public:
    Matrix(int rows, int cols)
        : data_(rows * cols, 0.0), rows_(rows), cols_(cols) {}

    double get(int r, int c) const {
        if (r < 0 || r >= rows_ || c < 0 || c >= cols_)
            throw std::out_of_range("Index out of bounds");
        return data_[r * cols_ + c];
    }

    void set(int r, int c, double val) {
        if (r < 0 || r >= rows_ || c < 0 || c >= cols_)
            throw std::out_of_range("Index out of bounds");
        data_[r * cols_ + c] = val;
    }

    int rows() const { return rows_; }
    int cols() const { return cols_; }

    Matrix multiply(const Matrix& other) const {
        if (cols_ != other.rows_)
            throw std::invalid_argument("Dimension mismatch");
        Matrix result(rows_, other.cols_);
        for (int i = 0; i < rows_; ++i)
            for (int j = 0; j < other.cols_; ++j)
                for (int k = 0; k < cols_; ++k)
                    result.data_[i * other.cols_ + j] +=
                        data_[i * cols_ + k] * other.data_[k * other.cols_ + j];
        return result;
    }

    std::string toString() const {
        std::ostringstream ss;
        for (int i = 0; i < rows_; ++i) {
            for (int j = 0; j < cols_; ++j) {
                if (j > 0) ss << ", ";
                ss << data_[i * cols_ + j];
            }
            ss << "\n";
        }
        return ss.str();
    }
};

EMSCRIPTEN_BINDINGS(matrix_module) {
    emscripten::class_<Matrix>("Matrix")
        .constructor<int, int>()
        .function("get", &Matrix::get)
        .function("set", &Matrix::set)
        .function("rows", &Matrix::rows)
        .function("cols", &Matrix::cols)
        .function("multiply", &Matrix::multiply)
        .function("toString", &Matrix::toString);

    // Register std::vector for JS array interop
    emscripten::register_vector<double>("VectorDouble");
}
```

The JavaScript usage is clean - you get a class that behaves like a normal JS object. Just remember the `.delete()` calls at the end, because Wasm memory is not garbage collected:

```javascript
// JavaScript - automatic type marshaling
const m1 = new Module.Matrix(2, 3);
m1.set(0, 0, 1); m1.set(0, 1, 2); m1.set(0, 2, 3);
m1.set(1, 0, 4); m1.set(1, 1, 5); m1.set(1, 2, 6);

const m2 = new Module.Matrix(3, 2);
m2.set(0, 0, 7); m2.set(0, 1, 8);
m2.set(1, 0, 9); m2.set(1, 1, 10);
m2.set(2, 0, 11); m2.set(2, 1, 12);

const result = m1.multiply(m2);  // Returns Matrix (C++ object wrapped in JS)
console.log(result.toString());
// 58, 64
// 139, 154

// C++ exceptions map to JS Error:
try { m1.get(10, 10); } catch(e) { console.log(e.message); }
// "Index out of bounds"

m1.delete(); m2.delete(); result.delete();  // Must free!
```

### Q3: Explain the memory model: WASM has a linear memory shared with JavaScript via TypedArrays

**Answer:**

This is worth understanding properly because it affects how you move data between C++ and JavaScript. Wasm has a single contiguous block of memory - no separate heap and stack segment from JavaScript's perspective. That block is laid out roughly like this:

```cpp
WebAssembly Linear Memory:
┌────────────────────────────────────────────────────────────┐
│ Address 0                                                    │
│ ┌──────────┐                                                │
│ │ Static    │  Global/static C++ variables                   │
│ │ Data      │  String literals, vtables                      │
│ ├──────────┤                                                │
│ │ Stack ↓   │  Local variables, function frames              │
│ │           │  Grows downward from STACK_BASE                │
│ ├──────────┤                                                │
│ │ (unused)  │                                                │
│ ├──────────┤                                                │
│ │ Heap ↑    │  malloc/new allocations                        │
│ │           │  Grows upward (sbrk)                           │
│ └──────────┘                                                │
│ Address: memory.buffer.byteLength (e.g., 16 MB)             │
└────────────────────────────────────────────────────────────┘

JavaScript sees this as one ArrayBuffer:
  Module.HEAP8     = new Int8Array(buffer)
  Module.HEAPU8    = new Uint8Array(buffer)
  Module.HEAP16    = new Int16Array(buffer)
  Module.HEAPU16   = new Uint16Array(buffer)
  Module.HEAP32    = new Int32Array(buffer)
  Module.HEAPU32   = new Uint32Array(buffer)
  Module.HEAPF32   = new Float32Array(buffer)
  Module.HEAPF64   = new Float64Array(buffer)
```

The important practical consequence is that a C++ pointer is just a byte offset into this buffer. When you read `Module.HEAP32[(ptr >> 2) + i]`, you're dividing the byte offset by 4 to get the 32-bit integer index - it's direct access to the same memory your C++ code is using:

```javascript
// Allocate 100 ints in Wasm memory
const ptr = Module._malloc(100 * 4);  // 4 bytes per int32

// Write to Wasm memory from JS
for (let i = 0; i < 100; i++) {
    Module.HEAP32[(ptr >> 2) + i] = i * i;  // >> 2 converts byte offset to int32 index
}

// Read back from Wasm memory
console.log(Module.HEAP32[(ptr >> 2) + 5]);  // 25

// IMPORTANT: free when done
Module._free(ptr);

// Memory growth warning:
// With -sALLOW_MEMORY_GROWTH:
//   When heap runs out, Emscripten grows the ArrayBuffer
//   WARNING: All TypedArray views (HEAP8, HEAP32, etc.) become INVALIDATED!
//   Always re-read Module.HEAP* after any allocation that might grow memory
```

There are a few rules here that will bite you if you forget them:

1. Wasm memory is a **single contiguous ArrayBuffer** - no separate heap/stack segments from JS's view.
2. JS and Wasm share this buffer - changes on either side are immediately visible to the other.
3. With `ALLOW_MEMORY_GROWTH`, the buffer can be reallocated, which invalidates all existing JS TypedArray views.
4. Default stack size is 64KB; set `-sSTACK_SIZE=1048576` if you need a 1MB stack.
5. C++ pointers are just **byte offsets** into this ArrayBuffer, not JS references.

---

## Notes

- Embind auto-marshals `std::string` to/from JS `string` via UTF-8 - this allocates on each conversion.
- Use `register_vector<T>()` and `register_map<K,V>()` for container interop.
- JS to Wasm string copies are expensive; for hot paths, pass raw pointers and length instead.
- `emscripten::val` lets you manipulate any JS value from C++ (DOM, promises, etc.).
- Default initial memory is 16MB; set with `-sINITIAL_MEMORY=33554432` (32MB).
- Memory is NOT garbage collected - C++ `new`/`delete` rules apply in full.
