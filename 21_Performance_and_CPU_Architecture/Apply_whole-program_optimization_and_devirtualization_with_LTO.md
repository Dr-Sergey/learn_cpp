# Apply whole-program optimization and devirtualization with LTO

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/LinkTimeOptimization.html>  

---

## Topic Overview

When you compile a C++ project with separate translation units, the compiler optimizes each `.cpp` file in isolation. That means it can only inline a function if it can see the function body in the same translation unit - and it can only devirtualize a virtual call if it can prove at compile time which concrete type is being used. Both of those are often impossible when your program is split across files.

Link-Time Optimization (LTO) breaks that boundary. Instead of emitting machine code during compilation, the compiler emits an intermediate representation (IR). At link time, the linker hands all that IR to the optimizer as one giant program, enabling cross-file inlining, dead code elimination, and - with the right flags - whole-program devirtualization of virtual calls.

| Feature | Without LTO | With LTO |
| --- | --- | --- |
| Inlining | Within TU only | Cross-TU |
| Devirtualization | Limited to visible classes | Whole-program analysis |
| Dead code elimination | Per-TU | Whole-program |
| Build time | Faster | Slower (2-3x) |
| Binary size | Larger | Smaller (DCE) |

There are two modes worth knowing. Full LTO merges everything into a single giant compilation unit; ThinLTO does a smarter summary-based approach that runs in parallel and scales much better on large codebases.

| LTO Mode | Description |
| --- | --- |
| `-flto` (Full) | Merges all TUs into one, full optimization |
| `-flto=thin` | Parallel, scalable LTO with summary-based optimization |

---

## Self-Assessment

### Q1: Enable ThinLTO with whole-program vtables

This example shows the most impactful case: a virtual call that the compiler cannot devirtualize without LTO because the concrete type lives in a different file. The build command comments walk you through how to see the difference in the generated assembly.

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// === file: shape.h ===
#pragma once
struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

// === file: circle.cpp ===
#include "shape.h"
#include <cmath>
struct Circle : Shape {
    double radius;
    Circle(double r) : radius(r) {}
    double area() const override { return std::numbers::pi * radius * radius; }
};
Shape* make_circle(double r) { return new Circle(r); }

// === file: main.cpp ===
#include "shape.h"
#include <iostream>
extern Shape* make_circle(double r);

int main() {
    Shape* s = make_circle(5.0);
    // Without LTO: virtual call (vtable lookup)
    // With LTO:    devirtualized to Circle::area() direct call
    std::cout << s->area() << '\n';  // 78.5398
    delete s;
}

// Build commands:
//
// Without LTO:
//   g++ -O2 -c circle.cpp -o circle.o
//   g++ -O2 -c main.cpp -o main.o
//   g++ -O2 circle.o main.o -o app
//   // s->area() = virtual call through vtable
//
// With ThinLTO:
//   clang++ -O2 -flto=thin -fwhole-program-vtables -c circle.cpp -o circle.o
//   clang++ -O2 -flto=thin -fwhole-program-vtables -c main.cpp -o main.o
//   clang++ -O2 -flto=thin -fwhole-program-vtables circle.o main.o -o app
//   // s->area() devirtualized to direct call!
//
// Verify with:
//   objdump -d app | grep -A5 'area'
```

With LTO enabled, the linker can see that `make_circle` is the only factory function and `Circle` is the only concrete `Shape` in the whole program. That lets it replace the vtable dispatch with a plain direct call - and potentially inline `Circle::area()` into `main` entirely.

### Q2: LTO enables cross-TU inlining

Even without virtual calls, LTO gives you a big win on any small utility function that lives in its own file. Without LTO the compiler sees only an `extern` declaration and has to emit a real function call. With LTO it can see the body and inline it directly into the hot loop.

```cpp
// === file: math_utils.cpp ===
int square(int x) { return x * x; }
int cube(int x) { return x * x * x; }

// === file: main.cpp ===
#include <iostream>
extern int square(int);
extern int cube(int);

int main() {
    int result = 0;
    for (int i = 0; i < 1000000; ++i) {
        result += square(i) + cube(i % 10);
    }
    std::cout << result << '\n';
}

// WITHOUT LTO:
// - square() and cube() are in a different TU
// - Compiler sees only declarations (extern)
// - Cannot inline -> function call overhead per iteration
// - Assembly: call square; call cube;
//
// WITH LTO (-flto):
// - Linker passes all IR to optimizer
// - Optimizer sees function bodies across TUs
// - Inlines square() and cube() into the loop
// - Assembly: imul + imul (no function calls)
//
// Performance difference for tight loops: 2-5x speedup
//
// Build:
//   g++ -O2 -flto -c math_utils.cpp -o math_utils.o
//   g++ -O2 -flto -c main.cpp -o main.o
//   g++ -O2 -flto math_utils.o main.o -o app
//
// ThinLTO (parallel, better for large projects):
//   clang++ -O2 -flto=thin -c math_utils.cpp -o math_utils.o
//   clang++ -O2 -flto=thin -c main.cpp -o main.o
//   clang++ -O2 -flto=thin math_utils.o main.o -o app
```

For a loop that calls `square` and `cube` a million times, replacing two function calls per iteration with two multiply instructions is easily a 2-5x speedup - and that is just one hot loop. Real programs benefit across many call sites simultaneously.

### Q3: Virtual call devirtualized only with LTO

This is the classic scenario where LTO pays off most dramatically: a tight loop that calls a virtual method a million times through a base-class pointer. Without LTO, each call goes through the vtable. With LTO and `-fwhole-program-vtables`, the optimizer can prove only one concrete type exists and replace all the dispatches with direct calls (or even inline them).

```cpp
// === file: logger.h ===
#pragma once
#include <string>
struct Logger {
    virtual void log(const std::string& msg) = 0;
    virtual ~Logger() = default;
};

// === file: console_logger.cpp ===
#include "logger.h"
#include <iostream>
struct ConsoleLogger : Logger {
    void log(const std::string& msg) override {
        std::cout << msg << '\n';
    }
};
Logger* create_logger() { return new ConsoleLogger(); }

// === file: app.cpp ===
#include "logger.h"
extern Logger* create_logger();

void run() {
    Logger* logger = create_logger();
    // This loop calls log() 1M times:
    for (int i = 0; i < 1000000; ++i) {
        logger->log("tick");
    }
    delete logger;
}

// WITHOUT LTO:
// - Compiler in app.cpp sees Logger* (abstract base)
// - Cannot know the concrete type is ConsoleLogger
// - Every call: load vtable ptr -> load function ptr -> indirect call
// - Cost: ~5ns per call overhead (branch mispredict + icache miss)
//
// WITH LTO + -fwhole-program-vtables:
// - Linker sees ALL classes derived from Logger
// - Only one: ConsoleLogger
// - Devirtualizes: logger->log() becomes ConsoleLogger::log()
// - May even INLINE ConsoleLogger::log() into the loop
// - Cost: 0ns overhead (direct call or inlined)
//
// Verify devirtualization:
//   clang++ -O2 -flto=thin -fwhole-program-vtables \
//           -Rpass=devirt app.cpp console_logger.cpp -o app
//   // Remark: devirtualized call to ConsoleLogger::log
```

The 5 ns overhead per virtual call comes from a chain of three memory indirections: load the vtable pointer from the object, load the function pointer from the vtable, then do an indirect call through that pointer. That last step is an indirect branch, which the CPU's branch target buffer may mispredict. Over a million iterations those nanoseconds add up fast.

---

## Notes

- ThinLTO (`-flto=thin`) scales better than full LTO for large projects.
- `-fwhole-program-vtables` is Clang-specific; GCC uses `-fdevirtualize` (on by default).
- LTO increases link time significantly; use for release builds only.
- CMake: `set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)` enables LTO.
- `-Rpass=devirt` (Clang) reports which calls were devirtualized.
- Full LTO can reduce binary size by 10-20% through cross-TU dead code elimination.
