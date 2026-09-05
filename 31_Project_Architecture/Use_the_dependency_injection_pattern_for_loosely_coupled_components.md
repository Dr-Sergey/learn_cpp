# Use the dependency injection pattern for loosely coupled components

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Dependency Injection (DI)** means passing a class its dependencies from the outside rather than letting the class create them internally. Instead of writing `db_ = new SqliteDatabase()` inside a class, you accept a database reference in the constructor. The caller decides which concrete implementation to provide.

This sounds like a small mechanical change, but it has large consequences: the class becomes testable (you can pass a fake), its requirements become visible (the constructor signature tells you everything it needs), and you can swap implementations without changing the class at all.

There are three main DI styles in C++. Constructor injection is the default. Setter injection is used for optional or runtime-changeable dependencies. Template injection eliminates vtable overhead entirely at the cost of compile-time binding.

### DI Styles Comparison

| Style | When to Use | Pros | Cons |
| --- | --- | --- | --- |
| **Constructor** | Required dependencies | Immutable, clear contract | Long parameter lists |
| **Setter** | Optional dependencies | Flexible, change at runtime | Nullable state |
| **Interface** | Plugin systems | Open for extension | More boilerplate |
| **Template** | Compile-time DI | Zero overhead, inlined | No runtime swap |

---

## Self-Assessment

### Q1: Implement constructor-based DI for a service

**Answer:**

The structure here is: define interfaces for each dependency, write a service that takes those interfaces by reference in its constructor, and then wire up the concrete implementations in `main`. The service never mentions `ConsoleLogger`, `InMemoryDb`, or `SmtpSender` - it only knows about `ILogger`, `IDatabase`, and `IEmailSender`.

```cpp
#include <memory>
#include <string>
#include <iostream>
#include <unordered_map>

// === Interfaces (dependencies) ===
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& msg) = 0;
};

class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual void save(const std::string& key, const std::string& value) = 0;
    virtual std::string load(const std::string& key) = 0;
};

class IEmailSender {
public:
    virtual ~IEmailSender() = default;
    virtual void send(const std::string& to, const std::string& body) = 0;
};

// === Service with constructor injection ===
class UserRegistrationService {
public:
    // All dependencies injected via constructor
    UserRegistrationService(IDatabase& db, ILogger& logger, IEmailSender& email)
        : db_(db), logger_(logger), email_(email) {}

    void register_user(const std::string& name, const std::string& email_addr) {
        logger_.log("Registering user: " + name);
        db_.save("user:" + name, email_addr);
        email_.send(email_addr, "Welcome, " + name + "!");
        logger_.log("User registered successfully");
    }

private:
    IDatabase& db_;
    ILogger& logger_;
    IEmailSender& email_;
};

// === Concrete implementations ===
class ConsoleLogger : public ILogger {
public:
    void log(const std::string& msg) override {
        std::cout << "[LOG] " << msg << "\n";
    }
};

class InMemoryDb : public IDatabase {
public:
    void save(const std::string& key, const std::string& value) override {
        data_[key] = value;
    }
    std::string load(const std::string& key) override {
        return data_[key];
    }
private:
    std::unordered_map<std::string, std::string> data_;
};

class SmtpSender : public IEmailSender {
public:
    void send(const std::string& to, const std::string& body) override {
        // Real SMTP sending
    }
};

// === Composition root ===
int main() {
    ConsoleLogger logger;
    InMemoryDb db;
    SmtpSender email;

    // Wire dependencies
    UserRegistrationService service(db, logger, email);
    service.register_user("Alice", "alice@example.com");
}
```

`main` is the composition root - the one place in the program that knows all the concrete types and wires them together. Everything else in the codebase talks to interfaces. This is what makes DI scale well: adding a new service means adding it to the composition root, not touching existing classes.

### Q2: Use template-based DI for zero-overhead injection

**Answer:**

Virtual dispatch has a real cost: an indirect function call through a vtable pointer. For most application code the cost is negligible, but in tight inner loops or on embedded hardware it matters. Template-based DI resolves everything at compile time - no vtable, no indirection, the compiler can inline across the dependency boundary.

The trade-off is that you lose runtime flexibility: you can't swap a `SmtpSender` for a `MockSender` at runtime, only at compile time. For production/test separation via build configuration, that's usually fine.

```cpp
#include <string>
#include <iostream>
#include <unordered_map>

// === Template DI: dependencies resolved at compile time ===
template<typename DB, typename Logger, typename Email>
class UserRegistrationServiceT {
public:
    UserRegistrationServiceT(DB& db, Logger& logger, Email& email)
        : db_(db), logger_(logger), email_(email) {}

    void register_user(const std::string& name, const std::string& addr) {
        logger_.log("Registering: " + name);
        db_.save("user:" + name, addr);
        email_.send(addr, "Welcome!");
    }

private:
    DB& db_;
    Logger& logger_;
    Email& email_;
};

// No virtual dispatch overhead - everything is inlined.
// Trade-off: can't swap implementations at runtime.

// === Usage in production ===
// auto service = UserRegistrationServiceT(real_db, real_logger, real_email);

// === Usage in tests ===
struct FakeDb {
    void save(const std::string&, const std::string&) {}
    std::string load(const std::string&) { return {}; }
};
struct FakeLogger {
    std::vector<std::string> messages;
    void log(const std::string& msg) { messages.push_back(msg); }
};
struct FakeEmail {
    int send_count = 0;
    void send(const std::string&, const std::string&) { send_count++; }
};

// Test:
// FakeDb db; FakeLogger log; FakeEmail email;
// auto svc = UserRegistrationServiceT(db, log, email);
// svc.register_user("Test", "test@test.com");
// EXPECT_EQ(email.send_count, 1);
```

Notice that the fakes don't need to inherit from any interface - they just need to have the right method signatures. The template instantiation checks compatibility at compile time. This is duck typing at the type level.

### Q3: Build a simple DI container

**Answer:**

As a project grows, manually wiring everything in `main` can become verbose. A DI container automates that wiring: you register how each interface should be created (a factory lambda or a singleton instance), and then call `resolve<T>()` to get a fully wired instance with all its dependencies provided automatically.

This is more machinery than most C++ projects need, but it's worth understanding so you recognize the pattern when you encounter it.

```cpp
#include <any>
#include <functional>
#include <memory>
#include <typeindex>
#include <unordered_map>
#include <stdexcept>

// === Lightweight DI Container ===
class Container {
public:
    // Register a factory for an interface type
    template<typename Interface>
    void bind(std::function<std::shared_ptr<Interface>(Container&)> factory) {
        factories_[typeid(Interface)] = [factory](Container& c) -> std::any {
            return std::static_pointer_cast<void>(factory(c));
        };
    }

    // Register a singleton
    template<typename Interface>
    void bind_singleton(std::shared_ptr<Interface> instance) {
        singletons_[typeid(Interface)] = std::static_pointer_cast<void>(instance);
    }

    // Resolve a dependency
    template<typename Interface>
    std::shared_ptr<Interface> resolve() {
        // Check singletons first
        auto sit = singletons_.find(typeid(Interface));
        if (sit != singletons_.end())
            return std::static_pointer_cast<Interface>(
                std::any_cast<std::shared_ptr<void>>(sit->second));

        // Check factories
        auto fit = factories_.find(typeid(Interface));
        if (fit == factories_.end())
            throw std::runtime_error("No binding for type");

        auto result = fit->second(*this);
        return std::static_pointer_cast<Interface>(
            std::any_cast<std::shared_ptr<void>>(result));
    }

private:
    std::unordered_map<std::type_index,
        std::function<std::any(Container&)>> factories_;
    std::unordered_map<std::type_index, std::any> singletons_;
};

// === Usage ===
void setup_production(Container& c) {
    c.bind_singleton<ILogger>(std::make_shared<ConsoleLogger>());
    c.bind<IDatabase>([](Container&) {
        return std::make_shared<InMemoryDb>();
    });
    c.bind<IEmailSender>([](Container&) {
        return std::make_shared<SmtpSender>();
    });
}

// In main:
// Container c;
// setup_production(c);
// auto logger = c.resolve<ILogger>();
// auto db = c.resolve<IDatabase>();
```

The container resolves singletons by returning the same shared instance every time, and resolves factory-bound types by calling the lambda fresh. The factory lambda receives the container itself, so a factory for a complex type can call `c.resolve<IDependency>()` to get its own dependencies - this is how the automatic wiring works.

---

## Notes

- **Constructor injection** is the default choice - dependencies are explicit, immutable after construction, and visible at the call site without reading the implementation.
- **Template DI** gives zero overhead but loses runtime flexibility - use it for performance-critical paths or on constrained embedded targets.
- A DI container is optional in C++ - manual wiring in `main()` is often sufficient for all but the largest projects. Don't add a container just because it exists.
- If constructor parameter lists grow long (more than four or five arguments), that's a signal to split the class into smaller, more focused units rather than a signal to switch to setter injection.
- DI makes testing straightforward: inject fakes or mocks instead of real implementations, and your tests run fast without any real I/O or network access.
- Prefer passing dependencies by reference rather than `shared_ptr` when the dependency outlives the object - references communicate non-ownership clearly and avoid unnecessary reference counting.
- The **composition root** (`main.cpp`) is the single place in the program that knows all concrete types - everything else in the codebase depends only on abstractions.
