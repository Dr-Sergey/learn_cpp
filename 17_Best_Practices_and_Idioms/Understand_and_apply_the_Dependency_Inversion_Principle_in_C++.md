# Understand and apply the Dependency Inversion Principle in C++

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Ri-abstract>  

---

## Topic Overview

**Dependency Inversion Principle (DIP):** High-level modules should not depend on low-level modules. Both should depend on abstractions. In plain terms: your `Service` should not reach down and directly construct a `FileLogger`. Instead, it should ask for "something that logs" - and the actual logger type gets decided at the call site, not inside the service.

The payoff is testability. When your high-level code depends on a concrete type, you cannot swap it out. When it depends on an abstraction, you can inject a mock in tests and a real implementation in production.

### Without DIP vs With DIP

```cpp
WITHOUT DIP:                       WITH DIP:
┌──────────┐                       ┌──────────┐
│ Service  │───depends on──>│      │ Service  │───depends on──>│ ILogger │
│          │                │      │          │                └────┬────┘
└──────────┘                │      └──────────┘                     │
                    ┌───────┴──┐                         ┌─────────┴─────────┐
                    │FileLogger│                         │FileLogger│MockLogger│
                    └──────────┘                         └──────────┴──────────┘
Hard to test!                      Easy to test with MockLogger
```

### Two Flavors in C++

C++ gives you two distinct ways to apply DIP, with different tradeoffs:

| Style | Mechanism | Overhead | When to use |
| --- | --- | --- | --- |
| Dynamic DIP | Virtual interfaces | vtable dispatch (~1ns) | Runtime plugin swap |
| Static DIP | Templates + concepts | Zero overhead | Compile-time selection |

---

## Self-Assessment

### Q1: Refactor a class that directly instantiates a Logger to accept a Logger interface

The bad version hard-codes `FileLogger` as a member. You cannot test `ServiceBad` without actually running the logger - and you cannot swap it for anything else without changing the class. The fix is to accept any `ILogger` by reference.

```cpp
#include <iostream>
#include <memory>
#include <string>

// BAD: Service directly depends on concrete FileLogger
class FileLogger {
public:
    void log(const std::string& msg) { std::cout << "[FILE] " << msg << '\n'; }
};

class ServiceBad {
    FileLogger logger_;  // hard-coded dependency!
public:
    void process(int data) {
        logger_.log("Processing " + std::to_string(data));
        std::cout << "Result: " << data * 2 << '\n';
    }
};
// Can't test without writing to a file!

// GOOD: Depend on an abstraction
class ILogger {
public:
    virtual void log(const std::string& msg) = 0;
    virtual ~ILogger() = default;
};

class ConsoleLogger : public ILogger {
public:
    void log(const std::string& msg) override { std::cout << "[CONSOLE] " << msg << '\n'; }
};

class MockLogger : public ILogger {
public:
    std::string last_message;
    int call_count = 0;
    void log(const std::string& msg) override {
        last_message = msg;
        ++call_count;
    }
};

class ServiceGood {
    ILogger& logger_;  // depends on abstraction!
public:
    ServiceGood(ILogger& logger) : logger_(logger) {}
    void process(int data) {
        logger_.log("Processing " + std::to_string(data));
        std::cout << "Result: " << data * 2 << '\n';
    }
};

int main() {
    // Production:
    ConsoleLogger console;
    ServiceGood service(console);
    service.process(21);

    // Testing:
    MockLogger mock;
    ServiceGood testable(mock);
    testable.process(42);
    std::cout << "Mock received: " << mock.last_message << '\n';
    std::cout << "Mock call count: " << mock.call_count << '\n';
}
// Expected output:
// [CONSOLE] Processing 21
// Result: 42
// Result: 84
// Mock received: Processing 42
// Mock call count: 1
```

Notice the testing section - `MockLogger` captures the messages instead of printing them, so you can assert on them in a test without any output or filesystem side effects. That is the direct benefit of DIP.

### Q2: Show how templates (static DIP) differ from virtual interfaces (dynamic DIP) in overhead

The virtual dispatch path through `ILoggerDynamic` adds an indirect call - the CPU has to look up the function pointer in the vtable at runtime. With the template version, the call is resolved at compile time and the compiler can inline it entirely, leaving essentially nothing in the generated code.

```cpp
#include <chrono>
#include <iostream>
#include <string>

// Dynamic DIP: virtual dispatch
class ILoggerDynamic {
public:
    virtual void log(const std::string& msg) = 0;
    virtual ~ILoggerDynamic() = default;
};

class NullLoggerDynamic : public ILoggerDynamic {
public:
    void log(const std::string&) override {}  // intentionally empty
};

class DynamicService {
    ILoggerDynamic& logger_;
public:
    DynamicService(ILoggerDynamic& l) : logger_(l) {}
    void work() { logger_.log("work"); }
};

// Static DIP: templates + concepts (zero overhead)
template<typename T>
concept Logger = requires(T t, const std::string& s) {
    { t.log(s) };
};

class NullLoggerStatic {
public:
    void log(const std::string&) {}  // no virtual, inlinable
};

template<Logger L>
class StaticService {
    L& logger_;
public:
    StaticService(L& l) : logger_(l) {}
    void work() { logger_.log("work"); }
};

int main() {
    constexpr int N = 10'000'000;

    NullLoggerDynamic dyn_logger;
    DynamicService dyn_svc(dyn_logger);

    NullLoggerStatic sta_logger;
    StaticService sta_svc(sta_logger);

    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) dyn_svc.work();
    auto t2 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) sta_svc.work();
    auto t3 = std::chrono::high_resolution_clock::now();

    auto dyn_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto sta_ms = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::cout << "Dynamic DIP: " << dyn_ms << " ms\n";
    std::cout << "Static  DIP: " << sta_ms << " ms\n";
    std::cout << "Speedup: " << (dyn_ms / std::max(sta_ms, 0.001)) << "x\n";
}
// Typical output (-O2):
// Dynamic DIP: ~50 ms  (vtable lookup prevents full inlining)
// Static  DIP: ~0 ms   (call fully inlined away by compiler)
// Speedup: huge
```

The static version's call is completely optimized away because the compiler knows at compile time exactly which `log` will be called. Choose static DIP for performance-critical inner loops; use dynamic DIP when you need to swap implementations at runtime (plugins, configuration-driven behaviour).

### Q3: Inject dependencies via constructor parameters and verify testability improvement

You do not always need a full virtual interface. For lightweight components, `std::function` provides a clean DIP mechanism without the ceremony of defining an abstract base class.

```cpp
#include <functional>
#include <iostream>
#include <string>
#include <vector>

// Lightweight DIP with std::function (no virtual class needed)
class OrderProcessor {
public:
    using PaymentFn = std::function<bool(double amount)>;
    using NotifyFn = std::function<void(const std::string& msg)>;

private:
    PaymentFn charge_;
    NotifyFn notify_;

public:
    OrderProcessor(PaymentFn charge, NotifyFn notify)
        : charge_(std::move(charge)), notify_(std::move(notify)) {}

    bool place_order(const std::string& item, double price) {
        if (charge_(price)) {
            notify_("Order placed: " + item + " ($" + std::to_string(price) + ")");
            return true;
        }
        notify_("Payment failed for " + item);
        return false;
    }
};

int main() {
    // Production dependencies:
    auto real_payment = [](double amount) {
        std::cout << "Charging $" << amount << "...\n";
        return true;
    };
    auto real_notify = [](const std::string& msg) {
        std::cout << "[EMAIL] " << msg << '\n';
    };

    OrderProcessor prod(real_payment, real_notify);
    prod.place_order("Widget", 29.99);

    // Test dependencies:
    std::vector<std::string> notifications;
    bool payment_result = false;  // simulate failure

    OrderProcessor test(
        [&](double) { return payment_result; },
        [&](const std::string& msg) { notifications.push_back(msg); }
    );

    test.place_order("Gadget", 9.99);
    std::cout << "Test notification: " << notifications.back() << '\n';
    std::cout << "Result verified: payment failed as expected\n";
}
// Expected output:
// Charging $29.99...
// [EMAIL] Order placed: Widget ($29.990000)
// Test notification: Payment failed for Gadget
// Result verified: payment failed as expected
```

The test version injects lambdas that capture local variables - the payment always fails, and notifications go into a vector you can inspect. No mock framework, no interface hierarchy needed.

---

## Notes

- DIP is the "D" in SOLID. It's the foundation of testable, modular C++ code.
- Prefer static DIP (templates/concepts) in performance-critical paths.
- Use dynamic DIP (virtual interfaces) when plugins or runtime selection is needed.
- `std::function` is a lightweight DIP mechanism for small components.
- Constructor injection is the cleanest form: all dependencies visible at construction.
