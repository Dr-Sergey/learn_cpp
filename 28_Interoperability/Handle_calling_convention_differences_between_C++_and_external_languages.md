# Handle calling convention differences between C++ and external languages

**Category:** Interoperability  
**Standard:** Not version-specific  
**Reference:** <https://docs.microsoft.com/en-us/cpp/cpp/argument-passing-and-naming-conventions>  

---

## Topic Overview

A **calling convention** defines how functions receive parameters and return results at the binary level: which arguments go in registers, which go on the stack, who is responsible for cleaning the stack after the call, and how the function's name is decorated in the binary. This is one of those topics that feels academic until you get a crash or silent data corruption from a mismatch - at that point it becomes urgent.

Mismatched calling conventions between C++ and external languages cause **stack corruption, crashes, or silent data corruption** because both sides disagree on how the stack frame is supposed to look.

### Calling Conventions on Windows

On 32-bit Windows, calling conventions are a real concern because there are several in common use and the caller/callee disagree on who cleans up the stack:

| Convention | Push Order | Stack Cleanup | Registers | Use Case |
| --- | --- | --- | --- | --- |
| `__cdecl` | Right->Left | **Caller** | None | Default C/C++, variadic functions |
| `__stdcall` | Right->Left | **Callee** | None | Win32 API, COM |
| `__fastcall` | Right->Left | **Callee** | ECX, EDX (first 2 args) | Performance-critical |
| `__thiscall` | Right->Left | **Callee** | ECX = `this` | C++ member functions (MSVC) |
| `__vectorcall` | Right->Left | **Callee** | XMM0-XMM5 | SIMD/floating-point heavy |

### x86-64 (64-bit) Conventions

On 64-bit, life is simpler - each platform uses a single convention, and the main difference is in which registers get used for the first few arguments:

| Platform | Integer Args | Float Args | Caller/Callee Cleanup |
| --- | --- | --- | --- |
| **Windows x64** | RCX, RDX, R8, R9 | XMM0-XMM3 | Caller (+ 32-byte shadow space) |
| **System V (Linux/macOS)** | RDI, RSI, RDX, RCX, R8, R9 | XMM0-XMM7 | Caller |

---

## Self-Assessment

### Q1: Explain `__cdecl`, `__stdcall`, `__fastcall` and when each is required for Windows DLL exports

**Answer:**

Here's the practical breakdown of each convention and where you're required to use it. The comments explain the key distinction - mainly around who cleans the stack and whether variadic argument lists are supported:

```cpp
// __cdecl - default, supports variadic
// Caller cleans up the stack -> supports variable-argument functions
extern "C" __cdecl int add(int a, int b) { return a + b; }
// printf is __cdecl because it's variadic
// int printf(const char* fmt, ...);  // MUST be __cdecl

// __stdcall - Win32 API standard
// Callee cleans up -> smaller caller code (each call site saves ~2 bytes)
extern "C" __stdcall int WINAPI_Style(int x, int y) { return x * y; }
// REQUIRED for:
//   - Windows API callbacks (WNDPROC, DLGPROC)
//   - COM interface methods
//   - DLLs loaded by VB6/Delphi (they expect __stdcall)

// __fastcall - first 2 args in registers
extern "C" __fastcall int fast_op(int a, int b) { return a + b; }
// Faster for small functions (avoids memory access for first 2 params)
// Used when: performance-critical internal APIs

// Practical DLL example
// my_dll.h
#ifdef MY_DLL_EXPORTS
    #define MY_API __declspec(dllexport)
#else
    #define MY_API __declspec(dllimport)
#endif

// Default __cdecl - safe for C, Python ctypes, etc.
extern "C" MY_API int __cdecl compute(int x);

// __stdcall - required if DLL loaded from VB, Delphi, or .NET P/Invoke
extern "C" MY_API int __stdcall compute_stdcall(int x);

// WRONG: mismatch between declaration and definition
// Header says __stdcall, but .cpp compiled as __cdecl -> stack corruption!
```

To pick the right one, here's a quick decision table:

**When to use each:**

| Need | Convention |
| --- | --- |
| Default DLL API | `__cdecl` (safest, most portable) |
| Win32 API callback | `__stdcall` (CALLBACK macro = __stdcall) |
| COM interfaces | `__stdcall` |
| Variadic functions | `__cdecl` (ONLY option) |
| Hot internal functions | `__fastcall` or `__vectorcall` |
| Cross-platform | Just use `extern "C"` and let 64-bit handle it |

### Q2: Show how to pass structs across a C ABI boundary safely (only trivially copyable types)

**Answer:**

The reason this trips people up is that not all C++ types are safe to pass across a language boundary, even as function arguments. If a type has constructors, destructors, virtual functions, or allocator state, the other side doesn't know how to handle it. You need types that are just plain data - what C++ calls "trivially copyable" and "standard layout."

```cpp
#include <type_traits>
#include <cstdint>
#include <cstring>

// SAFE: trivially copyable struct
struct Point {
    double x, y, z;
};
static_assert(std::is_trivially_copyable_v<Point>);
static_assert(std::is_standard_layout_v<Point>);

// SAFE: packed with explicit layout
#pragma pack(push, 1)
struct NetworkPacket {
    uint32_t id;
    uint16_t type;
    uint8_t  payload[256];
    uint32_t checksum;
};
#pragma pack(pop)
static_assert(sizeof(NetworkPacket) == 4 + 2 + 256 + 4);  // No padding

// UNSAFE: contains non-trivial members
struct BadForABI {
    std::string name;       // Has constructor/destructor
    std::vector<int> data;  // Has allocator state
    virtual void foo();     // Has vtable pointer
};
// static_assert(std::is_trivially_copyable_v<BadForABI>);  // FAILS!

// ABI-safe wrapper
// Instead of passing std::string, pass char* + length
struct NameRecord {
    char name[64];          // Fixed-size buffer - trivially copyable
    int32_t age;
    double salary;
};
static_assert(std::is_trivially_copyable_v<NameRecord>);

// C ABI boundary functions
extern "C" {
    // GOOD: trivially copyable, explicit layout
    void process_point(const Point* p);         // By pointer
    Point create_point(double x, double y, double z);  // By value (small struct OK)

    // GOOD: pass raw buffer for strings
    void set_name(const char* name, int length);

    // BAD: never pass C++ types across C boundary
    // void process(std::string s);       // WRONG!
    // void process(std::vector<int> v);  // WRONG!
}

// Cross-language struct agreement
// Both C++ and external language MUST agree on:
// 1. Field order
// 2. Field sizes
// 3. Alignment/padding
// 4. Endianness (for network protocols)

// Example: matching Python ctypes definition
/*
# Python side:
class Point(ctypes.Structure):
    _fields_ = [
        ("x", ctypes.c_double),
        ("y", ctypes.c_double),
        ("z", ctypes.c_double),
    ]
*/
```

### Q3: Use `static_assert(std::is_trivially_copyable_v<T>)` to guard ABI boundary struct types

**Answer:**

Rather than remembering to check each struct manually, you can encode the requirement right in the code with a macro or a compile-time variable. This catches problems at compile time instead of at runtime when you're staring at a crash dump:

```cpp
#include <type_traits>
#include <cstdint>
#include <string>

// ABI-safety guard macro
#define ABI_SAFE(T) \
    static_assert(std::is_trivially_copyable_v<T>, \
        #T " must be trivially copyable for C ABI"); \
    static_assert(std::is_standard_layout_v<T>, \
        #T " must be standard layout for C ABI")

// Guarded structs
struct Color {
    uint8_t r, g, b, a;
};
ABI_SAFE(Color);  // Passes both checks

struct Rect {
    float x, y, width, height;
};
ABI_SAFE(Rect);  // Passes

struct Transform {
    float matrix[16];  // 4x4 matrix as flat array
};
ABI_SAFE(Transform);  // Passes

// Catches mistakes at compile time
struct Widget {
    std::string label;  // Non-trivial!
    int id;
};
// ABI_SAFE(Widget);  // COMPILE ERROR:
// "Widget must be trivially copyable for C ABI"

struct Base {
    virtual ~Base() = default;
    int value;
};
// ABI_SAFE(Base);  // COMPILE ERROR:
// virtual function -> not trivially copyable, not standard layout

// Template version for generic APIs
template<typename T>
extern "C" void send_to_foreign(const T* data, int count) {
    static_assert(std::is_trivially_copyable_v<T>,
        "Only trivially copyable types can cross ABI boundary");
    static_assert(alignof(T) <= 16,
        "Alignment must be <= 16 for cross-platform compatibility");
    // ... send raw bytes ...
}

// Full ABI contract check
template<typename T>
constexpr bool is_abi_safe_v =
    std::is_trivially_copyable_v<T> &&
    std::is_standard_layout_v<T> &&
    !std::is_pointer_v<T>;  // Pointers aren't meaningful across processes

struct Message {
    uint32_t type;
    uint32_t length;
    uint8_t  data[1024];
};
static_assert(is_abi_safe_v<Message>, "Message must be ABI-safe");
```

---

## Notes

- On 64-bit, `__cdecl`, `__stdcall`, and `__fastcall` are **identical** - the x64 convention is the only one. These distinctions only matter on 32-bit.
- `extern "C"` disables name mangling but does NOT change the calling convention.
- MSVC default is `__cdecl`; you can change it project-wide with `/Gz` (__stdcall) or `/Gr` (__fastcall).
- COM uses `__stdcall` - every COM interface method is `virtual HRESULT __stdcall Method(...)`.
- `CALLBACK`, `WINAPI`, and `APIENTRY` are all macros that expand to `__stdcall` on Windows.
- For cross-platform code: stick to `extern "C"` + trivially copyable structs + fixed-width integers.
