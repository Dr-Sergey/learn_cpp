# Apply the Observer pattern with type-safe signal/slot using templates

**Category:** Modern OOP Patterns  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/variadic_template>  

---

## Topic Overview

The Observer pattern decouples event producers from consumers. A modern C++ implementation uses **variadic templates** to create a type-safe `Signal<Args...>` that can connect to any callable matching the signature - lambdas, function pointers, bound member functions - without base classes or runtime type checks.

The key insight is that the signal's template parameters lock in the argument types at the point where you declare it. There's no casting, no `void*`, and no way to accidentally connect a slot with the wrong signature.

### Architecture

Here's the basic shape of the system. A signal holds a list of slots (stored as `std::function`), and calling `emit` fires all of them with the same arguments:

```cpp
Signal<int, std::string>
  ┌──────────────────────────────────────┐
  │ slots_: vector<function<void(int,string)>> │
  │                                      │
  │ connect(callable) -> id               │
  │ disconnect(id)                       │
  │ emit(42, "hello") ────┬──> slot[0](42,"hello") │
  │                       ├──> slot[1](42,"hello") │
  │                       └──> slot[2](42,"hello") │
  └──────────────────────────────────────┘

Type safety: Signal<int,string> ONLY accepts functions matching void(int,string)
```

### Why Templates Over Virtual Dispatch

| Approach | Pros | Cons |
| --- | --- | --- |
| Virtual `IObserver::update()` | Simple, familiar | Type-unsafe (void*), heavyweight |
| `std::function<void(Args...)>` | Type-safe, flexible | Slight overhead from type erasure |
| Template callbacks | Zero overhead, inlinable | Harder to store heterogeneous slots |

---

## Self-Assessment

### Q1: Implement a `Signal<Args...>` that notifies all connected functions with typed arguments

Notice how `SlotId` gives every connection a unique handle. Without it you'd have no way to disconnect a specific slot later.

**Solution - Type-Safe Signal:**

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
    struct Slot {
        SlotId id;
        SlotType callback;
    };
    std::vector<Slot> slots_;
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
                           [id](const Slot& s) { return s.id == id; }),
            slots_.end()
        );
    }

    // Emit: notify all connected slots
    void emit(Args... args) const {
        for (const auto& slot : slots_) {
            slot.callback(args...);
        }
    }

    // Shorthand: signal(args...) same as signal.emit(args...)
    void operator()(Args... args) const { emit(args...); }

    std::size_t slot_count() const { return slots_.size(); }
};

// Example: UI event system
class Button {
public:
    Signal<> clicked;               // no args
    Signal<int, int> mouse_moved;   // x, y
    Signal<const std::string&> text_changed; // new text

    void simulate_click() {
        std::cout << "[Button] clicked\n";
        clicked.emit();
    }

    void simulate_move(int x, int y) {
        mouse_moved.emit(x, y);
    }
};

int main() {
    Button btn;

    // Connect lambda
    auto id1 = btn.clicked.connect([] {
        std::cout << "  Handler 1: button was clicked!\n";
    });

    // Connect another lambda
    btn.clicked.connect([] {
        std::cout << "  Handler 2: logging click event\n";
    });

    // Connect to mouse_moved with typed arguments
    btn.mouse_moved.connect([](int x, int y) {
        std::cout << "  Mouse at (" << x << ", " << y << ")\n";
    });

    btn.simulate_click();
    btn.simulate_move(100, 200);

    // Disconnect first handler
    btn.clicked.disconnect(id1);
    std::cout << "\nAfter disconnecting handler 1:\n";
    btn.simulate_click();
}
// Expected output:
//   [Button] clicked
//     Handler 1: button was clicked!
//     Handler 2: logging click event
//     Mouse at (100, 200)
//
//   After disconnecting handler 1:
//   [Button] clicked
//     Handler 2: logging click event
```

After the disconnect, only Handler 2 fires. The `id1` returned by `connect` is the key - store it whenever you need the ability to unsubscribe.

---

### Q2: Show automatic disconnection using `weak_ptr`-based slot guards

Manual disconnect by ID works, but it requires the observer to hold onto the ID and remember to call `disconnect`. A cleaner approach ties slot lifetime to object lifetime: when the observer is destroyed, its slots vanish on the next `emit` automatically.

**Solution - RAII-Based Auto-Disconnect:**

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <memory>
#include <algorithm>
#include <string>

template <typename... Args>
class SafeSignal {
    using SlotType = std::function<void(Args...)>;

    struct SlotEntry {
        std::weak_ptr<void> guard;  // weak_ptr to track slot lifetime
        SlotType callback;
    };

    std::vector<SlotEntry> slots_;

public:
    // Connect with a lifetime guard - when guard expires, slot auto-disconnects
    void connect(std::shared_ptr<void> guard, SlotType slot) {
        slots_.push_back({guard, std::move(slot)});
    }

    void emit(Args... args) {
        // Remove expired slots, then call remaining
        slots_.erase(
            std::remove_if(slots_.begin(), slots_.end(),
                           [](const SlotEntry& s) { return s.guard.expired(); }),
            slots_.end()
        );

        for (const auto& slot : slots_) {
            if (auto locked = slot.guard.lock()) {
                slot.callback(args...);
            }
        }
    }

    std::size_t slot_count() const { return slots_.size(); }
};

// Connection guard - destructor triggers auto-disconnect on next emit
class Connection {
    std::shared_ptr<void> guard_;
public:
    Connection() : guard_(std::make_shared<int>(0)) {}
    std::shared_ptr<void> guard() const { return guard_; }
    // When Connection is destroyed, guard's refcount drops -> weak_ptr expires
};

class Logger {
    Connection conn_;  // destroyed with Logger -> auto-disconnects
public:
    void subscribe(SafeSignal<const std::string&>& signal) {
        signal.connect(conn_.guard(), [this](const std::string& msg) {
            std::cout << "  [Logger] " << msg << "\n";
        });
    }
};

int main() {
    SafeSignal<const std::string&> on_message;

    {
        Logger logger;
        logger.subscribe(on_message);

        on_message.emit("Hello");   // Logger receives this
        std::cout << "Slots before: " << on_message.slot_count() << "\n";
    }
    // logger destroyed here -> Connection destroyed -> guard expired

    on_message.emit("World");  // Logger does NOT receive this (auto-disconnected)
    std::cout << "Slots after cleanup: " << on_message.slot_count() << "\n";
}
// Expected output:
//   [Logger] Hello
//   Slots before: 1
//   Slots after cleanup: 0
```

The `weak_ptr` guard is the elegance here. The signal holds only a `weak_ptr`, so destroying the `Logger` (and its `Connection` member) is enough to invalidate it. No manual `disconnect` needed, no dangling function pointers.

---

### Q3: Compare with Qt signals/slots and Boost.Signals2 for lifecycle management

It's worth knowing what the established libraries look like so you understand the trade-offs when you choose to roll your own.

**Comparison Table:**

| Feature | Hand-rolled `Signal<Args...>` | Qt Signals/Slots | Boost.Signals2 |
| --- | --- | --- | --- |
| **Type safety** | Compile-time (templates) | Runtime (MOC generates glue code) | Compile-time |
| **Connection lifetime** | Manual or weak_ptr guard | `QObject` parent-child auto-disconnect | `scoped_connection` RAII |
| **Thread safety** | Not built-in (add mutex) | Auto queued connections across threads | Mutex-based by default |
| **Signature match** | Compile error on mismatch | Runtime warning | Compile error |
| **Dependencies** | None (STL only) | Qt framework (MOC, vtables) | Boost headers |
| **Overhead** | `std::function` copy | Virtual dispatch + string lookup | `shared_ptr` overhead |
| **Slot types** | Any callable | Declared slots (macros) | Any callable |
| **Disconnect** | By ID or RAII guard | `disconnect()` or `~QObject` | `connection::disconnect()` |

```cpp
// Qt style (requires MOC):
// class Button : public QObject {
//     Q_OBJECT
// signals:
//     void clicked();
// };
// connect(&button, &Button::clicked, &logger, &Logger::handle);

// Boost.Signals2 style:
// boost::signals2::signal<void(int)> sig;
// auto conn = sig.connect([](int x) { ... });
// conn.disconnect();  // or scoped_connection for RAII

// Our hand-rolled style:
// Signal<int> sig;
// auto id = sig.connect([](int x) { ... });
// sig.disconnect(id);  // manual
```

**When to Use Which:**

```cpp
Hand-rolled Signal<Args...>:
  // Small projects, no external dependencies
  // Learning exercise
  // When you need minimal overhead

Qt Signals/Slots:
  // Qt applications (already using the framework)
  // Cross-thread communication (auto-queued)
  // Rich tooling (Qt Designer, signal spy)

Boost.Signals2:
  // Non-Qt projects needing mature signal library
  // Thread-safe by default
  // Combiner support (aggregate return values)
```

---

## Notes

- **`std::function` overhead:** ~32-64 bytes per slot due to type erasure. For tight inner loops, consider templated callbacks instead.
- **Reentrancy:** If a slot modifies the signal (connects/disconnects) during emission, iterating `slots_` may invalidate iterators. Solution: copy the slot list before emitting, or use a deferred modification queue.
- **Return values:** Signals typically return `void`. For return value aggregation, see Boost.Signals2's "combiners".
- **Thread safety:** The basic `Signal` above is NOT thread-safe. Protect `connect`/`disconnect`/`emit` with a `std::mutex` for multi-threaded use.
- **C++20 improvement:** Use `std::erase_if` instead of the erase-remove idiom for cleaner code.
