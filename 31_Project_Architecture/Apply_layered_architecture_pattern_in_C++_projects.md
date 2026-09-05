# Apply layered architecture pattern in C++ projects

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Layered architecture** organizes code into horizontal layers where each layer only depends on the layer below it. This enforces separation of concerns and makes each layer independently testable and replaceable.

The reason this is so valuable is that change tends to cluster by concern. UI requirements change frequently; business rules change occasionally; infrastructure (databases, networking) changes rarely but disruptively. If all three are tangled together, any change can ripple everywhere. Layers are a fence between those change patterns.

### Classic Layers

```cpp
+-----------------------------+
|   Presentation / CLI / API  |  Layer 4: User interface
+-----------------------------+
|     Application Services    |  Layer 3: Use cases, orchestration
+-----------------------------+
|       Domain / Business     |  Layer 2: Core business rules
+-----------------------------+
|    Infrastructure / Data    |  Layer 1: DB, network, filesystem
+-----------------------------+

Dependency direction: TOP -> BOTTOM (each layer depends only on below)
```

| Layer | Responsibility | Dependencies | Changes when... |
| --- | --- | --- | --- |
| **Presentation** | UI, CLI, API endpoints | Application | UI requirements change |
| **Application** | Use case orchestration | Domain | Workflows change |
| **Domain** | Business rules, entities | None (pure) | Business rules change |
| **Infrastructure** | DB, files, network | Domain (interfaces) | Technology changes |

---

## Self-Assessment

### Q1: Implement a layered architecture in C++

**Answer:**

The folder structure is half the battle - you are making the layers visible and enforceable. Here is what that looks like:

```cpp
project/
├── include/
│   ├── domain/           # Layer 2: Pure business logic
│   │   ├── order.h
│   │   ├── product.h
│   │   └── i_order_repository.h
│   ├── application/      # Layer 3: Use cases
│   │   └── order_service.h
│   ├── infrastructure/   # Layer 1: Implementations
│   │   └── sqlite_order_repository.h
│   └── presentation/     # Layer 4: API/CLI
│       └── order_controller.h
```

The domain layer is the most important one to get right. It has zero dependencies on anything external - no database headers, no networking, no framework types. `Product` and `Order` are pure value types, and `IOrderRepository` defines what the domain needs from storage without specifying how storage works:

```cpp
// === domain/product.h (Layer 2 - no dependencies) ===
#pragma once
#include <string>

struct Product {
    int id;
    std::string name;
    double price;
};

// === domain/order.h ===
#pragma once
#include "domain/product.h"
#include <vector>
#include <numeric>

class Order {
public:
    int id() const { return id_; }
    void set_id(int id) { id_ = id; }

    void add_item(const Product& product, int quantity) {
        items_.push_back({product, quantity});
    }

    double total() const {
        return std::accumulate(items_.begin(), items_.end(), 0.0,
            [](double sum, const auto& item) {
                return sum + item.product.price * item.quantity;
            });
    }

    double total_with_discount(double discount_pct) const {
        return total() * (1.0 - discount_pct / 100.0);
    }

    struct LineItem {
        Product product;
        int quantity;
    };
    const std::vector<LineItem>& items() const { return items_; }

private:
    int id_ = 0;
    std::vector<LineItem> items_;
};

// === domain/i_order_repository.h (interface - Layer 2) ===
#pragma once
#include "domain/order.h"
#include <optional>
#include <vector>

class IOrderRepository {
public:
    virtual ~IOrderRepository() = default;
    virtual void save(const Order& order) = 0;
    virtual std::optional<Order> find_by_id(int id) = 0;
    virtual std::vector<Order> find_all() = 0;
    virtual void remove(int id) = 0;
};
```

The application layer (Layer 3) only knows about domain types. It applies use-case rules - like "you can't place an empty order" - and delegates persistence to whatever implements `IOrderRepository`. It never mentions SQLite:

```cpp
// === application/order_service.h (Layer 3 - depends on domain) ===
#pragma once
#include "domain/order.h"
#include "domain/i_order_repository.h"
#include <memory>
#include <stdexcept>

class OrderService {
public:
    explicit OrderService(IOrderRepository& repo) : repo_(repo) {}

    void place_order(Order& order) {
        if (order.items().empty())
            throw std::invalid_argument("Cannot place empty order");
        if (order.total() <= 0)
            throw std::invalid_argument("Order total must be positive");
        repo_.save(order);
    }

    std::optional<Order> get_order(int id) {
        return repo_.find_by_id(id);
    }

    void cancel_order(int id) {
        auto order = repo_.find_by_id(id);
        if (!order)
            throw std::runtime_error("Order not found");
        repo_.remove(id);
    }

private:
    IOrderRepository& repo_;
};
```

The infrastructure layer (Layer 1) is where the database actually lives. It implements the interface the domain declared. Notice how the dependency arrow points upward - infrastructure depends on domain, not the other way round. This is the Dependency Inversion Principle in action:

```cpp
// === infrastructure/sqlite_order_repository.h (Layer 1) ===
#pragma once
#include "domain/i_order_repository.h"

class SqliteOrderRepository : public IOrderRepository {
public:
    explicit SqliteOrderRepository(const std::string& db_path)
        : db_path_(db_path) {}

    void save(const Order& order) override {
        // INSERT INTO orders ... (real SQLite code)
    }
    std::optional<Order> find_by_id(int id) override {
        // SELECT FROM orders WHERE id = ...
        return std::nullopt;
    }
    std::vector<Order> find_all() override { return {}; }
    void remove(int id) override {}

private:
    std::string db_path_;
};
```

### Q2: Enforce layer dependencies with CMake

**Answer:**

CMake's `target_link_libraries` is your enforcement mechanism. If the domain library does not link to anything external, then domain code literally cannot compile if someone sneaks in an infrastructure include. The build system becomes your architecture guardian:

```cmake
# === CMakeLists.txt ===
# Domain layer: ZERO dependencies
add_library(domain INTERFACE)
target_include_directories(domain INTERFACE include/)

# Application layer: depends on domain only
add_library(application src/order_service.cpp)
target_link_libraries(application PUBLIC domain)

# Infrastructure layer: depends on domain (implements interfaces)
add_library(infrastructure src/sqlite_order_repository.cpp)
target_link_libraries(infrastructure PUBLIC domain PRIVATE sqlite3)

# Presentation layer: depends on application + infrastructure
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE application infrastructure)

# Tests: can test each layer independently
add_executable(domain_tests tests/test_order.cpp)
target_link_libraries(domain_tests PRIVATE domain GTest::gtest_main)

add_executable(service_tests tests/test_order_service.cpp)
target_link_libraries(service_tests PRIVATE application GTest::gtest_main GTest::gmock)
```

Notice that `sqlite3` is a `PRIVATE` dependency of infrastructure. That means domain and application layers never see the SQLite headers transitively - the boundary is enforced by the build graph itself.

### Q3: Test layers independently

**Answer:**

This is where the layered design pays off most visibly. The domain layer tests need nothing but the domain itself - no mocks, no fakes, just plain function calls:

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include "application/order_service.h"

// === Domain layer test: no mocks needed ===
TEST(OrderTest, CalculatesTotal) {
    Order order;
    order.add_item({1, "Widget", 10.0}, 3);
    order.add_item({2, "Gadget", 25.0}, 1);
    EXPECT_DOUBLE_EQ(order.total(), 55.0);
}

TEST(OrderTest, DiscountApplied) {
    Order order;
    order.add_item({1, "Widget", 100.0}, 1);
    EXPECT_DOUBLE_EQ(order.total_with_discount(10.0), 90.0);
}

// === Application layer test: mock infrastructure ===
class MockOrderRepo : public IOrderRepository {
public:
    MOCK_METHOD(void, save, (const Order&), (override));
    MOCK_METHOD(std::optional<Order>, find_by_id, (int), (override));
    MOCK_METHOD(std::vector<Order>, find_all, (), (override));
    MOCK_METHOD(void, remove, (int), (override));
};

TEST(OrderServiceTest, PlaceOrderSavesToRepo) {
    MockOrderRepo repo;
    OrderService service(repo);

    Order order;
    order.add_item({1, "Widget", 10.0}, 1);

    EXPECT_CALL(repo, save(testing::_)).Times(1);
    service.place_order(order);
}

TEST(OrderServiceTest, EmptyOrderRejected) {
    MockOrderRepo repo;
    OrderService service(repo);
    Order empty;
    EXPECT_THROW(service.place_order(empty), std::invalid_argument);
}
```

The `MockOrderRepo` replaces SQLite entirely for the application layer tests. You can exercise every code path in `OrderService` - including error paths - without a database anywhere in sight. Tests that do not touch I/O are fast, deterministic, and never fail due to environment issues.

---

## Notes

- **Domain layer must have zero external dependencies** - it is the core of your application, and everything else depends on it, not vice versa.
- Interface definitions (e.g., `IOrderRepository`) live in the domain layer; their implementations live in infrastructure.
- CMake `target_link_libraries` enforces layer boundaries at build time - if a layer accidentally depends on the wrong thing, the build fails.
- Each layer can be tested independently: domain with no mocks, application with mocked interfaces.
- Data flows down (domain does not know about databases), but control can be inverted via interfaces.
- Layered architecture is a stepping stone to hexagonal architecture (ports and adapters).
