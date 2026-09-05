# Implement the Observer pattern with modern C++ (signals and slots)

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

The **Observer pattern** is the fundamental tool for decoupling event sources (things that emit events) from event handlers (things that react to them). If you've used Qt or Boost.Signals2, you've already used it. The question in modern C++ isn't whether to use the pattern, but which mechanism to use to wire the two sides together.

Here's how the main approaches compare:

| Approach | Pros | Cons |
| --- | --- | --- |
| Virtual interface | Type-safe, clear contract | Coupling, inheritance required |
| `std::function` slots | Flexible, any callable | Lifetime management needed |
| Signal/slot library | Auto-disconnect, thread-safe | External dependency |
| `std::weak_ptr` observers | Auto-cleanup on destroy | Shared ownership required |

The biggest practical headache with observer systems is lifetime management: what happens when an observer is destroyed but is still registered? A dangling callback pointer is a crash waiting to happen. The modern C++ solutions below deal with that problem directly, using reference counting and weak pointers to make stale callbacks fail silently rather than catastrophically.

---

## Self-Assessment

### Q1: Implement a modern signal/slot system with auto-disconnect

The trick here is giving each subscription a `Connection` object - a lightweight handle that contains a `shared_ptr<bool>` flag. When the slot is fired, it checks whether the corresponding `Connection` is still alive via a `weak_ptr`. If the connection was disconnected (or the `Connection` object was destroyed), the slot is silently skipped and cleaned up. This way you never have to manually remember to unregister - just let the `Connection` go out of scope.

**Answer:**

```cpp
#include <functional>
#include <vector>
#include <algorithm>
#include <memory>
#include <iostream>

// Connection handle for disconnect
class Connection {
    std::shared_ptr<bool> alive_;
public:
    Connection() : alive_(std::make_shared<bool>(true)) {}
    void disconnect() { *alive_ = false; }
    bool connected() const { return *alive_; }
    std::weak_ptr<bool> token() const { return alive_; }
};

// Type-safe signal with auto-cleanup
template<typename... Args>
class Signal {
    struct Slot {
        std::function<void(Args...)> callback;
        std::weak_ptr<bool> token;
    };
    std::vector<Slot> slots_;

public:
    Connection connect(std::function<void(Args...)> cb) {
        Connection conn;
        slots_.push_back({std::move(cb), conn.token()});
        return conn;
    }

    void emit(Args... args) {
        // Remove dead connections and invoke live ones
        slots_.erase(
            std::remove_if(slots_.begin(), slots_.end(),
                [](const Slot& s) { return s.token.expired(); }),
            slots_.end());

        for (auto& slot : slots_) {
            if (auto lock = slot.token.lock()) {
                slot.callback(args...);
            }
        }
    }

    size_t slot_count() const { return slots_.size(); }
};

// Example: button click system
class Button {
public:
    Signal<int, int> on_click;      // x, y
    Signal<> on_hover;

    void click(int x, int y) { on_click.emit(x, y); }
    void hover() { on_hover.emit(); }
};

int main() {
    Button btn;
    auto c1 = btn.on_click.connect([](int x, int y) {
        std::cout << "Clicked at (" << x << ", " << y << ")\n";
    });

    auto c2 = btn.on_click.connect([](int x, int y) {
        std::cout << "Logger: click (" << x << ", " << y << ")\n";
    });

    btn.click(10, 20);  // Both handlers fire

    c1.disconnect();     // Remove first handler
    btn.click(30, 40);   // Only logger fires
    return 0;
}
```

Notice that `emit()` does two things in one pass: it erases expired slots (those whose `Connection` object was destroyed or explicitly disconnected) and then fires the survivors. This keeps the slot list from growing without bound if connections are frequently created and dropped, and it means you never need a separate "garbage collection" pass.

### Q2: Show an observer that auto-unregisters when destroyed

The pattern here is different from Q1: instead of an explicit `Connection` handle, the observer passes a `shared_ptr` to itself as a "lifetime token." The `Event` stores a `weak_ptr` to that token. When the observer object is destroyed (its `shared_ptr` goes to zero), the `weak_ptr` expires, and the next notification call silently removes the dead observer. This is particularly useful when the observer's lifetime is already managed by a `shared_ptr` and you just want the event subscription to track that lifetime automatically.

**Answer:**

```cpp
#include <functional>
#include <vector>
#include <memory>
#include <iostream>
#include <unordered_map>

template<typename... Args>
class Event {
    struct Observer {
        std::weak_ptr<void> lifetime;  // Observer's lifetime token
        std::function<void(Args...)> handler;
    };
    std::vector<Observer> observers_;
public:
    // Observer passes shared_ptr to itself as lifetime token
    template<typename T>
    void subscribe(std::shared_ptr<T> observer,
                   std::function<void(Args...)> handler) {
        observers_.push_back({observer, std::move(handler)});
    }

    void notify(Args... args) {
        // Auto-remove dead observers
        observers_.erase(
            std::remove_if(observers_.begin(), observers_.end(),
                [](const Observer& o) { return o.lifetime.expired(); }),
            observers_.end());
        for (auto& o : observers_)
            o.handler(args...);
    }
};

// Model with change notification
class DataModel {
public:
    Event<const std::string&, int> on_change;  // key, new_value

    void set(const std::string& key, int value) {
        data_[key] = value;
        on_change.notify(key, value);
    }
private:
    std::unordered_map<std::string, int> data_;
};

class DashboardWidget : public std::enable_shared_from_this<DashboardWidget> {
public:
    void bind(DataModel& model) {
        model.on_change.subscribe(
            shared_from_this(),
            [this](const std::string& key, int val) {
                std::cout << "Dashboard: " << key << " = " << val << "\n";
            });
    }
};

int main() {
    DataModel model;
    {
        auto widget = std::make_shared<DashboardWidget>();
        widget->bind(model);
        model.set("temperature", 42);  // Widget notified
    }
    // widget destroyed - auto-unregistered
    model.set("temperature", 50);  // No crash, no notification
    return 0;
}
```

The reason `DashboardWidget` inherits from `enable_shared_from_this` is so it can call `shared_from_this()` inside `bind()`. That call returns a `shared_ptr` to the same control block as the outer `make_shared`, so the lifetime token genuinely tracks the widget's lifetime. If you called `shared_ptr<DashboardWidget>(this)` instead, you'd create a separate control block and the lifetime tracking would be broken - the weak pointer would expire immediately. That's a common mistake when people first encounter this pattern.

### Q3: Build a thread-safe observable property

When multiple threads read and write the same observable value, you need to protect both the value and the handler list with a mutex. The classic mistake is calling observer callbacks while holding the lock - if a callback tries to set the property again (re-entry), you deadlock. The solution is to copy the handlers under the lock, release the lock, then notify. It's a subtle but critical point: the lock scope and the notification scope must be separate.

**Answer:**

```cpp
#include <functional>
#include <vector>
#include <mutex>
#include <shared_mutex>
#include <iostream>

template<typename T>
class ObservableProperty {
public:
    using ChangeHandler = std::function<void(const T& old_val, const T& new_val)>;

    explicit ObservableProperty(T initial)
        : value_(std::move(initial)) {}

    // Thread-safe read
    T get() const {
        std::shared_lock lock(mtx_);
        return value_;
    }

    // Thread-safe write with notification
    void set(T new_val) {
        std::vector<ChangeHandler> handlers_copy;
        T old_val;
        {
            std::unique_lock lock(mtx_);
            if (value_ == new_val) return;  // No change
            old_val = std::move(value_);
            value_ = std::move(new_val);
            handlers_copy = handlers_;  // Copy handlers under lock
        }
        // Notify OUTSIDE lock to prevent deadlocks
        for (auto& h : handlers_copy)
            h(old_val, value_);
    }

    void on_change(ChangeHandler handler) {
        std::unique_lock lock(mtx_);
        handlers_.push_back(std::move(handler));
    }

private:
    T value_;
    std::vector<ChangeHandler> handlers_;
    mutable std::shared_mutex mtx_;
};

int main() {
    ObservableProperty<int> counter(0);
    counter.on_change([](int old_v, int new_v) {
        std::cout << old_v << " -> " << new_v << "\n";
    });

    counter.set(1);   // 0 -> 1
    counter.set(5);   // 1 -> 5
    counter.set(5);   // No output (same value)
    return 0;
}
```

The `std::shared_mutex` allows multiple concurrent readers via `shared_lock` but gives exclusive access for writes via `unique_lock`. The crucial detail is that `handlers_copy = handlers_` happens inside the locked section, but the actual callback calls happen after the lock is released. This makes re-entrant callbacks safe - a handler that calls `set()` again will acquire the lock normally instead of deadlocking. It's slightly more expensive than calling handlers inside the lock, but the correctness guarantee is worth it.

---

## Notes

- Use `std::weak_ptr` for automatic observer cleanup - no manual unregister call needed, which removes an entire class of use-after-free bugs.
- Always notify outside the lock to prevent deadlocks - an observer callback might itself call into the subject, and if the lock is still held you'll deadlock.
- `Connection` objects give callers explicit control over subscription lifetime - useful when the observer and the subject have independent lifetimes not managed by `shared_ptr`.
- For production use, consider Boost.Signals2, which is thread-safe and handles connection management robustly.
- Prefer signals/slots over virtual observer interfaces - they're more flexible (any callable works, not just objects that inherit from a specific base) and less coupling.
- Be careful with lambda captures: if you capture `this` in a handler and the owning object is destroyed while still subscribed, you have a dangling pointer. The `weak_ptr` lifetime-token pattern in Q2 is the safeguard.
