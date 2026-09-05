# Use link-time optimization (LTO) to enable cross-TU inlining and devirtualization

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/LinkTimeOptimization.html>  

---

## Topic Overview

The compiler normally optimizes each translation unit (a `.cpp` file and its headers) in isolation. It has no visibility into other translation units when it compiles yours, so it cannot inline a function from another file, eliminate a virtual call that always resolves to the same override, or remove a function that turns out to be unused across the whole program. Link-time optimization (LTO) removes that constraint by deferring the final optimization pass to link time, when all translation units are visible together.

```cpp
Without LTO:                          With LTO:
  a.cpp -> a.o  \                      a.cpp -> a.o (IR) \
  b.cpp -> b.o   > linker -> binary    b.cpp -> b.o (IR)  > optimizer -> binary
  c.cpp -> c.o  /                      c.cpp -> c.o (IR) /
  (each .o optimized                   (all IR merged,
   independently)                       whole-program optimization)
```

The object files under LTO do not contain machine code - they contain the compiler's intermediate representation (IR). At link time, the linker hands all of that IR to the optimizer at once, which can then see across every function call boundary in the whole program.

| Mode | Full LTO | Thin LTO |
| --- | --- | --- |
| Flag | `-flto` | `-flto=thin` |
| Parallelism | Single-threaded link | Parallel link |
| Build time | Slow (large projects) | Much faster |
| Optimization | Best possible | Nearly as good |
| Memory | High (all IR in memory) | Low (per-module) |

---

## Self-Assessment

### Q1: Enable LTO in CMake and measure impact

CMake has first-class support for LTO through the `INTERPROCEDURAL_OPTIMIZATION` target property. The `CheckIPOSupported` module detects whether your current compiler supports it, so you can handle the fallback gracefully.

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(LTODemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)

# Check LTO support:
include(CheckIPOSupported)
check_ipo_supported(RESULT lto_supported OUTPUT lto_error)

add_executable(myapp main.cpp util.cpp math.cpp)

if(lto_supported)
    set_target_properties(myapp PROPERTIES
        INTERPROCEDURAL_OPTIMIZATION TRUE  # enables LTO
    )
    message(STATUS "LTO enabled")
else()
    message(WARNING "LTO not supported: ${lto_error}")
endif()
```

You can measure the impact directly by building with and without the flag and comparing binary sizes:

```bash
# Build without LTO:
cmake -B build-nolto -DCMAKE_BUILD_TYPE=Release
cmake --build build-nolto
ls -la build-nolto/myapp   # e.g., 245,000 bytes

# Build with LTO:
cmake -B build-lto -DCMAKE_BUILD_TYPE=Release
cmake --build build-lto
ls -la build-lto/myapp     # e.g., 198,000 bytes  (~20% smaller)

# Manual flags (GCC/Clang):
g++ -O2 -flto main.cpp util.cpp math.cpp -o myapp_lto
g++ -O2       main.cpp util.cpp math.cpp -o myapp_nolto

# ThinLTO (Clang only, faster builds):
clang++ -O2 -flto=thin main.cpp util.cpp math.cpp -o myapp_thinlto
```

The binary size reduction happens because LTO can see the whole program at once and eliminate code that is unreachable or can be inlined away. The actual performance improvement varies by workload, but programs with many small functions spread across translation units benefit the most.

### Q2: Full LTO vs ThinLTO

Full LTO and ThinLTO reach similar optimization results through very different processes. Understanding the trade-off helps you choose the right one for a given situation.

```cpp
Full LTO pipeline:
  a.cpp -> a.bc \
  b.cpp -> b.bc  > merge all IR -> single optimization pass -> codegen
  c.cpp -> c.bc /  (sequential, memory-intensive)

ThinLTO pipeline:
  a.cpp -> a.bc \   analyze
  b.cpp -> b.bc  > thin link -> per-module optimization (parallel!) -> codegen
  c.cpp -> c.bc /   (summaries only, low memory)
```

| Aspect | Full LTO | ThinLTO |
| --- | --- | --- |
| Build speed | Slow (sequential) | Fast (parallel) |
| Memory usage | High (all IR loaded) | Low (summaries + on-demand) |
| Optimization quality | Best | 95-99% of full LTO |
| Incremental builds | None (must redo all) | Partial (only changed modules) |
| Best for | Small projects, final release | Large projects, CI |

ThinLTO's key insight is that most cross-module optimization decisions only require a lightweight summary of each module, not the full IR. The summaries are computed quickly, distributed to per-module optimization jobs that run in parallel, and each job only loads the full IR of the modules it actually needs. For large codebases this can cut link time by an order of magnitude compared to full LTO.

```bash
# ThinLTO with parallelism:
clang++ -O2 -flto=thin -Wl,--thinlto-jobs=8 main.cpp util.cpp -o app

# CMake with ThinLTO:
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -flto=thin")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} -flto=thin")

# Cache ThinLTO results:
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--thinlto-cache-dir=thinlto-cache")
```

The ThinLTO cache directory stores the results of previous link-time optimizations so that unchanged modules do not need to be reprocessed on the next build - this gives you something approaching incremental build behavior even with LTO enabled.

### Q3: Devirtualization with LTO

Devirtualization is one of the most compelling reasons to enable LTO in programs with polymorphism. A virtual call normally requires an indirect dispatch through the vtable at runtime. If LTO can prove that the dynamic type of an object is always the same concrete class, it can replace the indirect call with a direct call - and then inline that call and constant-fold through it.

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// === shape.h ===
struct Shape {
    virtual int area() const = 0;
    virtual ~Shape() = default;
};

// === circle.cpp ===
#include "shape.h"
struct Circle : Shape {
    int r;
    Circle(int r) : r(r) {}
    int area() const override { return 3 * r * r; }  // simplified
};
Shape* make_circle(int r) { return new Circle(r); }

// === main.cpp ===
#include "shape.h"
#include <iostream>
extern Shape* make_circle(int r);

int main() {
    Shape* s = make_circle(5);
    int a = s->area();         // virtual call
    std::cout << a << '\n';
    delete s;
}
```

The difference in generated code is dramatic:

```bash
# Without LTO - virtual call remains:
$ g++ -O2 -c circle.cpp -o circle.o
$ g++ -O2 -c main.cpp -o main.o
$ g++ main.o circle.o -o app_nolto
# Assembly: call [vtable + offset]  <-- indirect call through vtable

# With LTO - devirtualized:
$ g++ -O2 -flto circle.cpp main.cpp -o app_lto
# Assembly: mov eax, 75   <-- computed at compile time (3*5*5=75)
#           No vtable lookup, no indirect call

# Why: LTO sees that make_circle always returns Circle,
# so area() always calls Circle::area(). The virtual call
# is devirtualized and then the function is inlined and
# constant-folded
```

Without LTO, the compiler compiles `main.cpp` without knowing what `make_circle` returns - it only knows it returns `Shape*`, so the virtual dispatch is unavoidable. With LTO, the optimizer can see both files at once and figure out that `make_circle` always constructs a `Circle`. From there, `s->area()` can be devirtualized to `Circle::area()`, inlined, and the whole expression evaluated at compile time to the constant 75.

---

## Notes

- LTO requires that all object files were compiled with the same `-flto` flag. Mixing LTO and non-LTO objects at link time will work, but the non-LTO objects will not benefit from cross-module optimization.
- GCC and Clang LTO IR formats are not interchangeable - you cannot mix object files from the two compilers in an LTO build.
- Static libraries work well with LTO; shared libraries have limited benefit because the optimizer cannot assume who will call into the library.
- Combining debug info with LTO via `-g -flto` is supported but significantly increases link time.
- MSVC uses `/GL` at compile time and `/LTCG` at link time for its equivalent of LTO.
