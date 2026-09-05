# Link C and C++ objects and match calling conventions on Windows

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/language_linkage>  

---

## Topic Overview

This second topic on C interop focuses on **practical cross-language linking**: exporting C++ functions to C consumers, reading symbol tables, and understanding which calling convention to use on Windows.

The big idea is that C and C++ can share compiled object files, but only if they agree on symbol names and calling conventions. This file shows you how to make that work reliably.

### Linking Workflow

Here is the full picture of how a C main program links against a C++ library. Notice that the key requirement is that the symbol name in the C++ object file (`math.o`) must match what the C object file (`main.o`) expects - and `extern "C"` is the bridge that makes that happen.

```cpp
┌─────────────────┐     g++ -c         ┌──────────────┐
│  math.cpp       │ ──────────────────► │  math.o      │
│  extern "C"     │                     │  symbol: add │
│  int add(...)   │                     └──────┬───────┘
└─────────────────┘                            │
                                         ld / link.exe
┌─────────────────┐     gcc -c          ┌──────┴───────┐
│  main.c         │ ──────────────────► │  main.o      │
│  calls add()    │                     │  ref: add    │
└─────────────────┘                     └──────┬───────┘
                                               │
                                        ┌──────▼───────┐
                                        │  a.out / .exe│
                                        └──────────────┘
```

### Calling Conventions on Windows

Calling conventions are agreements between caller and callee about how arguments get passed and who cleans up the stack afterward. If you get this wrong on x86 Windows, you will get silent stack corruption - one of the worst bugs to debug. The table below summarises the conventions you will encounter. If the table feels like a lot, just remember: on x64, none of this matters because there is only one convention.

| Convention | Stack Cleanup | Arguments | Variadic | Used By |
| --- | :---: | :---: | :---: | --- |
| `__cdecl` | Caller | Stack (x86), RCX/RDX/R8/R9 (x64) | Yes | Default C/C++ |
| `__stdcall` | Callee | Stack | No | Win32 API, COM |
| `__fastcall` | Callee | ECX, EDX + stack | No | Performance-critical |
| `__vectorcall` | Callee | XMM0-5 + registers | No | SIMD-heavy code |
| `__thiscall` | Callee | ECX = `this` + stack | No | C++ member functions |

> On x64 Windows, `__cdecl` is the only convention - `__stdcall`/`__fastcall` are silently ignored.

---

## Self-Assessment

### Q1: Use extern "C" to export a C++ function with C linkage and call it from a C program

**Answer:**

The trick here is a shared header that works in both C and C++. When a C++ compiler sees it, the `#ifdef __cplusplus` guard wraps everything in `extern "C"`. When a C compiler sees it, those guards disappear and it reads as plain C. This is the standard pattern for writing cross-language APIs.

```cpp
// ═══════════ stringlib.h — shared between C and C++ ═══════════
#ifndef STRINGLIB_H
#define STRINGLIB_H

#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

// String utilities with C linkage
size_t string_length(const char* s);
int    string_compare(const char* a, const char* b);
char*  string_duplicate(const char* s);  // caller must free()
void   string_free(char* s);

#ifdef __cplusplus
}
#endif

#endif
```

The C++ implementation can use the full standard library internally - C++ features are fine inside the function bodies. The `extern "C"` block only controls the exported symbol names, not what code is allowed inside.

```cpp
// ═══════════ stringlib.cpp — C++ implementation ═══════════
#include "stringlib.h"
#include <cstring>
#include <cstdlib>
#include <algorithm>

extern "C" {

size_t string_length(const char* s) {
    if (!s) return 0;
    return std::strlen(s);  // use C++ std library internally
}

int string_compare(const char* a, const char* b) {
    if (!a || !b) return a ? 1 : (b ? -1 : 0);
    return std::strcmp(a, b);
}

char* string_duplicate(const char* s) {
    if (!s) return nullptr;
    size_t len = std::strlen(s) + 1;
    char* copy = static_cast<char*>(std::malloc(len));
    if (copy) std::memcpy(copy, s, len);
    return copy;
}

void string_free(char* s) {
    std::free(s);
}

}  // extern "C"
```

The C consumer includes the same header and calls the functions without any knowledge that C++ is involved. From C's perspective, these are just regular C functions.

```c
// ═══════════ main.c — Pure C consumer ═══════════
#include "stringlib.h"
#include <stdio.h>

int main(void) {
    const char* hello = "Hello, World!";

    printf("Length: %zu\n", string_length(hello));
    // Output: Length: 13

    char* copy = string_duplicate(hello);
    printf("Copy: %s\n", copy);
    printf("Compare: %d\n", string_compare(hello, copy));
    // Output: Copy: Hello, World!
    // Output: Compare: 0

    string_free(copy);
    return 0;
}
```

Compiling requires two separate steps: compile the C++ implementation with `g++`, then link it together with the C main using `gcc`. The `-lstdc++` flag brings in the C++ runtime.

```bash
# Build and run:
g++ -c stringlib.cpp -o stringlib.o      # Compile C++ as object
gcc main.c stringlib.o -lstdc++ -o app   # Link C main with C++ object
./app
```

### Q2: Show the mangled vs unmangled symbol using nm on the compiled object

**Answer:**

To really appreciate what `extern "C"` does, it helps to look directly at the symbol table. This example defines both mangled and unmangled versions of a function so you can see the contrast side-by-side.

```cpp
// symbols_demo.cpp
#include <cmath>

// Regular C++ function — will be mangled
double compute(double x, int n) {
    return std::pow(x, n);
}

// Overloaded — different mangled names
double compute(double x) {
    return x * x;
}

// extern "C" — unmangled
extern "C" double c_compute(double x, int n) {
    return std::pow(x, n);
}
```

After compiling, `nm` shows the raw symbol names in the object file. The mangled names encode the full function signature: `_Z7computed` means "compute taking a double", `_Z7computedi` means "compute taking a double and an int". The `c++filt` tool decodes them back to human-readable form. Notice that `c_compute` is identical in all views - that is the whole point.

```bash
# Compile and inspect:
g++ -c symbols_demo.cpp -o symbols_demo.o

# ═══════════ View raw symbols ═══════════
nm symbols_demo.o | grep compute
# 0000000000000000 T _Z7computed       <- compute(double) mangled
# 0000000000000020 T _Z7computedi      <- compute(double, int) mangled
# 0000000000000040 T c_compute         <- unmangled

# ═══════════ Demangle to see original names ═══════════
nm symbols_demo.o | c++filt | grep compute
# 0000000000000000 T compute(double)
# 0000000000000020 T compute(double, int)
# 0000000000000040 T c_compute

# ═══════════ On Windows with MSVC ═══════════
# dumpbin /symbols symbols_demo.obj | findstr compute
# ?compute@@YANN@Z          <- MSVC mangling for compute(double)
# ?compute@@YANNH@Z         <- MSVC mangling for compute(double, int)
# c_compute                 <- unmangled

# ═══════════ Mangling scheme comparison ═══════════
#
# Function:          GCC/Clang:           MSVC:
# compute(double)    _Z7computed          ?compute@@YANN@Z
# compute(double,int) _Z7computedi       ?compute@@YANNH@Z
# extern "C"         c_compute            c_compute
#                    (identical across all compilers)
```

The mangling schemes between GCC/Clang and MSVC are completely different, which is why you cannot link a GCC-compiled object with MSVC-compiled code using C++ symbols. `extern "C"` is the only cross-compiler-portable way to expose a stable symbol name.

### Q3: Explain calling convention differences (cdecl, stdcall, fastcall) and when they matter on Windows

**Answer:**

The diagrams below show what actually happens at the assembly level. The key difference is who adjusts the stack pointer after the call - the caller or the callee. Getting this wrong means the stack pointer ends up at the wrong place after the call returns, corrupting everything that follows.

```cpp
__cdecl (default C/C++):
┌────────┐   push arg2   ┌──────────┐
│ Caller │──push arg1──►│  Callee  │
│        │   call func   │          │
│        │◄──────────────│  ret     │
│ cleans │   add esp, 8  └──────────┘
│ stack  │
└────────┘

__stdcall (Win32 API):
┌────────┐   push arg2   ┌──────────┐
│ Caller │──push arg1──►│  Callee  │
│        │   call func   │  cleans  │
│        │◄──────────────│  ret 8   │  <- callee pops args
└────────┘               └──────────┘

__fastcall:
┌────────┐   ECX=arg1    ┌──────────┐
│ Caller │──EDX=arg2───►│  Callee  │
│        │   call func   │  cleans  │
│        │◄──────────────│  ret     │
└────────┘               └──────────┘
```

Here are the practical situations where conventions actually matter. Pay special attention to function pointer typedefs - if the convention keyword is wrong on a pointer, the call will silently corrupt the stack.

```cpp
// ═══════════ 1. Calling Win32 API (always __stdcall) ═══════════
#include <windows.h>

// Win32 API declared as __stdcall via WINAPI macro:
// BOOL WINAPI CloseHandle(HANDLE);  <- expands to __stdcall

// If you get the convention wrong (e.g., function pointer):
typedef BOOL (__stdcall *CloseHandleFn)(HANDLE);  // CORRECT
// typedef BOOL (__cdecl *CloseHandleFn)(HANDLE);  // WRONG — stack corruption!

// ═══════════ 2. DLL exports: specify explicitly ═══════════
// mylib.h
#ifdef MYLIB_EXPORTS
#define MYLIB_API __declspec(dllexport)
#else
#define MYLIB_API __declspec(dllimport)
#endif

// Use __cdecl (default) for C interop — caller cleans up, supports variadic:
extern "C" MYLIB_API int __cdecl my_function(int x);

// Use __stdcall for COM-style interfaces:
extern "C" MYLIB_API HRESULT __stdcall DllGetClassObject(
    REFCLSID rclsid, REFIID riid, LPVOID* ppv);

// ═══════════ 3. Callback function pointers — must match! ═══════════
// Windows callbacks are __stdcall:
LRESULT CALLBACK WindowProc(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp) {
    // CALLBACK = __stdcall
    return DefWindowProc(hwnd, msg, wp, lp);
}

// qsort callback is __cdecl:
int compare(const void* a, const void* b) {
    // __cdecl by default — matches what qsort expects
    return *(const int*)a - *(const int*)b;
}

// ═══════════ 4. x64: conventions don't matter ═══════════
// On x64 Windows there is ONLY one convention (Microsoft x64):
//   - First 4 args: RCX, RDX, R8, R9 (or XMM0-3 for floats)
//   - Remaining: stack
//   - Caller cleans up
//   - __stdcall, __cdecl, __fastcall all compile to the same code
//
// On x64 Linux (System V AMD64):
//   - First 6 integer args: RDI, RSI, RDX, RCX, R8, R9
//   - First 8 float args: XMM0-7
//   - Calling convention differences are x86-only concerns
```

The x64 note is important: if you only ever target 64-bit platforms, calling convention keywords in your source code are essentially decorative - they all resolve to the same ABI. The risk is real only when you support 32-bit x86.

---

## Notes

- On x86 Windows, mismatched conventions cause **stack corruption** (hard to debug).
- On x64, all conventions compile identically - convention keywords are ignored.
- Use `__declspec(dllexport)` + `.def` files for DLL exports on Windows.
- Linux uses System V AMD64 ABI universally - no convention mismatches possible.
- `WINAPI`, `CALLBACK`, and `APIENTRY` are all macros that expand to `__stdcall`.
- Always put the explicit convention keyword on function pointer typedefs to prevent silent mismatches.
