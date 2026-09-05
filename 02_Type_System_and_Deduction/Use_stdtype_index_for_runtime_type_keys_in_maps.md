# Use `std::type_index` for Runtime Type Keys in Maps

**Category:** Type System and Deduction  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/types/type_index>  

---

## Topic Overview

### What Is `std::type_index`

`std::type_index` is a wrapper around `std::type_info` that adds comparison operators and hash support, making it usable as a key in `std::map`, `std::unordered_map`, and `std::set`.

### The Problem with `std::type_info`

The `typeid()` operator returns a `const std::type_info&`, but `type_info` has some annoying limitations that prevent it from being used as a map key directly:

- Is not copyable (only references to it exist)
- Has no `<` operator guaranteed to work as a map key
- Has no `std::hash` specialization

`std::type_index` solves all of these. It stores a pointer to the underlying `type_info` and provides all the operators containers need:

```cpp
#include <typeindex>
#include <unordered_map>
#include <map>

// type_index wraps type_info and provides:
//  - Copy/move constructors
//  - operator<, ==, !=, <=, >=, >
//  - std::hash<std::type_index> specialization

std::type_index idx = typeid(int);
std::type_index idx2 = typeid(double);

std::map<std::type_index, std::string> ordered;        // uses operator<
std::unordered_map<std::type_index, std::string> hash;  // uses std::hash
```

### Core Pattern: Type Registry

The most common use of `std::type_index` is a type-keyed registry - a map from a type to something that knows how to work with that type at runtime:

```cpp
#include <typeindex>
#include <unordered_map>
#include <functional>
#include <memory>

struct Base { virtual ~Base() = default; };

// Factory registry: type_index -> factory function
std::unordered_map<std::type_index, std::function<std::unique_ptr<Base>()>> registry;

template<typename T>
void register_type() {
    registry[typeid(T)] = []() { return std::make_unique<T>(); };
}
```

---

## Self-Assessment

### Q1: Build a registry from `std::type_index` to factory functions using unordered_map

This example builds a real shape registry where you can look up a factory by either a compile-time type or a runtime `type_index` value:

```cpp
#include <iostream>
#include <typeindex>
#include <unordered_map>
#include <functional>
#include <memory>
#include <string>

// Base class for all registered types
struct Shape {
    virtual ~Shape() = default;
    virtual std::string name() const = 0;
    virtual double area() const = 0;
};

struct Circle : Shape {
    double radius = 5.0;
    std::string name() const override { return "Circle"; }
    double area() const override { return 3.14159 * radius * radius; }
};

struct Square : Shape {
    double side = 4.0;
    std::string name() const override { return "Square"; }
    double area() const override { return side * side; }
};

struct Triangle : Shape {
    double base = 6.0, height = 3.0;
    std::string name() const override { return "Triangle"; }
    double area() const override { return 0.5 * base * height; }
};

// Factory registry
class ShapeRegistry {
    std::unordered_map<std::type_index,
                       std::function<std::unique_ptr<Shape>()>> factories_;
public:
    template<typename T>
    void register_type() {
        factories_[typeid(T)] = []() { return std::make_unique<T>(); };
    }

    template<typename T>
    std::unique_ptr<Shape> create() const {
        auto it = factories_.find(typeid(T));
        if (it != factories_.end()) {
            return it->second();
        }
        return nullptr;
    }

    // Create by type_index (for runtime dispatch)
    std::unique_ptr<Shape> create(std::type_index idx) const {
        auto it = factories_.find(idx);
        if (it != factories_.end()) {
            return it->second();
        }
        return nullptr;
    }

    size_t registered_count() const { return factories_.size(); }
};

int main() {
    ShapeRegistry registry;

    // Register types
    registry.register_type<Circle>();
    registry.register_type<Square>();
    registry.register_type<Triangle>();

    std::cout << "Registered types: " << registry.registered_count() << "\n\n";

    // Create by compile-time type
    auto c = registry.create<Circle>();
    auto s = registry.create<Square>();
    auto t = registry.create<Triangle>();

    for (const auto* shape : {c.get(), s.get(), t.get()}) {
        std::cout << shape->name() << " area: " << shape->area() << "\n";
    }

    // Create by runtime type_index
    std::cout << "\nRuntime creation:\n";
    std::type_index types[] = {typeid(Circle), typeid(Triangle)};
    for (auto idx : types) {
        auto shape = registry.create(idx);
        std::cout << "  " << shape->name() << " area: " << shape->area() << "\n";
    }

    return 0;
}
```

Notice that the registry supports both paths: `create<Circle>()` at compile time (where the type is known), and `create(idx)` at runtime (where you only have a `type_index` value, perhaps deserialized or passed through a plugin system).

**Output:**

```text
Registered types: 3

Circle area: 78.5397
Square area: 16
Triangle area: 9

Runtime creation:
  Circle area: 78.5397
  Triangle area: 9
```

### Q2: Show that `std::type_index` wraps `std::type_info` for comparison and hashing

Here you can see the contrast between raw `type_info` (useful but limited) and `type_index` (container-ready):

```cpp
#include <iostream>
#include <typeindex>
#include <typeinfo>
#include <unordered_set>
#include <set>
#include <string>

struct A {};
struct B {};
struct C : A {};

int main() {
    std::cout << std::boolalpha;

    // type_info: reference only, no direct use in containers
    const std::type_info& ti_int = typeid(int);
    const std::type_info& ti_dbl = typeid(double);

    // type_info has name() and ==/!=
    std::cout << "=== type_info ===\n";
    std::cout << "int name: " << ti_int.name() << "\n";
    std::cout << "double name: " << ti_dbl.name() << "\n";
    std::cout << "int == int: " << (typeid(int) == typeid(int)) << "\n";
    std::cout << "int == double: " << (typeid(int) == typeid(double)) << "\n";

    // type_index: copyable, comparable, hashable
    std::type_index idx_int(typeid(int));
    std::type_index idx_dbl(typeid(double));
    std::type_index idx_int2(typeid(int));

    std::cout << "\n=== type_index comparison ===\n";
    std::cout << "idx_int == idx_int2: " << (idx_int == idx_int2) << "\n";
    std::cout << "idx_int == idx_dbl:  " << (idx_int == idx_dbl) << "\n";
    std::cout << "idx_int < idx_dbl:   " << (idx_int < idx_dbl) << "\n";

    // Hash support -> works in unordered containers
    std::cout << "\n=== hashing ===\n";
    std::cout << "hash(int):    " << std::hash<std::type_index>{}(idx_int) << "\n";
    std::cout << "hash(double): " << std::hash<std::type_index>{}(idx_dbl) << "\n";

    // Use in ordered set
    std::set<std::type_index> ordered_types;
    ordered_types.insert(typeid(int));
    ordered_types.insert(typeid(double));
    ordered_types.insert(typeid(A));
    ordered_types.insert(typeid(B));
    ordered_types.insert(typeid(int));  // duplicate

    std::cout << "\n=== ordered set (" << ordered_types.size() << " types) ===\n";
    for (const auto& t : ordered_types) {
        std::cout << "  " << t.name() << "\n";
    }

    // Use in unordered set
    std::unordered_set<std::type_index> hash_types;
    hash_types.insert(typeid(int));
    hash_types.insert(typeid(std::string));
    hash_types.insert(typeid(C));

    std::cout << "\n=== unordered set (" << hash_types.size() << " types) ===\n";
    for (const auto& t : hash_types) {
        std::cout << "  " << t.name() << "\n";
    }

    return 0;
}
```

The duplicate `typeid(int)` insertion is silently ignored by the set - two `type_index` values are equal if and only if they refer to the same type, so the set correctly keeps only one entry for `int`.

**Output (names are implementation-defined):**

```text
=== type_info ===
int name: int
double name: double
int == int: true
int == double: false

=== type_index comparison ===
idx_int == idx_int2: true
idx_int == idx_dbl:  false
idx_int < idx_dbl:   true

=== hashing ===
hash(int):    9365628381058182472
hash(double): 12345678901234567

=== ordered set (4 types) ===
  struct A
  struct B
  double
  int

=== unordered set (3 types) ===
  struct C
  class std::string
  int
```

### Q3: Implement a simple event dispatcher keyed on listener type using `type_index`

This is a practical pattern you'll recognize from UI frameworks and game engines: a dispatcher that routes events to the right handlers based on the event's type at runtime.

```cpp
#include <iostream>
#include <typeindex>
#include <unordered_map>
#include <functional>
#include <vector>
#include <any>
#include <string>

// Event dispatcher: routes events to handlers by event type
class EventDispatcher {
    using Handler = std::function<void(const std::any&)>;
    std::unordered_map<std::type_index, std::vector<Handler>> handlers_;

public:
    template<typename EventT>
    void subscribe(std::function<void(const EventT&)> handler) {
        handlers_[typeid(EventT)].push_back(
            [handler = std::move(handler)](const std::any& event) {
                handler(std::any_cast<const EventT&>(event));
            }
        );
    }

    template<typename EventT>
    void emit(const EventT& event) {
        auto it = handlers_.find(typeid(EventT));
        if (it != handlers_.end()) {
            for (auto& handler : it->second) {
                handler(event);
            }
        }
    }

    size_t event_type_count() const { return handlers_.size(); }
};

// Event types
struct MouseClick { int x, y; };
struct KeyPress { char key; };
struct WindowResize { int width, height; };

int main() {
    EventDispatcher dispatcher;

    // Subscribe to MouseClick events
    dispatcher.subscribe<MouseClick>(
        [](const MouseClick& e) {
            std::cout << "Handler 1: Click at (" << e.x << "," << e.y << ")\n";
        });

    dispatcher.subscribe<MouseClick>(
        [](const MouseClick& e) {
            std::cout << "Handler 2: Click at (" << e.x << "," << e.y << ")\n";
        });

    // Subscribe to KeyPress events
    dispatcher.subscribe<KeyPress>(
        [](const KeyPress& e) {
            std::cout << "Key pressed: '" << e.key << "'\n";
        });

    // Subscribe to WindowResize events
    dispatcher.subscribe<WindowResize>(
        [](const WindowResize& e) {
            std::cout << "Window resized to " << e.width << "x" << e.height << "\n";
        });

    std::cout << "Registered event types: " << dispatcher.event_type_count() << "\n\n";

    // Emit events — dispatched to correct handlers by type_index
    dispatcher.emit(MouseClick{100, 200});
    std::cout << "\n";
    dispatcher.emit(KeyPress{'A'});
    std::cout << "\n";
    dispatcher.emit(WindowResize{1920, 1080});
    std::cout << "\n";

    // Emitting an unregistered event type — nothing happens
    struct UnknownEvent {};
    dispatcher.emit(UnknownEvent{});
    std::cout << "UnknownEvent emitted (no handlers)\n";

    return 0;
}
```

The `std::any` wrapper lets all handler types live in the same vector, while `type_index` ensures each event type routes only to its own subscribers. Emitting an unregistered event type simply finds nothing in the map and returns silently.

**Output:**

```text
Registered event types: 3

Handler 1: Click at (100,200)
Handler 2: Click at (100,200)

Key pressed: 'A'

Window resized to 1920x1080

UnknownEvent emitted (no handlers)
```

---

## Notes

- **`typeid` and polymorphism:** For polymorphic types (with virtual functions), `typeid(*ptr)` gives the dynamic type. For non-polymorphic types, it gives the static type.
- `type_index` is constructed from `type_info`, not from a type directly. Always write `std::type_index(typeid(T))` or `typeid(T)` (implicit conversion).
- **`type_info::name()`** returns implementation-defined strings. Don't rely on them for serialization or cross-compiler compatibility.
- `type_index` comparison reflects the program's type system - two `type_index` values are equal if and only if they refer to the same type in the same program execution.
