# Understand the Itanium ABI name mangling and extern C

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://itanium-cxx-abi.github.io/cxx-abi/abi.html>  

---

## Topic Overview

C++ allows function overloading - multiple functions with the same name but different parameters. Since the linker only sees symbol names, the compiler **mangles** function names to encode parameter types, namespaces, and templates into the symbol.

### What Is Name Mangling

Here is what the same base name `foo` looks like to the linker under the Itanium ABI - each overload becomes a distinct symbol.

```cpp
void foo(int);           // mangled: _Z3fooi
void foo(double);        // mangled: _Z3food
void foo(int, double);   // mangled: _Z3fooId
namespace ns {
    void foo(int);       // mangled: _ZN2ns3fooEi
}
```

The Itanium ABI (used by GCC, Clang, and most non-MSVC compilers) defines a specific mangling scheme:

| Element | Mangling |
| --- | --- |
| `_Z` | Prefix for all mangled names |
| `3foo` | Function name (length + name) |
| `i` | `int` parameter |
| `d` | `double` parameter |
| `N...E` | Namespace scope |
| `I...E` | Template arguments |

### extern "C" - Disable Mangling

When you need a function to be callable from C - or from any language that can link against a C ABI - you tell the compiler to skip mangling entirely.

```cpp
// Without extern "C": name is mangled
void cpp_func(int x);      // symbol: _Z8cpp_funci

// With extern "C": name is NOT mangled (C linkage)
extern "C" void c_func(int x);  // symbol: c_func

// Block form for multiple declarations:
extern "C" {
    void init();
    void cleanup();
    int process(const char* data, int len);
}
```

The block form is convenient for declaring a whole C API at once.

### Why extern "C" Matters

The classic pattern for a header that works in both C and C++ puts the `extern "C"` block behind a preprocessor guard.

```cpp
// header.h — used by both C and C++
#ifdef __cplusplus
extern "C" {
#endif

void my_library_init(void);
int  my_library_process(const char* data);

#ifdef __cplusplus
}
#endif
```

C compilers see no `extern "C"` (they don't need it); C++ compilers wrap the declarations to suppress mangling, so the symbols match the ones the C compiler produced.

---

## Self-Assessment

### Q1: Use c++filt to demangle a mangled symbol name from a linker error

When the linker complains about an undefined reference, the name looks like line noise. `c++filt` translates it back to human-readable C++.

```cpp
// Consider this C++ code:
namespace graphics {
    class Renderer {
    public:
        void draw(int x, int y);
        void draw(double x, double y);
    };
}
// If you forget to define draw(double, double), the linker says:
//   undefined reference to `_ZN8graphics8Renderer4drawEdd'
//
// Demangle it:
//   $ c++filt _ZN8graphics8Renderer4drawEdd
//   graphics::Renderer::draw(double, double)
//
// Common demangling tools:
//   c++filt (Linux/macOS)               — pipe linker output through it
//   undname (MSVC)                       — demangles MSVC-style names
//   llvm-cxxfilt                          — LLVM version of c++filt
//   Compiler Explorer                     — shows both mangled and demangled

// Example mangled names and their demangled forms:
// _Z3fooi                    -> foo(int)
// _Z3food                    -> foo(double)
// _ZN2ns3barEid              -> ns::bar(int, double)
// _ZN5MyApp6Widget4drawERKSt6vectorIiSaIiEE
//                             -> MyApp::Widget::draw(std::vector<int, std::allocator<int>> const&)
// _Z8templateIiEvT_           -> void template_<int>(int)

// Practical usage: pipe linker errors through c++filt
// $ g++ main.cpp -o main 2>&1 | c++filt
```

Once you know this trick, linker errors become much easier to read. The mangled name tells you exactly which overload was missing.

**How this works:**

- Linker errors show mangled names because the linker works with raw symbols.
- `c++filt` decodes the Itanium ABI mangling back to readable C++ names.
- The mangled name encodes: namespace, class, function name, parameter types, const/volatile, templates.

### Q2: Explain why overloaded functions have different mangled names

**Answer:**

The linker identifies functions **solely by their symbol name**. If two functions had the same symbol, the linker couldn't distinguish them. Name mangling solves this by encoding the parameter types into the symbol name.

Notice how the four `process` overloads below produce four completely different symbols for the linker.

```cpp
// These four functions all have the same base name "process"
// but the linker sees four DIFFERENT symbols:

void process(int x);           // _Z7processi
void process(double x);        // _Z7processd
void process(int x, int y);    // _Z7processii
void process(const char* s);   // _Z7processPKc

// The mangling encodes:
// - Number of characters in the name (7 for "process")
// - Parameter types: i=int, d=double, PKc=pointer-to-const-char
// - This is why the linker can resolve the correct overload
```

**What's encoded:**

- Parameter types (including const, volatile, references, pointers)
- Namespace/class scope
- Template parameters
- Return type (for template specializations only)
- `const` qualifier on member functions

**What's NOT encoded:**

- Return type (for non-template functions) - this is why C++ doesn't allow overloading by return type alone
- Parameter names

### Q3: Show that extern "C" disables name mangling and the implications for function overloading

The trade-off is direct: no mangling means the linker can't tell overloads apart, so `extern "C"` functions can't be overloaded. The C-compatible API pattern at the bottom shows how to expose a clean C interface while keeping the implementation in full C++.

```cpp
#include <iostream>

// With extern "C": no mangling -> no overloading!
extern "C" void greet(int x) {
    std::cout << "greet(int): " << x << "\n";
}

// extern "C" void greet(double x) { }  // ERROR: symbol 'greet' already defined!
// Can't overload — both would produce symbol name 'greet'

// C++ functions CAN overload (mangled differently)
void hello(int x)    { std::cout << "hello(int)\n"; }
void hello(double x) { std::cout << "hello(double)\n"; }
// These produce _Z5helloi and _Z5hellod — different symbols

// Common pattern: C-compatible API with C++ implementation
extern "C" {
    // These are the public C API (no mangling)
    void* create_engine();
    void  engine_process(void* engine, const char* data);
    void  destroy_engine(void* engine);
}

// Implementation uses full C++
struct Engine {
    std::string state;
    void process(const std::string& data) { state += data; }
};

void* create_engine() { return new Engine{}; }
void engine_process(void* engine, const char* data) {
    static_cast<Engine*>(engine)->process(data);
}
void destroy_engine(void* engine) { delete static_cast<Engine*>(engine); }

int main() {
    greet(42);         // Calls the extern "C" function
    hello(42);         // Calls the mangled C++ overload
    hello(3.14);       // Calls the other mangled C++ overload

    // Use the C API
    void* e = create_engine();
    engine_process(e, "hello");
    destroy_engine(e);
}
```

The opaque `void*` handle is the standard trick for hiding a C++ object behind a C API - callers never see the `Engine` type, just the plain functions.

**How this works:**

- `extern "C"` produces unmangled symbols - the linker sees just `greet`, not `_Z5greeti`.
- Without mangling, there's no way to distinguish overloads - so overloading is forbidden with `extern "C"`.
- Templates and namespaced functions cannot be `extern "C"` (they require mangling).
- The C-compatible API pattern wraps C++ objects in opaque `void*` pointers with `extern "C"` functions.

---

## Notes

- MSVC uses a different mangling scheme (not Itanium ABI) - names start with `?` instead of `_Z`.
- `extern "C"` still allows using C++ features in the function body - only the linkage/name changes.
- `extern "C"` functions can still throw exceptions (but C callers won't handle them - use `noexcept`).
- You can nest `extern "C"` inside namespaces, but the symbol won't include the namespace.
- Use `__attribute__((visibility("default")))` (GCC/Clang) or `__declspec(dllexport)` (MSVC) to control which symbols are exported from shared libraries.
