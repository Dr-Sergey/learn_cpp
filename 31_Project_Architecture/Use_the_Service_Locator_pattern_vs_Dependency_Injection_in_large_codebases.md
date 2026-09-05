# Use the Service Locator pattern vs Dependency Injection in large codebases

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Service Locator** is a registry where objects reach out and ask for their dependencies at runtime. **Dependency Injection** pushes dependencies into objects from the outside, typically through the constructor. Both patterns solve the same problem - decoupling a class from the concrete types it depends on - but with very different trade-offs.

The reason this topic is worth understanding carefully is that Service Locator looks simpler at first. Any class can call the locator and get what it needs, with no constructor changes required. But this hides dependencies inside implementations instead of declaring them up front, and that hidden-ness makes code genuinely harder to reason about, test, and maintain. DI makes dependencies visible and verifiable at compile time. In modern C++, DI is the default choice - but Service Locator has real legitimate uses and you'll encounter it in existing codebases.

### Pattern Comparison

| Aspect | Service Locator | Dependency Injection |
| --- | --- | --- |
| **Dependency visibility** | Hidden (ask the locator) | Explicit (in constructor) |
| **Testability** | Must configure locator | Pass mocks directly |
| **Compile-time safety** | Runtime resolution | Compile-time verification |
| **Coupling** | All code depends on locator | Only composition root wires |
| **Legacy migration** | Easy to retrofit | Requires refactoring |
| **Plugin systems** | Natural fit | Awkward with dynamic loading |

---

## Self-Assessment

### Q1: Implement a Service Locator and show its drawbacks

**Answer:**

Here's a standard Service Locator implementation. Read through it and then look closely at `PaymentProcessor` - the problem becomes obvious when you try to construct one in a test.

```cpp
#include <any>
#include <typeindex>
#include <unordered_map>
#include <memory>
#include <stdexcept>

// === Service Locator ===
class ServiceLocator {
public:
    static ServiceLocator& instance() {
        static ServiceLocator sl;
        return sl;
    }

    template<typename T>
    void register_service(std::shared_ptr<T> service) {
        services_[typeid(T)] = std::move(service);
    }

    template<typename T>
    std::shared_ptr<T> resolve() {
        auto it = services_.find(typeid(T));
        if (it == services_.end())
            throw std::runtime_error("Service not registered");
        return std::any_cast<std::shared_ptr<T>>(it->second);
    }

    void clear() { services_.clear(); }

private:
    ServiceLocator() = default;
    std::unordered_map<std::type_index, std::any> services_;
};

// === Usage: dependencies are HIDDEN ===
class PaymentProcessor {
public:
    void process(double amount) {
        // BAD: dependency is invisible from the outside
        auto logger = ServiceLocator::instance().resolve<ILogger>();
        auto gateway = ServiceLocator::instance().resolve<IPaymentGateway>();
        logger->log("Processing $" + std::to_string(amount));
        gateway->charge(amount);
    }
    // Problem: the caller has no idea this needs ILogger and IPaymentGateway.
    // Reading the constructor tells you NOTHING about dependencies.
};

// === Same with DI: dependencies are EXPLICIT ===
class PaymentProcessorDI {
public:
    PaymentProcessorDI(ILogger& logger, IPaymentGateway& gateway)
        : logger_(logger), gateway_(gateway) {}

    void process(double amount) {
        logger_.log("Processing $" + std::to_string(amount));
        gateway_.charge(amount);
    }
    // CLEAR: constructor tells you exactly what this class needs.

private:
    ILogger& logger_;
    IPaymentGateway& gateway_;
};
```

The difference is stark. With `PaymentProcessor`, to write a test you have to know - by reading the implementation - that it needs `ILogger` and `IPaymentGateway`, then pre-configure the global locator before calling `process()`. Forget to register one of them and you get a runtime exception. With `PaymentProcessorDI`, the constructor tells you everything you need to know at a glance, and the compiler will reject any test that fails to provide the right types.

### Q2: Show a legitimate use for Service Locator (plugin system)

**Answer:**

So when should you actually use a Service Locator? The genuine use case is dynamic plugin systems. When plugins are loaded at runtime via `dlopen`/`LoadLibrary`, the host application has no idea what plugin types will appear - they're not in the codebase. You can't inject dependencies through constructors because constructors aren't part of a stable binary ABI the way a C function pointer is. Passing a context object that plugins can query is the natural solution here.

```cpp
// === Plugin system where DI is impractical ===
// Plugins are loaded at runtime via dlopen/LoadLibrary.
// We can't inject dependencies through constructors because
// the host doesn't know plugin types at compile time.

class PluginContext {
public:
    template<typename T>
    void provide(std::shared_ptr<T> service) {
        services_[typeid(T)] = std::move(service);
    }

    template<typename T>
    T& get() {
        auto it = services_.find(typeid(T));
        if (it == services_.end())
            throw std::runtime_error("Service unavailable");
        return *std::any_cast<std::shared_ptr<T>>(it->second);
    }

private:
    std::unordered_map<std::type_index, std::any> services_;
};

// Plugin interface
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual void initialize(PluginContext& ctx) = 0;
    virtual void execute() = 0;
};

// Host provides services to plugins via locator
void load_plugins(PluginContext& ctx,
                   const std::vector<std::string>& plugin_paths) {
    ctx.provide<ILogger>(std::make_shared<FileLogger>());
    ctx.provide<IConfig>(std::make_shared<AppConfig>());

    for (const auto& path : plugin_paths) {
        auto* handle = dlopen(path.c_str(), RTLD_LAZY);
        auto create = reinterpret_cast<IPlugin*(*)()>(
            dlsym(handle, "create_plugin"));
        auto plugin = std::unique_ptr<IPlugin>(create());
        plugin->initialize(ctx);  // Plugin fetches what it needs
    }
}

// Inside a plugin:
class MyPlugin : public IPlugin {
    ILogger* logger_ = nullptr;
public:
    void initialize(PluginContext& ctx) override {
        logger_ = &ctx.get<ILogger>();  // Service Locator is natural here
    }
    void execute() override {
        logger_->log("Plugin running");
    }
};
```

Notice this version is scoped (it's a `PluginContext` object, not a global singleton), and the set of services it holds is small and well-defined. That's the key difference from the anti-pattern: this is a bounded locator with a clear purpose, not a global grab-bag for all application dependencies.

### Q3: Migrate from Service Locator to DI incrementally

**Answer:**

If you're working in a legacy codebase that already uses Service Locator everywhere, a big-bang refactoring is usually too risky. The four-step incremental migration below lets you move toward DI one class at a time, keeping the code working throughout.

```cpp
// === Step 1: Wrap SL calls in constructors ===
// Before:
class OrderService {
    void save(Order& o) {
        auto db = ServiceLocator::instance().resolve<IDatabase>();
        db->insert(o);
    }
};

// After step 1: Move SL call to constructor (still uses SL, but concentrated)
class OrderService {
public:
    OrderService()
        : db_(*ServiceLocator::instance().resolve<IDatabase>()) {}

    void save(Order& o) { db_.insert(o); }
private:
    IDatabase& db_;
};

// === Step 2: Add DI constructor alongside SL constructor ===
class OrderService {
public:
    // Legacy path (still works with SL)
    OrderService()
        : db_(*ServiceLocator::instance().resolve<IDatabase>()) {}

    // New path (pure DI)
    explicit OrderService(IDatabase& db) : db_(db) {}

    void save(Order& o) { db_.insert(o); }
private:
    IDatabase& db_;
};

// === Step 3: Update callers to use DI constructor ===
// Old: auto svc = OrderService();           // Uses SL
// New: auto svc = OrderService(real_db);     // Uses DI

// === Step 4: Remove default constructor once all callers migrated ===
class OrderService {
public:
    explicit OrderService(IDatabase& db) : db_(db) {}
    void save(Order& o) { db_.insert(o); }
private:
    IDatabase& db_;
};
```

Step 1 is the most important one. By moving all locator calls to the constructor, you at least centralize the dependency acquisition in one place. The class body becomes clean. Steps 2-4 can then proceed at whatever pace the codebase allows.

---

## Notes

- **Default to Dependency Injection** for all new code - explicit dependencies in constructors are easier to read, test, and refactor.
- Service Locator is an anti-pattern when used as a global registry for all application dependencies - it's the "hidden dependency" smell taken to its logical extreme.
- Service Locator is legitimate for plugin systems, legacy migration paths, and dynamic service discovery where the set of available services isn't known at compile time.
- The key smell to watch for: if you can't tell what a class needs without reading its implementation body, it has hidden dependencies.
- DI combined with a composition root (usually `main.cpp`) gives you compile-time verification of the entire dependency graph - if you forget to wire something, you get a compiler error, not a runtime crash.
- In embedded C++ without dynamic loading, there's almost never a reason for Service Locator - the entire program is known at compile time, so DI works cleanly.
