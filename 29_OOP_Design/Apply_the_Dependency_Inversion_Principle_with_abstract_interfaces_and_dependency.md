# Apply the Dependency Inversion Principle with abstract interfaces and dependency injection

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

The **Dependency Inversion Principle (DIP)** says: high-level modules should not depend on low-level modules. Both should depend on abstractions. In plain English, your business logic should not reach out and grab a concrete `SqlDatabase` or `SmtpMailer` directly. Instead, it should talk to an interface, and whoever builds the application decides which concrete class plugs in.

This is the difference between wiring your components together at construction time (flexible, testable) versus baking the concrete dependencies into your class body (rigid, untestable).

```cpp
WITHOUT DIP:                           WITH DIP:
┌──────────┐                          ┌──────────┐
│ Business  │──────► SqlDatabase       │ Business  │──────► IDatabase (abstract)
│ Logic     │──────► SmtpMailer        │ Logic     │──────► IMailer (abstract)
└──────────┘                          └──────────┘
Hard dependency on                      │                    ▲
concrete implementations               │        ┌───────────┴──────────┐
                                        │        │                      │
                                        │   SqlDatabase            MockDatabase
                                        │   SmtpMailer             MockMailer
```

The diagram makes the core idea vivid: with DIP, the arrow from your business logic points at an abstraction (an interface), not at a concrete class. That means you can swap the concrete class without touching the business logic at all.

### Injection Styles in C++

There are several ways to hand a dependency to a class. The right choice depends on whether you need runtime flexibility, zero overhead, or just simplicity.

| Style | Syntax | When to use |
| --- | --- | --- |
| Constructor injection | `Service(std::unique_ptr<IDep>)` | Default choice - explicit deps |
| Template injection | `template<typename Dep> class Service` | Zero-overhead, compile-time |
| `std::function` injection | `Service(std::function<int()>)` | Simple callbacks |
| Setter injection | `void set_logger(ILogger*)` | Optional/late-bound deps |

Constructor injection is the default because every dependency is visible at the call site - there's no mystery about what a class needs to function.

---

## Self-Assessment

### Q1: Implement constructor-based dependency injection with interfaces

The pattern here is: define pure virtual interfaces for each dependency, then pass `unique_ptr` to those interfaces into the constructor. The high-level service never touches a concrete type.

**Answer:**

```cpp
#include <memory>
#include <string>
#include <vector>
#include <iostream>

// Abstract interfaces (the "abstractions")
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(std::string_view msg) = 0;
};

class IUserRepository {
public:
    virtual ~IUserRepository() = default;
    virtual void save(const std::string& name, const std::string& email) = 0;
    virtual std::vector<std::string> find_all() = 0;
};

class IEmailService {
public:
    virtual ~IEmailService() = default;
    virtual bool send(const std::string& to, const std::string& subject,
                      const std::string& body) = 0;
};

// High-level module: depends ONLY on abstractions
class UserRegistrationService {
    std::unique_ptr<IUserRepository> repo_;
    std::unique_ptr<IEmailService> email_;
    std::unique_ptr<ILogger> logger_;

public:
    // Constructor injection: ALL dependencies are explicit
    UserRegistrationService(
        std::unique_ptr<IUserRepository> repo,
        std::unique_ptr<IEmailService> email,
        std::unique_ptr<ILogger> logger)
        : repo_(std::move(repo))
        , email_(std::move(email))
        , logger_(std::move(logger)) {}

    bool register_user(const std::string& name, const std::string& email) {
        logger_->log("Registering user: " + name);

        repo_->save(name, email);

        if (!email_->send(email, "Welcome!", "Hello " + name)) {
            logger_->log("Failed to send welcome email to " + email);
            return false;
        }

        logger_->log("User registered successfully: " + name);
        return true;
    }
};

// Production implementations
class ConsoleLogger : public ILogger {
public:
    void log(std::string_view msg) override {
        std::cout << "[LOG] " << msg << "\n";
    }
};

class SqlUserRepository : public IUserRepository {
public:
    void save(const std::string& name, const std::string& email) override {
        // INSERT INTO users ...
    }
    std::vector<std::string> find_all() override { return {}; }
};

class SmtpEmailService : public IEmailService {
public:
    bool send(const std::string& to, const std::string& subject,
              const std::string& body) override {
        // Connect to SMTP...
        return true;
    }
};

// Test mocks
class MockLogger : public ILogger {
public:
    std::vector<std::string> messages;
    void log(std::string_view msg) override {
        messages.emplace_back(msg);
    }
};

class MockRepo : public IUserRepository {
public:
    std::vector<std::pair<std::string, std::string>> saved;
    void save(const std::string& name, const std::string& email) override {
        saved.emplace_back(name, email);
    }
    std::vector<std::string> find_all() override { return {}; }
};

class MockEmail : public IEmailService {
public:
    bool should_fail = false;
    int send_count = 0;
    bool send(const std::string&, const std::string&, const std::string&) override {
        ++send_count;
        return !should_fail;
    }
};

// Composition root (wiring)
void production_main() {
    auto service = UserRegistrationService(
        std::make_unique<SqlUserRepository>(),
        std::make_unique<SmtpEmailService>(),
        std::make_unique<ConsoleLogger>()
    );
    service.register_user("Alice", "alice@example.com");
}

void test_registration() {
    auto logger = std::make_unique<MockLogger>();
    auto repo = std::make_unique<MockRepo>();
    auto email = std::make_unique<MockEmail>();

    auto* repo_ptr = repo.get();
    auto* email_ptr = email.get();

    auto service = UserRegistrationService(
        std::move(repo), std::move(email), std::move(logger));

    service.register_user("Bob", "bob@test.com");
    assert(repo_ptr->saved.size() == 1);
    assert(email_ptr->send_count == 1);
}
```

Notice that `UserRegistrationService` has no `#include <SqlUserRepository>` anywhere in it. It simply calls `repo_->save(...)` and trusts the interface. In tests, you swap in `MockRepo` and verify it was called correctly without touching the service code at all. That is exactly what DIP buys you.

### Q2: Show template-based dependency injection for zero-overhead DIP

When the concrete dependency type is known at compile time and you want zero vtable cost, you can inject via template parameters. C++20 concepts let you express the interface contract cleanly.

**Answer:**

```cpp
#include <string>
#include <vector>
#include <iostream>
#include <concepts>
#include <string_view>
#include <utility>
#include <cassert>

// Template DIP: compile-time injection, zero vtable overhead

// "Concepts" define the interface contract (C++20)
template<typename T>
concept Logger = requires(T& t, std::string_view msg) {
    { t.log(msg) };
};

template<typename T>
concept Repository = requires(T& t, const std::string& s) {
    { t.save(s) };
    { t.find_all() } -> std::same_as<std::vector<std::string>>;
};

// High-level module: template parameters instead of virtual interfaces
template<Logger LoggerT, Repository RepoT>
class OrderService {
    LoggerT logger_;
    RepoT repo_;

public:
    OrderService(LoggerT logger, RepoT repo)
        : logger_(std::move(logger)), repo_(std::move(repo)) {}

    void place_order(const std::string& item) {
        logger_.log("Placing order: " + item);
        repo_.save(item);
        logger_.log("Order placed");
    }

    auto all_orders() { return repo_.find_all(); }
};

// Production implementations (no virtual, no inheritance needed)
struct FileLogger {
    void log(std::string_view msg) {
        std::cout << "[FILE] " << msg << "\n";
    }
};

struct InMemoryRepo {
    std::vector<std::string> items;
    void save(const std::string& s) { items.push_back(s); }
    std::vector<std::string> find_all() { return items; }
};

// Test doubles (also no inheritance)
struct NullLogger {
    void log(std::string_view) {}  // Discard
};

struct SpyRepo {
    std::vector<std::string> saved;
    void save(const std::string& s) { saved.push_back(s); }
    std::vector<std::string> find_all() { return saved; }
};

int main() {
    // Production: FileLogger + real repo
    auto prod = OrderService<FileLogger, InMemoryRepo>({}, {});
    prod.place_order("Widget");

    // Test: NullLogger + spy repo
    auto test_svc = OrderService<NullLogger, SpyRepo>({}, {});
    test_svc.place_order("Test Item");
    // Verify: spy has the item
    assert(test_svc.all_orders().size() == 1);
    return 0;
}
```

The reason to reach for templates instead of virtuals is performance: the compiler sees the exact concrete type, inlines the calls, and eliminates the vtable dispatch entirely. The tradeoff is that you lose the ability to swap implementations at runtime and binary size grows from monomorphization. The table below helps you pick:

**Virtual vs Template DIP:**

| Aspect | Virtual (runtime) | Template (compile-time) |
| --- | :---: | :---: |
| Overhead | vtable + indirection | Zero |
| Binary size | Smaller | Larger (monomorphization) |
| Swap at runtime | Yes | No |
| Plugin loading | Yes | No |
| Compile times | Faster | Slower |
| Error messages | Clear | Can be complex (use concepts!) |

### Q3: Show a composition root that wires an entire application

The **composition root** is the single place in the codebase that knows about concrete types and builds the full object graph. Everything else talks only to interfaces. Keeping all that wiring in one spot makes substituting implementations trivial - you change one factory method, not a dozen scattered files.

**Answer:**

```cpp
#include <memory>
#include <string>

// Composition Root pattern - wire deps in ONE place

// Forward declare interfaces
class IConfig;
class IDatabase;
class ICache;
class ILogger;
class IAuthService;
class IOrderService;
class IPaymentGateway;

// Application: holds the entire object graph
class Application {
    // Infrastructure layer
    std::unique_ptr<IConfig> config_;
    std::unique_ptr<ILogger> logger_;
    std::unique_ptr<IDatabase> db_;
    std::unique_ptr<ICache> cache_;

    // Service layer
    std::unique_ptr<IAuthService> auth_;
    std::unique_ptr<IOrderService> orders_;
    std::unique_ptr<IPaymentGateway> payments_;

public:
    // The COMPOSITION ROOT: all wiring happens here
    static Application create_production() {
        Application app;
        // Infrastructure
        app.config_ = make_file_config("/etc/app/config.yaml");
        app.logger_ = make_syslog_logger(app.config_.get());
        app.db_ = make_postgres_db(app.config_.get(), app.logger_.get());
        app.cache_ = make_redis_cache(app.config_.get());

        // Services - injected with their dependencies
        app.auth_ = make_auth_service(app.db_.get(), app.cache_.get());
        app.orders_ = make_order_service(app.db_.get(), app.logger_.get());
        app.payments_ = make_stripe_gateway(app.config_.get(), app.logger_.get());

        return app;
    }

    static Application create_test() {
        Application app;
        app.config_ = make_in_memory_config();
        app.logger_ = make_null_logger();
        app.db_ = make_sqlite_db(":memory:");
        app.cache_ = make_no_cache();
        app.auth_ = make_auth_service(app.db_.get(), app.cache_.get());
        app.orders_ = make_order_service(app.db_.get(), app.logger_.get());
        app.payments_ = make_mock_gateway();
        return app;
    }

    void run() {
        // Start HTTP server, event loop, etc.
    }
};

int main(int argc, char* argv[]) {
    // ONE place where concrete types are chosen
    auto app = (argc > 1 && std::string(argv[1]) == "--test")
        ? Application::create_test()
        : Application::create_production();
    app.run();
}

// RULES:
// 1. ONLY the composition root knows concrete types
// 2. Business logic NEVER creates its own dependencies
// 3. Dependencies flow DOWN (high-level <- receives <- low-level)
// 4. All classes receive deps through constructors
```

Notice how `create_test()` and `create_production()` share the exact same shape - they differ only in which concrete implementations they plug in. All business-logic classes are completely untouched by this choice.

---

## Notes

- The **composition root** should be as close to `main()` as possible - it's the only place that knows concrete types.
- Constructor injection is the default; use setter injection only for optional dependencies.
- In embedded systems, DIP via templates gives zero overhead while keeping testability.
- Avoid "service locators" - they hide dependencies and make code harder to reason about.
- DIP is **the enabler** of testability. Without it, you can't mock or stub dependencies.
