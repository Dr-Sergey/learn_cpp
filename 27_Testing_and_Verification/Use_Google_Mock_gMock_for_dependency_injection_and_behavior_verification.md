# Use Google Mock (gMock) for dependency injection and behavior verification

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://google.github.io/googletest/gmock_cook_book.html>  

---

## Topic Overview

This file focuses on the **dependency injection pattern** with gMock - designing classes so mock objects can replace real dependencies, and verifying that the system interacts correctly with them. (See file #584 for MOCK_METHOD syntax, ON_CALL basics, and test double taxonomy.)

The whole point of dependency injection in testable code is that your class should not decide which implementation of a dependency to use. It should receive the dependency from the outside. When it receives an interface rather than a concrete type, you can hand it a mock in tests and a real implementation in production - the class never knows the difference.

### Dependency Injection Pattern

Here is a side-by-side comparison. The left side is what most codebases start with: a hard-coded dependency that you cannot replace in tests. The right side shows the same class redesigned to accept the dependency through its constructor.

```cpp
Without DI (untestable):            With DI (testable):
┌──────────────┐                    ┌──────────────┐
│ OrderService │                    │ OrderService │
│  Database db │ <- hard-coded      │  IDatabase&  │ <- injected interface
│  Emailer em  │                    │  IEmailer&   │
└──────────────┘                    └──────┬───────┘
                                           │
                   ┌───────────────────────┼──────────────────┐
                   │ Production:           │ Test:            │
                   │ RealDatabase          │ MockDatabase     │
                   │ SmtpEmailer           │ MockEmailer      │
                   └───────────────────────┴──────────────────┘
```

### EXPECT_CALL Matchers Quick Reference

Matchers are how you tell gMock exactly what argument values you care about. You can be as specific or as loose as the test requires - use `_` when you genuinely do not care, and use a precise matcher when the exact value matters to the behavior you are verifying.

| Matcher | Matches |
| --- | --- |
| `_` | Any value |
| `Eq(x)` | Equal to x |
| `Gt(x)`, `Lt(x)`, `Ge(x)`, `Le(x)` | Comparison |
| `HasSubstr(s)` | String containing s |
| `StartsWith(s)` | String starting with s |
| `AllOf(m1, m2)` | Both matchers match |
| `AnyOf(m1, m2)` | Either matcher matches |
| `Not(m)` | Inverse of matcher |
| `Property(&Class::member, m)` | Match object's member |

---

## Self-Assessment

### Q1: Create a mock class for an interface using MOCK_METHOD and inject it in the system under test

This example is a checkout flow that touches three separate dependencies: a payment gateway, an inventory system, and an email service. All three are injected through the constructor, which means in a test you can hand in mocks for all three and verify exactly what the `CheckoutService` does when everything works correctly.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>
#include <memory>
#include <vector>

// Interfaces (abstractions for DI)
class IPaymentGateway {
public:
    virtual ~IPaymentGateway() = default;
    virtual bool charge(const std::string& card_token, double amount) = 0;
    virtual bool refund(const std::string& transaction_id, double amount) = 0;
};

class IInventory {
public:
    virtual ~IInventory() = default;
    virtual int stock(const std::string& sku) const = 0;
    virtual void reserve(const std::string& sku, int qty) = 0;
    virtual void release(const std::string& sku, int qty) = 0;
};

class IEmailService {
public:
    virtual ~IEmailService() = default;
    virtual void send(const std::string& to, const std::string& subject,
                      const std::string& body) = 0;
};

// Mock implementations
class MockPaymentGateway : public IPaymentGateway {
public:
    MOCK_METHOD(bool, charge, (const std::string&, double), (override));
    MOCK_METHOD(bool, refund, (const std::string&, double), (override));
};

class MockInventory : public IInventory {
public:
    MOCK_METHOD(int, stock, (const std::string&), (const, override));
    MOCK_METHOD(void, reserve, (const std::string&, int), (override));
    MOCK_METHOD(void, release, (const std::string&, int), (override));
};

class MockEmailService : public IEmailService {
public:
    MOCK_METHOD(void, send, (const std::string&, const std::string&,
                              const std::string&), (override));
};

// System under test - accepts interfaces via DI
class CheckoutService {
    IPaymentGateway& payment_;
    IInventory& inventory_;
    IEmailService& email_;
public:
    CheckoutService(IPaymentGateway& p, IInventory& i, IEmailService& e)
        : payment_(p), inventory_(i), email_(e) {}

    struct Result { bool success; std::string message; };

    Result checkout(const std::string& user_email,
                    const std::string& sku, int qty,
                    const std::string& card_token) {
        // Check stock
        if (inventory_.stock(sku) < qty)
            return {false, "out of stock"};

        // Reserve inventory
        inventory_.reserve(sku, qty);

        // Charge payment
        double amount = qty * 29.99;
        if (!payment_.charge(card_token, amount)) {
            inventory_.release(sku, qty);
            return {false, "payment failed"};
        }

        // Send confirmation
        email_.send(user_email, "Order Confirmed",
                    "Your order for " + std::to_string(qty) + "x " + sku);
        return {true, "order placed"};
    }
};

// Test: inject mocks
using ::testing::Return;
using ::testing::_;

TEST(CheckoutService, SuccessfulOrder) {
    MockPaymentGateway payment;
    MockInventory inventory;
    MockEmailService email;

    // Inject all three mocks
    CheckoutService service(payment, inventory, email);

    ON_CALL(inventory, stock("SKU-001")).WillByDefault(Return(10));
    EXPECT_CALL(inventory, reserve("SKU-001", 2)).Times(1);
    EXPECT_CALL(payment, charge("tok_abc", 59.98)).WillOnce(Return(true));
    EXPECT_CALL(email, send("user@test.com", "Order Confirmed", _)).Times(1);

    auto result = service.checkout("user@test.com", "SKU-001", 2, "tok_abc");
    EXPECT_TRUE(result.success);
}
```

Notice the split between `ON_CALL` and `EXPECT_CALL`. Stock checking is handled permissively with `ON_CALL` because the test cares about what happens after a stock check passes, not about how many times it is called. The `reserve`, `charge`, and `send` calls are strict expectations because those are the side effects this test is actually asserting.

### Q2: Set expectations with EXPECT_CALL and verify call counts and argument values

Now let's look at the failure path. When payment fails, the service must release inventory - this is a critical correctness requirement. We use precise matchers and call counts to assert that the right things happen and that email is never sent.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>

using ::testing::_;
using ::testing::Gt;
using ::testing::Le;
using ::testing::HasSubstr;
using ::testing::Return;
using ::testing::InSequence;
using ::testing::AtLeast;

// Reusing the interfaces from Q1

TEST(CheckoutService, PaymentFailureReleasesInventory) {
    MockPaymentGateway payment;
    MockInventory inventory;
    MockEmailService email;
    CheckoutService service(payment, inventory, email);

    // Verify EXACT call count and argument values
    EXPECT_CALL(inventory, stock("SKU-002"))
        .Times(1)
        .WillOnce(Return(5));

    EXPECT_CALL(inventory, reserve("SKU-002", 3))
        .Times(1);  // Must be called exactly once

    EXPECT_CALL(payment, charge("tok_bad", Gt(0.0)))  // amount > 0
        .WillOnce(Return(false));  // Simulate failure

    // Inventory must be released on payment failure
    EXPECT_CALL(inventory, release("SKU-002", 3))
        .Times(1);

    // Email must NOT be sent
    EXPECT_CALL(email, send(_, _, _))
        .Times(0);

    auto result = service.checkout("user@test.com", "SKU-002", 3, "tok_bad");
    EXPECT_FALSE(result.success);
    EXPECT_EQ(result.message, "payment failed");
}

TEST(CheckoutService, OrderSequenceIsCorrect) {
    MockPaymentGateway payment;
    MockInventory inventory;
    MockEmailService email;
    CheckoutService service(payment, inventory, email);

    // Enforce call ORDER
    {
        InSequence seq;
        EXPECT_CALL(inventory, stock(_)).WillOnce(Return(100));
        EXPECT_CALL(inventory, reserve(_, _));
        EXPECT_CALL(payment, charge(_, _)).WillOnce(Return(true));
        EXPECT_CALL(email, send(_, _, _));
    }

    service.checkout("a@b.com", "X", 1, "tok");
}
```

The `InSequence` block in the second test is worth highlighting. Without it, gMock verifies that each call happens, but does not care about order. With `InSequence`, the test will fail if `payment.charge` is called before `inventory.reserve`, for example. Use this when the order of interactions is part of the contract you are enforcing.

### Q3: Use ON_CALL to define default behaviors for stubs that don't need strict verification

The second test in this section shows a more advanced technique: using `Invoke` to make a mock stateful. Instead of returning a fixed value, you wire the mock to a lambda that updates a local map. This is a middle ground between a strict mock and a full fake - you get realistic behavior without writing an entire `IInventory` implementation.

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>
#include <unordered_map>

using ::testing::Return;
using ::testing::_;
using ::testing::Invoke;

// When focused testing needs permissive stubs

class MockInventory;  // Defined above

TEST(CheckoutService, FocusOnEmailContent) {
    // We only care about the email in this test
    MockPaymentGateway payment;
    MockInventory inventory;
    MockEmailService email;
    CheckoutService service(payment, inventory, email);

    // Set defaults - these won't cause test failure
    ON_CALL(inventory, stock(_)).WillByDefault(Return(999));
    ON_CALL(payment, charge(_, _)).WillByDefault(Return(true));

    // Only set strict expectation on what we're testing
    EXPECT_CALL(email, send(
        "alice@test.com",                       // exact recipient
        "Order Confirmed",                      // exact subject
        testing::HasSubstr("3x WIDGET-A")       // body contains this
    )).Times(1);

    service.checkout("alice@test.com", "WIDGET-A", 3, "tok");
    // inventory and payment calls happen but don't fail
}

// ON_CALL with Invoke for stateful stubs
TEST(CheckoutService, InventoryDecreasesAfterReserve) {
    MockPaymentGateway payment;
    MockInventory inventory;
    MockEmailService email;
    CheckoutService service(payment, inventory, email);

    // Stateful stub using Invoke
    std::unordered_map<std::string, int> fake_stock = {{"A", 5}};

    ON_CALL(inventory, stock(_))
        .WillByDefault(Invoke([&](const std::string& sku) {
            return fake_stock[sku];
        }));
    ON_CALL(inventory, reserve(_, _))
        .WillByDefault(Invoke([&](const std::string& sku, int qty) {
            fake_stock[sku] -= qty;
        }));
    ON_CALL(inventory, release(_, _))
        .WillByDefault(Invoke([&](const std::string& sku, int qty) {
            fake_stock[sku] += qty;
        }));
    ON_CALL(payment, charge(_, _)).WillByDefault(Return(true));

    service.checkout("x@y.com", "A", 2, "tok");
    EXPECT_EQ(fake_stock["A"], 3);  // 5 - 2 = 3
}
```

---

## Notes

- Constructor injection (pass `Interface&`) is the simplest DI pattern in C++
- For classes that can't take references, use `std::unique_ptr<Interface>` or `std::function`
- `NiceMock<MockFoo>` suppresses warnings for methods without `EXPECT_CALL` - use with `ON_CALL` defaults
- `StrictMock<MockFoo>` fails on ANY unmatched call - use sparingly
- Mock objects auto-verify in their destructor - no explicit `verify()` call needed
