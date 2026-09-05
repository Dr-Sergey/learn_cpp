# Implement type-safe event systems using std::variant and visitor dispatch

**Category:** Modern OOP Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

A **type-safe event system** uses `std::variant` to represent events as a closed set of types, and `std::visit` to dispatch handlers. Unlike virtual-dispatch event hierarchies, the compiler **guarantees** every event type is handled - adding a new event type produces compile errors everywhere a visitor is incomplete.

The key idea here is "closed set." With a virtual base class you can always add a new subclass later and the rest of the code compiles silently, potentially ignoring the new event entirely. With `std::variant` the set of event types is locked in the type definition, so the compiler can check exhaustiveness at every call site.

### Variant Event vs Virtual Event

Here's a side-by-side feel for the two approaches:

```cpp
Virtual Event Hierarchy:              Variant Event System:
┌──────────────┐                     using Event = std::variant<
│ Event (base) │                         MouseClick,
│ virtual      │                         KeyPress,
│ handle() = 0 │                         WindowResize
└──┬───────────┘                     >;
   │   │   │
   v   v   v                        std::visit(overloaded{
Mouse Key  Resize                       [](MouseClick e)   { ... },
Click Press                             [](KeyPress e)     { ... },
                                        [](WindowResize e) { ... }
Open set: new types                  }, event);
compile fine silently.
                                     Closed set: ADD a new type ->
                                     COMPILER ERROR if not handled!
```

---

## Self-Assessment

### Q1: Define `Event = std::variant<MouseClick, KeyPress, WindowResize>` and dispatch with `std::visit`

The `overloaded` helper is a small but important piece of machinery here. It takes a set of lambdas and inherits from all of them, combining their `operator()` overloads into one struct that `std::visit` can dispatch through.

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <vector>

// Event types - plain structs, no base class needed
struct MouseClick {
    int x, y;
    int button;  // 0=left, 1=right
};

struct KeyPress {
    char key;
    bool shift;
    bool ctrl;
};

struct WindowResize {
    int width, height;
};

// Closed event type
using Event = std::variant<MouseClick, KeyPress, WindowResize>;

// Helper: overloaded pattern (C++17)
template <typename... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <typename... Ts>
overloaded(Ts...) -> overloaded<Ts...>;  // deduction guide (not needed in C++20)

// Event handler using std::visit
void handle_event(const Event& event) {
    std::visit(overloaded{
        [](const MouseClick& e) {
            std::cout << "Mouse click at (" << e.x << "," << e.y
                      << ") button=" << e.button << "\n";
        },
        [](const KeyPress& e) {
            std::cout << "Key press: '" << e.key << "'"
                      << (e.shift ? " +Shift" : "")
                      << (e.ctrl ? " +Ctrl" : "") << "\n";
        },
        [](const WindowResize& e) {
            std::cout << "Window resized to " << e.width
                      << "x" << e.height << "\n";
        }
    }, event);
}

int main() {
    // Event queue
    std::vector<Event> events = {
        MouseClick{100, 200, 0},
        KeyPress{'A', true, false},
        WindowResize{1920, 1080},
        MouseClick{50, 75, 1},
        KeyPress{'s', false, true}  // Ctrl+S
    };

    std::cout << "Processing event queue:\n";
    for (const auto& event : events)
        handle_event(event);
}
// Expected output:
//   Processing event queue:
//   Mouse click at (100,200) button=0
//   Key press: 'A' +Shift
//   Window resized to 1920x1080
//   Mouse click at (50,75) button=1
//   Key press: 's' +Ctrl
```

Notice that the event structs themselves have no base class, no virtual functions, and no vtable. They're just data. All the dispatch logic lives in the visitor, not the events.

---

### Q2: Add a new event type and show that the compiler forces you to handle it in all visitors

When you add `FileDrop` to the variant, every `std::visit` call in your codebase that doesn't handle `FileDrop` immediately becomes a compile error. That's the exhaustiveness guarantee in action - the compiler is doing a job that a code review might miss.

```cpp
#include <iostream>
#include <variant>
#include <string>

struct MouseClick { int x, y; };
struct KeyPress { char key; };
struct WindowResize { int w, h; };

// NEW event type added:
struct FileDrop {
    std::string path;
    size_t size_bytes;
};

// Updated variant - FileDrop added
using Event = std::variant<MouseClick, KeyPress, WindowResize, FileDrop>;

template <typename... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <typename... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

void handle_event(const Event& event) {
    std::visit(overloaded{
        [](const MouseClick& e) {
            std::cout << "Click at (" << e.x << "," << e.y << ")\n";
        },
        [](const KeyPress& e) {
            std::cout << "Key: '" << e.key << "'\n";
        },
        [](const WindowResize& e) {
            std::cout << "Resize: " << e.w << "x" << e.h << "\n";
        },
        // If you REMOVE this line, you get a compile error:
        // "no matching function for call to 'overloaded<...>::operator()(const FileDrop&)"
        [](const FileDrop& e) {
            std::cout << "File dropped: " << e.path
                      << " (" << e.size_bytes << " bytes)\n";
        }
    }, event);
}

// Alternative: use auto catch-all for extensibility
void handle_with_default(const Event& event) {
    std::visit(overloaded{
        [](const MouseClick& e) {
            std::cout << "Click at (" << e.x << "," << e.y << ")\n";
        },
        [](const auto& e) {
            // Catch-all - handles KeyPress, WindowResize, FileDrop
            // WARNING: defeats the purpose of exhaustive checking!
            std::cout << "Unhandled event type\n";
        }
    }, event);
}

int main() {
    Event e1 = FileDrop{"/home/user/photo.jpg", 1048576};
    handle_event(e1);

    Event e2 = MouseClick{42, 99};
    handle_with_default(e2);

    handle_with_default(e1);  // falls through to catch-all
}
// Expected output:
//   File dropped: /home/user/photo.jpg (1048576 bytes)
//   Click at (42,99)
//   Unhandled event type
```

The `auto` catch-all in `handle_with_default` is a deliberate escape hatch - it silences the compiler error but brings you right back to the "silently ignores new types" problem that virtual dispatch has. Use it only when you genuinely want a fallback.

**Compiler error if handler missing (clang):**

```cpp
error: no type named 'type' in 'std::invoke_result<overloaded<...>, const FileDrop&>'
note: in instantiation of function 'std::visit<overloaded<...>, const Event&>'
```

---

### Q3: Compare this approach with a virtual `handle()` double-dispatch hierarchy

The right choice between `std::variant` and virtual dispatch depends heavily on which dimension of the problem changes more often - event types or event handlers. The table below captures the tradeoffs clearly.

| Aspect | `std::variant` + `std::visit` | Virtual Dispatch Hierarchy |
| --- | --- | --- |
| **Type set** | Closed - all types in variant | Open - add new subclass anytime |
| **Exhaustiveness** | **Compile-time enforced** | No - can forget to override |
| **Adding new event** | Modify variant -> compiler finds all missing handlers | Add subclass -> compiles even with missing handlers |
| **Adding new handler** | Add a new `visit` call (easy) | Add virtual method to base -> must update all subclasses |
| **Performance** | Jump table or if-chain (no heap, no vtable) | Virtual call (pointer chase) |
| **Memory layout** | Stack-allocated, cache-friendly | Heap-allocated polymorphic objects |
| **Serialization** | Easy - `index()` gives you the type tag | Needs RTTI or manual type ID |

This is the classic **Expression Problem**:

- **Variant:** Easy to add new operations (visitors), hard to add new types (must update variant typedef).
- **Virtual:** Easy to add new types (subclasses), hard to add new operations (must add to all classes).

The virtual double-dispatch version below shows how much more infrastructure the virtual approach requires for the same result:

```cpp
// Virtual dispatch approach - for comparison
#include <iostream>
#include <memory>
#include <vector>

class EventHandler;  // forward declare

class Event {
public:
    virtual ~Event() = default;
    virtual void accept(EventHandler& handler) const = 0;
};

class MouseClick2 : public Event {
public:
    int x, y;
    MouseClick2(int x, int y) : x(x), y(y) {}
    void accept(EventHandler& handler) const override;
};

class KeyPress2 : public Event {
public:
    char key;
    explicit KeyPress2(char k) : key(k) {}
    void accept(EventHandler& handler) const override;
};

class EventHandler {
public:
    virtual ~EventHandler() = default;
    virtual void handle(const MouseClick2& e) = 0;
    virtual void handle(const KeyPress2& e) = 0;
    // Adding FileDrop means updating THIS interface AND all implementations!
};

void MouseClick2::accept(EventHandler& h) const { h.handle(*this); }
void KeyPress2::accept(EventHandler& h) const { h.handle(*this); }

class PrintHandler : public EventHandler {
public:
    void handle(const MouseClick2& e) override {
        std::cout << "Virtual: Click at (" << e.x << "," << e.y << ")\n";
    }
    void handle(const KeyPress2& e) override {
        std::cout << "Virtual: Key '" << e.key << "'\n";
    }
};

int main() {
    std::vector<std::unique_ptr<Event>> events;
    events.push_back(std::make_unique<MouseClick2>(10, 20));
    events.push_back(std::make_unique<KeyPress2>('X'));

    PrintHandler handler;
    for (const auto& e : events)
        e->accept(handler);
}
// Expected output:
//   Virtual: Click at (10,20)
//   Virtual: Key 'X'
```

Compare how many classes and virtual functions it takes to dispatch two event types here versus the variant approach in Q1. For a closed, known-ahead-of-time set of events (like a UI input system), `std::variant` is almost always the cleaner choice.

---

## Notes

- **`overloaded` helper** becomes a standard library feature candidate - until then, define it yourself (4 lines).
- **`std::variant` size:** `sizeof(variant)` = max sizeof(alternatives) + alignment + discriminant (~8 bytes overhead).
- **`std::visit` performance:** Compilers generate a jump table for small variant sizes. For >10 alternatives, consider measuring vs. if-else chains.
- **`std::monostate`:** Use as the first variant alternative to represent "no event" without using `std::optional<variant<...>>`.
- **Pattern matching (C++26 proposal):** `inspect(event) { ... }` will replace `std::visit(overloaded{...})` with cleaner syntax.
- **Recursive variants** (e.g., AST nodes) need `std::unique_ptr` inside the variant or a library like Boost.Spirit for indirect variants.
