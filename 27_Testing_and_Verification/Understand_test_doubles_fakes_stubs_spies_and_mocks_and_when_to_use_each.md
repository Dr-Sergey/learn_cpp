# Understand test doubles: fakes, stubs, spies, and mocks and when to use each

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://martinfowler.com/bliki/TestDouble.html>  

---

## Topic Overview

A **test double** is any object that stands in for a real dependency during testing. The reason you need them is simple: real dependencies (databases, network clients, email servers, file systems) are slow, unreliable, or impossible to control in a test. A test double replaces the dependency with something predictable.

The four main kinds differ in complexity and what they let you verify. The reason this trips people up is that "mock" is often used informally to mean any test double, but technically a mock is a specific kind with strict expectation checking. Using the right double for the job keeps your tests simple and avoids brittleness.

```cpp
                    +---------------------------------------------+
                    |           Test Doubles Spectrum              |
                    +---------+----------+----------+-------------+
                    |  Dummy  |  Stub    |  Spy     |  Mock       |
                    | (unused)| (canned) | (records)| (expects)   |
                    |         |          |          |             |
Complexity:         |  lowest |   low    |  medium  |  highest    |
Behavior verified:  |  none   |  state   |  state   |  interaction|
                    +---------+----------+----------+-------------+
                                    |
                               +----+
                               |
                    +----------------------+
                    |  Fake                |
                    | (working impl)       |
                    |  medium complexity   |
                    |  state verification  |
                    +----------------------+
```

| Double | Has Logic? | Records Calls? | Asserts Expectations? | Example |
| --- | :---: | :---: | :---: | --- |
| **Dummy** | No | No | No | Null object passed to satisfy API |
| **Stub** | No (canned) | No | No | Always returns `{"ok": true}` |
| **Spy** | Maybe | Yes | No (you check after) | Records every method call |
| **Mock** | Maybe | Yes | Yes (auto-fails) | `EXPECT_CALL(...).Times(1)` |
| **Fake** | Yes (simplified) | No | No | In-memory database |

---

## Self-Assessment

### Q1: Implement a Fake (working in-memory implementation) for a database interface

**Answer:**

A fake has real logic - it just uses a simplified backing store. The classic example is an in-memory database that behaves like the real thing (insert, find, remove all work correctly) but uses a `std::unordered_map` instead of disk. This lets you write tests that exercise realistic sequences of operations without touching a real database.

The key design point: the fake implements the same interface as the real dependency, so you can swap it in via dependency injection without changing the production code at all.

```cpp
#include <string>
#include <unordered_map>
#include <optional>
#include <vector>
#include <stdexcept>
#include <cassert>
#include <iostream>

// Interface: what production code depends on
class IUserRepository {
public:
    virtual ~IUserRepository() = default;
    virtual void save(const std::string& id, const std::string& name) = 0;
    virtual std::optional<std::string> find(const std::string& id) const = 0;
    virtual void remove(const std::string& id) = 0;
    virtual std::vector<std::string> all_ids() const = 0;
};

// FAKE: working in-memory implementation
class FakeUserRepository : public IUserRepository {
    std::unordered_map<std::string, std::string> store_;
public:
    void save(const std::string& id, const std::string& name) override {
        store_[id] = name;  // Real insert/update logic
    }

    std::optional<std::string> find(const std::string& id) const override {
        auto it = store_.find(id);
        if (it == store_.end()) return std::nullopt;
        return it->second;  // Real lookup logic
    }

    void remove(const std::string& id) override {
        store_.erase(id);  // Real removal logic
    }

    std::vector<std::string> all_ids() const override {
        std::vector<std::string> ids;
        ids.reserve(store_.size());
        for (auto& [id, _] : store_) ids.push_back(id);
        return ids;
    }

    // Test helper - not in interface
    size_t size() const { return store_.size(); }
};

// Production code under test
class UserService {
    IUserRepository& repo_;
public:
    explicit UserService(IUserRepository& repo) : repo_(repo) {}

    void register_user(const std::string& id, const std::string& name) {
        if (repo_.find(id).has_value())
            throw std::runtime_error("user already exists");
        repo_.save(id, name);
    }

    std::string get_user_name(const std::string& id) {
        auto name = repo_.find(id);
        if (!name) throw std::runtime_error("user not found");
        return *name;
    }
};

// Tests using the Fake
int main() {
    FakeUserRepository fake_repo;
    UserService service(fake_repo);

    // Test: register and retrieve
    service.register_user("u1", "Alice");
    assert(service.get_user_name("u1") == "Alice");

    // Test: duplicate registration throws
    try {
        service.register_user("u1", "Bob");
        assert(false && "should have thrown");
    } catch (const std::runtime_error& e) {
        assert(std::string(e.what()) == "user already exists");
    }

    // Test: find non-existent user throws
    try {
        service.get_user_name("u999");
        assert(false);
    } catch (const std::runtime_error&) {}

    // Fake has full behavior - we can inspect state
    assert(fake_repo.size() == 1);
    assert(fake_repo.all_ids().size() == 1);

    std::cout << "All Fake tests passed!\n";
}
```

Notice the `size()` helper method at the bottom of `FakeUserRepository`. It's not on the `IUserRepository` interface, but it's fine to add test-helper methods to the fake itself. Since the tests have the concrete `FakeUserRepository` type, they can call methods the interface doesn't expose.

### Q2: Implement a Stub (returns canned values) for a network client in unit tests

**Answer:**

A stub is simpler than a fake - it doesn't have real logic, it just returns pre-configured responses. The stub below lets you configure what the GET and POST responses will be before the test runs, and then the production code under test exercises its own logic against those fixed responses. The stub itself doesn't care what URL you pass; it always returns the same thing.

This is the right tool when you only care about how the production code reacts to different responses, not about the HTTP logic itself.

```cpp
#include <string>
#include <vector>
#include <cassert>
#include <iostream>

// Interface
struct HttpResponse {
    int status_code;
    std::string body;
};

class IHttpClient {
public:
    virtual ~IHttpClient() = default;
    virtual HttpResponse get(const std::string& url) = 0;
    virtual HttpResponse post(const std::string& url, const std::string& body) = 0;
};

// STUB: returns pre-configured responses
class StubHttpClient : public IHttpClient {
    HttpResponse get_response_{200, R"({"status":"ok"})"};
    HttpResponse post_response_{201, R"({"id":42})"};
public:
    // Configure canned responses
    void set_get_response(int code, std::string body) {
        get_response_ = {code, std::move(body)};
    }
    void set_post_response(int code, std::string body) {
        post_response_ = {code, std::move(body)};
    }

    HttpResponse get(const std::string& /*url*/) override {
        return get_response_;  // Always the same, regardless of URL
    }
    HttpResponse post(const std::string& /*url*/,
                       const std::string& /*body*/) override {
        return post_response_;
    }
};

// Production code
class ApiClient {
    IHttpClient& http_;
public:
    explicit ApiClient(IHttpClient& http) : http_(http) {}

    bool health_check() {
        auto resp = http_.get("/health");
        return resp.status_code == 200;
    }

    int create_resource(const std::string& data) {
        auto resp = http_.post("/resources", data);
        if (resp.status_code != 201)
            throw std::runtime_error("creation failed");
        // Parse id from response (simplified)
        return 42;
    }
};

// Tests
int main() {
    StubHttpClient stub;
    ApiClient client(stub);

    // Test: health check succeeds
    assert(client.health_check() == true);

    // Test: health check fails with 503
    stub.set_get_response(503, "Service Unavailable");
    assert(client.health_check() == false);

    // Test: create resource
    stub.set_post_response(201, R"({"id":42})");
    assert(client.create_resource("test") == 42);

    // Test: create resource fails
    stub.set_post_response(500, "Internal Error");
    try {
        client.create_resource("test");
        assert(false);
    } catch (const std::runtime_error&) {}

    std::cout << "All Stub tests passed!\n";
}
```

The stub approach makes it easy to test error handling paths that would be hard to trigger with a real HTTP client. "What happens when the server returns 503?" is a one-liner with a stub: `stub.set_get_response(503, ...)`.

### Q3: Distinguish a Spy (records calls) from a Mock (also verifies expectations)

**Answer:**

The spy vs mock distinction is the one that confuses people most. Both record what happened, but they differ in *when* you assert and *who* decides what counts as a failure.

A **spy** is passive: it records every call, and you inspect the recording after the test with your own `assert` statements. You decide what matters.

A **mock** is active: you declare expectations *before* the test runs, and the mock automatically fails if those expectations aren't met. The mock enforces the contract; you don't need to write the assertions yourself.

```cpp
#include <string>
#include <vector>
#include <cassert>
#include <iostream>
#include <functional>

// Interface
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& level, const std::string& msg) = 0;
};

// SPY: records all calls for later inspection
class SpyLogger : public ILogger {
public:
    struct Call {
        std::string level;
        std::string message;
    };
    std::vector<Call> calls;

    void log(const std::string& level, const std::string& msg) override {
        calls.push_back({level, msg});  // Just record - no assertions
    }

    // Inspection helpers
    size_t call_count() const { return calls.size(); }
    bool was_called_with(const std::string& level) const {
        for (auto& c : calls)
            if (c.level == level) return true;
        return false;
    }
};

// MOCK: sets expectations and auto-fails
class MockLogger : public ILogger {
    struct Expectation {
        std::string level;
        std::string message_contains;
        int expected_times;
        int actual_times = 0;
    };
    std::vector<Expectation> expectations_;
    bool verified_ = false;

public:
    // Set expectation BEFORE the test runs
    void expect_call(const std::string& level,
                     const std::string& msg_contains,
                     int times = 1) {
        expectations_.push_back({level, msg_contains, times, 0});
    }

    void log(const std::string& level, const std::string& msg) override {
        for (auto& exp : expectations_) {
            if (exp.level == level &&
                msg.find(exp.message_contains) != std::string::npos) {
                exp.actual_times++;
                return;
            }
        }
        // Unexpected call - mock would fail here
        throw std::runtime_error("Unexpected log call: " + level + ": " + msg);
    }

    // Call at end of test - verifies all expectations met
    void verify() {
        for (auto& exp : expectations_) {
            if (exp.actual_times != exp.expected_times) {
                throw std::runtime_error(
                    "Expected " + std::to_string(exp.expected_times) +
                    " calls with level=" + exp.level +
                    " but got " + std::to_string(exp.actual_times));
            }
        }
        verified_ = true;
    }

    ~MockLogger() {
        if (!verified_)
            std::cerr << "Warning: MockLogger destroyed without verify()\n";
    }
};

// Code under test
class PaymentProcessor {
    ILogger& logger_;
public:
    explicit PaymentProcessor(ILogger& logger) : logger_(logger) {}

    bool process(double amount) {
        logger_.log("INFO", "Processing payment: $" + std::to_string(amount));
        if (amount <= 0) {
            logger_.log("ERROR", "Invalid amount");
            return false;
        }
        logger_.log("INFO", "Payment successful");
        return true;
    }
};

int main() {
    // SPY usage: inspect AFTER the test
    {
        SpyLogger spy;
        PaymentProcessor proc(spy);
        proc.process(50.0);

        // YOU decide what to check after the fact
        assert(spy.call_count() == 2);
        assert(spy.was_called_with("INFO"));
        assert(!spy.was_called_with("ERROR"));
        // Spy is passive - it never fails on its own
    }

    // MOCK usage: set expectations BEFORE
    {
        MockLogger mock;
        // Expectations set up front
        mock.expect_call("INFO", "Processing payment", 1);
        mock.expect_call("INFO", "Payment successful", 1);

        PaymentProcessor proc(mock);
        proc.process(50.0);

        mock.verify();  // Fails if expectations not met
    }

    // MOCK catches unexpected calls
    {
        MockLogger mock;
        mock.expect_call("INFO", "Processing payment", 1);
        mock.expect_call("ERROR", "Invalid amount", 1);

        PaymentProcessor proc(mock);
        proc.process(-10.0);  // Should log ERROR

        mock.verify();
    }

    std::cout << "All Spy vs Mock tests passed!\n";
}

// Summary:
// Spy:  Record calls -> inspect later -> YOU write assert()s
// Mock: Set expectations -> run code -> verify() auto-checks
// Use Spy when you want flexible, after-the-fact assertions.
// Use Mock when you want strict interaction verification.
```

The practical guidance: use spies more often than mocks. Mocks are powerful but they encode interaction details into the test, which makes refactoring harder. If you rename a method or reorder two calls that don't affect behavior, a spy-based test still passes - a mock-based test may not. Reach for mocks when you specifically need to verify that a dependency was called a precise number of times, or when an unexpected call should be a hard failure.

---

## Notes

- In real projects, use **Google Mock** or **trompeloeil** instead of hand-rolling mocks
- **Fakes** are best for integration-like tests where you need realistic behavior (in-memory DB, fake filesystem)
- **Stubs** are the simplest - use them when you just need a dependency to "not crash"
- **Mocks** can make tests brittle if they over-specify (testing implementation, not behavior)
- Rule of thumb: prefer **state verification** (stubs/fakes + assert result) over **interaction verification** (mocks + verify calls)
