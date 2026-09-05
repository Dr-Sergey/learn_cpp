# Design a clean separation between business logic and infrastructure code

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Business logic** encodes the rules and processes of your domain. **Infrastructure code** handles external concerns: databases, networks, file I/O, frameworks. Mixing them creates code that is hard to test, hard to change, and hard to reason about. Clean separation means business logic has zero infrastructure dependencies.

Here is the practical test: can you run your business rules without a database? If the answer is no, your business logic and infrastructure are tangled. The moment you need a database to verify that a 10% discount applies to orders over $100, something has gone wrong. That rule should live in a pure function that a test can call with a list of items and get back a number - no connection strings, no mock setup, just math.

### Separation Signals

| Clean Separation | Mixed Code |
| --- | --- |
| Domain classes have no `#include <sqlite3.h>` | `Order::save()` writes to database directly |
| Business rules testable without mocks | Tests require database setup |
| Can swap database without touching domain | Database change requires editing 50 files |
| Domain code compiles standalone | Domain code will not compile without infra headers |

---

## Self-Assessment

### Q1: Refactor mixed code into separated layers

**Answer:**

Here is what the mixed version looks like - everything living inside one class, business logic right next to raw SQL calls and SMTP calls:

```cpp
// BAD: Business logic mixed with infrastructure
class OrderProcessor_Bad {
public:
    void process_order(int order_id) {
        // Infrastructure: database query
        sqlite3_stmt* stmt;
        sqlite3_prepare_v2(db_, "SELECT * FROM orders WHERE id=?", -1, &stmt, 0);
        sqlite3_bind_int(stmt, 1, order_id);
        // ... parse results ...

        // Business logic: calculate discount
        double total = 0;
        // ... sum items ...
        double discount = (total > 100) ? 0.1 : 0.0;
        double final_amount = total * (1.0 - discount);

        // Infrastructure: send email
        smtp_send("customer@example.com", "Order total: $" + std::to_string(final_amount));

        // Infrastructure: write to database
        sqlite3_exec(db_, "UPDATE orders SET total=...", 0, 0, 0);
    }
private:
    sqlite3* db_;
};
```

Now here is the same functionality split into clean layers. The domain objects and business rules are pure C++ - no external headers, no side effects. The port interfaces (`IOrderStore`, `INotifier`) are how the domain declares what it needs from the outside world without saying how it will be provided:

```cpp
// GOOD: Separated into clean layers

// --- Domain layer (pure business logic, zero infra deps) ---
struct OrderItem {
    std::string name;
    double price;
    int quantity;
};

class OrderCalculator {
public:
    struct Result {
        double subtotal;
        double discount_rate;
        double discount_amount;
        double total;
    };

    static Result calculate(const std::vector<OrderItem>& items) {
        double subtotal = 0;
        for (const auto& item : items)
            subtotal += item.price * item.quantity;

        double discount_rate = (subtotal > 100) ? 0.10 :
                               (subtotal > 50)  ? 0.05 : 0.0;
        double discount_amount = subtotal * discount_rate;
        return {subtotal, discount_rate, discount_amount,
                subtotal - discount_amount};
    }
};

// --- Port interfaces (defined by domain, implemented by infra) ---
class IOrderStore {
public:
    virtual ~IOrderStore() = default;
    virtual std::vector<OrderItem> get_items(int order_id) = 0;
    virtual void update_total(int order_id, double total) = 0;
};

class INotifier {
public:
    virtual ~INotifier() = default;
    virtual void notify_order_processed(const std::string& email,
                                         double total) = 0;
};

// --- Application service (orchestrates domain + ports) ---
class OrderProcessor {
public:
    OrderProcessor(IOrderStore& store, INotifier& notifier)
        : store_(store), notifier_(notifier) {}

    OrderCalculator::Result process(int order_id,
                                     const std::string& customer_email) {
        auto items = store_.get_items(order_id);
        auto result = OrderCalculator::calculate(items);
        store_.update_total(order_id, result.total);
        notifier_.notify_order_processed(customer_email, result.total);
        return result;
    }

private:
    IOrderStore& store_;
    INotifier& notifier_;
};
```

The term "port" comes from hexagonal architecture (also called ports and adapters). A port is an interface that the domain defines but does not implement. The infrastructure layer provides the adapters - concrete classes that implement those interfaces using SQLite, SMTP, or whatever technology you choose. Swapping technologies means writing a new adapter; the domain never changes.

### Q2: Test business logic without any infrastructure

**Answer:**

This is where the payoff becomes obvious. The `OrderCalculator` tests do not need a single mock - they just call a static function:

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>

// === Pure domain test: no mocks needed ===
TEST(OrderCalculatorTest, NoDiscountUnder50) {
    auto result = OrderCalculator::calculate({{"A", 10.0, 3}});  // $30
    EXPECT_DOUBLE_EQ(result.discount_rate, 0.0);
    EXPECT_DOUBLE_EQ(result.total, 30.0);
}

TEST(OrderCalculatorTest, FivePercentOver50) {
    auto result = OrderCalculator::calculate({{"A", 20.0, 3}});  // $60
    EXPECT_DOUBLE_EQ(result.discount_rate, 0.05);
    EXPECT_DOUBLE_EQ(result.total, 57.0);
}

TEST(OrderCalculatorTest, TenPercentOver100) {
    auto result = OrderCalculator::calculate({{"A", 50.0, 3}});  // $150
    EXPECT_DOUBLE_EQ(result.discount_rate, 0.10);
    EXPECT_DOUBLE_EQ(result.total, 135.0);
}

// === Application service test: mock infrastructure ===
class MockStore : public IOrderStore {
public:
    MOCK_METHOD(std::vector<OrderItem>, get_items, (int), (override));
    MOCK_METHOD(void, update_total, (int, double), (override));
};

class MockNotifier : public INotifier {
public:
    MOCK_METHOD(void, notify_order_processed,
                (const std::string&, double), (override));
};

TEST(OrderProcessorTest, ProcessesOrderEndToEnd) {
    MockStore store;
    MockNotifier notifier;
    OrderProcessor processor(store, notifier);

    EXPECT_CALL(store, get_items(42))
        .WillOnce(testing::Return(
            std::vector<OrderItem>{{"Widget", 50.0, 3}}));
    EXPECT_CALL(store, update_total(42, 135.0));
    EXPECT_CALL(notifier, notify_order_processed("a@b.com", 135.0));

    auto result = processor.process(42, "a@b.com");
    EXPECT_DOUBLE_EQ(result.total, 135.0);
}
```

The domain tests run in microseconds and never fail due to external state. The application service tests use mocks precisely because the service is supposed to call the store and notifier - you are verifying the orchestration. If `OrderProcessor` forgot to call `update_total`, the test catches it.

### Q3: Enforce separation with CMake build structure

**Answer:**

The same architecture you drew in the code should be reflected in the build system. If the domain library does not link to `sqlite3`, then domain code cannot accidentally include `<sqlite3.h>` - the linker would fail. CMake becomes your architectural guard:

```cmake
# Domain library: ZERO external dependencies
add_library(domain
    src/domain/order_calculator.cpp
)
target_include_directories(domain PUBLIC include/domain)
# NOTE: domain does NOT link_libraries to anything external

# Application library: depends on domain only (+ port interfaces)
add_library(application
    src/application/order_processor.cpp
)
target_link_libraries(application PUBLIC domain)

# Infrastructure library: implements ports, depends on external libs
add_library(infrastructure
    src/infrastructure/sqlite_order_store.cpp
    src/infrastructure/smtp_notifier.cpp
)
target_link_libraries(infrastructure
    PUBLIC domain          # Implements domain interfaces
    PRIVATE sqlite3 curl   # External dependencies
)

# Domain tests: FAST, no external dependencies
add_executable(domain_tests tests/test_order_calculator.cpp)
target_link_libraries(domain_tests PRIVATE domain GTest::gtest_main)

# Application tests: mocked infrastructure
add_executable(app_tests tests/test_order_processor.cpp)
target_link_libraries(app_tests PRIVATE application GTest::gtest_main GTest::gmock)
```

The `sqlite3` and `curl` dependencies are `PRIVATE` to the infrastructure library. That means they do not propagate to anything that links against infrastructure - the application layer and domain layer are completely shielded from those headers. Any violation (someone including `<sqlite3.h>` in a domain file) becomes a compile error immediately.

---

## Notes

- **The domain library must compile with zero external dependencies** - this is the clearest litmus test for whether your separation is real.
- Business rules in pure functions (like `OrderCalculator::calculate`) are trivially testable and never flaky.
- Infrastructure interfaces are defined by the domain and implemented by infrastructure - this is the Dependency Inversion Principle.
- CMake enforces boundaries: if the domain accidentally includes `<sqlite3.h>`, the build breaks.
- Domain tests are fast (milliseconds) and never flaky because they do not touch I/O.
- The application service orchestrates the workflow but contains no business rules itself.
- This pattern is the foundation of hexagonal architecture and clean architecture.
