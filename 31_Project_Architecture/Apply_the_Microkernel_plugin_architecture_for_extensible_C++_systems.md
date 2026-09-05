# Apply the Microkernel (plugin) architecture for extensible C++ systems

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

The **Microkernel architecture** keeps a minimal core system that provides essential services (plugin loading, messaging, lifecycle management) while all features are implemented as plugins. Unlike monolithic designs, the core knows nothing about specific functionality - it only manages the plugin infrastructure. This is the architecture behind IDEs, game engines, and audio workstations.

The key idea is that the kernel is deliberately kept dumb. It does not know how to log, process HTTP requests, or manage users - all it knows is how to load plugins, give them a communication channel, and call them in the right order. Every useful capability comes from a plugin. That means you can add or remove features by shipping a plugin, not by recompiling the core.

### Microkernel vs Monolithic

| Aspect | Microkernel | Monolithic |
| --- | --- | --- |
| Core size | Minimal (loader + bus) | Everything built-in |
| Adding features | Ship a new plugin | Recompile everything |
| Third-party extensions | Natural | Difficult |
| Startup complexity | Plugin discovery/ordering | Single init path |
| Inter-feature coupling | Message-based | Direct calls |

---

## Self-Assessment

### Q1: Implement a microkernel core with extension points

**Answer:**

The kernel has three components: an `ExtensionRegistry` (named hooks where plugins register capabilities), a `MessageBus` (asynchronous communication between plugins), and the `Kernel` itself (lifecycle management). Notice how thin the kernel is - it contains zero business logic:

```cpp
#include <memory>
#include <string>
#include <vector>
#include <unordered_map>
#include <functional>
#include <iostream>
#include <any>

// === Extension Point: named hook where plugins register ===
class ExtensionRegistry {
public:
    using Handler = std::function<std::any(const std::any&)>;

    void register_handler(const std::string& point,
                           const std::string& plugin_name,
                           Handler handler) {
        extensions_[point].push_back({plugin_name, std::move(handler)});
    }

    std::vector<std::any> invoke_all(const std::string& point,
                                      const std::any& arg) {
        std::vector<std::any> results;
        auto it = extensions_.find(point);
        if (it != extensions_.end()) {
            for (auto& [name, handler] : it->second) {
                results.push_back(handler(arg));
            }
        }
        return results;
    }

    std::any invoke_first(const std::string& point,
                           const std::any& arg) {
        auto it = extensions_.find(point);
        if (it != extensions_.end() && !it->second.empty())
            return it->second.front().handler(arg);
        return {};
    }

private:
    struct Entry {
        std::string plugin_name;
        Handler handler;
    };
    std::unordered_map<std::string, std::vector<Entry>> extensions_;
};

// === Message Bus: inter-plugin communication ===
class MessageBus {
public:
    using Subscriber = std::function<void(const std::any&)>;

    void subscribe(const std::string& topic, Subscriber sub) {
        subscribers_[topic].push_back(std::move(sub));
    }

    void publish(const std::string& topic, const std::any& message) {
        auto it = subscribers_.find(topic);
        if (it != subscribers_.end())
            for (auto& sub : it->second)
                sub(message);
    }

private:
    std::unordered_map<std::string,
        std::vector<Subscriber>> subscribers_;
};

// === Microkernel Core ===
class Kernel {
public:
    // Services available to plugins
    ExtensionRegistry& extensions() { return extensions_; }
    MessageBus& bus() { return bus_; }

    // Plugin lifecycle
    void register_plugin(std::unique_ptr<IPlugin> plugin) {
        plugins_.push_back(std::move(plugin));
    }

    void start() {
        // Phase 1: Initialize all plugins (register extensions)
        for (auto& p : plugins_)
            p->initialize(*this);
        // Phase 2: Start all plugins (begin processing)
        for (auto& p : plugins_)
            p->start();

        bus_.publish("kernel.started", {});
    }

    void stop() {
        bus_.publish("kernel.stopping", {});
        for (auto it = plugins_.rbegin(); it != plugins_.rend(); ++it)
            (*it)->stop();
    }

private:
    ExtensionRegistry extensions_;
    MessageBus bus_;
    std::vector<std::unique_ptr<IPlugin>> plugins_;
};

// === Plugin interface ===
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual const char* name() const = 0;
    virtual void initialize(Kernel& kernel) = 0;
    virtual void start() = 0;
    virtual void stop() = 0;
};
```

The two-phase start (`initialize` then `start`) is intentional. During `initialize`, all plugins register their extension handlers. Only after every plugin has registered does `start` run, so a plugin's startup code can safely call extension points registered by other plugins.

### Q2: Write feature plugins that communicate via the kernel

**Answer:**

Here is how two concrete plugins - logging and HTTP - are implemented without any reference to each other. The `HttpPlugin` sends log messages via the extension registry rather than calling `LoggingPlugin` directly. If you replace the logging plugin with a different one tomorrow, `HttpPlugin` does not change:

```cpp
// === LoggingPlugin: provides logging extension point ===
class LoggingPlugin : public IPlugin {
public:
    const char* name() const override { return "logging"; }

    void initialize(Kernel& kernel) override {
        kernel_ = &kernel;
        // Register as the logging handler
        kernel.extensions().register_handler("log", name(),
            [this](const std::any& arg) -> std::any {
                auto msg = std::any_cast<std::string>(arg);
                std::cout << "[LOG] " << msg << "\n";
                log_count_++;
                return {};
            });
    }

    void start() override {
        kernel_->bus().publish("log",
            std::string("Logging plugin started"));
    }
    void stop() override {}
private:
    Kernel* kernel_ = nullptr;
    int log_count_ = 0;
};

// === HttpPlugin: uses logging, provides HTTP extension ===
class HttpPlugin : public IPlugin {
public:
    const char* name() const override { return "http"; }

    void initialize(Kernel& kernel) override {
        kernel_ = &kernel;

        // Subscribe to requests
        kernel.bus().subscribe("http.request",
            [this](const std::any& msg) {
                auto url = std::any_cast<std::string>(msg);
                handle_request(url);
            });

        // Register route extension point
        kernel.extensions().register_handler("http.route", name(),
            [](const std::any&) -> std::any { return std::string("OK"); });
    }

    void start() override {
        // Log via extension point (communicates with LoggingPlugin)
        kernel_->extensions().invoke_all("log",
            std::string("HTTP server starting"));
    }
    void stop() override {}

private:
    void handle_request(const std::string& url) {
        kernel_->extensions().invoke_all("log",
            std::string("Request: " + url));
    }
    Kernel* kernel_ = nullptr;
};

// === Startup (composition root) ===
int main() {
    Kernel kernel;
    kernel.register_plugin(std::make_unique<LoggingPlugin>());
    kernel.register_plugin(std::make_unique<HttpPlugin>());
    kernel.start();
    // ... run application ...
    kernel.stop();
}
```

The `main` function is the only place that knows which plugins exist. Everything else is mediated through the kernel. This is the composition root pattern - the one place where you wire things together, so the rest of the code stays decoupled.

### Q3: Handle plugin dependencies and ordered startup

**Answer:**

In real systems, plugins have dependencies. A `HttpPlugin` might need `LoggingPlugin` to be initialized first. You can handle this by declaring dependencies explicitly and sorting the startup order with a topological sort. Kahn's algorithm is a clean fit here:

```cpp
// === Plugin with declared dependencies ===
class IPluginEx : public IPlugin {
public:
    // Override to declare dependency names
    virtual std::vector<std::string> dependencies() const { return {}; }
};

// === Topological sort for startup order ===
#include <algorithm>
#include <queue>

class KernelEx : public Kernel {
public:
    void start() {
        auto ordered = topological_sort();
        for (auto* p : ordered)
            p->initialize(*this);
        for (auto* p : ordered)
            p->start();
    }

private:
    std::vector<IPluginEx*> topological_sort() {
        // Build adjacency and in-degree
        std::unordered_map<std::string, IPluginEx*> by_name;
        std::unordered_map<std::string, int> in_degree;
        std::unordered_map<std::string, std::vector<std::string>> adj;

        for (auto& p : plugins_) {
            auto* pe = dynamic_cast<IPluginEx*>(p.get());
            if (!pe) continue;
            by_name[pe->name()] = pe;
            in_degree[pe->name()];  // ensure entry exists
            for (auto& dep : pe->dependencies()) {
                adj[dep].push_back(pe->name());
                in_degree[pe->name()]++;
            }
        }

        // Kahn's algorithm
        std::queue<std::string> q;
        for (auto& [name, deg] : in_degree)
            if (deg == 0) q.push(name);

        std::vector<IPluginEx*> result;
        while (!q.empty()) {
            auto name = q.front(); q.pop();
            result.push_back(by_name[name]);
            for (auto& dependent : adj[name]) {
                if (--in_degree[dependent] == 0)
                    q.push(dependent);
            }
        }

        if (result.size() != by_name.size())
            throw std::runtime_error("Circular plugin dependency");

        return result;
    }
};
```

The reason this trips people up: Kahn's algorithm processes nodes with zero in-degree first (no unresolved dependencies), then reduces the in-degree of everything that depended on them. If the sorted result is shorter than the plugin list, you have a cycle - which becomes a clean `runtime_error` rather than a mysterious hang.

---

## Notes

- The **kernel should contain zero business logic** - only plugin management, messaging, and extension points.
- Extension points are named hooks: plugins register handlers, and other plugins invoke them by name.
- The message bus enables loose coupling: publishers do not know who their subscribers are.
- Plugin ordering via topological sort prevents initialization race conditions when one plugin depends on another.
- For ABI safety with dynamic loading, keep `extern "C"` factory functions and C-compatible data at the library boundary.
- Real-world examples: JUCE (audio), Qt (plugins), Unreal Engine (modules).
- Errors in one plugin should not crash the kernel - wrap plugin calls in `try`/`catch`.
