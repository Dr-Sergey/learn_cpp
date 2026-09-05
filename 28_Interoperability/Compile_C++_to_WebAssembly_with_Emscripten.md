# Compile C++ to WebAssembly with Emscripten

**Category:** Interoperability  
**Standard:** C++20  
**Reference:** <https://emscripten.org/>  

---

## Topic Overview

**Emscripten** is a toolchain that compiles C/C++ to **WebAssembly (Wasm)**, letting you run native code directly in the browser at near-native speed. It provides a full POSIX-like environment, ships libc/libc++, and includes bridges to JavaScript APIs so you don't have to build that glue yourself.

### Compilation Pipeline

The pipeline goes from your C++ source all the way down to a `.wasm` binary and a JavaScript loader, handled entirely by the `emcc` driver:

```cpp
   C++ source (.cpp)
        │
        ▼  emcc (Clang/LLVM)
   LLVM IR (.bc)
        │
        ▼  LLVM Wasm backend
   WebAssembly (.wasm) + JavaScript glue (.js)
        │
        ▼  Browser / Node.js
   Runs in Wasm VM
```

### Key Emscripten Features

Here's a quick reference for the major features you'll reach for most often:

| Feature | Flag / API | Purpose |
| --- | --- | --- |
| `cwrap`/`ccall` | `-sEXPORTED_FUNCTIONS` | Call C-style functions from JS |
| Embind | `--bind` | Expose C++ classes to JS |
| Asyncify | `-sASYNCIFY` | Async JS calls from sync C++ |
| File system | `-sFORCE_FILESYSTEM` | Virtual FS (MEMFS, IDBFS) |
| Threads | `-pthread` | SharedArrayBuffer + Web Workers |

---

## Self-Assessment

### Q1: Compile a C++ library to .wasm with emcc and call it from JavaScript using cwrap

**Answer:**

The simplest path into Wasm is wrapping plain C-style functions with `extern "C"` and marking them with `EMSCRIPTEN_KEEPALIVE` so the dead-code eliminator doesn't strip them out. Here's a small math library:

**C++ source (math_lib.cpp):**

```cpp
#include <emscripten.h>
#include <cmath>

// EMSCRIPTEN_KEEPALIVE prevents dead code elimination
extern "C" {

EMSCRIPTEN_KEEPALIVE
int factorial(int n) {
    if (n <= 1) return 1;
    int result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}

EMSCRIPTEN_KEEPALIVE
double distance(double x1, double y1, double x2, double y2) {
    double dx = x2 - x1, dy = y2 - y1;
    return std::sqrt(dx * dx + dy * dy);
}

EMSCRIPTEN_KEEPALIVE
int fibonacci(int n) {
    if (n <= 0) return 0;
    if (n == 1) return 1;
    int a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        int tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}

}  // extern "C"
```

Compile it with `emcc`. The `-sEXPORTED_FUNCTIONS` list tells Emscripten which symbols to keep, and each name gets a leading underscore because that's how C symbols appear at the Wasm level:

**Compile:**

```bash
emcc math_lib.cpp -o math_lib.js \
    -sEXPORTED_FUNCTIONS='["_factorial","_distance","_fibonacci"]' \
    -sEXPORTED_RUNTIME_METHODS='["cwrap","ccall"]' \
    -O2
# Produces: math_lib.js + math_lib.wasm
```

On the JavaScript side, `cwrap` creates a reusable wrapper function so you don't have to type argument types on every call. `ccall` is useful when you only need to call something once. Both are only available after the Wasm module finishes initializing, which is why everything goes inside `onRuntimeInitialized`:

**JavaScript usage:**

```html
<script src="math_lib.js"></script>
<script>
Module.onRuntimeInitialized = () => {
    // cwrap: create a reusable JS function wrapper
    const factorial = Module.cwrap('factorial', 'number', ['number']);
    const distance  = Module.cwrap('distance', 'number',
                                   ['number', 'number', 'number', 'number']);

    console.log(factorial(10));            // 3628800
    console.log(distance(0, 0, 3, 4));    // 5.0

    // ccall: one-shot call (no wrapper)
    const fib = Module.ccall('fibonacci', 'number', ['number'], [20]);
    console.log(fib);                      // 6765
};
</script>
```

### Q2: Use Embind to expose a C++ class as a JavaScript class with Emscripten

**Answer:**

`cwrap` only works with free C functions. When you want to expose a full C++ class - with constructors, properties, and methods - Embind is the right tool. You register the class in a special `EMSCRIPTEN_BINDINGS` block, and Emscripten generates all the glue automatically:

```cpp
// vector2d.cpp
#include <emscripten/bind.h>
#include <cmath>
#include <string>

class Vector2D {
public:
    double x, y;

    Vector2D() : x(0), y(0) {}
    Vector2D(double x, double y) : x(x), y(y) {}

    double length() const { return std::sqrt(x * x + y * y); }

    Vector2D normalized() const {
        double len = length();
        if (len == 0) return {0, 0};
        return {x / len, y / len};
    }

    Vector2D operator+(const Vector2D& o) const { return {x + o.x, y + o.y}; }
    Vector2D operator*(double s) const { return {x * s, y * s}; }

    double dot(const Vector2D& o) const { return x * o.x + y * o.y; }

    std::string to_string() const {
        return "(" + std::to_string(x) + ", " + std::to_string(y) + ")";
    }
};

// Embind registration - maps C++ names to JS names
EMSCRIPTEN_BINDINGS(vector_module) {
    emscripten::class_<Vector2D>("Vector2D")
        .constructor<>()
        .constructor<double, double>()
        .property("x", &Vector2D::x)
        .property("y", &Vector2D::y)
        .function("length", &Vector2D::length)
        .function("normalized", &Vector2D::normalized)
        .function("add", &Vector2D::operator+)        // Rename operator
        .function("scale", &Vector2D::operator*)
        .function("dot", &Vector2D::dot)
        .function("toString", &Vector2D::to_string);
}
```

Compile with `--bind` to activate Embind:

```bash
emcc vector2d.cpp -o vector2d.js --bind -O2
```

On the JS side, `Vector2D` behaves like a native class. The key thing to remember is that each object lives in Wasm's heap, not JS's garbage-collected heap, so you must call `.delete()` when you're done or you'll leak memory:

```javascript
// JavaScript - Vector2D is a real JS class!
Module.onRuntimeInitialized = () => {
    const v1 = new Module.Vector2D(3, 4);
    console.log(v1.length());       // 5.0
    console.log(v1.toString());     // "(3.000000, 4.000000)"

    const v2 = new Module.Vector2D(1, 0);
    const v3 = v1.add(v2);         // Vector2D(4, 4)
    console.log(v3.x, v3.y);      // 4, 4

    const norm = v1.normalized();
    console.log(norm.length());    // ~1.0

    // IMPORTANT: prevent memory leaks - delete C++ objects
    v1.delete();
    v2.delete();
    v3.delete();
    norm.delete();
};
```

### Q3: Explain the Asyncify transform and how it enables co_await-style async in Wasm C++

**Answer:**

This one's a bit mind-bending. WebAssembly executes synchronously by design - there's no native `sleep()` or blocking I/O. If you call `sleep(1000)` in a normal Wasm build, the entire browser tab freezes for a second. Asyncify solves this by transforming your compiled code so it can pause execution, return control to the JavaScript event loop, and then resume from exactly where it left off once the async operation completes.

Here's the conceptual difference:

```cpp
Without Asyncify:
  C++ calls sleep(1000) -> ENTIRE browser tab freezes for 1s

With Asyncify:
  C++ calls sleep(1000) -> Asyncify pauses C++ execution
                         -> Returns control to JS event loop
                         -> setTimeout(1000ms) fires
                         -> Asyncify resumes C++ from where it paused
```

The reason this is possible is that Asyncify instruments every function with save/restore logic for the call stack and local variables. When an async trigger fires, Asyncify unwinds the C++ stack by saving everything to memory. When the async operation completes, it reloads that saved state and picks up right where it left off:

```cpp
1. Compiler instruments every function call with save/restore state
2. When async operation triggered:

   ┌──────────────┐
   │ C++ function  │
   │ ...           │
   │ fetch(url)    │ <- triggers async
   │ ...           │
   └──────┬───────┘
          │ save stack + locals to memory
          ▼
   Return to JS event loop
          │ ... network request completes ...
          ▼
   Restore stack + locals from memory
   ┌──────────────┐
   │ C++ resumes   │
   │ at fetch()+1  │
   │ ...           │
   └──────────────┘
```

To your C++ code, this looks completely synchronous. The `emscripten_sleep` call below pauses for 2 seconds without freezing the browser, and the fetch happens without callbacks:

```cpp
// async_example.cpp
#include <emscripten.h>
#include <emscripten/fetch.h>
#include <cstdio>

// This C++ code looks synchronous, but Asyncify makes it non-blocking!
extern "C" EMSCRIPTEN_KEEPALIVE void run_demo() {
    printf("Step 1: starting\n");

    // emscripten_sleep pauses C++ but doesn't freeze the browser
    emscripten_sleep(2000);  // "sleep" 2 seconds (non-blocking)

    printf("Step 2: after sleep\n");

    // Fetch URL synchronously (Asyncify unwinding happens here)
    emscripten_fetch_attr_t attr;
    emscripten_fetch_attr_init(&attr);
    attr.attributes = EMSCRIPTEN_FETCH_LOAD_TO_MEMORY | EMSCRIPTEN_FETCH_SYNCHRONOUS;
    strcpy(attr.requestMethod, "GET");

    emscripten_fetch_t* fetch = emscripten_fetch(&attr, "https://api.example.com/data");
    printf("Step 3: got %llu bytes\n", fetch->numBytes);
    emscripten_fetch_close(fetch);
}
```

```bash
emcc async_example.cpp -o async.js \
    -sASYNCIFY \
    -sASYNCIFY_STACK_SIZE=65536 \
    -sFETCH \
    -O2
```

**Asyncify cost:** roughly 10% code size increase and about 5% runtime overhead for instrumented functions. It's not free, but for code that needs to do blocking-style async work in a browser, it's a very clean solution.

---

## Notes

- `emcc` is a drop-in replacement for `gcc`/`clang` - it accepts the same flags plus Emscripten-specific `-s` options.
- Use `-O2` or `-O3` for production; use `-g` for debugging with DWARF info.
- Embind objects MUST be `.delete()`d from JS to avoid C++ memory leaks.
- `-sALLOW_MEMORY_GROWTH` enables dynamic heap resizing (slight performance cost).
- Emscripten supports most of C++20 (std::format, ranges, concepts) via libc++.
- For file I/O, Emscripten provides MEMFS (in-memory), IDBFS (IndexedDB), and NODEFS (Node.js).
