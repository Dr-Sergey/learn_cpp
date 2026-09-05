# Use inline namespaces for ABI versioning without breaking client code

**Category:** API and Library Design  
**Standard:** C++11, C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/namespace#Inline_namespaces>  

---

## Topic Overview

Inline namespaces let a library have multiple ABI versions coexist in the same header while keeping client code unchanged. The key benefit is safety: the linker catches ABI mismatches as unresolved symbols instead of silently letting mismatched code run and corrupt memory.

The reason this trips people up is that ABI breaks are invisible at the source level. If you change a struct's layout - add a field, change alignment, reorder members - a client compiled against the old layout will still compile cleanly against the new header. The code "looks right." But at runtime, the client passes a pointer to a 16-byte struct and your library reads it as a 24-byte struct. That's silent corruption, and it's one of the nastiest bugs to diagnose.

Inline namespaces turn that silent corruption into a linker error you catch immediately.

### How It Works

The `inline` keyword on a namespace makes everything inside it visible as if it were declared in the enclosing namespace. That's what lets client code say `mylib::Widget` and get `mylib::v2::Widget` without knowing `v2` exists. Meanwhile the linker-visible mangled names still include `v2`, so an old client looking for `v1` symbols will simply fail to link:

```cpp
// mylib/version.hpp
namespace mylib {
    inline namespace v2 {  // "inline" makes v2 the default
        struct Widget {
            int x;
            int y;
            int z;  // Added in v2 - changes sizeof(Widget)!
        };
        void process(Widget& w);
    }

    namespace v1 {  // Old ABI - still available explicitly
        struct Widget {
            int x;
            int y;
        };
        void process(Widget& w);
    }
}

// Client code - no changes needed:
// mylib::Widget w;           // Resolves to mylib::v2::Widget
// mylib::process(w);         // Resolves to mylib::v2::process
//
// Client compiled against v1 looking for mylib::v1::Widget
// will get a LINKER ERROR (not silent ABI mismatch!)
```

The old `v1` namespace remains non-inline so clients that explicitly opt in to the old ABI (`mylib::v1::Widget`) can still do so - useful for maintaining backward compatibility in rare cases.

### Name Mangling Makes It Safe

The safety guarantee comes from C++ name mangling. Every function's "real" symbol name in the object file encodes its full namespace path. When you change which namespace is `inline`, you change the default symbol names clients expect to find:

```cpp
// The mangled symbol names include the namespace:
// mylib::v1::process -> _ZN5mylib2v17processERNS0_6WidgetE
// mylib::v2::process -> _ZN5mylib2v27processERNS0_6WidgetE
//
// A v1-compiled client calls the v1 symbol.
// If linked against v2 library (which only exports v2 symbol):
// -> LINKER ERROR: undefined reference to mylib::v1::process
// This is MUCH better than silently using the wrong struct layout!
```

Without inline namespaces, both versions would mangle to the same symbol (`mylib::process`). The linker would happily link them together - and your program would silently read the wrong struct layout at runtime.

### Practical Pattern

In real libraries you usually want to select the ABI version at build time via a preprocessor define. This way a single set of headers can build against any supported ABI:

```cpp
// config.hpp - choose ABI version at build time
#if MYLIB_ABI_VERSION == 1
  #define MYLIB_ABI_NS v1
#elif MYLIB_ABI_VERSION == 2
  #define MYLIB_ABI_NS v2
#else
  #define MYLIB_ABI_NS v3
#endif

// api.hpp
namespace mylib {
    inline namespace MYLIB_ABI_NS {
        class Connection {
            // Layout changes between ABI versions
        public:
            void send(const char* data, size_t len);
            void close();
        };
    }
}
// Clients see: mylib::Connection
// Linker sees: mylib::v3::Connection
```

Clients that don't define `MYLIB_ABI_VERSION` get `v3` (the default). Clients that need to pin to an older ABI for compatibility can set the define in their build system.

---

## Self-Assessment

### Q1: Show what happens when a client compiled against v1 links with a v2 library

The client's object files reference mangled symbols containing `v1` (for example, `_ZN5mylib2v17processERNS0_6WidgetE`). The v2 library exports symbols containing `v2` (for example, `_ZN5mylib2v27processERNS0_6WidgetE`). The linker cannot resolve the v1 references and reports "undefined reference to `mylib::v1::process`" - a clear, actionable error. Without inline namespaces, both would mangle to the same `mylib::process`, the linker would succeed, but the v1 client would be passing a 16-byte struct where the library expects a 24-byte struct, causing memory corruption.

### Q2: Can you have multiple inline namespaces active simultaneously

No. Only one version should be `inline` at a time - that's the "current" or "default" ABI. Other versions live as non-inline namespaces that clients can name explicitly if they need to opt into an older ABI: `mylib::v1::Widget`. Having two versions simultaneously inline would create ambiguity for names that exist in both, and that's exactly what you're trying to avoid.

### Q3: How do the standard library implementations use inline namespaces

The standard library uses this exact technique for real ABI transitions. Here's the most famous example:

```cpp
// libstdc++ uses inline namespace __cxx11 for ABI-breaking changes:
namespace std {
    inline namespace __cxx11 {
        class string { /* SSO implementation */ };
    }
    // Old ABI: std::string
    // New ABI: std::__cxx11::string
    // -D_GLIBCXX_USE_CXX11_ABI=0 switches to old ABI
}

// libc++ uses inline namespace __1: std::__1::string
// MSVC STL uses inline namespaces for ABI version tracking
```

When GCC 5 introduced the new `std::string` implementation (with the small string optimization), it put it in `inline namespace __cxx11`. Code compiled with GCC 5 looks for `std::__cxx11::string`. Code compiled with GCC 4 looks for `std::string` (the old one, in no inline namespace). Mixing the two in the same program produces a linker error - as intended.

---

## Notes

- Every shared library that might change its struct layouts should use inline namespaces for ABI versioning - it turns silent corruption into safe linker errors.
- The `inline` keyword on a namespace has nothing to do with `inline` on functions; it just means "treat names inside as if they're in the enclosing namespace."
- libstdc++'s `__cxx11` ABI break in GCC 5 is the canonical real-world example of this technique used at scale.
- Combine inline namespace versioning with symbol visibility control (`__attribute__((visibility("default")))`) for the cleanest possible shared library interface.
