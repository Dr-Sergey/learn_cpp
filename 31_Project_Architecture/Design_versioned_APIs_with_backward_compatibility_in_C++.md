# Design versioned APIs with backward compatibility in C++

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**API versioning** allows evolving a library or service interface without breaking existing consumers. In C++ this applies to shared libraries (ABI compatibility), header-only APIs (source compatibility), and network protocols (wire compatibility). The key challenge is adding capabilities while preserving old behavior.

Here's why this is hard in C++: the ABI (Application Binary Interface) is the compiled representation of your types - struct layouts, vtable orders, name-mangled symbols. If you add a member variable to a class in a shared library, its layout changes, and any existing code compiled against the old layout will read garbage. This is fundamentally different from most higher-level languages where you can just add a method and redeploy. In C++ you need deliberate techniques to evolve without breaking things.

### Versioning Strategies

| Strategy | Scope | Breaking Changes | Example |
| --- | --- | --- | --- |
| **Inline namespace** | Source/ABI | New namespace = new ABI | `v1::Widget`, `v2::Widget` |
| **PIMPL** | ABI | Add members freely | Private Impl class |
| **Function overloads** | Source | Add parameters with defaults | `create()`, `create(options)` |
| **Capability negotiation** | Wire | Query supported features | `version()` + `supports()` |
| **Semantic versioning** | Release | Major = breaking | `libfoo.so.2.1.0` |

---

## Self-Assessment

### Q1: Use inline namespaces for API versioning

An `inline namespace` is transparent to callers: `mylib::Widget` automatically resolves to `mylib::v1::Widget` because `v1` is inline. The old code that was already written just works. New code can opt in to `mylib::v2::Widget` explicitly. When you're ready to make v2 the default, you change which namespace is `inline` - a one-line change that makes the new version the default while still letting anyone who needs the old behavior write `mylib::v1::Widget` explicitly.

**Answer:**

```cpp
// === mylib/widget.h ===
#pragma once

namespace mylib {

// v1: original API (default)
inline namespace v1 {
    class Widget {
    public:
        Widget(int id);
        void process();
        int id() const;
    private:
        int id_;
    };
}

// v2: extended API (opt-in)
namespace v2 {
    struct WidgetOptions {
        int id;
        std::string name = "";
        int priority = 0;
        bool async = false;
    };

    class Widget {
    public:
        explicit Widget(WidgetOptions opts);
        void process();  // Enhanced processing
        int id() const;
        const std::string& name() const;
        int priority() const;
    private:
        WidgetOptions opts_;
    };
}

}  // namespace mylib

// === Usage ===
void old_code() {
    mylib::Widget w(42);     // Uses v1 (inline = default)
    w.process();
}

void new_code() {
    mylib::v2::Widget w({.id = 42, .name = "main", .priority = 5});
    w.process();
}

// To switch default version in a future release:
// 1. Remove 'inline' from v1
// 2. Add 'inline' to v2
// 3. Old code using mylib::Widget now gets v2
// 4. Old code can still explicitly use mylib::v1::Widget
```

This is a clean migration strategy. The library ships with both versions; consumers can test their code against v2 before you flip the default; and nothing breaks at the flip point unless the consumer was using `mylib::Widget` without thinking about versions (which is fine - they just get the new default, which you've designed to be backward-compatible at the source level).

### Q2: Design ABI-stable API with PIMPL and C interface

PIMPL (Pointer to IMPLementation) is the fundamental ABI stability technique in C++. The public header exposes only a pointer to a forward-declared `Impl` class. All the actual data members live in `Impl`, which is defined only in the `.cpp` file. From the ABI perspective, the public class's binary layout never changes - it's just a `unique_ptr`. You can add any number of new fields to `Impl`, add new methods to the public class, and existing binaries continue to work without recompilation.

The C API at the bottom goes even further: it has no C++ types at all, which makes it stable across any C++ compiler and even callable from C, Python (via ctypes), and other languages.

**Answer:**

```cpp
// === Public header: stable across library versions ===
// include/mylib/engine.h
#pragma once
#include "mylib_export.h"
#include <memory>
#include <cstdint>

namespace mylib {

// Version information
struct MYLIB_API Version {
    int major, minor, patch;
    static Version current();
    bool is_compatible_with(Version other) const {
        return major == other.major;  // Same major = compatible
    }
};

// ABI-stable class via PIMPL
class MYLIB_API Engine {
public:
    Engine();
    ~Engine();
    Engine(Engine&&) noexcept;
    Engine& operator=(Engine&&) noexcept;

    // v1.0 API
    bool start();
    void stop();
    int process(const uint8_t* data, size_t len);

    // v1.1 API: new methods (ABI-compatible addition)
    void set_option(const char* key, const char* value);
    const char* get_option(const char* key) const;

    // v1.2 API: more additions
    struct Stats {
        uint64_t processed;
        uint64_t errors;
        double avg_latency_ms;
    };
    Stats get_stats() const;

private:
    class Impl;
    std::unique_ptr<Impl> impl_;
};

}  // namespace mylib

// === Optional C API for maximum ABI stability ===
extern "C" {
    typedef struct mylib_engine mylib_engine;

    MYLIB_API mylib_engine* mylib_engine_create();
    MYLIB_API void mylib_engine_destroy(mylib_engine* e);
    MYLIB_API int mylib_engine_start(mylib_engine* e);
    MYLIB_API void mylib_engine_stop(mylib_engine* e);
    MYLIB_API int mylib_engine_process(mylib_engine* e,
        const uint8_t* data, size_t len);

    // Version query
    MYLIB_API int mylib_version_major();
    MYLIB_API int mylib_version_minor();
}
```

Notice that the destructor, move constructor, and move assignment are all explicitly declared even though they'd be generated by default. With PIMPL you must declare them because the compiler needs to generate the destructor of `unique_ptr<Impl>` where `Impl` is fully defined - if you let the destructor be implicitly inlined into the header, the compiler will complain that it can't destroy an incomplete type.

### Q3: Deprecate old API gracefully

Deprecation is a communication strategy: you warn existing users that something will be removed, tell them what to use instead, and give them a release window to migrate. The `[[deprecated]]` attribute (C++14) produces a compiler warning with a custom message. That message should tell the user exactly what to do, not just say "don't use this."

The `Capabilities::supports()` function is the runtime equivalent: instead of compile-time deprecation, it lets callers probe whether a feature exists before trying to use it, which is useful for library code that needs to work with multiple installed versions.

**Answer:**

```cpp
// === Deprecation with clear migration path ===

// Compiler-specific deprecation macros
#if defined(__cplusplus) && __cplusplus >= 201402L
    #define MYLIB_DEPRECATED(msg) [[deprecated(msg)]]
#elif defined(_MSC_VER)
    #define MYLIB_DEPRECATED(msg) __declspec(deprecated(msg))
#elif defined(__GNUC__)
    #define MYLIB_DEPRECATED(msg) __attribute__((deprecated(msg)))
#else
    #define MYLIB_DEPRECATED(msg)
#endif

namespace mylib {

// Old API: deprecated with migration guide
MYLIB_DEPRECATED("Use Engine::set_option() instead. Will be removed in v3.0")
MYLIB_API void configure_engine(Engine& e, int mode, int flags);

MYLIB_DEPRECATED("Use Engine(EngineOptions) constructor instead")
MYLIB_API Engine create_engine(int type);

// New API replaced old one:
struct EngineOptions {
    std::string name;
    int mode = 0;
    int flags = 0;
    // NEW in v2.0
    int worker_count = 4;
    bool enable_metrics = false;
};

// === Version negotiation for dynamic features ===
class MYLIB_API Capabilities {
public:
    static bool supports(const char* feature) {
        static const std::unordered_set<std::string> features = {
            "async_process",    // Added in v1.1
            "batch_mode",       // Added in v1.2
            "metrics",          // Added in v2.0
            "compression",      // Added in v2.1
        };
        return features.count(feature) > 0;
    }

    static int api_version() { return 2; }  // Bumped on breaking changes
};

}  // namespace mylib

// === Consumer code with version checking ===
void safe_usage(mylib::Engine& engine) {
    if (mylib::Capabilities::supports("metrics")) {
        auto stats = engine.get_stats();
        // ... use v2.0 feature
    }
    // Fallback for older versions
}
```

The multi-compiler macro fallback chain ensures deprecation warnings work everywhere: `[[deprecated]]` for standard C++14 compilers, `__declspec(deprecated)` for older MSVC, and `__attribute__((deprecated))` for GCC/Clang before full C++14 support. The `#else` fallback makes the macro silently disappear on unknown compilers, which is better than a compile error.

---

## Notes

- **Inline namespaces** are the cleanest C++ versioning mechanism: default version is transparent to callers.
- **PIMPL** preserves ABI: you can add fields, change implementation, add methods without breaking `.so` compatibility.
- **C API** provides maximum ABI stability: C has no name mangling, no vtable, no exceptions.
- **Semantic versioning**: major bump = breaking change, minor = additions, patch = fixes.
- `[[deprecated]]` gives compile-time warnings to users: they see the migration path before removal.
- Never remove or change field numbers in protobuf, never reorder vtable entries in COM-style interfaces.
- SO versioning: `SOVERSION` = major version; consumers link to `libfoo.so.2`, symlink resolves to `libfoo.so.2.1.0`.
