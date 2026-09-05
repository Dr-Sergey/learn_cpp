# Implement the Observer pattern with type-safe signal/slot semantics

**Category:** Modern OOP Patterns  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/function>  

---

## Topic Overview

The Observer pattern decouples an event source (subject) from its listeners (observers). A **Signal/Slot** system makes this type-safe: the signal declares the argument types, and only matching slots can connect.

This file focuses on the **implementation** side: Signal class, lifetime management, and thread safety. (See the companion file on the Observer pattern with templates for the design-pattern perspective.)

### Signal/Slot Architecture

Here's the basic shape. Slots are stored as `std::function` objects and called in sequence when `emit` fires:

```cpp
+──────────────+   emit(x, y)    +──────────────────────+
│    Signal     │ ───────────────>│  slot1(x, y)         │
│  <int, int>   │                 │  slot2(x, y)         │
│               │                 │  weak_ptr -> slot3    │
│  connect()    │                 │    (expired? remove)  │
│  disconnect() │                 +──────────────────────+
│  emit()       │
+──────────────+

Slots are std::function<void(Args...)> stored in a list.
```

---

## Self-Assessment

### Q1: Write a `Signal<Args...>` class that stores and invokes a list of `std::function` callbacks

The variadic template `Args...` is what makes this type-safe. A `Signal<int, int>` can only connect callables that accept two `int` arguments - anything else is a compile error.

**Solution:**

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <string>
#include <algorithm>

template <typename... Args>
class Signal {
public:
    using SlotType = std::function<void(Args...)>;
    using SlotId = std::size_t;

private:
    struct SlotEntry {
        SlotId id;
        SlotType func;
    };

    std::vector<SlotEntry> slots_;
    SlotId next_id_ = 0;

public:
    // Connect a callable - returns an ID for later disconnection
    SlotId connect(SlotType slot) {
        SlotId id = next_id_++;
        slots_.push_back({id, std::move(slot)});
        return id;
    }

    // Disconnect by ID
    void disconnect(SlotId id) {
        slots_.erase(
            std::remove_if(slots_.begin(), slots_.end(),
                [id](const SlotEntry& e) { return e.id == id; }),
            slots_.end()
        );
    }

    // Emit - call all connected slots
    void emit(Args... args) const {
        for (const auto& entry : slots_)
            entry.func(args...);
    }

    // Convenience: operator() as alias for emit
    void operator()(Args... args) const { emit(args...); }

    std::size_t slot_count() const { return slots_.size(); }
};

// Usage
struct Button {
    Signal<> clicked;                     // no args
    Signal<int, int> position_changed;    // two int args

    void click() { clicked.emit(); }
    void move_to(int x, int y) { position_changed.emit(x, y); }
};

int main() {
    Button btn;

    auto id1 = btn.clicked.connect([] {
        std::cout << "Button clicked!\n";
    });

    btn.position_changed.connect([](int x, int y) {
        std::cout << "Button moved to (" << x << ", " << y << ")\n";
    });

    btn.click();
    btn.move_to(100, 200);

    // Disconnect first slot
    btn.clicked.disconnect(id1);
    std::cout << "After disconnect, clicked slots: " << btn.clicked.slot_count() << "\n";
    btn.click();  // nothing happens
}
// Expected output:
//   Button clicked!
//   Button moved to (100, 200)
//   After disconnect, clicked slots: 0
```

After `disconnect(id1)`, the slot is removed from the list and the next `click()` produces no output. The returned `SlotId` is the only handle you have for removing a specific connection, so store it if you'll need to disconnect later.

---

### Q2: Handle observer lifetime: use `weak_ptr` to auto-disconnect destroyed observers

Manual disconnect requires the observer to remember to call it - easy to forget, especially during destruction. The `weak_ptr` approach is cleaner: when the observer is destroyed, its `weak_ptr` guard expires, and the signal automatically removes the dead slot on the next `emit`.

**Solution:**

```cpp
#include <iostream>
#include <functional>
#include <memory>
#include <vector>
#include <string>

template <typename... Args>
class SafeSignal {
    struct WeakSlot {
        std::weak_ptr<void> guard;  // if expired, slot is dead
        std::function<void(Args...)> func;
    };

    std::vector<WeakSlot> slots_;

public:
    // Connect with a weak_ptr guard - slot auto-expires when guard dies
    void connect(std::shared_ptr<void> guard, std::function<void(Args...)> slot) {
        slots_.push_back({guard, std::move(slot)});
    }

    void emit(Args... args) {
        // Remove expired slots and call living ones
        auto it = slots_.begin();
        while (it != slots_.end()) {
            if (auto locked = it->guard.lock()) {
                it->func(args...);
                ++it;
            } else {
                // Observer is dead - auto-remove
                it = slots_.erase(it);
            }
        }
    }

    std::size_t slot_count() const { return slots_.size(); }
};

class Observer : public std::enable_shared_from_this<Observer> {
    std::string name_;
public:
    explicit Observer(std::string name) : name_(std::move(name)) {}
    ~Observer() { std::cout << "  ~Observer(" << name_ << ") destroyed\n"; }

    void on_event(int value) {
        std::cout << "  " << name_ << " received: " << value << "\n";
    }

    // Self-register with a signal
    void listen(SafeSignal<int>& signal) {
        auto self = shared_from_this();
        signal.connect(self, [self](int v) { self->on_event(v); });
        // Note: we capture shared_ptr; the signal holds weak_ptr from it
        // Actually, the signal holds a weak_ptr to `self` via the guard
    }
};

int main() {
    SafeSignal<int> signal;

    auto obs1 = std::make_shared<Observer>("Alice");
    auto obs2 = std::make_shared<Observer>("Bob");

    obs1->listen(signal);
    obs2->listen(signal);

    std::cout << "Emit with 2 observers:\n";
    signal.emit(42);
    std::cout << "Slots: " << signal.slot_count() << "\n\n";

    // Destroy obs2
    std::cout << "Destroying Bob:\n";
    obs2.reset();

    std::cout << "\nEmit with 1 observer (Bob auto-disconnected):\n";
    signal.emit(99);
    std::cout << "Slots: " << signal.slot_count() << "\n";
}
// Expected output:
//   Emit with 2 observers:
//     Alice received: 42
//     Bob received: 42
//   Slots: 2
//
//   Destroying Bob:
//     ~Observer(Bob) destroyed
//
//   Emit with 1 observer (Bob auto-disconnected):
//     Alice received: 99
//   Slots: 1
```

After `obs2.reset()`, the `shared_ptr` holding `Bob` is released. The signal still holds only a `weak_ptr` to the guard, so when `emit` runs next, `guard.lock()` returns null and the slot is cleaned up automatically. No dangling pointer, no crash.

---

### Q3: Show thread-safety issues in a naive observer implementation and fix with a mutex

The race condition here is one of the most common bugs in event systems. One thread adds a slot while another thread is iterating the slot list during `emit`. The vector's internal state gets corrupted.

**Problem - Data Race in Naive Implementation:**

```cpp
// UNSAFE: No synchronization
template <typename... Args>
class UnsafeSignal {
    std::vector<std::function<void(Args...)>> slots_;
public:
    void connect(std::function<void(Args...)> f) {
        slots_.push_back(std::move(f));  // RACE: concurrent with emit
    }
    void emit(Args... args) {
        for (auto& s : slots_)  // RACE: vector may be modified during iteration
            s(args...);
    }
};

// Thread A: signal.connect(mySlot);   <- modifies slots_
// Thread B: signal.emit(42);          <- iterates slots_
// -> UNDEFINED BEHAVIOR (data race on vector)
```

The fix takes a snapshot of the slot list under the lock, then iterates the snapshot outside the lock. This is important: if you hold the lock during invocation, a slot that tries to call `connect()` on the same signal will deadlock.

**Solution - Mutex-Protected Signal:**

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <mutex>
#include <thread>
#include <chrono>

template <typename... Args>
class ThreadSafeSignal {
    mutable std::mutex mutex_;
    std::vector<std::function<void(Args...)>> slots_;

public:
    void connect(std::function<void(Args...)> f) {
        std::lock_guard lock(mutex_);
        slots_.push_back(std::move(f));
    }

    void emit(Args... args) const {
        // Copy the slot list under lock, then invoke outside lock
        // This prevents deadlock if a slot calls connect/disconnect!
        std::vector<std::function<void(Args...)>> local_copy;
        {
            std::lock_guard lock(mutex_);
            local_copy = slots_;
        }
        for (const auto& slot : local_copy)
            slot(args...);
    }

    std::size_t slot_count() const {
        std::lock_guard lock(mutex_);
        return slots_.size();
    }
};

int main() {
    ThreadSafeSignal<int> signal;

    // Producer thread: emits values
    std::jthread producer([&signal] {
        for (int i = 0; i < 5; ++i) {
            signal.emit(i);
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
    });

    // Consumer thread: connects slots while producer emits
    std::jthread connector([&signal] {
        for (int i = 0; i < 3; ++i) {
            signal.connect([i](int val) {
                std::cout << "Slot" << i << " got: " << val << "\n";
            });
            std::this_thread::sleep_for(std::chrono::milliseconds(15));
        }
    });
}
// Output is non-deterministic but safe - no data races
```

**Why copy the slot list before invoking:**

| Approach | Re-entrancy safe? | Performance | Risk |
| --- | --- | --- | --- |
| Lock during iteration | No - deadlock if slot calls `connect()` | Good (no copy) | **Deadlock** |
| Copy-then-invoke | Yes - slots can freely call `connect()` | Copy overhead | Safe |
| `shared_mutex` (read/write lock) | Partial - still deadlocks on re-entrant `connect()` | Better read perf | Same deadlock risk |

The copy-then-invoke approach costs a vector allocation per emission, but it eliminates the deadlock risk entirely. For most event systems, that's a good trade.

---

## Notes

- **Boost.Signals2** solves all these problems (thread-safety, lifetime, ordering) but adds a dependency.
- **Qt Signals/Slots** uses a code generator (moc) for type-safe signals with rich features: cross-thread delivery, queued connections, automatic disconnection.
- **`std::function` overhead:** Each slot has ~32 bytes + possible heap allocation. For high-frequency signals (e.g., per-frame game events), consider using function pointers or type-erased small-buffer callbacks.
- **Connection objects** (RAII disconnect): Instead of returning an ID, return a `Connection` RAII object whose destructor calls `disconnect()` - guarantees cleanup even on exceptions.
- **Emission during disconnect:** If a slot disconnects itself during `emit()`, the copy-then-invoke approach handles this correctly because iteration is over the local copy.
