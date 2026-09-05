# Know how to design classes for testability with dependency injection

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**Dependency Injection (DI)** means supplying a class's collaborators from the outside rather than having the class create or locate them internally. This is the single most impactful design decision for testability in C++ because it lets you substitute mocks, fakes, or stubs during testing without modifying production code. The rule of thumb is simple: if a class creates its own database connection, HTTP client, or logger, you can't test it without those real services running. If those things are injected, you can hand in a fake and test the business logic in complete isolation.

### DI Injection Styles

| Style | Mechanism | When to Use | Testability |
| --- | --- | --- | --- |
| **Constructor injection** | Pass dependency via ctor | Always preferred - makes deps explicit | Excellent |
| **Method injection** | Pass dependency per call | When dep varies per invocation | Good |
| **Template injection** | Type parameter for dep | When virtual dispatch cost is unacceptable | Excellent (compile-time) |
| **Setter injection** | Set dep after construction | Legacy code; two-phase init | Moderate |
| **Service locator** | Global registry lookup | Last resort; migration from singletons | Poor (hidden dep) |

### Design for Testability Principles

```cpp
                    +-----------------+
                    |  IRepository     |  <- Abstract interface
                    +--------+--------+
                             |
              +--------------+--------------+
              |              |              |
    +---------+--+  +--------+-----+  +----+---------+
    | SqlRepo     |  | InMemoryRepo |  | MockRepo     |
    | (production)|  | (integration)|  | (unit test)  |
    +------------+  +--------------+  +--------------+
```

**Key rules:**

1. Depend on **abstractions** (interfaces), not concretions.
2. **No `new` inside business logic** - receive ready-made objects.
3. **No globals or singletons** as implicit dependencies.
4. **Thin constructors** - don't do work in the constructor.
5. **Single Responsibility** - each class does one thing, testable in isolation.

---

## Self-Assessment

### Q1: Refactor a tightly-coupled class to be testable via constructor injection

The "before" version below is a pattern you'll recognize from legacy codebases: the class creates its own database connection internally. This means every test that exercises `OrderProcessor` requires a real database to be running, configured, and populated with the right test data. That's slow, brittle, and couples your unit tests to your infrastructure. The "after" version receives the database as an interface reference - now you can hand in a mock and test all the business logic instantly.

**Answer:**

```cpp
#include <string>
#include <vector>
#include <memory>
#include <cassert>
#include <iostream>
#include <optional>
#include <utility>

// === BEFORE: Untestable - hard-coded dependency ===
/*
class OrderProcessor {
    SqlDatabase db_;  // Created internally - can't mock!
public:
    OrderProcessor() : db_("prod_connection_string") {}
    bool process(int orderId) {
        auto order = db_.query("SELECT * FROM orders WHERE id=" + std::to_string(orderId));
        // business logic...
        db_.execute("UPDATE orders SET status='processed' WHERE id=" + std::to_string(orderId));
        return true;
    }
};
// Testing this requires a real database. Every test is slow, flaky, and
// requires setup/teardown of test data.
*/

// === AFTER: Testable - dependency injected ===

// Step 1: Define abstract interface for the dependency
class IOrderRepository {
public:
    virtual ~IOrderRepository() = default;

    struct Order {
        int id;
        std::string product;
        double amount;
        std::string status;
    };

    virtual std::optional<Order> find(int orderId) = 0;
    virtual void updateStatus(int orderId, const std::string& status) = 0;
};

// Step 2: Business logic depends only on the interface
class OrderProcessor {
public:
    // Constructor injection - dependency is explicit and required
    explicit OrderProcessor(IOrderRepository& repo) : repo_(repo) {}

    bool process(int orderId) {
        auto order = repo_.find(orderId);
        if (!order) return false;

        if (order->amount <= 0) return false;  // Business rule

        repo_.updateStatus(orderId, "processed");
        return true;
    }

private:
    IOrderRepository& repo_;  // Reference: non-owning, must outlive this
};

// Step 3: Production implementation
class SqlOrderRepository : public IOrderRepository {
public:
    std::optional<Order> find(int orderId) override {
        // Real SQL query...
        return Order{orderId, "Widget", 42.0, "pending"};
    }
    void updateStatus(int orderId, const std::string& status) override {
        // Real SQL update...
    }
};

// Step 4: Test mock - no framework needed for simple cases
class MockOrderRepository : public IOrderRepository {
public:
    std::optional<Order> find_result;
    std::vector<std::pair<int, std::string>> update_calls;

    std::optional<Order> find(int orderId) override {
        return find_result;
    }

    void updateStatus(int orderId, const std::string& status) override {
        update_calls.emplace_back(orderId, status);
    }
};

// Step 5: Unit test - fast, deterministic, no I/O
void test_process_valid_order() {
    MockOrderRepository mock;
    mock.find_result = IOrderRepository::Order{1, "Widget", 42.0, "pending"};

    OrderProcessor processor(mock);
    assert(processor.process(1) == true);
    assert(mock.update_calls.size() == 1);
    assert(mock.update_calls[0].second == "processed");
}

void test_process_missing_order() {
    MockOrderRepository mock;
    mock.find_result = std::nullopt;

    OrderProcessor processor(mock);
    assert(processor.process(999) == false);
    assert(mock.update_calls.empty());
}

void test_process_zero_amount_order() {
    MockOrderRepository mock;
    mock.find_result = IOrderRepository::Order{1, "Free item", 0.0, "pending"};

    OrderProcessor processor(mock);
    assert(processor.process(1) == false);
    assert(mock.update_calls.empty());  // Never updated
}

int main() {
    test_process_valid_order();
    test_process_missing_order();
    test_process_zero_amount_order();
    std::cout << "All tests passed\n";
}
```

### Q2: Compare DI approaches - constructor, template, and type-erased - for testability

There's more than one way to inject a dependency. The virtual interface approach is the most familiar but has a small runtime cost from vtable dispatch. The template approach has zero overhead but requires the dependency type to be known at compile time. The `std::function` approach is the most lightweight for single-operation dependencies. Here's how they look side by side.

**Answer:**

| Aspect | Virtual Interface (runtime DI) | Template Parameter (compile-time DI) | `std::function` / Type-erased |
| --- | --- | --- | --- |
| **Overhead** | vtable dispatch (~1-3ns) | Zero (inlined) | `std::function` may heap-allocate |
| **Mockability** | Derive mock from interface | Instantiate with mock type | Pass lambda/functor |
| **Binary size** | One instantiation | Template bloat possible | One instantiation |
| **ABI stability** | Stable across shared libs | Headers must be visible | Stable |
| **Readability** | Familiar OOP style | Can be verbose | Natural for simple callbacks |

```cpp
// === Template-based DI (zero-cost, compile-time) ===
template<typename Repository>
class OrderProcessorT {
    Repository& repo_;
public:
    explicit OrderProcessorT(Repository& repo) : repo_(repo) {}

    bool process(int orderId) {
        auto order = repo_.find(orderId);
        if (!order) return false;
        if (order->amount <= 0) return false;
        repo_.updateStatus(orderId, "processed");
        return true;
    }
};

// In production:
// SqlOrderRepository sql_repo(conn_string);
// OrderProcessorT processor(sql_repo);

// In tests - no virtual, no overhead:
struct FakeRepo {
    struct Order { int id; std::string product; double amount; std::string status; };
    std::optional<Order> find_result;
    int update_count = 0;

    std::optional<Order> find(int) { return find_result; }
    void updateStatus(int, const std::string&) { ++update_count; }
};

// FakeRepo fake;
// OrderProcessorT processor(fake);

// === std::function-based DI (for simple cases) ===
class NotificationService {
public:
    using SendFn = std::function<bool(const std::string& to, const std::string& body)>;

    explicit NotificationService(SendFn sender) : send_(std::move(sender)) {}

    bool notify(const std::string& user, const std::string& event) {
        return send_(user, "Event: " + event);
    }

private:
    SendFn send_;
};

// In test:
// bool captured = false;
// NotificationService svc([&](auto&, auto&) { captured = true; return true; });
```

**Guidelines:**

- Default to **virtual interface DI** - simplest, most readable.
- Use **template DI** in hot paths where vtable cost matters (measured, not assumed).
- Use **`std::function`** when the dependency is a single operation, not a role.

### Q3: Design a complete testable subsystem with multiple injected dependencies

This is a realistic example of what dependency injection looks like at scale. The `CheckoutService` has three dependencies - payment, inventory, and notification - all injected through the constructor. The mock objects record what was called so you can assert on behavior, not just output. This is the pattern you'd use in a real project.

**Answer:**

```cpp
#include <memory>
#include <string>
#include <vector>
#include <iostream>
#include <cassert>

// === Interfaces for all external dependencies ===

class IPaymentGateway {
public:
    virtual ~IPaymentGateway() = default;
    virtual bool charge(const std::string& token, double amount) = 0;
    virtual bool refund(const std::string& transactionId) = 0;
};

class IInventoryService {
public:
    virtual ~IInventoryService() = default;
    virtual bool reserve(const std::string& sku, int qty) = 0;
    virtual void release(const std::string& sku, int qty) = 0;
};

class INotifier {
public:
    virtual ~INotifier() = default;
    virtual void send(const std::string& to, const std::string& message) = 0;
};

// === Business logic with all dependencies injected ===

class CheckoutService {
public:
    // All dependencies injected via constructor
    CheckoutService(IPaymentGateway& payment,
                    IInventoryService& inventory,
                    INotifier& notifier)
        : payment_(payment), inventory_(inventory), notifier_(notifier) {}

    struct LineItem { std::string sku; int qty; double price; };

    bool checkout(const std::string& userId,
                  const std::string& paymentToken,
                  const std::vector<LineItem>& items) {
        // 1. Reserve inventory (rollback on failure)
        std::vector<const LineItem*> reserved;
        for (const auto& item : items) {
            if (!inventory_.reserve(item.sku, item.qty)) {
                // Rollback previously reserved
                for (auto* r : reserved)
                    inventory_.release(r->sku, r->qty);
                return false;
            }
            reserved.push_back(&item);
        }

        // 2. Charge payment
        double total = 0;
        for (const auto& item : items)
            total += item.price * item.qty;

        if (!payment_.charge(paymentToken, total)) {
            // Rollback inventory
            for (const auto& item : items)
                inventory_.release(item.sku, item.qty);
            return false;
        }

        // 3. Notify user
        notifier_.send(userId, "Order confirmed! Total: $" + std::to_string(total));
        return true;
    }

private:
    IPaymentGateway& payment_;
    IInventoryService& inventory_;
    INotifier& notifier_;
};

// === Test doubles ===

class MockPayment : public IPaymentGateway {
public:
    bool charge_result = true;
    int charge_count = 0;
    double last_amount = 0;

    bool charge(const std::string&, double amount) override {
        ++charge_count;
        last_amount = amount;
        return charge_result;
    }
    bool refund(const std::string&) override { return true; }
};

class MockInventory : public IInventoryService {
public:
    bool reserve_result = true;
    int reserve_count = 0;
    int release_count = 0;

    bool reserve(const std::string&, int) override {
        ++reserve_count;
        return reserve_result;
    }
    void release(const std::string&, int) override { ++release_count; }
};

class MockNotifier : public INotifier {
public:
    std::vector<std::pair<std::string, std::string>> sent;
    void send(const std::string& to, const std::string& msg) override {
        sent.emplace_back(to, msg);
    }
};

// === Tests ===

void test_successful_checkout() {
    MockPayment pay;
    MockInventory inv;
    MockNotifier notify;
    CheckoutService svc(pay, inv, notify);

    std::vector<CheckoutService::LineItem> items{{"SKU1", 2, 10.0}, {"SKU2", 1, 5.0}};
    assert(svc.checkout("user@test.com", "tok_123", items));

    assert(inv.reserve_count == 2);
    assert(pay.charge_count == 1);
    assert(pay.last_amount == 25.0);
    assert(notify.sent.size() == 1);
    assert(inv.release_count == 0);  // No rollback
}

void test_payment_failure_rollback() {
    MockPayment pay;
    pay.charge_result = false;  // Simulate payment failure
    MockInventory inv;
    MockNotifier notify;
    CheckoutService svc(pay, inv, notify);

    std::vector<CheckoutService::LineItem> items{{"SKU1", 2, 10.0}};
    assert(svc.checkout("user@test.com", "tok_bad", items) == false);

    assert(inv.reserve_count == 1);   // Tried to reserve
    assert(inv.release_count == 1);   // Rolled back
    assert(notify.sent.empty());      // No notification
}

void test_inventory_failure_partial_rollback() {
    MockPayment pay;
    MockInventory inv;
    MockNotifier notify;
    CheckoutService svc(pay, inv, notify);

    // First reserve succeeds, second fails
    int call = 0;
    // Override with stateful lambda via a specialized mock
    struct PartialMockInventory : public IInventoryService {
        int call_count = 0;
        int release_count = 0;
        bool reserve(const std::string&, int) override {
            return (call_count++ == 0);  // First succeeds, second fails
        }
        void release(const std::string&, int) override { ++release_count; }
    } partial_inv;

    CheckoutService svc2(pay, partial_inv, notify);
    std::vector<CheckoutService::LineItem> items{{"A", 1, 10.0}, {"B", 1, 20.0}};
    assert(svc2.checkout("user", "tok", items) == false);
    assert(partial_inv.release_count == 1);  // Only first item rolled back
}

int main() {
    test_successful_checkout();
    test_payment_failure_rollback();
    test_inventory_failure_partial_rollback();
    std::cout << "All checkout tests passed\n";
}
```

Each test function sets up exactly the scenario it needs - failure, success, partial failure - and the mock objects record every call so you can assert on what happened. No database, no network, no configuration files required. This is the payoff for the DI discipline: tests that run in microseconds and give you pinpoint information about what went wrong.

---

## Notes

- **Constructor injection** is the gold standard - all dependencies are visible in the constructor signature.
- Store injected dependencies as **references** (non-owning) or **`unique_ptr`** (owning) depending on lifetime.
- Use `unique_ptr<IInterface>` when the class owns the dependency and needs polymorphic deletion.
- Use references when the caller manages the dependency's lifetime.
- Avoid setter injection - it creates a two-phase initialization window where the object is in an invalid state.
- For C++ without virtual overhead: use template parameters (policy-based DI) or concepts (C++20).
- Popular C++ DI frameworks: Boost.DI, fruit - but manual DI is often sufficient.
- The **Composition Root** pattern: wire all dependencies at the top level (main), pass them down.
