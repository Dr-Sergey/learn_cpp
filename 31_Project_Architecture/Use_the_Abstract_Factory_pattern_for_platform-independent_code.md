# Use the Abstract Factory pattern for platform-independent code

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

The **Abstract Factory** pattern provides an interface for creating families of related objects without specifying concrete types. In C++ projects, it's the standard approach for isolating platform-specific code (Windows/Linux/macOS, different hardware targets) behind a single creation interface, so the rest of the codebase stays platform-agnostic.

The key insight is "family of objects." It's not just about creating one thing abstractly - it's about making sure that when you're running on Linux, you get a Linux filesystem, a Linux network client, and a Linux timer together. You never accidentally mix a Windows timer with a Linux filesystem, because the factory is what hands you the whole consistent set.

### When to Use Abstract Factory

| Scenario | Alternative | Why Abstract Factory Wins |
| --- | --- | --- |
| Multi-platform GUI/IO | `#ifdef` everywhere | Centralizes platform deps |
| Embedded multi-target | Separate builds | Shared business logic |
| Plugin-created objects | Direct construction | Decouples plugin types |
| Test doubles family | Individual mocks | Consistent mock families |

---

## Self-Assessment

### Q1: Implement Abstract Factory for platform-independent I/O

**Answer:**

The structure has two layers. First you define the abstract product interfaces - `IFileSystem`, `INetworkClient`, `ITimer` - which are the things your application logic will actually use. Then you define the abstract factory `IPlatformFactory` whose job is to create one consistent set of those products. Business logic only ever sees the interfaces; it has no idea which platform it's running on.

```cpp
#include <memory>
#include <string>
#include <iostream>
#include <fstream>

// === Abstract product interfaces ===
class IFileSystem {
public:
    virtual ~IFileSystem() = default;
    virtual bool exists(const std::string& path) const = 0;
    virtual std::string read(const std::string& path) const = 0;
    virtual void write(const std::string& path,
                        const std::string& data) = 0;
};

class INetworkClient {
public:
    virtual ~INetworkClient() = default;
    virtual std::string get(const std::string& url) = 0;
    virtual void post(const std::string& url,
                       const std::string& body) = 0;
};

class ITimer {
public:
    virtual ~ITimer() = default;
    virtual void start(int ms, std::function<void()> callback) = 0;
    virtual void stop() = 0;
};

// === Abstract Factory ===
class IPlatformFactory {
public:
    virtual ~IPlatformFactory() = default;
    virtual std::unique_ptr<IFileSystem> create_filesystem() = 0;
    virtual std::unique_ptr<INetworkClient> create_network() = 0;
    virtual std::unique_ptr<ITimer> create_timer() = 0;
};

// === Linux implementations ===
class LinuxFileSystem : public IFileSystem {
public:
    bool exists(const std::string& path) const override {
        return access(path.c_str(), F_OK) == 0;
    }
    std::string read(const std::string& path) const override {
        std::ifstream f(path);
        return {std::istreambuf_iterator<char>(f), {}};
    }
    void write(const std::string& path, const std::string& data) override {
        std::ofstream f(path);
        f << data;
    }
};

// (LinuxNetworkClient, LinuxTimer similarly)

class LinuxFactory : public IPlatformFactory {
public:
    std::unique_ptr<IFileSystem> create_filesystem() override {
        return std::make_unique<LinuxFileSystem>();
    }
    std::unique_ptr<INetworkClient> create_network() override {
        return std::make_unique<LinuxNetworkClient>();
    }
    std::unique_ptr<ITimer> create_timer() override {
        return std::make_unique<LinuxTimer>();
    }
};

// === Windows factory similarly ===
// class WindowsFactory : public IPlatformFactory { ... };

// === Application code: platform-agnostic ===
class Application {
public:
    explicit Application(IPlatformFactory& factory)
        : fs_(factory.create_filesystem()),
          net_(factory.create_network()),
          timer_(factory.create_timer()) {}

    void run() {
        auto config = fs_->read("config.json");
        auto data = net_->get("https://api.example.com/data");
        // ... business logic using platform abstractions ...
    }

private:
    std::unique_ptr<IFileSystem> fs_;
    std::unique_ptr<INetworkClient> net_;
    std::unique_ptr<ITimer> timer_;
};
```

Notice that `Application` receives the factory by reference in its constructor and immediately creates all three products. After construction, `Application::run()` never touches the factory again - it only uses the products. This is the pattern working as intended: the platform decision is made once at startup (in `main`, the composition root), and then the application proceeds with zero knowledge of which platform it's on.

### Q2: Create a test factory for the entire product family

**Answer:**

Here's where the Abstract Factory pattern really pays off in testing. Instead of mocking individual classes case by case, you create one `TestFactory` that hands you an entire consistent family of fakes. The `last_fs_`, `last_net_`, and `last_timer_` raw pointers let you reach into the fakes from the test to set up pre-conditions and verify post-conditions.

```cpp
// === In-memory test doubles ===
class FakeFileSystem : public IFileSystem {
public:
    bool exists(const std::string& path) const override {
        return files_.count(path) > 0;
    }
    std::string read(const std::string& path) const override {
        return files_.at(path);
    }
    void write(const std::string& path, const std::string& data) override {
        files_[path] = data;
    }
    // Test helper: pre-populate files
    void seed(const std::string& path, const std::string& data) {
        files_[path] = data;
    }
private:
    std::unordered_map<std::string, std::string> files_;
};

class FakeNetworkClient : public INetworkClient {
public:
    std::string get(const std::string& url) override {
        return responses_[url];
    }
    void post(const std::string& url, const std::string& body) override {
        posts_.emplace_back(url, body);
    }
    // Test helpers
    void stub_response(const std::string& url, const std::string& resp) {
        responses_[url] = resp;
    }
    std::vector<std::pair<std::string, std::string>> posts_;
private:
    std::unordered_map<std::string, std::string> responses_;
};

class FakeTimer : public ITimer {
public:
    void start(int ms, std::function<void()> cb) override {
        callback_ = cb; ms_ = ms;
    }
    void stop() override { callback_ = nullptr; }
    void fire() { if (callback_) callback_(); }  // Manual trigger in tests
private:
    std::function<void()> callback_;
    int ms_ = 0;
};

// === Test factory ===
class TestFactory : public IPlatformFactory {
public:
    std::unique_ptr<IFileSystem> create_filesystem() override {
        auto fs = std::make_unique<FakeFileSystem>();
        last_fs_ = fs.get();
        return fs;
    }
    std::unique_ptr<INetworkClient> create_network() override {
        auto net = std::make_unique<FakeNetworkClient>();
        last_net_ = net.get();
        return net;
    }
    std::unique_ptr<ITimer> create_timer() override {
        auto timer = std::make_unique<FakeTimer>();
        last_timer_ = timer.get();
        return timer;
    }
    // Access created fakes for test setup/verification
    FakeFileSystem* last_fs_ = nullptr;
    FakeNetworkClient* last_net_ = nullptr;
    FakeTimer* last_timer_ = nullptr;
};

// === Test ===
TEST(ApplicationTest, LoadsConfigAndFetchesData) {
    TestFactory factory;
    Application app(factory);

    factory.last_fs_->seed("config.json", R"({"api": "url"})");
    factory.last_net_->stub_response("https://api.example.com/data",
                                       R"({"status": "ok"})");
    app.run();  // Uses fakes - fast, deterministic, no I/O
}
```

The test runs fast, touches no disk, makes no network calls, and is completely deterministic. That's the payoff: you can run thousands of tests like this in a second.

### Q3: Compile-time factory selection with zero overhead

**Answer:**

When you're in an embedded or performance-critical context and you know the platform at build time, you can eliminate all the virtual dispatch overhead entirely using a template-based factory. Each "platform" is just a struct that declares type aliases, and the application template instantiates those concrete types directly.

```cpp
// === Compile-time Abstract Factory using templates ===
struct LinuxPlatform {
    using FileSystem = LinuxFileSystem;
    using Network    = LinuxNetworkClient;
    using Timer      = LinuxTimer;
};

struct WindowsPlatform {
    using FileSystem = WindowsFileSystem;
    using Network    = WindowsNetworkClient;
    using Timer      = WindowsTimer;
};

struct TestPlatform {
    using FileSystem = FakeFileSystem;
    using Network    = FakeNetworkClient;
    using Timer      = FakeTimer;
};

template<typename Platform>
class ApplicationT {
public:
    void run() {
        auto config = fs_.read("config.json");
        // ... zero virtual dispatch overhead
    }
private:
    typename Platform::FileSystem fs_;
    typename Platform::Network net_;
    typename Platform::Timer timer_;
};

// Select at compile time:
#ifdef _WIN32
using App = ApplicationT<WindowsPlatform>;
#elif defined(__linux__)
using App = ApplicationT<LinuxPlatform>;
#endif
// Tests: using TestApp = ApplicationT<TestPlatform>;
```

The trade-off is that you can't swap implementations at runtime - but for embedded targets that's usually exactly what you want. The compiler can see through every call, inline aggressively, and produce tighter code than a vtable-based design.

---

## Notes

- Abstract Factory ensures you get a **consistent family** of objects (all Linux or all Windows, never mixed) - this is the guarantee that makes the pattern worth its complexity.
- Virtual interface factories give you runtime flexibility at the cost of a slight vtable overhead per call.
- Template-based factories give you zero overhead and compile-time selection - the preferred choice for embedded or performance-sensitive code.
- Keep factories in the composition root; business logic should never call factory methods directly - it only uses the products it was given.
- Test factories let you write fast, deterministic tests for the entire application without any real I/O.
- Avoid proliferating factory interfaces - one factory per platform-variable product family is enough; adding more usually means you're solving the wrong problem.
