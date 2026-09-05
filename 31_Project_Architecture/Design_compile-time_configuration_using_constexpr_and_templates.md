# Design compile-time configuration using constexpr and templates

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Compile-time configuration** uses `constexpr`, `if constexpr`, template parameters, and preprocessor defines to select behavior at compile time rather than at runtime. The key benefit is that dead code disappears entirely - if logging is disabled, the logging function body is empty, not just skipped at runtime. The optimizer sees the condition as a compile-time constant and removes the whole branch.

This is especially valuable in embedded systems, where every byte and cycle matters. But it also applies to desktop/server software: compile-time configuration catches misconfiguration as a compile error instead of a runtime crash, and policy-class templates give you zero-overhead customization points with no vtable involved.

### Configuration Mechanisms

| Mechanism | Flexibility | Safety | Overhead |
| --- | --- | --- | --- |
| `constexpr` variables | Compile-time values | Type-safe | Zero |
| `if constexpr` | Branch elimination | Dead code removed | Zero |
| Template parameters | Type/value selection | Compile-time checked | Zero |
| `#define` | Preprocessor switching | Weak (text replacement) | Zero |
| Policy classes | Behavior injection | Strong (concepts) | Zero |

---

## Self-Assessment

### Q1: Design a compile-time configuration system

**Answer:**

The `Config` struct below is the single source of truth for all build-time settings. Because every member is `static constexpr`, the compiler can evaluate any expression involving them at compile time - including `POOL_SIZE`, which is derived from `IS_EMBEDDED`. The `static_assert` calls at the bottom turn invalid configurations into compile errors, which is exactly where you want to catch them.

```cpp
#include <cstdint>
#include <type_traits>

// === Configuration struct (all constexpr) ===
struct Config {
    static constexpr bool ENABLE_LOGGING = true;
    static constexpr bool ENABLE_ASSERTIONS = true;
    static constexpr int MAX_CONNECTIONS = 128;
    static constexpr size_t BUFFER_SIZE = 4096;
    static constexpr bool USE_TLS = true;
    static constexpr int LOG_LEVEL = 2;  // 0=off, 1=error, 2=info, 3=debug

    // Platform detection
    static constexpr bool IS_EMBEDDED =
#ifdef PLATFORM_EMBEDDED
        true;
#else
        false;
#endif

    // Derived configurations
    static constexpr size_t POOL_SIZE = IS_EMBEDDED ? 16 : 256;
    static constexpr bool USE_HEAP = !IS_EMBEDDED;
};

// === Zero-overhead logging ===
template<int Level>
void log_impl(const char* msg) {
    if constexpr (Config::ENABLE_LOGGING && Level <= Config::LOG_LEVEL) {
        // This entire function body is eliminated when logging is disabled
        if constexpr (Level == 1) {
            fprintf(stderr, "[ERROR] %s\n", msg);
        } else if constexpr (Level == 2) {
            printf("[INFO] %s\n", msg);
        } else if constexpr (Level == 3) {
            printf("[DEBUG] %s\n", msg);
        }
    }
    // When ENABLE_LOGGING is false, this function is empty = no overhead
}

#define LOG_ERROR(msg) log_impl<1>(msg)
#define LOG_INFO(msg)  log_impl<2>(msg)
#define LOG_DEBUG(msg) log_impl<3>(msg)

// === Compile-time assert for config validation ===
static_assert(Config::MAX_CONNECTIONS > 0,
    "MAX_CONNECTIONS must be positive");
static_assert(Config::BUFFER_SIZE >= 256,
    "BUFFER_SIZE too small");
static_assert(!Config::IS_EMBEDDED || Config::POOL_SIZE <= 64,
    "Pool too large for embedded");
```

The reason `if constexpr` beats a regular `if` here is that with a regular `if`, both branches must compile even if one is never taken. With `if constexpr`, the compiler only instantiates the branch that is actually active for this build. That means the debug logging code never appears in a release binary at all - not as dead code, not as eliminated code, simply not compiled.

### Q2: Use policy classes for compile-time behavior injection

**Answer:**

Policy classes are a way of injecting behavior into a class at compile time by making the behavior a template parameter. The class you are customizing does not know or care which policy it gets - it just calls the policy's interface. The compiler selects the right implementation and inlines everything.

```cpp
// === Policy classes: inject behavior at compile time ===

// Allocation policy
struct HeapAllocation {
    template<typename T>
    static T* allocate(size_t n) {
        return new T[n];
    }
    template<typename T>
    static void deallocate(T* p) {
        delete[] p;
    }
};

struct PoolAllocation {
    template<typename T>
    static T* allocate(size_t n) {
        static std::array<std::byte, 4096> pool;
        static size_t offset = 0;
        T* ptr = reinterpret_cast<T*>(pool.data() + offset);
        offset += sizeof(T) * n;
        return ptr;
    }
    template<typename T>
    static void deallocate(T*) {
        // Pool doesn't free individual allocations
    }
};

// Threading policy
struct SingleThreaded {
    struct Lock { Lock() {} };
    // Zero-overhead: no mutex, no atomic
};

struct MultiThreaded {
    struct Lock {
        Lock() { mutex_.lock(); }
        ~Lock() { mutex_.unlock(); }
        static std::mutex mutex_;
    };
};

// === Class parameterized by policies ===
template<typename AllocPolicy = HeapAllocation,
         typename ThreadPolicy = MultiThreaded>
class ConnectionPool {
public:
    void add_connection() {
        typename ThreadPolicy::Lock lock;
        auto* conn = AllocPolicy::template allocate<Connection>(1);
        connections_.push_back(conn);
    }

    void remove_connection(Connection* conn) {
        typename ThreadPolicy::Lock lock;
        // ...
        AllocPolicy::template deallocate(conn);
    }

private:
    std::vector<Connection*> connections_;
};

// Compile-time configuration:
using ProdPool     = ConnectionPool<HeapAllocation, MultiThreaded>;
using EmbeddedPool = ConnectionPool<PoolAllocation, SingleThreaded>;
using TestPool     = ConnectionPool<HeapAllocation, SingleThreaded>;
```

Notice that `SingleThreaded::Lock` does nothing - its constructor and destructor are empty. When the compiler inlines a `SingleThreaded` policy, the lock acquisition disappears entirely. There is no branch, no atomic, no mutex - just zero overhead. That is the promise of policy-based design.

### Q3: Compile-time feature flags with concepts

**Answer:**

Taking the policy idea one step further, you can encode an entire build configuration as a single type. Each configuration type bundles all its feature flags together, and the application is parameterized by the configuration type. The `if constexpr` checks inside then select exactly the right behavior for each build variant.

```cpp
#include <concepts>

// === Feature flags as types ===
struct FeatureEnabled  { static constexpr bool value = true; };
struct FeatureDisabled { static constexpr bool value = false; };

// === Build configuration as a type bundle ===
struct ProductionConfig {
    using Logging   = FeatureEnabled;
    using Metrics   = FeatureEnabled;
    using Tracing   = FeatureDisabled;
    using DebugUI   = FeatureDisabled;
    static constexpr size_t WorkerCount = 16;
};

struct DebugConfig {
    using Logging   = FeatureEnabled;
    using Metrics   = FeatureEnabled;
    using Tracing   = FeatureEnabled;
    using DebugUI   = FeatureEnabled;
    static constexpr size_t WorkerCount = 2;
};

struct EmbeddedConfig {
    using Logging   = FeatureDisabled;
    using Metrics   = FeatureDisabled;
    using Tracing   = FeatureDisabled;
    using DebugUI   = FeatureDisabled;
    static constexpr size_t WorkerCount = 1;
};

// === Application parameterized by config ===
template<typename Cfg>
class Application {
public:
    void initialize() {
        if constexpr (Cfg::Logging::value) {
            init_logging();
        }
        if constexpr (Cfg::Metrics::value) {
            init_metrics();
        }
        if constexpr (Cfg::Tracing::value) {
            init_tracing();  // Only in debug builds
        }
        if constexpr (Cfg::DebugUI::value) {
            init_debug_ui(); // Only in debug builds
        }

        // Worker count known at compile time
        std::array<Worker, Cfg::WorkerCount> workers;
        for (auto& w : workers) w.start();
    }
};

// Select at build time:
#ifdef NDEBUG
using App = Application<ProductionConfig>;
#else
using App = Application<DebugConfig>;
#endif

// Feature checks in code:
template<typename Cfg>
void maybe_log(const char* msg) {
    if constexpr (Cfg::Logging::value) {
        printf("%s\n", msg);
    }
    // When disabled: function body is EMPTY, completely optimized away
}
```

The worker array (`std::array<Worker, Cfg::WorkerCount>`) is a good example of why compile-time values matter more than they might seem. Because the size is a compile-time constant, the compiler can allocate the array on the stack and unroll loops over it. If it were a runtime variable, none of that would be possible.

---

## Notes

- `if constexpr` eliminates dead branches entirely - the compiler does not even check the syntax of the unused branch for dependent expressions, which is different from a regular `if`.
- `static_assert` validates configuration at compile time, catching errors before the program ever runs.
- Policy classes combined with templates give zero-overhead customization with no vtable and full inlining.
- Prefer `constexpr` variables over `#define` - they are type-safe, scoped to their namespace, and visible in the debugger.
- CMake passes configuration into the build with `target_compile_definitions(app PRIVATE PLATFORM_EMBEDDED)`.
- For embedded targets, compile-time configuration avoids runtime overhead and reduces binary size, which is often a hard constraint.
- C++20 concepts can constrain policy types so that a wrong policy gives a readable error message instead of a wall of template noise.
