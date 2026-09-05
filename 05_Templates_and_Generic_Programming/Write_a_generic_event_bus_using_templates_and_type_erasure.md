# Write a Generic Event Bus Using Templates and Type Erasure

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/types/type_index>  

---

## Topic Overview

### What Is a Generic Event Bus

An **event bus** is a publish-subscribe messaging system where:

- **Publishers** emit events of various types
- **Subscribers** register handlers for specific event types
- The bus **dispatches** each event to all matching handlers

A **generic** event bus uses templates so any struct can be an event - no base class required. **Type erasure** stores heterogeneous handlers in a single container.

Here is the reason this design is tricky: you want to store handlers for `MouseClick`, `KeyPress`, `WindowResize`, and any other event type in a single map. Those handlers all have different types, so you cannot put them in a `std::vector<handler_type>` directly. Type erasure is how you solve that - you hide the concrete handler type behind a uniform wrapper.

```cpp
  Publisher                    Event Bus                    Subscribers
  ┌────────┐   publish<E>()   ┌──────────────┐             ┌──────────┐
  │ Code A ├──────────────────►│              ├────────────►│ Handler1 │
  └────────┘                  │  type_index   │             └──────────┘
                              │     ↓         │             ┌──────────┐
  ┌────────┐   publish<F>()   │  handlers     ├────────────►│ Handler2 │
  │ Code B ├──────────────────►│  map          │             └──────────┘
  └────────┘                  └──────────────┘
```

### Key Components

| Component | Role |
| --- | --- |
| `std::type_index` | Runtime type identifier - used as map key to route events |
| `std::any` | Type-erased storage for handler functions |
| `std::function<void(const Event&)>` | Typed handler wrapper |
| `subscribe<E>(handler)` | Register a handler for event type `E` |
| `publish<E>(event)` | Dispatch event to all `E` handlers |

---

## Self-Assessment

### Q1: Implement `EventBus::subscribe<EventT>(handler)` and `EventBus::publish<EventT>(event)`

The central trick is using `std::type_index(typeid(EventT))` as a map key. Each event type gets its own bucket in the map, and handlers are stored as `std::any` wrapping a `std::function`. When you publish, you cast back to the right `std::function` specialization.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <functional>
#include <typeindex>
#include <any>

class EventBus {
    // Map from event type -> list of type-erased handlers
    std::unordered_map<std::type_index, std::vector<std::any>> handlers_;

public:
    // Subscribe a handler for event type EventT
    template <typename EventT>
    void subscribe(std::function<void(const EventT&)> handler) {
        auto key = std::type_index(typeid(EventT));
        handlers_[key].push_back(std::move(handler));
    }

    // Convenience: accept lambdas directly
    template <typename EventT, typename F>
    void on(F&& f) {
        subscribe<EventT>(std::function<void(const EventT&)>(std::forward<F>(f)));
    }

    // Publish an event to all subscribed handlers
    template <typename EventT>
    void publish(const EventT& event) const {
        auto key = std::type_index(typeid(EventT));
        auto it = handlers_.find(key);
        if (it == handlers_.end()) return;

        for (const auto& any_handler : it->second) {
            // Cast back to the concrete handler type
            const auto& handler =
                std::any_cast<const std::function<void(const EventT&)>&>(any_handler);
            handler(event);
        }
    }

    // Check subscriber count for a type
    template <typename EventT>
    std::size_t subscriber_count() const {
        auto key = std::type_index(typeid(EventT));
        auto it = handlers_.find(key);
        return it != handlers_.end() ? it->second.size() : 0;
    }
};

// === Event types - just plain structs, no base class needed ===
struct MouseClick {
    int x, y;
    std::string button;
};

struct KeyPress {
    char key;
    bool shift;
};

struct WindowResize {
    int width, height;
};

int main() {
    EventBus bus;

    // Subscribe handlers
    bus.on<MouseClick>([](const MouseClick& e) {
        std::cout << "Click handler 1: " << e.button
                  << " at (" << e.x << "," << e.y << ")\n";
    });

    bus.on<MouseClick>([](const MouseClick& e) {
        std::cout << "Click handler 2: logging click at (" << e.x << "," << e.y << ")\n";
    });

    bus.on<KeyPress>([](const KeyPress& e) {
        std::cout << "Key handler: '" << e.key << "'"
                  << (e.shift ? " +Shift" : "") << "\n";
    });

    bus.on<WindowResize>([](const WindowResize& e) {
        std::cout << "Resize handler: " << e.width << "x" << e.height << "\n";
    });

    // Publish events
    std::cout << "=== Publishing MouseClick ===\n";
    bus.publish(MouseClick{100, 200, "left"});

    std::cout << "\n=== Publishing KeyPress ===\n";
    bus.publish(KeyPress{'A', true});

    std::cout << "\n=== Publishing WindowResize ===\n";
    bus.publish(WindowResize{1920, 1080});

    std::cout << "\nMouse subscribers: " << bus.subscriber_count<MouseClick>() << "\n";
    std::cout << "Key subscribers:   " << bus.subscriber_count<KeyPress>() << "\n";

    return 0;
}
```

Notice that event types are plain structs - they don't inherit from anything. The bus works with any type at all, because `typeid` gives you a unique identity for every type.

**Expected output:**

```text
=== Publishing MouseClick ===
Click handler 1: left at (100,200)
Click handler 2: logging click at (100,200)

=== Publishing KeyPress ===
Key handler: 'A' +Shift

=== Publishing WindowResize ===
Resize handler: 1920x1080

Mouse subscribers: 2
Key subscribers:   1
```

### Q2: Use `std::type_index` to dispatch published events to the correct handler list

`std::type_index` deserves a closer look. It wraps a `std::type_info` reference and adds two things the raw `type_info` lacks: value semantics (copyable, assignable) and `std::hash` support so it can be used as a key in `unordered_map`. That makes it the natural choice for a type-keyed dispatch table.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <functional>
#include <typeindex>
#include <any>

// === Demonstrating std::type_index as a dispatch key ===

class TypedEventBus {
    // std::type_index provides:
    // - operator== for comparison
    // - std::hash<std::type_index> for use in unordered containers
    // - Generated from typeid(T) at compile/runtime
    using HandlerList = std::vector<std::any>;
    std::unordered_map<std::type_index, HandlerList> dispatch_table_;

public:
    template <typename E>
    void subscribe(std::function<void(const E&)> h) {
        // typeid(E) -> std::type_info& -> std::type_index (hashable key)
        auto key = std::type_index(typeid(E));
        dispatch_table_[key].push_back(std::move(h));
    }

    template <typename E>
    void publish(const E& event) const {
        auto key = std::type_index(typeid(E));
        auto it = dispatch_table_.find(key);  // O(1) lookup by type
        if (it == dispatch_table_.end()) {
            std::cout << "  No handlers for type: " << typeid(E).name() << "\n";
            return;
        }
        std::cout << "  Dispatching to " << it->second.size()
                  << " handler(s) for type_index: " << key.name() << "\n";
        for (const auto& h : it->second) {
            std::any_cast<const std::function<void(const E&)>&>(h)(event);
        }
    }

    void dump_registry() const {
        std::cout << "=== Dispatch Table ===\n";
        for (const auto& [ti, handlers] : dispatch_table_) {
            std::cout << "  " << ti.name() << " -> " << handlers.size() << " handlers\n";
        }
    }
};

struct LogEvent { std::string message; int level; };
struct MetricEvent { std::string name; double value; };
struct AlertEvent { std::string description; };

int main() {
    TypedEventBus bus;

    bus.subscribe<LogEvent>([](const LogEvent& e) {
        std::cout << "    [LOG] level=" << e.level << ": " << e.message << "\n";
    });
    bus.subscribe<MetricEvent>([](const MetricEvent& e) {
        std::cout << "    [METRIC] " << e.name << "=" << e.value << "\n";
    });
    bus.subscribe<AlertEvent>([](const AlertEvent& e) {
        std::cout << "    [ALERT] " << e.description << "\n";
    });

    bus.dump_registry();

    std::cout << "\n--- Publishing LogEvent ---\n";
    bus.publish(LogEvent{"Server started", 1});

    std::cout << "\n--- Publishing MetricEvent ---\n";
    bus.publish(MetricEvent{"cpu_usage", 75.3});

    // Publishing an unregistered type:
    struct UnknownEvent {};
    std::cout << "\n--- Publishing UnknownEvent ---\n";
    bus.publish(UnknownEvent{});

    return 0;
}
```

### Q3: Show that handlers are called in subscription order and that type safety is enforced at compile time

Two things to verify here. First, handlers fire in the order they were subscribed, because a `std::vector` preserves insertion order. Second, the type template parameter prevents you from accidentally wiring a `PaymentEvent` handler to an `OrderEvent` subscription - the wrong signature simply won't compile.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <functional>
#include <typeindex>
#include <any>

class OrderedEventBus {
    std::unordered_map<std::type_index, std::vector<std::any>> handlers_;

public:
    template <typename E, typename F>
    void on(F&& f) {
        handlers_[std::type_index(typeid(E))].push_back(
            std::function<void(const E&)>(std::forward<F>(f)));
    }

    template <typename E>
    void publish(const E& event) const {
        auto it = handlers_.find(std::type_index(typeid(E)));
        if (it == handlers_.end()) return;
        // Handlers are stored in a vector - iteration preserves insertion order
        for (const auto& h : it->second) {
            std::any_cast<const std::function<void(const E&)>&>(h)(event);
        }
    }
};

struct OrderEvent { int id; };
struct PaymentEvent { double amount; };

int main() {
    OrderedEventBus bus;

    // Subscribe in a specific order
    bus.on<OrderEvent>([](const OrderEvent& e) {
        std::cout << "  [1st] Validate order #" << e.id << "\n";
    });
    bus.on<OrderEvent>([](const OrderEvent& e) {
        std::cout << "  [2nd] Log order #" << e.id << "\n";
    });
    bus.on<OrderEvent>([](const OrderEvent& e) {
        std::cout << "  [3rd] Notify warehouse about order #" << e.id << "\n";
    });

    bus.on<PaymentEvent>([](const PaymentEvent& e) {
        std::cout << "  [1st] Process payment $" << e.amount << "\n";
    });
    bus.on<PaymentEvent>([](const PaymentEvent& e) {
        std::cout << "  [2nd] Send receipt for $" << e.amount << "\n";
    });

    std::cout << "=== Publishing OrderEvent - handlers in subscription order ===\n";
    bus.publish(OrderEvent{42});

    std::cout << "\n=== Publishing PaymentEvent - handlers in subscription order ===\n";
    bus.publish(PaymentEvent{99.95});

    // === Type safety ===
    // The template parameter ensures you can only subscribe with the correct signature:
    //   bus.on<OrderEvent>([](const PaymentEvent& e) { });  // BAD: Won't compile!
    //   - std::function<void(const OrderEvent&)> can't be constructed from
    //     a lambda taking const PaymentEvent&

    // Publishing an OrderEvent never invokes PaymentEvent handlers and vice versa:
    std::cout << "\n=== Type isolation: PaymentEvent doesn't trigger OrderEvent handlers ===\n";
    bus.publish(PaymentEvent{50.00});
    // Only payment handlers fire - order handlers are never called

    std::cout << "\nType safety is enforced:\n";
    std::cout << "  - subscribe<E> only accepts handlers matching void(const E&)\n";
    std::cout << "  - publish<E> only invokes handlers registered for type E\n";
    std::cout << "  - Mismatched types fail at compile time, not runtime\n";

    return 0;
}
```

**Expected output:**

```text
=== Publishing OrderEvent - handlers in subscription order ===
  [1st] Validate order #42
  [2nd] Log order #42
  [3rd] Notify warehouse about order #42

=== Publishing PaymentEvent - handlers in subscription order ===
  [1st] Process payment $99.95
  [2nd] Send receipt for $99.95

=== Type isolation: PaymentEvent doesn't trigger OrderEvent handlers ===
  [1st] Process payment $50
  [2nd] Send receipt for $50

Type safety is enforced:
  - subscribe<E> only accepts handlers matching void(const E&)
  - publish<E> only invokes handlers registered for type E
  - Mismatched types fail at compile time, not runtime
```

---

## Notes

- `std::type_index` wraps `std::type_info` and provides hashing - ideal as `unordered_map` key.
- `std::any` provides type erasure to store different `std::function` specializations in one container.
- `std::any_cast` throws `std::bad_any_cast` on type mismatch - but our design ensures correct types.
- Subscription order is preserved because `std::vector` maintains insertion order.
- For thread-safe event buses, protect `handlers_` with `std::shared_mutex` (read-write lock).
- Consider `std::move_only_function` (C++23) if you need non-copyable handlers.
