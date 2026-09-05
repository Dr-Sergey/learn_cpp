# Use mixins through CRTP or inheritance to compose behavior

**Category:** Modern OOP Patterns  
**Standard:** C++11, C++20 (CRTP)  
**Reference:** <https://en.cppreference.com/w/cpp/language/crtp>  

---

## Topic Overview

A **mixin** is a class that provides a set of operations to a derived class, composing behavior without the overhead of virtual dispatch. CRTP (Curiously Recurring Template Pattern) is the classic C++ mixin mechanism: a base class is templated on the derived class so it can call derived methods statically.

The reason CRTP exists is that sometimes you want a base class to call methods that only exist on the derived type - but without the runtime cost of a virtual call. By passing the derived type as a template argument, the base class can `static_cast<Derived&>(*this)` and call derived methods directly. The compiler resolves everything at compile time, so there is no vtable involved.

The nice part is that each mixin is a distinct template instantiation, so combining multiple mixins does not create a diamond inheritance problem.

```cpp
┌──────────────────────────────────┐
│  Mixin Composition               │
│                                  │
│  template<typename D>            │
│  struct Printable { ... };       │     Each mixin is independent
│                                  │     -- no diamond problem!
│  template<typename D>            │
│  struct Serializable { ... };    │
│                                  │
│  struct Widget : Printable<Widget>,  │
│                  Serializable<Widget>│
│  { ... };                        │
└──────────────────────────────────┘
```

### Key Benefits

| Feature | CRTP Mixin | Virtual Mixin | Multiple Inheritance |
| --- | --- | --- | --- |
| Dispatch | Static (compile-time) | Dynamic (vtable) | Dynamic |
| Overhead | Zero | vptr + indirect call | vptr + possible diamond |
| Extensibility | Open (new derived) | Open | Complex |
| Binary size | May increase (templates) | Smaller | Moderate |

---

## Self-Assessment

### Q1: Implement a Comparable mixin that provides <=, >, >=, != from a single < operator

The goal is to let a class define only `operator<` and get the rest for free. The mixin derives all other comparisons from that one. Watch how `Temperature` only writes one operator, yet all five work:

```cpp
#include <iostream>
#include <string>
#include <compare>

// CRTP Comparable mixin: derive all relational ops from operator<
template <typename Derived>
struct Comparable {
    friend bool operator>(const Derived& a, const Derived& b)  { return b < a; }
    friend bool operator<=(const Derived& a, const Derived& b) { return !(b < a); }
    friend bool operator>=(const Derived& a, const Derived& b) { return !(a < b); }
    friend bool operator!=(const Derived& a, const Derived& b) { return a < b || b < a; }
    friend bool operator==(const Derived& a, const Derived& b) { return !(a < b) && !(b < a); }
};

struct Temperature : Comparable<Temperature> {
    double celsius;

    // Only define this ONE operator -- mixin provides the rest!
    friend bool operator<(const Temperature& a, const Temperature& b) {
        return a.celsius < b.celsius;
    }
};

struct Priority : Comparable<Priority> {
    int level;
    std::string name;

    friend bool operator<(const Priority& a, const Priority& b) {
        return a.level < b.level;
    }
};

int main() {
    Temperature boiling{100.0}, freezing{0.0}, body{36.6};

    std::cout << std::boolalpha;
    std::cout << "boiling > freezing: " << (boiling > freezing) << "\n";
    std::cout << "freezing >= body:   " << (freezing >= body) << "\n";
    std::cout << "body == body:       " << (body == body) << "\n";
    std::cout << "body != boiling:    " << (body != boiling) << "\n";

    Priority high{10, "High"}, low{1, "Low"};
    std::cout << "high > low: " << (high > low) << "\n";
    std::cout << "low <= high: " << (low <= high) << "\n";
}
// Expected output:
//   boiling > freezing: true
//   freezing >= body:   false
//   body == body:       true
//   body != boiling:    true
//   high > low: true
//   low <= high: true
```

Both `Temperature` and `Priority` share the same mixin definition but get completely independent operator sets because they are separate template instantiations.

> **C++20 note:** With `<=>`, you can use `auto operator<=>(const T&) const = default;` directly. The CRTP mixin approach is valuable for pre-C++20 code or when you need custom comparison logic.

---

### Q2: Show how multiple CRTP bases can be composed without diamond inheritance issues

The key insight here is in the comment after the code: `Printable<Sensor>`, `Serializable<Sensor>`, and `Cloneable<Sensor>` are three completely different types, so there is no shared base, and therefore no diamond:

```cpp
#include <iostream>
#include <string>
#include <sstream>
#include <memory>

// Mixin 1: Printable -- adds print() based on to_string()
template <typename Derived>
struct Printable {
    void print() const {
        const auto& self = static_cast<const Derived&>(*this);
        std::cout << self.to_string() << "\n";
    }
};

// Mixin 2: Serializable -- adds serialize() based on to_string()
template <typename Derived>
struct Serializable {
    std::string serialize() const {
        const auto& self = static_cast<const Derived&>(*this);
        return "{\"data\":\"" + self.to_string() + "\"}";
    }
};

// Mixin 3: Cloneable -- adds clone() returning unique_ptr
template <typename Derived>
struct Cloneable {
    std::unique_ptr<Derived> clone() const {
        const auto& self = static_cast<const Derived&>(*this);
        return std::make_unique<Derived>(self);  // copy ctor
    }
};

// Compose ALL THREE mixins -- no diamond!
// Each CRTP base is a DIFFERENT template instantiation:
//   Printable<Sensor>, Serializable<Sensor>, Cloneable<Sensor>
struct Sensor : Printable<Sensor>,
                Serializable<Sensor>,
                Cloneable<Sensor> {
    std::string name;
    double value;

    std::string to_string() const {
        std::ostringstream oss;
        oss << name << "=" << value;
        return oss.str();
    }
};

int main() {
    Sensor s{"temperature", 23.5};

    // From Printable mixin
    s.print();

    // From Serializable mixin
    std::cout << s.serialize() << "\n";

    // From Cloneable mixin
    auto copy = s.clone();
    copy->value = 99.0;
    copy->print();

    // Original unchanged
    s.print();
}
// Expected output:
//   temperature=23.5
//   {"data":"temperature=23.5"}
//   temperature=99
//   temperature=23.5
```

**Why no diamond:** Each CRTP base `Mixin<Derived>` is a distinct type. There's no shared base class, so no ambiguity:

```cpp
  Printable<Sensor>  Serializable<Sensor>  Cloneable<Sensor>
         \                  |                   /
          \                 |                  /
           ─────────── Sensor ────────────────
```

---

### Q3: Compare mixin composition with multiple inheritance and explain the tradeoffs

The following example implements the same `LoggableMixin` and `CountableMixin` behaviors two different ways - once with CRTP (no virtual, fully static) and once with traditional multiple inheritance (virtual dispatch):

```cpp
#include <iostream>
#include <string>

// --- Approach 1: CRTP Mixin Composition ---
template <typename D>
struct LoggableMixin {
    void log(const std::string& msg) const {
        const auto& self = static_cast<const D&>(*this);
        std::cout << "[" << self.get_name() << "] " << msg << "\n";
    }
};

template <typename D>
struct CountableMixin {
    inline static int count_ = 0;
    CountableMixin()  { ++count_; }
    ~CountableMixin() { --count_; }
    static int alive() { return count_; }
};

struct ServiceA : LoggableMixin<ServiceA>, CountableMixin<ServiceA> {
    std::string get_name() const { return "ServiceA"; }
};

// --- Approach 2: Traditional Multiple Inheritance ---
struct Loggable {
    virtual std::string get_name() const = 0;
    void log(const std::string& msg) const {
        std::cout << "[" << get_name() << "] " << msg << "\n";
    }
    virtual ~Loggable() = default;
};

struct Countable {
    inline static int count_ = 0;
    Countable()  { ++count_; }
    virtual ~Countable() { --count_; }
    static int alive() { return count_; }
};

struct ServiceB : Loggable, Countable {
    std::string get_name() const override { return "ServiceB"; }
};

int main() {
    // CRTP mixin -- no virtual dispatch
    ServiceA a;
    a.log("started");
    std::cout << "ServiceA alive: " << ServiceA::alive() << "\n";

    // Multiple inheritance -- uses virtual dispatch
    ServiceB b;
    b.log("started");
    std::cout << "ServiceB alive: " << ServiceB::alive() << "\n";
}
// Expected output:
//   [ServiceA] started
//   ServiceA alive: 1
//   [ServiceB] started
//   ServiceB alive: 1
```

Both approaches produce the same output, but `ServiceA` calls `get_name()` as a direct function call with zero overhead, while `ServiceB` goes through a vtable lookup. The tradeoff is that `ServiceA` cannot be stored behind a `Loggable*` pointer.

**Tradeoffs comparison:**

| Aspect | CRTP Mixin | Multiple Inheritance |
| --- | --- | --- |
| **Dispatch type** | Static (zero overhead) | Dynamic (vtable) |
| **Runtime polymorphism** | No (cannot hold base pointer) | Yes (`Loggable* p = &b;`) |
| **Diamond problem** | Impossible (distinct base types) | Possible (needs virtual inheritance) |
| **Per-type state** | Natural (`CountableMixin<A>` != `CountableMixin<B>`) | Shared static (`Countable::count_` is ONE counter) |
| **Code bloat** | Yes - each instantiation duplicates code | No - single implementation |
| **Compilation time** | Slower (template instantiation) | Faster |
| **Debugging** | Harder (template error messages) | Easier |
| **Suitable for** | Performance-critical, library internals | Public APIs, plugin systems |

---

## Notes

- **CRTP mixin rule:** The derived class must appear as the template argument: `struct X : Mixin<X>`. Passing the wrong type compiles but causes UB.
- **C++23 alternative:** Use deducing-this (`this auto& self`) to write mixins without templates.
- **Empty base optimization (EBO):** CRTP mixin bases with no data members take zero extra bytes.
- **Variadic mixin pattern:** `template<typename... Mixins> struct Composed : Mixins...{};` - combine arbitrary mixins at the call site.
- **Scope:** Mixins shouldn't be instantiated directly. Consider making their constructors `protected`.
