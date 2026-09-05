# Apply CQRS (Command Query Responsibility Segregation) in C++ systems

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**CQRS** separates read (query) and write (command) operations into distinct models. The write side handles validation, business rules, and state mutations; the read side is optimized for specific query patterns with denormalized views. This separation lets you scale, optimize, and evolve each side completely independently - a huge deal when your read and write traffic look nothing alike.

The core intuition is simple: the shape of data you need for a complex write operation (validation, business rule enforcement, consistency checks) is very different from the shape you need when rendering a list on a screen. Forcing both through one model is always a compromise. CQRS says "stop compromising - just build two."

### CQRS vs Traditional CRUD

| Aspect | Traditional CRUD | CQRS |
| --- | --- | --- |
| Model | Single model for read/write | Separate read/write models |
| Optimization | Compromise for both | Each optimized independently |
| Complexity | Lower | Higher |
| Scalability | Limited by single model | Read/write scale independently |
| Consistency | Immediate | Eventually consistent (typically) |
| Best for | Simple domains | Complex domains, high read:write ratio |

---

## Self-Assessment

### Q1: Implement basic CQRS with command and query separation

**Answer:**

The pattern starts by drawing a hard line between the two sides. Commands carry intent ("create this order", "cancel this order") and go through handlers that apply business rules. Queries are simple requests for data and go through handlers that read from a purpose-built read store. Here's what that looks like in practice:

```cpp
#include <string>
#include <vector>
#include <memory>
#include <unordered_map>
#include <functional>
#include <variant>
#include <optional>

// === Commands (write side) ===
struct CreateOrderCommand {
    std::string customer_id;
    std::vector<std::pair<std::string, int>> items;  // product_id, qty
};

struct CancelOrderCommand {
    std::string order_id;
    std::string reason;
};

struct ShipOrderCommand {
    std::string order_id;
    std::string tracking_number;
};

using Command = std::variant<CreateOrderCommand, CancelOrderCommand,
                             ShipOrderCommand>;

// === Queries (read side) ===
struct GetOrderQuery {
    std::string order_id;
};

struct ListOrdersByCustomerQuery {
    std::string customer_id;
    int page = 0;
    int page_size = 20;
};

// === Read model (denormalized for fast queries) ===
struct OrderView {
    std::string order_id;
    std::string customer_id;
    std::string status;
    double total_amount;
    std::string created_at;
    std::vector<std::string> item_names;  // Denormalized
};

// === Command handler (write side) ===
class OrderCommandHandler {
public:
    std::string handle(const CreateOrderCommand& cmd) {
        Order order;
        order.id = generate_id();
        order.customer_id = cmd.customer_id;
        order.status = OrderStatus::Pending;

        for (const auto& [product_id, qty] : cmd.items) {
            auto product = product_repo_.find(product_id);
            if (!product) throw std::runtime_error("Product not found");
            order.add_item(*product, qty);
        }

        order.validate();  // Business rules
        write_repo_.save(order);

        // Publish event for read-side projection
        event_bus_.publish(OrderCreatedEvent{order});
        return order.id;
    }

    void handle(const CancelOrderCommand& cmd) {
        auto order = write_repo_.load(cmd.order_id);
        order.cancel(cmd.reason);  // Domain logic + validation
        write_repo_.save(order);
        event_bus_.publish(OrderCancelledEvent{cmd.order_id, cmd.reason});
    }

private:
    OrderWriteRepository write_repo_;  // Normalized, optimized for writes
    ProductRepository product_repo_;
    EventBus event_bus_;
};

// === Query handler (read side) ===
class OrderQueryHandler {
public:
    std::optional<OrderView> handle(const GetOrderQuery& q) {
        return read_store_.find_order(q.order_id);
    }

    std::vector<OrderView> handle(const ListOrdersByCustomerQuery& q) {
        return read_store_.orders_by_customer(
            q.customer_id, q.page, q.page_size);
    }

private:
    OrderReadStore read_store_;  // Denormalized, optimized for queries
};
```

Notice that the command handler owns a normalized write repository (structured for integrity) while the query handler owns a denormalized read store (structured for speed). They never touch each other's data stores. The bridge between them is the event bus: when a command succeeds it fires an event, and the read side listens to that event to update its own view.

### Q2: Build read-side projections from events

**Answer:**

This is where the "eventual" in eventual consistency lives. The projection class listens for domain events and updates the read store accordingly. The read store itself is shaped entirely around how queries will be made - not around how the data is stored on the write side.

```cpp
// === Event-driven read model projection ===
struct OrderCreatedEvent {
    std::string order_id;
    std::string customer_id;
    double total;
    std::vector<std::string> item_names;
    std::string timestamp;
};

struct OrderShippedEvent {
    std::string order_id;
    std::string tracking_number;
};

struct OrderCancelledEvent {
    std::string order_id;
    std::string reason;
};

// Projection: updates denormalized read store from events
class OrderProjection {
public:
    explicit OrderProjection(OrderReadStore& store) : store_(store) {}

    void on(const OrderCreatedEvent& e) {
        OrderView view;
        view.order_id = e.order_id;
        view.customer_id = e.customer_id;
        view.total_amount = e.total;
        view.status = "pending";
        view.item_names = e.item_names;
        view.created_at = e.timestamp;

        store_.upsert(view);

        // Also update customer summary (denormalized)
        store_.increment_customer_order_count(e.customer_id);
    }

    void on(const OrderShippedEvent& e) {
        store_.update_status(e.order_id, "shipped");
        store_.set_tracking(e.order_id, e.tracking_number);
    }

    void on(const OrderCancelledEvent& e) {
        store_.update_status(e.order_id, "cancelled");
    }

private:
    OrderReadStore& store_;
};

// === Read store: optimized for query patterns ===
class OrderReadStore {
public:
    void upsert(const OrderView& view) {
        orders_[view.order_id] = view;
        customer_orders_[view.customer_id].push_back(view.order_id);
    }

    std::optional<OrderView> find_order(const std::string& id) {
        auto it = orders_.find(id);
        if (it != orders_.end()) return it->second;
        return std::nullopt;
    }

    std::vector<OrderView> orders_by_customer(
        const std::string& customer_id, int page, int page_size) {
        std::vector<OrderView> result;
        auto it = customer_orders_.find(customer_id);
        if (it == customer_orders_.end()) return result;

        const auto& ids = it->second;
        int start = page * page_size;
        int end = std::min(start + page_size, (int)ids.size());

        for (int i = start; i < end; ++i) {
            if (auto view = find_order(ids[i]))
                result.push_back(*view);
        }
        return result;
    }

private:
    std::unordered_map<std::string, OrderView> orders_;
    std::unordered_map<std::string, std::vector<std::string>> customer_orders_;
};
```

The key design decision here is the `customer_orders_` index. On the write side you would never store this - it duplicates data. But on the read side it means a customer's order list is a single map lookup, not a scan. That is exactly the kind of freedom CQRS gives you.

### Q3: Command dispatcher with middleware

**Answer:**

In a larger system you want a single entry point that routes any command to the right handler, and you want to be able to add cross-cutting concerns (logging, validation, timing) without touching the handlers themselves. A middleware chain achieves that cleanly:

```cpp
// === Generic command dispatcher ===
class CommandDispatcher {
public:
    using Middleware = std::function<void(const Command&, std::function<void()>)>;

    template<typename Cmd>
    void register_handler(std::function<void(const Cmd&)> handler) {
        handlers_[typeid(Cmd).hash_code()] = [handler](const Command& cmd) {
            handler(std::get<Cmd>(cmd));
        };
    }

    void add_middleware(Middleware mw) {
        middlewares_.push_back(std::move(mw));
    }

    void dispatch(const Command& cmd) {
        auto it = handlers_.find(std::visit(
            [](const auto& c) { return typeid(c).hash_code(); }, cmd));
        if (it == handlers_.end())
            throw std::runtime_error("No handler for command");

        // Build middleware chain
        auto final_handler = [&]() { it->second(cmd); };

        auto chain = final_handler;
        for (auto rit = middlewares_.rbegin(); rit != middlewares_.rend(); ++rit) {
            auto& mw = *rit;
            chain = [&mw, &cmd, chain]() { mw(cmd, chain); };
        }
        chain();
    }

private:
    std::unordered_map<size_t, std::function<void(const Command&)>> handlers_;
    std::vector<Middleware> middlewares_;
};

// === Middleware examples ===
auto logging_middleware = [](const Command& cmd, std::function<void()> next) {
    std::cout << "Executing command...\n";
    auto start = std::chrono::steady_clock::now();
    next();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();
    std::cout << "Command completed in " << ms << "ms\n";
};

auto validation_middleware = [](const Command& cmd, std::function<void()> next) {
    // Validate before executing
    std::visit([](const auto& c) {
        // Each command type has its own validation
        validate(c);
    }, cmd);
    next();
};

// Setup:
// dispatcher.add_middleware(logging_middleware);
// dispatcher.add_middleware(validation_middleware);
// dispatcher.dispatch(CreateOrderCommand{...});
```

The chain is built in reverse so the first middleware added wraps the outermost call. Each middleware receives a `next` function it must call to continue the chain - familiar if you have worked with Express or ASP.NET middleware. The real win is that you can add transaction management, auditing, or rate limiting here without touching a single handler.

---

## Notes

- CQRS shines when read and write patterns differ significantly (e.g., complex writes, simple reads).
- The read model can be a SQL view, Redis cache, Elasticsearch index, or in-memory map - whatever makes your queries fastest.
- **Event Sourcing** pairs naturally with CQRS: events drive both write-side persistence and read-side projections.
- Start with separate classes/methods; only evolve to separate databases when scaling actually demands it.
- Eventually consistent read models are acceptable for most UIs - just show "processing..." for recent writes.
- Avoid CQRS for simple CRUD domains. The added complexity is real, and it is only worth it when the domain justifies it.
