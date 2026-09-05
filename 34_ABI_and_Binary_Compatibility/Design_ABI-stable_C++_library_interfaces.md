# Design ABI-Stable C++ Library Interfaces

**Category:** ABI and Binary Compatibility  
**Standard:** C++11, C++14, C++17, C++20  
**Reference:** https://www.kdab.com/cpp-api-design/  

---

## Topic Overview

Designing an ABI-stable library interface means creating a public API that can evolve - adding features, fixing bugs, changing internal implementation - without requiring users to recompile their applications. This is the gold standard for shared libraries distributed as binaries (system libraries, commercial SDKs, plugin systems). Achieving ABI stability in C++ requires deliberate architectural decisions because the language's compilation model naturally couples binary layout to header declarations.

The reason this is hard in C++ is that a header file is not just documentation - it drives code generation. The moment a user includes your header, the compiler uses it to decide `sizeof`, member offsets, vtable layout, and calling conventions for your types. If you ship a new version of the library and change anything the header describes, every binary that included the old header is now broken.

The core techniques for ABI stability are:

| Technique              | Mechanism                                   | Trade-off                          |
| --- | --- | --- |
| **Pimpl idiom**        | Hide data behind opaque pointer             | Heap allocation, indirection cost  |
| **Abstract interface** | Pure virtual interface + factory             | vtable dispatch overhead           |
| **C wrapper layer**    | `extern "C"` functions + opaque handles     | No C++ ergonomics in C API         |
| **Versioned structs**  | Size field in struct for forward compat      | Manual version checking            |
| **Inline namespace**   | Mangle versions into symbol names           | Detects mismatch, doesn't prevent  |

If you are not sure which technique to choose, this decision tree helps:

```cpp
// Need cross-language interop? --YES--> C wrapper layer + opaque handles
//          | NO
// Plugin architecture? --YES--> Abstract interface + factory
//          | NO
// Single implementation? --YES--> Pimpl idiom
//          | NO
// Need maximum performance? --YES--> Versioned struct + C API
```

The most robust libraries combine multiple techniques: an internal C++ implementation using Pimpl, a public C++ interface using abstract classes, and a C wrapper layer for FFI consumers. Qt, gRPC, and SQLite are excellent examples of these patterns in production.

---

## Self-Assessment

### Q1: Implement a full Pimpl-based library class that can safely evolve across versions without ABI breaks

The key insight of Pimpl is that the class visible to users contains exactly one data member - a pointer to a hidden `Impl` struct. The size of a pointer never changes, so `sizeof(ComputeEngine)` is frozen at eight bytes on 64-bit regardless of what you add to the implementation. Here is the full pattern applied to a real-looking library class:

```cpp
// === mathlib/engine.h (PUBLIC HEADER - ships to users) ===
#pragma once
#include <memory>

#ifdef _WIN32
    #ifdef MATHLIB_EXPORTS
        #define MATHLIB_API __declspec(dllexport)
    #else
        #define MATHLIB_API __declspec(dllimport)
    #endif
#else
    #define MATHLIB_API __attribute__((visibility("default")))
#endif

namespace mathlib {

class MATHLIB_API ComputeEngine {
public:
    // Construction / destruction
    ComputeEngine();
    ~ComputeEngine();

    // Move-only (Pimpl doesn't copy well without extra effort)
    ComputeEngine(ComputeEngine&& other) noexcept;
    ComputeEngine& operator=(ComputeEngine&& other) noexcept;

    ComputeEngine(const ComputeEngine&) = delete;
    ComputeEngine& operator=(const ComputeEngine&) = delete;

    // Public API - adding new methods is ABI-safe
    void set_precision(int digits);
    double compute(double a, double b, int operation);

    // v1.1: added without ABI break
    void set_thread_count(int n);
    double compute_batch(const double* values, int count);

    // v1.2: added without ABI break
    const char* last_error() const;
    void reset();

private:
    // This is the ONLY data member - sizeof never changes
    struct Impl;
    std::unique_ptr<Impl> impl_;
    // sizeof(ComputeEngine) = sizeof(unique_ptr<Impl>) = 8 bytes (64-bit)
    // This is FROZEN - it will never change regardless of internal evolution
};

}  // namespace mathlib


// === mathlib/engine.cpp (INTERNAL - never shipped to users) ===
// #include "engine.h"
#include <vector>
#include <string>
#include <cmath>
#include <cstdio>
#include <thread>

namespace mathlib {

struct ComputeEngine::Impl {
    int precision = 6;
    int thread_count = 1;
    std::string last_error;

    // v1.1 additions:
    std::vector<double> batch_results;

    // v1.2 additions:
    bool has_error = false;

    // v2.0 additions (still no ABI break!):
    // std::unordered_map<std::string, double> cache;
    // std::mutex mtx;
    // PerformanceCounters perf;

    double do_compute(double a, double b, int op) {
        switch (op) {
            case 0: return a + b;
            case 1: return a - b;
            case 2: return a * b;
            case 3:
                if (b == 0.0) {
                    last_error = "division by zero";
                    has_error = true;
                    return 0.0;
                }
                return a / b;
            default:
                last_error = "unknown operation";
                has_error = true;
                return 0.0;
        }
    }
};

ComputeEngine::ComputeEngine() : impl_(std::make_unique<Impl>()) {}
ComputeEngine::~ComputeEngine() = default;
ComputeEngine::ComputeEngine(ComputeEngine&&) noexcept = default;
ComputeEngine& ComputeEngine::operator=(ComputeEngine&&) noexcept = default;

void ComputeEngine::set_precision(int digits) { impl_->precision = digits; }

double ComputeEngine::compute(double a, double b, int operation) {
    return impl_->do_compute(a, b, operation);
}

void ComputeEngine::set_thread_count(int n) { impl_->thread_count = n; }

double ComputeEngine::compute_batch(const double* values, int count) {
    double sum = 0;
    for (int i = 0; i < count; ++i) sum += values[i];
    return sum;
}

const char* ComputeEngine::last_error() const {
    return impl_->has_error ? impl_->last_error.c_str() : nullptr;
}

void ComputeEngine::reset() {
    impl_->has_error = false;
    impl_->last_error.clear();
}

}  // namespace mathlib
```

Notice that the destructor must be defined in the `.cpp` file even if it is `= default`. If you default it in the header, the compiler tries to generate it there, but `Impl` is incomplete at that point and `unique_ptr`'s destructor fails. Moving the destructor definition to the `.cpp` is not optional with Pimpl.

### Q2: Design a plugin system using abstract interfaces with factory registration that maintains ABI stability

Abstract interfaces work differently from Pimpl. Instead of hiding data, you expose only pure-virtual methods and ship a factory function. Plugins implement the interface, and the host loads them without knowing or caring about the concrete type. The tricky part is versioning: once you publish an interface, you can never reorder or remove virtual methods - you can only append.

```cpp
#include <cstdio>
#include <memory>
#include <cstring>

// === plugin_api.h (shipped to plugin developers) ===

// ABI-stable abstract interface
// Rules: NEVER reorder, remove, or change signatures of existing virtuals
// Only ADD new virtuals at the END
class IPlugin {
public:
    virtual ~IPlugin() = default;

    // v1.0 interface
    virtual const char* name() const = 0;
    virtual const char* version() const = 0;
    virtual int initialize(const char* config) = 0;
    virtual void shutdown() = 0;
    virtual int process(const void* input, int input_size,
                        void* output, int output_capacity) = 0;

    // v1.1: added at END - ABI-safe for host built with v1.1+
    // Plugins built against v1.0 will have garbage here!
    // Solution: query interface pattern (see below)
};

// ABI version negotiation - critical for safety
struct PluginInfo {
    int struct_size;          // = sizeof(PluginInfo), for forward compat
    int abi_version;          // incrementing ABI version number
    const char* name;
    const char* description;
    const char* author;
};

// C ABI entry points - every plugin exports these
extern "C" {
    typedef IPlugin* (*CreatePluginFn)();
    typedef void (*DestroyPluginFn)(IPlugin*);
    typedef const PluginInfo* (*GetPluginInfoFn)();
}

// === Versioned interface querying (COM-inspired pattern) ===

enum class InterfaceId : int {
    IPlugin_v1   = 1,
    IPlugin_v1_1 = 2,
    IBatchPlugin = 3,
};

class IQueryInterface {
public:
    virtual ~IQueryInterface() = default;
    // Returns nullptr if interface not supported
    virtual void* query_interface(InterfaceId id) = 0;
};

// Extended interface - only available from v1.1 plugins
class IBatchPlugin {
public:
    virtual ~IBatchPlugin() = default;
    virtual int process_batch(const void** inputs, const int* sizes,
                              int count, void* output, int capacity) = 0;
};


// === Example plugin implementation ===

class MyPlugin : public IPlugin, public IQueryInterface, public IBatchPlugin {
public:
    const char* name() const override { return "MyPlugin"; }
    const char* version() const override { return "1.1.0"; }

    int initialize(const char* config) override {
        std::printf("MyPlugin initialized with: %s\n", config);
        return 0;
    }

    void shutdown() override {
        std::printf("MyPlugin shutdown\n");
    }

    int process(const void* input, int input_size,
                void* output, int output_capacity) override {
        int copy_size = input_size < output_capacity ? input_size : output_capacity;
        std::memcpy(output, input, copy_size);
        return copy_size;
    }

    // IQueryInterface
    void* query_interface(InterfaceId id) override {
        switch (id) {
            case InterfaceId::IPlugin_v1:   return static_cast<IPlugin*>(this);
            case InterfaceId::IPlugin_v1_1: return static_cast<IPlugin*>(this);
            case InterfaceId::IBatchPlugin: return static_cast<IBatchPlugin*>(this);
            default: return nullptr;
        }
    }

    // IBatchPlugin
    int process_batch(const void** inputs, const int* sizes,
                      int count, void* output, int capacity) override {
        std::printf("Batch processing %d items\n", count);
        return 0;
    }
};

// C entry points
extern "C" {
    IPlugin* create_plugin() { return new MyPlugin(); }
    void destroy_plugin(IPlugin* p) { delete p; }

    const PluginInfo* get_plugin_info() {
        static const PluginInfo info{
            sizeof(PluginInfo), 2, "MyPlugin", "A demo plugin", "Author"
        };
        return &info;
    }
}
```

The `query_interface` pattern is COM-inspired and solves the "can't add virtuals to a frozen interface" problem. The host calls `query_interface(InterfaceId::IBatchPlugin)` and checks for null - if null, the plugin is old and doesn't support batch processing; if non-null, it does. No vtable ordering is disturbed.

### Q3: Implement the versioned struct pattern for C-compatible configuration structures that can grow over time

This pattern is used by Vulkan, Win32 API, and DirectX. The idea is to put a `struct_size` field first. Old callers set it to the size they know about; new library code checks the size before touching any field to determine which version of the struct it received. No version enum, no separate version parameter - the struct carries its own version as its size.

```cpp
#include <cstdint>
#include <cstring>
#include <cstdio>

// === Versioned struct pattern ===
// The struct begins with a size field. The library checks the size
// to determine which version the caller compiled against.

struct EngineConfig {
    // v1.0 fields (NEVER reorder or remove)
    std::uint32_t struct_size;      // MUST be first - set to sizeof(EngineConfig)
    std::uint32_t flags;
    std::uint32_t max_threads;
    std::uint32_t queue_depth;

    // v1.1 fields (appended at end)
    std::uint32_t timeout_ms;
    std::uint32_t retry_count;

    // v1.2 fields (appended at end)
    std::uint64_t max_memory_bytes;
    const char*   log_file_path;
};

// Helper macro for version detection
#define CONFIG_HAS_FIELD(cfg, field) \
    ((cfg)->struct_size >= offsetof(EngineConfig, field) + sizeof((cfg)->field))

// Library-side initialization - handles any version of the struct
int engine_init(const EngineConfig* cfg) {
    if (!cfg || cfg->struct_size < 16) {
        std::fprintf(stderr, "Invalid config: too small\n");
        return -1;
    }

    // v1.0 fields - always present
    std::printf("flags: 0x%x, threads: %u, queue: %u\n",
                cfg->flags, cfg->max_threads, cfg->queue_depth);

    // v1.1 fields - check before accessing
    if (CONFIG_HAS_FIELD(cfg, timeout_ms)) {
        std::printf("timeout: %u ms, retries: %u\n",
                    cfg->timeout_ms, cfg->retry_count);
    } else {
        std::printf("Using default timeout (no v1.1 fields)\n");
    }

    // v1.2 fields - check before accessing
    if (CONFIG_HAS_FIELD(cfg, max_memory_bytes)) {
        std::printf("max memory: %llu bytes\n",
                    static_cast<unsigned long long>(cfg->max_memory_bytes));
        if (cfg->log_file_path) {
            std::printf("log file: %s\n", cfg->log_file_path);
        }
    } else {
        std::printf("Using default memory limit (no v1.2 fields)\n");
    }

    return 0;
}

int main() {
    // Modern caller (v1.2) - fills everything
    EngineConfig cfg{};
    cfg.struct_size = sizeof(EngineConfig);
    cfg.flags = 0x01;
    cfg.max_threads = 8;
    cfg.queue_depth = 256;
    cfg.timeout_ms = 5000;
    cfg.retry_count = 3;
    cfg.max_memory_bytes = 1ULL << 30;  // 1 GB
    cfg.log_file_path = "/var/log/engine.log";

    std::printf("=== Full v1.2 config ===\n");
    engine_init(&cfg);

    // Simulating a legacy v1.0 caller that only knows about
    // the first 16 bytes of the struct
    EngineConfig legacy{};
    legacy.struct_size = 16;  // Only knows v1.0 fields
    legacy.flags = 0x02;
    legacy.max_threads = 4;
    legacy.queue_depth = 64;

    std::printf("\n=== Legacy v1.0 config ===\n");
    engine_init(&legacy);

    return 0;
}
```

The `CONFIG_HAS_FIELD` macro calculates whether the struct is large enough to contain a given field by comparing `struct_size` against the field's offset plus its own size. A v1.0 caller sets `struct_size = 16` and the library gracefully falls back to defaults for anything it does not know about. Both the old caller and the new library can upgrade independently.

---

## Notes

- Pimpl is the simplest ABI stability technique - it guarantees `sizeof(Class)` never changes because the only member is a pointer to the hidden implementation.
- Abstract interfaces work well for plugin systems but require careful vtable management - never reorder or remove virtual functions, only append.
- The versioned struct pattern (used by Vulkan, Win32, DirectX) puts `struct_size` as the first field to enable forward and backward compatibility.
- Always provide factory functions (not constructors) in the public API - `create_widget()` instead of `new Widget()` - so the library controls allocation and can return different implementations.
- COM's `QueryInterface` pattern solves the "can't add virtuals" problem by allowing clients to discover extended interfaces at runtime.
- Combine techniques: use Pimpl for classes, C wrapper for FFI, and inline namespaces for version detection - defense in depth.
- Test ABI stability with `abi-compliance-checker` or `abidiff` in CI to catch accidental breaks before release.
