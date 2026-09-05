# Control symbol visibility with attributes and declspec for shared libraries

**Category:** API and Library Design  
**Standard:** C++17  
**Reference:** <https://gcc.gnu.org/wiki/Visibility>  

---

## Topic Overview

By default, every symbol in a shared library is exported - functions, internal helpers, template instantiations, all of it. This causes real problems: bloated symbol tables, slower dynamic linking at load time, symbol name clashes between libraries loaded into the same process, and ABI fragility because internal details become part of the public interface. Controlling visibility is the solution: you explicitly mark only the symbols that form your public API as exported, and everything else stays hidden.

The reason this trips people up is that the mechanism is platform-specific. Windows has always required explicit export/import declarations (`__declspec`), while Linux/GCC defaulted to exporting everything and added visibility attributes later. The cross-platform macro pattern below bridges that gap.

### Cross-Platform Visibility Macro

This is the standard pattern for wrapping platform differences. When you build the library, define `MYLIB_EXPORTS` and every `MYLIB_API`-tagged symbol gets exported. When a user includes your header, `MYLIB_API` becomes `dllimport` on Windows (telling the linker to expect it from a DLL) or the default visibility attribute on Linux.

```cpp
#pragma once

#if defined(_WIN32)
  #ifdef MYLIB_EXPORTS
    #define MYLIB_API __declspec(dllexport)
  #else
    #define MYLIB_API __declspec(dllimport)
  #endif
  #define MYLIB_LOCAL
#else
  #if __GNUC__ >= 4
    #define MYLIB_API   __attribute__((visibility("default")))
    #define MYLIB_LOCAL  __attribute__((visibility("hidden")))
  #else
    #define MYLIB_API
    #define MYLIB_LOCAL
  #endif
#endif

// Usage:
namespace mylib {
    MYLIB_API void public_function();   // Exported
    MYLIB_LOCAL void internal_helper(); // Hidden - not in symbol table

    class MYLIB_API PublicClass {
    public:
        void method();         // Exported (class is exported)
    private:
        MYLIB_LOCAL void impl(); // Hidden even in exported class
    };
}
```

Notice that you can mark an entire class as `MYLIB_API` and then individually hide specific methods with `MYLIB_LOCAL`. This gives you fine-grained control even within a public type.

### CMake Integration

The compiler flag `-fvisibility=hidden` flips the default from "export everything" to "hide everything", so only the symbols you explicitly tag with `MYLIB_API` become visible. You also want `-fvisibility-inlines-hidden` to suppress all the inline function and template instantiation symbols that would otherwise leak out.

```cmake
add_library(mylib SHARED src/api.cpp src/internal.cpp)

# -fvisibility=hidden makes all symbols hidden by default
# Only MYLIB_API-marked symbols are exported
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-fvisibility=hidden -fvisibility-inlines-hidden>
)

target_compile_definitions(mylib PRIVATE MYLIB_EXPORTS)
```

The `MYLIB_EXPORTS` definition is what activates `dllexport` mode in the header. Without it, the header would generate `dllimport` declarations in your own build, which would be wrong.

---

## Self-Assessment

### Q1: Measure the symbol table reduction from visibility control

The reduction can be dramatic. A real library might go from thousands of exported symbols down to just the handful that form the actual public API. Here is how to measure it:

```bash
# Before: all symbols exported
nm -D libmylib.so | wc -l    # e.g., 2500 symbols

# After: -fvisibility=hidden + explicit MYLIB_API
nm -D libmylib.so | wc -l    # e.g., 45 symbols

# Benefits:
# - ~98% fewer exported symbols
# - Faster dynamic linking (fewer symbols to resolve)
# - No symbol clashes between libraries
# - Compiler can optimize hidden functions more aggressively
```

That last benefit is worth dwelling on: because the compiler knows a hidden function cannot be called from outside the library, it is free to inline it, change its calling convention, or remove it entirely if nothing inside the library uses it.

### Q2: Why use `-fvisibility-inlines-hidden`

Inline functions and template instantiations generate many symbols. With `-fvisibility-inlines-hidden`, these are hidden by default, preventing them from polluting the shared library's symbol table. This is almost always what you want - inline functions should be resolved at compile time, not at link time, and exposing their symbols creates needless ABI coupling.

### Q3: How to export a class template from a shared library

You cannot export a template definition directly - templates have no compiled form until they are instantiated. The solution is explicit instantiation: you force the compiler to compile a specific instantiation inside your library and then export that compiled form.

```cpp
// Explicit instantiation with export
template class MYLIB_API std::vector<MyType>; // Export specific instantiation

// Template DEFINITIONS stay in headers
// Only explicit instantiations are exported from the .so
```

This is a more advanced pattern mostly needed when you want to guarantee a single, shared instantiation of a template type across a dynamic library boundary.

---

## Notes

- Always compile shared libraries with `-fvisibility=hidden` and mark public API explicitly - the default of exporting everything is a trap.
- On Windows, `__declspec(dllexport/dllimport)` is required; there is no "hidden by default" mode like Linux has.
- `__attribute__((visibility("default")))` on GCC/Clang is equivalent to `dllexport` for the purposes of making a symbol accessible outside the shared library.
- Exception types thrown across library boundaries must be visible (exported), otherwise the dynamic linker cannot match the thrown type to the catch clause on the other side.
