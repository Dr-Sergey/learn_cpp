# Use Google Mock for expressive test doubles in C++ unit tests

**Category:** Testing and Verification  
**Standard:** C++23  
**Reference:** <https://google.github.io/googletest/gmock_cook_book.html>  

---

## Topic Overview

**Google Mock** (gMock) gives you a set of macros for creating mock objects from interfaces, wiring up expectations on method calls, and controlling what those methods return - all tightly integrated with Google Test. If you have ever written a manual stub class that returns hard-coded values and had to update it every time the interface changed, gMock is what you have been missing.

### Key Macros

The table below is your go-to reference while writing tests. The most important distinction to keep in mind is `EXPECT_CALL` versus `ON_CALL` - the first is a strict expectation that fails your test if not satisfied, while the second is a default behavior that won't cause failures on its own.

| Macro | Purpose |
| --- | --- |
| `MOCK_METHOD(RetType, Name, (Args), (Qualifiers))` | Declare a mock method |
| `EXPECT_CALL(obj, Method(matchers)).Times(n)` | Set strict expectation |
| `ON_CALL(obj, Method(matchers)).WillByDefault(action)` | Set default behavior (no failure) |
| `EXPECT_CALL(...).WillOnce(Return(x))` | Return `x` once |
| `EXPECT_CALL(...).WillRepeatedly(action)` | Return every time |
| `testing::_` | Match any argument |
| `testing::Gt(n)` | Match if > n |
| `testing::HasSubstr("x")` | Match string containing "x" |

### MOCK_METHOD Syntax (C++17+)

The old numbered macros (`MOCK_METHOD1`, `MOCK_METHOD2`, ...) are deprecated. The new four-argument form handles everything cleanly. Here is the before and after so you can recognize both in existing code:

```cpp
// Old style (deprecated):
MOCK_METHOD1(foo, int(double));

// New style:
MOCK_METHOD(int, foo, (double), (override));
// MOCK_METHOD(return_type, name, (arg_types...), (qualifiers...))
```

The qualifiers slot is where you put `override`, `const`, or both - for example `(const, override)` for a const method.

---

## Self-Assessment

### Q1: Write a mock Logger with MOCK_METHOD and set expectations using EXPECT_CALL

The idea here is to take an interface (`ILogger`), create a mock implementation using `MOCK_METHOD`, then inject that mock into `OrderProcessor` and use `EXPECT_CALL` to assert that the processor sends exactly the right log messages in each scenario. Notice how each test focuses on one specific behavior of the system.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>
#include <memory>

// Interface
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& level, const std::string& msg) = 0;
    virtual void flush() = 0;
    virtual int pending_count() const = 0;
};

// Mock
class MockLogger : public ILogger {
public:
    MOCK_METHOD(void, log, (const std::string&, const std::string&), (override));
    MOCK_METHOD(void, flush, (), (override));
    MOCK_METHOD(int, pending_count, (), (const, override));
};

// Code under test
class OrderProcessor {
    ILogger& logger_;
public:
    explicit OrderProcessor(ILogger& logger) : logger_(logger) {}

    bool process(int order_id, double amount) {
        logger_.log("INFO", "Processing order " + std::to_string(order_id));

        if (amount <= 0) {
            logger_.log("ERROR", "Invalid amount");
            return false;
        }

        if (amount > 10000) {
            logger_.log("WARN", "Large order requires review");
        }

        logger_.log("INFO", "Order completed");
        logger_.flush();
        return true;
    }
};

// Tests
using ::testing::_;
using ::testing::HasSubstr;

TEST(OrderProcessor, NormalOrderLogsCorrectly) {
    MockLogger mock;

    // Set expectations in ORDER (gMock verifies reverse order by default)
    EXPECT_CALL(mock, log("INFO", HasSubstr("Processing")))
        .Times(1);
    EXPECT_CALL(mock, log("INFO", "Order completed"))
        .Times(1);
    EXPECT_CALL(mock, flush())
        .Times(1);

    OrderProcessor proc(mock);
    EXPECT_TRUE(proc.process(42, 99.99));
    // MockLogger auto-verifies expectations in destructor
}

TEST(OrderProcessor, InvalidAmountLogsError) {
    MockLogger mock;

    EXPECT_CALL(mock, log("INFO", _)).Times(1);   // Processing message
    EXPECT_CALL(mock, log("ERROR", "Invalid amount")).Times(1);
    EXPECT_CALL(mock, flush()).Times(0);  // Should NOT flush on error

    OrderProcessor proc(mock);
    EXPECT_FALSE(proc.process(1, -5.0));
}

TEST(OrderProcessor, LargeOrderLogsWarning) {
    MockLogger mock;

    EXPECT_CALL(mock, log("INFO", _)).Times(2);
    EXPECT_CALL(mock, log("WARN", HasSubstr("Large order"))).Times(1);
    EXPECT_CALL(mock, flush()).Times(1);

    OrderProcessor proc(mock);
    EXPECT_TRUE(proc.process(99, 50000.0));
}
```

One thing to notice: the `MockLogger` destructor automatically verifies that all `EXPECT_CALL` expectations were actually fulfilled. You do not need to call any explicit `verify()` method - the test fails the moment `mock` goes out of scope if something was not called as expected.

### Q2: Use ON_CALL to provide default behaviour for a mock without failing on unexpected calls

Sometimes a test cares deeply about one method and needs other methods to just work without getting in the way. `ON_CALL` is how you set up that permissive default. The key difference from `EXPECT_CALL` is that `ON_CALL` sets a return value but will not fail if the method is never called, called zero times, or called many times.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

// Interface
class ICache {
public:
    virtual ~ICache() = default;
    virtual bool has(const std::string& key) const = 0;
    virtual std::string get(const std::string& key) const = 0;
    virtual void put(const std::string& key, const std::string& val) = 0;
    virtual size_t size() const = 0;
};

class MockCache : public ICache {
public:
    MOCK_METHOD(bool, has, (const std::string&), (const, override));
    MOCK_METHOD(std::string, get, (const std::string&), (const, override));
    MOCK_METHOD(void, put, (const std::string&, const std::string&), (override));
    MOCK_METHOD(size_t, size, (), (const, override));
};

// Code under test
class DataService {
    ICache& cache_;
public:
    explicit DataService(ICache& c) : cache_(c) {}

    std::string fetch(const std::string& key) {
        if (cache_.has(key))
            return cache_.get(key);
        // Simulate fetching from DB
        std::string value = "db_value_for_" + key;
        cache_.put(key, value);
        return value;
    }
};

using ::testing::Return;
using ::testing::_;

TEST(DataService, FetchFromCacheWhenPresent) {
    MockCache mock;

    // ON_CALL: default behaviors - won't FAIL on unexpected calls
    ON_CALL(mock, has(_)).WillByDefault(Return(true));
    ON_CALL(mock, get(_)).WillByDefault(Return("cached_value"));
    ON_CALL(mock, size()).WillByDefault(Return(5u));

    // Only EXPECT what we specifically care about:
    EXPECT_CALL(mock, has("user:42")).WillOnce(Return(true));
    EXPECT_CALL(mock, get("user:42")).WillOnce(Return("Alice"));
    // put() is NOT expected - but won't fail if called thanks to ON_CALL defaults

    DataService service(mock);
    EXPECT_EQ(service.fetch("user:42"), "Alice");
}

TEST(DataService, FetchFromDbWhenMissing) {
    MockCache mock;

    // Default: cache always misses
    ON_CALL(mock, has(_)).WillByDefault(Return(false));
    ON_CALL(mock, get(_)).WillByDefault(Return(""));

    // We care specifically about the put() call
    EXPECT_CALL(mock, has("user:99")).WillOnce(Return(false));
    EXPECT_CALL(mock, put("user:99", "db_value_for_user:99")).Times(1);

    DataService service(mock);
    EXPECT_EQ(service.fetch("user:99"), "db_value_for_user:99");
}

// Key difference:
// EXPECT_CALL -> test FAILS if not called (or called wrong)
// ON_CALL     -> sets default, no failure if not called
// Use ON_CALL for "I don't care" methods to avoid brittle tests
```

The last comment in the code really is the mental model to keep. Use `EXPECT_CALL` to assert behavior your test owns. Use `ON_CALL` to keep the other methods from getting in the way.

### Q3: Distinguish mocks, stubs, fakes, and spies and show a C++ example of each

This is one of the most commonly confused areas in testing. All four are "test doubles" - stand-ins for real dependencies - but they serve different purposes. Here is a concrete example of each, all sharing the same `INotifier` interface so you can compare them side by side:

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>
#include <vector>
#include <unordered_map>

// Shared interface
class INotifier {
public:
    virtual ~INotifier() = default;
    virtual bool send(const std::string& to, const std::string& msg) = 0;
};

// 1. STUB: Returns canned values, no logic
class StubNotifier : public INotifier {
    bool return_value_ = true;
public:
    void set_return(bool v) { return_value_ = v; }
    bool send(const std::string&, const std::string&) override {
        return return_value_;  // Always returns pre-set value
    }
};

// 2. FAKE: Working simplified implementation
class FakeNotifier : public INotifier {
public:
    struct Message { std::string to; std::string body; };
    std::vector<Message> inbox;  // In-memory "sent" mailbox

    bool send(const std::string& to, const std::string& msg) override {
        if (to.empty()) return false;  // Real validation!
        inbox.push_back({to, msg});
        return true;
    }
};

// 3. SPY: Records calls for later inspection
class SpyNotifier : public INotifier {
public:
    struct Call { std::string to; std::string msg; };
    std::vector<Call> calls;

    bool send(const std::string& to, const std::string& msg) override {
        calls.push_back({to, msg});
        return true;
    }
};

// 4. MOCK: Sets expectations, auto-verifies
class MockNotifier : public INotifier {
public:
    MOCK_METHOD(bool, send, (const std::string&, const std::string&), (override));
};

// Tests showing each
using ::testing::Return;

TEST(TestDoubles, StubExample) {
    StubNotifier stub;
    stub.set_return(false);  // Simulate failure
    EXPECT_FALSE(stub.send("alice", "hi"));
    // Stub: no logic, just returns what you tell it
}

TEST(TestDoubles, FakeExample) {
    FakeNotifier fake;
    EXPECT_TRUE(fake.send("alice", "hello"));
    EXPECT_FALSE(fake.send("", "no recipient"));  // Fake has real validation
    ASSERT_EQ(fake.inbox.size(), 1u);
    EXPECT_EQ(fake.inbox[0].to, "alice");
    // Fake: working implementation, test actual behavior
}

TEST(TestDoubles, SpyExample) {
    SpyNotifier spy;
    spy.send("bob", "msg1");
    spy.send("carol", "msg2");
    // Spy: check what was called AFTER the fact
    ASSERT_EQ(spy.calls.size(), 2u);
    EXPECT_EQ(spy.calls[0].to, "bob");
    EXPECT_EQ(spy.calls[1].msg, "msg2");
}

TEST(TestDoubles, MockExample) {
    MockNotifier mock;
    // Mock: expectations set BEFORE the test
    EXPECT_CALL(mock, send("dave", "urgent"))
        .WillOnce(Return(true));
    // If send() is NOT called with these args, test FAILS
    EXPECT_TRUE(mock.send("dave", "urgent"));
}
```

The key mental distinction is about *when* and *how* you verify behavior. A stub just returns values - you never assert anything about it. A fake is a simplified but real implementation - you test the outputs. A spy lets you record calls and check them after the fact. A mock sets up assertions *before* the code runs and verifies them automatically when it goes out of scope.

---

## Notes

- Link with: `target_link_libraries(tests GTest::gmock_main)`
- `NiceMock<T>` suppresses "uninteresting call" warnings for unmatched `EXPECT_CALL`s
- `StrictMock<T>` fails on ANY call without a matching `EXPECT_CALL`
- `InSequence` enforces call ordering: `InSequence seq; EXPECT_CALL(m, A()); EXPECT_CALL(m, B());`
- Prefer `ON_CALL` + focused `EXPECT_CALL` over setting expectations for every method
