# Test error handling paths - exceptions, error codes, and edge cases

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Error handling code is the most under-tested part of most C++ projects, yet it runs in exactly the situations that matter most - when something goes wrong in production. It's tempting to focus your tests on the happy path because that's where your feature lives, but your error paths need the same rigor. Bugs in error handling code lead to crashes, data corruption, or resource leaks at the worst possible moments.

Testing error paths means exercising exception throw and catch, `std::error_code` propagation, `std::expected`/`std::optional` failure cases, boundary conditions, and resource cleanup under failure. Each of these mechanisms has different failure modes and requires different testing approaches.

### Error Handling Mechanisms to Test

| Mechanism | How to Trigger in Tests | What to Verify |
| --- | --- | --- |
| **Exceptions** | Mock dependency throws | Correct type, message, no resource leak |
| **Error codes** | Return error from mock | Caller propagates/handles correctly |
| **`std::optional`** | Return `nullopt` | Caller handles empty case |
| **`std::expected`** (C++23) | Return `unexpected(err)` | Error value accessible |
| **errno-style** | Set global state | Caller checks and resets |

### Edge Cases to Always Test

Here's a quick mental checklist of the edge cases that catch most real-world bugs. If you test all of these for every non-trivial function, you'll catch the overwhelming majority of latent errors:

```cpp
┌─────────────────────────────────────────────┐
│ Boundary values:  0, -1, MAX, MIN, empty    │
│ Null/empty:       nullptr, "", {}, nullopt   │
│ Resource failure: allocation, I/O, timeout   │
│ Concurrency:      race condition, deadlock   │
│ Overflow:         integer, buffer, stack     │
│ Invalid state:    double-free, use-after-move│
└─────────────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: Test exception handling paths thoroughly

The trick with exception testing is to combine Google Mock's `Throw` action with `EXPECT_CALL`. This lets you make a specific mock method throw a specific exception type on demand, without any special test infrastructure. Notice how the test verifies not just the return value but also that the logger was called with the right message - that's the part most developers forget to check.

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include <stdexcept>
#include <string>
#include <memory>

using ::testing::_;
using ::testing::Throw;
using ::testing::Return;

// === Interfaces ===
class IPaymentGateway {
public:
    virtual ~IPaymentGateway() = default;
    virtual std::string charge(double amount) = 0;  // Returns transaction ID
};

class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void error(const std::string& msg) = 0;
};

// === Production code ===
class PaymentService {
public:
    PaymentService(IPaymentGateway& gateway, ILogger& logger)
        : gateway_(gateway), logger_(logger) {}

    struct Result {
        bool success;
        std::string transaction_id;
        std::string error_message;
    };

    Result process_payment(double amount) {
        if (amount <= 0)
            return {false, "", "Invalid amount"};

        try {
            auto txn_id = gateway_.charge(amount);
            return {true, txn_id, ""};
        } catch (const std::runtime_error& e) {
            logger_.error(std::string("Payment failed: ") + e.what());
            return {false, "", e.what()};
        } catch (const std::exception& e) {
            logger_.error(std::string("Unexpected error: ") + e.what());
            return {false, "", "Internal error"};
        }
    }

private:
    IPaymentGateway& gateway_;
    ILogger& logger_;
};

// === Mocks ===
class MockGateway : public IPaymentGateway {
public:
    MOCK_METHOD(std::string, charge, (double), (override));
};

class MockLogger : public ILogger {
public:
    MOCK_METHOD(void, error, (const std::string&), (override));
};

// === Tests ===

TEST(PaymentServiceTest, SuccessfulPayment) {
    MockGateway gateway;
    MockLogger logger;
    EXPECT_CALL(gateway, charge(42.0)).WillOnce(Return("txn_123"));
    EXPECT_CALL(logger, error(_)).Times(0);  // No errors expected

    PaymentService svc(gateway, logger);
    auto result = svc.process_payment(42.0);

    EXPECT_TRUE(result.success);
    EXPECT_EQ(result.transaction_id, "txn_123");
    EXPECT_TRUE(result.error_message.empty());
}

TEST(PaymentServiceTest, GatewayThrowsRuntimeError) {
    MockGateway gateway;
    MockLogger logger;

    EXPECT_CALL(gateway, charge(_))
        .WillOnce(Throw(std::runtime_error("Connection timeout")));
    EXPECT_CALL(logger, error(::testing::HasSubstr("Connection timeout")));

    PaymentService svc(gateway, logger);
    auto result = svc.process_payment(42.0);

    EXPECT_FALSE(result.success);
    EXPECT_EQ(result.error_message, "Connection timeout");
    EXPECT_TRUE(result.transaction_id.empty());
}

TEST(PaymentServiceTest, GatewayThrowsUnexpectedException) {
    MockGateway gateway;
    MockLogger logger;

    EXPECT_CALL(gateway, charge(_))
        .WillOnce(Throw(std::bad_alloc()));  // Unexpected type
    EXPECT_CALL(logger, error(::testing::HasSubstr("Unexpected error")));

    PaymentService svc(gateway, logger);
    auto result = svc.process_payment(42.0);

    EXPECT_FALSE(result.success);
    EXPECT_EQ(result.error_message, "Internal error");
}

TEST(PaymentServiceTest, InvalidAmountZero) {
    MockGateway gateway;
    MockLogger logger;
    EXPECT_CALL(gateway, charge(_)).Times(0);  // Should never reach gateway

    PaymentService svc(gateway, logger);
    auto result = svc.process_payment(0.0);

    EXPECT_FALSE(result.success);
    EXPECT_EQ(result.error_message, "Invalid amount");
}

TEST(PaymentServiceTest, InvalidAmountNegative) {
    MockGateway gateway;
    MockLogger logger;
    EXPECT_CALL(gateway, charge(_)).Times(0);

    PaymentService svc(gateway, logger);
    auto result = svc.process_payment(-10.0);

    EXPECT_FALSE(result.success);
}
```

The `GatewayThrowsUnexpectedException` test is the one people usually forget. You verified the `runtime_error` path, but what about `std::bad_alloc` or some other unexpected exception type? Those hit the second catch clause and produce a different error message. Both paths need coverage.

### Q2: Test error code and std::expected error paths

`std::error_code` is the C++ standard's type-safe error reporting mechanism for operations that can fail without throwing. The reason this trips people up is that `error_code` comparisons use `operator==` between two `error_code` objects from the same category - comparing against a plain integer won't work. The test below also shows the boundary value discipline: test both endpoints of the valid range (1 and 65535) plus both just-outside values (0 and 65536).

```cpp
#include <gtest/gtest.h>
#include <system_error>
#include <string>
#include <variant>

// === Error code approach ===

enum class FileError {
    NotFound = 1,
    PermissionDenied,
    DiskFull,
    IoError
};

class FileErrorCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "FileError"; }
    std::string message(int ev) const override {
        switch (static_cast<FileError>(ev)) {
            case FileError::NotFound: return "File not found";
            case FileError::PermissionDenied: return "Permission denied";
            case FileError::DiskFull: return "Disk full";
            case FileError::IoError: return "I/O error";
            default: return "Unknown";
        }
    }
};

std::error_code make_error_code(FileError e) {
    static FileErrorCategory cat;
    return {static_cast<int>(e), cat};
}

// Function returning error_code
std::error_code save_file(const std::string& path, const std::string& content) {
    if (path.empty()) return make_error_code(FileError::NotFound);
    if (content.size() > 1024 * 1024) return make_error_code(FileError::DiskFull);
    return {};  // Success
}

TEST(ErrorCodeTest, EmptyPathReturnsNotFound) {
    auto ec = save_file("", "data");
    EXPECT_TRUE(ec.operator bool());  // Has error
    EXPECT_EQ(ec, make_error_code(FileError::NotFound));
    EXPECT_EQ(ec.message(), "File not found");
}

TEST(ErrorCodeTest, LargeContentReturnsDiskFull) {
    std::string large(2 * 1024 * 1024, 'x');
    auto ec = save_file("/tmp/test", large);
    EXPECT_EQ(ec, make_error_code(FileError::DiskFull));
}

TEST(ErrorCodeTest, ValidInputReturnsSuccess) {
    auto ec = save_file("/tmp/test", "data");
    EXPECT_FALSE(ec.operator bool());  // No error
}

// === std::expected-style (C++23 or tl::expected) ===

template<typename T, typename E>
using Expected = std::variant<T, E>;  // Simplified pre-C++23

template<typename T, typename E>
bool has_value(const Expected<T, E>& exp) {
    return std::holds_alternative<T>(exp);
}

Expected<int, std::string> parse_port(const std::string& s) {
    try {
        int port = std::stoi(s);
        if (port < 1 || port > 65535)
            return std::string{"Port out of range: " + s};
        return port;
    } catch (...) {
        return std::string{"Invalid port: " + s};
    }
}

TEST(ExpectedTest, ValidPort) {
    auto result = parse_port("8080");
    ASSERT_TRUE(has_value<int, std::string>(result));
    EXPECT_EQ(std::get<int>(result), 8080);
}

TEST(ExpectedTest, PortOutOfRange) {
    auto result = parse_port("99999");
    ASSERT_TRUE(std::holds_alternative<std::string>(result));
    EXPECT_THAT(std::get<std::string>(result),
                ::testing::HasSubstr("out of range"));
}

TEST(ExpectedTest, InvalidInput) {
    auto result = parse_port("abc");
    ASSERT_TRUE(std::holds_alternative<std::string>(result));
}

TEST(ExpectedTest, BoundaryValues) {
    EXPECT_TRUE(has_value<int, std::string>(parse_port("1")));
    EXPECT_TRUE(has_value<int, std::string>(parse_port("65535")));
    EXPECT_FALSE(has_value<int, std::string>(parse_port("0")));
    EXPECT_FALSE(has_value<int, std::string>(parse_port("65536")));
}
```

The `BoundaryValues` test is doing four checks in one go - the two valid endpoints and the two invalid neighbors. This pattern is useful any time you have a range validation: the exact boundary value is almost always where off-by-one bugs live.

### Q3: Test resource cleanup and RAII under error conditions

This is where RAII testing gets interesting. You want to verify that destructors actually run when an exception unwinds the stack - not just assume that they do. The logging trick here is elegant: the `Resource` class writes to a `vector<string>` on construction and destruction, so after catching the exception you can inspect the log to confirm the right things happened in the right order.

```cpp
#include <gtest/gtest.h>
#include <memory>
#include <vector>
#include <functional>

// === Verify RAII cleanup happens on exception ===

class Resource {
public:
    explicit Resource(std::vector<std::string>& log, const std::string& name)
        : log_(log), name_(name) {
        log_.push_back(name_ + " acquired");
    }

    ~Resource() {
        log_.push_back(name_ + " released");
    }

    void use() { log_.push_back(name_ + " used"); }

private:
    std::vector<std::string>& log_;
    std::string name_;
};

void operation_that_throws(std::vector<std::string>& log) {
    Resource a(log, "A");
    Resource b(log, "B");
    a.use();
    throw std::runtime_error("failure");
    // B and A should still be cleaned up
}

TEST(RAIITest, ResourcesCleanedUpOnException) {
    std::vector<std::string> log;

    EXPECT_THROW(operation_that_throws(log), std::runtime_error);

    // Verify acquisition and release happened in correct order
    ASSERT_EQ(log.size(), 5);
    EXPECT_EQ(log[0], "A acquired");
    EXPECT_EQ(log[1], "B acquired");
    EXPECT_EQ(log[2], "A used");
    EXPECT_EQ(log[3], "B released");  // LIFO destruction
    EXPECT_EQ(log[4], "A released");
}

// === Test scope guard cleanup ===
class ScopeGuard {
public:
    explicit ScopeGuard(std::function<void()> cleanup)
        : cleanup_(std::move(cleanup)), active_(true) {}

    ~ScopeGuard() { if (active_) cleanup_(); }

    void dismiss() { active_ = false; }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;

private:
    std::function<void()> cleanup_;
    bool active_;
};

TEST(ScopeGuardTest, RunsCleanupOnException) {
    bool cleaned_up = false;

    try {
        ScopeGuard guard([&] { cleaned_up = true; });
        throw std::runtime_error("oops");
    } catch (...) {}

    EXPECT_TRUE(cleaned_up);
}

TEST(ScopeGuardTest, DismissedGuardDoesNotCleanUp) {
    bool cleaned_up = false;

    {
        ScopeGuard guard([&] { cleaned_up = true; });
        guard.dismiss();  // Success path - don't clean up
    }

    EXPECT_FALSE(cleaned_up);
}
```

The LIFO destruction order check (`B released` before `A released`) is important. C++ guarantees destructors run in reverse construction order, but a broken RAII wrapper could violate that. The log makes the guarantee visible and testable. The `ScopeGuard` tests cover both branches: the guard fires on exception, but `dismiss()` prevents it on the success path.

---

## Notes

- Test the error path as thoroughly as the happy path - errors happen more often in production than you think, and nobody tests them as carefully as the main flow.
- Use `EXPECT_THROW(expr, type)` for specific exception types, and `EXPECT_ANY_THROW` when you just need to confirm something throws without caring about the exact type.
- Use `EXPECT_THAT` with matchers like `HasSubstr` to verify exception messages contain the right information.
- Always test that state is consistent after an error - no partial updates, no half-initialized objects left behind.
- Test boundary values religiously: 0, -1, empty, MAX, MIN, and off-by-one are where most logic bugs hide.
- Test resource leaks explicitly by logging or counting constructor/destructor calls - don't assume RAII is working correctly.
- Sanitizers (ASan, UBSan) complement error path tests by catching undefined behavior that your assertions might miss.
- `EXPECT_NO_THROW` is useful for verifying that valid inputs don't accidentally throw.
