# Use LTO (Link-Time Optimization) to enable cross-TU optimization

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://llvm.org/docs/LinkTimeOptimization.html>  

---

## Topic Overview

The fundamental problem LTO solves is that the compiler normally works one translation unit at a time. By the time it finishes compiling `main.cpp`, it has already discarded everything it knew about `util.cpp`. The linker then patches addresses between object files, but it cannot inline a function across that boundary because `util.o` is already machine code - there is nothing left to analyze.

LTO defers that final optimization step. Instead of emitting machine code, the compiler emits its intermediate representation (LLVM IR for Clang, GIMPLE for GCC). The linker then invokes the optimizer on the combined IR from all translation units together, which enables **cross-TU inlining**, **dead code elimination**, and as a bonus, **ODR violation detection**.

```cpp
Normal compilation:          LTO compilation:
a.cpp -> a.o (machine code)  a.cpp -> a.o (LLVM IR / GIMPLE)
b.cpp -> b.o (machine code)  b.cpp -> b.o (LLVM IR / GIMPLE)
     \  /                         \  /
   linker                     LTO optimizer
     |                             |
   binary                       binary (optimized across TUs)
```

---

## Self-Assessment

### Q1: Enable LTO in CMake and measure the effect

The cleanest way to enable LTO in CMake is through the `INTERPROCEDURAL_OPTIMIZATION` target property, combined with a check that the toolchain actually supports it:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(LTOExample LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# util.cpp contains helper functions called from main.cpp
add_executable(app main.cpp util.cpp)

# Enable LTO:
include(CheckIPOSupported)
check_ipo_supported(RESULT supported)
if(supported)
    set_target_properties(app PROPERTIES
        INTERPROCEDURAL_OPTIMIZATION TRUE
    )
endif()
```

Here is the example code that demonstrates the benefit. `main.cpp` calls two functions defined in `util.cpp` - without LTO the calls cannot be inlined; with LTO the entire computation collapses to a constant:

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// util.h
#pragma once
int compute(int x);
bool is_valid(int x);

// util.cpp
#include "util.h"
int compute(int x) { return x * x + 2 * x + 1; }  // (x+1)^2
bool is_valid(int x) { return x >= 0 && x < 1000; }

// main.cpp
#include "util.h"
#include <iostream>

int main() {
    int val = 5;
    if (is_valid(val))
        std::cout << compute(val) << '\n';  // Without LTO: call compute()
                                              // With LTO: cout << 36
}
// Expected output: 36
```

Build both variants and compare the resulting binary sizes:

```bash
# Compare:
g++ -O2       main.cpp util.cpp -o app_nolto && ls -la app_nolto
g++ -O2 -flto main.cpp util.cpp -o app_lto   && ls -la app_lto
# LTO version: smaller binary, compute() and is_valid() inlined into main()
```

### Q2: Why LTO enables cross-TU inlining

The reason this trips people up is the question: "why can't the linker just inline things?" The answer is that the linker traditionally works only at the address-patching level. It sees two blobs of machine code and connects them. There is no way to inline machine code into other machine code after the fact. LTO sidesteps this by never generating machine code until the very end.

Here is the transformation visualized at the IR level:

```cpp
Without LTO:
  main.o: call compute   ; linker resolves to address, but no inlining
  util.o: compute: ...   ; already compiled to machine code

  The linker just patches addresses. It cannot inline because
  util.o is already machine code, not analyzable IR.

With LTO:
  main.o: IR: call compute   ; still in IR form
  util.o: IR: define compute { ... }

  LTO optimizer merges IR:
    define main() {
      %val = 5
      %valid = icmp sge %val, 0   ; is_valid inlined
      %result = 36                 ; compute inlined + constant-folded
      call cout << %result
    }
```

The optimizer sees the whole picture at once and can apply constant folding, dead code elimination, and devirtualization that would be impossible with the per-TU model:

| Without LTO | With LTO |
| --- | --- |
| Each TU compiled independently | All TUs merged at IR level |
| No cross-TU inlining | Functions inlined across TUs |
| Dead functions remain in binary | Dead functions eliminated |
| Virtual calls stay virtual | Devirtualization possible |
| Can't detect ODR violations | Some ODR violations detected |

### Q3: ODR violation detected only with LTO

The One Definition Rule says that if a type or function is defined in multiple translation units, all definitions must be identical. Violations are undefined behavior, and the normal linker cannot detect them because it only sees symbol names, not type layouts. LTO can detect at least some violations because it processes the full IR and can compare type definitions across TUs.

Here is a classic example of an ODR violation that silently compiles and links without LTO:

```cpp
// === a.cpp ===
struct Config {
    int timeout = 30;    // timeout in seconds
};

int get_timeout() {
    Config c;
    return c.timeout;
}

// === b.cpp ===
struct Config {
    double timeout = 5.0;   // ODR VIOLATION! Different definition
};

double get_default() {
    Config c;
    return c.timeout;
}

// === main.cpp ===
#include <iostream>
extern int get_timeout();
extern double get_default();

int main() {
    std::cout << get_timeout() << '\n';
    std::cout << get_default() << '\n';
}
```

```bash
# Without LTO - links fine, silent undefined behavior:
g++ -O2 a.cpp b.cpp main.cpp -o app
./app
# Output: 30 and 5 (appears to work, but UB!)

# With LTO - detects the conflict:
g++ -O2 -flto a.cpp b.cpp main.cpp -o app
# warning: type 'struct Config' has incompatible definitions in different
#          translation units
# Or: error: ODR violation

# Clang with LTO:
clang++ -O2 -flto a.cpp b.cpp main.cpp -o app
# Similar ODR diagnostic
```

Why normal linking misses it:

- The linker only sees machine code in `.o` files.
- Both `Config` types are local (no external linkage symbol conflict).
- LTO sees the full IR and can compare type definitions.

The fact that the non-LTO build "appears to work" is exactly the problem with silent UB. The two `Config` types have different sizes and different member types, so depending on which definition the linker uses for any given call, you might read garbage.

---

## Notes

- Always use LTO for release builds (`CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE`).
- ThinLTO (`-flto=thin`) is faster for large projects with nearly equal optimization.
- LTO objects from different compilers (GCC/Clang) are **incompatible**.
- MSVC equivalent: `/GL` (compile) + `/LTCG` (link).
- LTO significantly increases link time; use ThinLTO cache to mitigate.
