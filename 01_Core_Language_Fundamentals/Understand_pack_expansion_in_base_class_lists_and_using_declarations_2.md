# Understand pack expansion in base class lists and using declarations - Part 2

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/parameter_pack>  

---

## Topic Overview

This continues the pack expansion topic, focusing on variadic CRTP, bringing methods from all bases into scope, and mixin composition patterns.

### Variadic CRTP (Curiously Recurring Template Pattern)

When you want static polymorphism from multiple mixins, you can combine CRTP with pack expansion. Each mixin takes the derived type as a template parameter, so it can call back into the derived class without virtual functions:

```cpp
// Each mixin takes the Derived type for static polymorphism
template<typename Derived>
struct Printable {
    void print() const {
        static_cast<const Derived*>(this)->do_print();
    }
};

template<typename Derived>
struct Serializable {
    std::string serialize() const {
        return static_cast<const Derived*>(this)->do_serialize();
    }
};

// Variadic CRTP: inherit from all CRTP bases
template<template<typename> class... Mixins>
struct Entity : Mixins<Entity<Mixins...>>... {
    std::string name;

    void do_print() const { std::cout << name; }
    std::string do_serialize() const { return name; }
};

Entity<Printable, Serializable> e;
e.name = "Player";
e.print();       // calls Printable<Entity>::print() -> Entity::do_print()
e.serialize();   // calls Serializable<Entity>::serialize() -> Entity::do_serialize()
```

### Using Pack Expansion to Bring Methods Into Scope

When bases all have a function with the same name, you need the `using` expansion or name hiding will block all but one. Here's the pattern in its simplest form:

```cpp
template<typename... Bases>
struct Combined : Bases... {
    using Bases::method...;  // C++17: all 'method' overloads visible
};

struct IntProcessor  { void method(int x)  { std::cout << "int: " << x << "\n"; } };
struct StrProcessor  { void method(const std::string& s) { std::cout << "str: " << s << "\n"; } };

Combined<IntProcessor, StrProcessor> c;
c.method(42);       // IntProcessor::method
c.method("hello");  // StrProcessor::method
```

### Mixin Composition Pattern

Fold expressions and pack expansion work together to call a method on each policy base in order - no manual unrolling needed:

```cpp
template<typename... Policies>
struct Widget : Policies... {
    using Policies::apply...;  // bring all apply() overloads

    void configure() {
        // Call apply() on each base
        (Policies::apply(), ...);  // fold expression
    }
};
```

---

## Self-Assessment

### Q1: Use variadic CRTP to inherit from a parameter pack of CRTP mixin bases

The `GameEntity` below composes any combination of CRTP mixins. Each mixin accesses the concrete derived type through `static_cast`, so all the dispatch is resolved at compile time with no virtual function overhead.

```cpp
#include <iostream>
#include <string>

// CRTP mixins
template<typename Derived>
struct HasName {
    void show_name() const {
        std::cout << "Name: " << static_cast<const Derived*>(this)->name << "\n";
    }
};

template<typename Derived>
struct HasHealth {
    void show_health() const {
        std::cout << "HP: " << static_cast<const Derived*>(this)->hp << "\n";
    }
};

template<typename Derived>
struct HasPosition {
    void show_pos() const {
        auto& self = *static_cast<const Derived*>(this);
        std::cout << "Pos: (" << self.x << ", " << self.y << ")\n";
    }
};

// Variadic CRTP
template<template<typename> class... Mixins>
struct GameEntity : Mixins<GameEntity<Mixins...>>... {
    std::string name = "Hero";
    int hp = 100;
    float x = 0.0f, y = 0.0f;
};

int main() {
    GameEntity<HasName, HasHealth, HasPosition> entity;
    entity.name = "Warrior";
    entity.hp = 150;
    entity.x = 10.5f;
    entity.y = 20.3f;

    entity.show_name();    // Name: Warrior
    entity.show_health();  // HP: 150
    entity.show_pos();     // Pos: (10.5, 20.3)

    // Can compose with subset:
    GameEntity<HasName> simple;
    simple.show_name();    // Name: Hero
    // simple.show_health(); // ERROR - no HasHealth base
}
```

**How this works:**

- Each CRTP base receives the derived type, enabling static polymorphism.
- `GameEntity<Mixins...>` expands to `GameEntity : HasName<GameEntity>, HasHealth<GameEntity>, HasPosition<GameEntity>`.
- Each mixin can access derived members via `static_cast<const Derived*>(this)`.
- No virtual functions needed - all dispatch is resolved at compile time.

### Q2: Use `using Bases::method...;` to bring methods from all base classes into scope

Without the `using` expansion, the `Dispatcher` would only see one `process` - name hiding would block the others. The pack expansion in the using-declaration is what makes the whole design work:

```cpp
#include <iostream>
#include <string>

struct HandleInt {
    void process(int x) { std::cout << "int: " << x << "\n"; }
};

struct HandleDouble {
    void process(double x) { std::cout << "double: " << x << "\n"; }
};

struct HandleString {
    void process(const std::string& s) { std::cout << "string: " << s << "\n"; }
};

// Combine all handlers with pack expansion in using-declaration
template<typename... Handlers>
struct Dispatcher : Handlers... {
    using Handlers::process...;  // Bring all process() overloads into scope
};

int main() {
    Dispatcher<HandleInt, HandleDouble, HandleString> d;

    d.process(42);        // int: 42
    d.process(3.14);      // double: 3.14
    d.process("hello");   // string: hello

    // Overload resolution works across all bases
}
```

**How this works:**

- Without `using Handlers::process...;`, each `process` would hide the others (name hiding).
- The pack expansion in the using-declaration brings all `process` overloads from all bases into `Dispatcher`'s scope.
- Standard overload resolution selects the best match.

### Q3: Show a mixin composition that applies N independent policy bases via pack expansion

The fold expression `(Policies::apply(), ...)` calls `apply()` on each base in left-to-right order. The composition is entirely type-driven - adding or removing a policy is just changing the template argument list.

```cpp
#include <iostream>
#include <string>

// Independent policy mixins
struct LogPolicy {
    void apply() { std::cout << "  [LogPolicy] Logging enabled\n"; }
};

struct CachePolicy {
    void apply() { std::cout << "  [CachePolicy] Cache initialized\n"; }
};

struct AuthPolicy {
    void apply() { std::cout << "  [AuthPolicy] Auth required\n"; }
};

struct RateLimitPolicy {
    void apply() { std::cout << "  [RateLimitPolicy] Rate limiting active\n"; }
};

// Compose N policies via pack expansion
template<typename... Policies>
struct Service : Policies... {
    void configure() {
        std::cout << "Configuring service:\n";
        // Fold expression: call apply() on each policy base
        (Policies::apply(), ...);
        std::cout << "Service ready!\n";
    }
};

int main() {
    // Full service
    Service<LogPolicy, CachePolicy, AuthPolicy, RateLimitPolicy> full;
    full.configure();
    // Configuring service:
    //   [LogPolicy] Logging enabled
    //   [CachePolicy] Cache initialized
    //   [AuthPolicy] Auth required
    //   [RateLimitPolicy] Rate limiting active
    // Service ready!

    std::cout << "\n";

    // Minimal service - just logging and auth
    Service<LogPolicy, AuthPolicy> minimal;
    minimal.configure();
    // Configuring service:
    //   [LogPolicy] Logging enabled
    //   [AuthPolicy] Auth required
    // Service ready!
}
```

**How this works:**

- Each policy is an independent struct with an `apply()` method.
- `Service<Policies...>` inherits from all policies via pack expansion.
- `(Policies::apply(), ...)` is a fold expression that calls each policy's `apply()` in order.
- The composition is fully type-safe and resolved at compile time - no virtual function overhead.
- Adding or removing policies is just changing the template arguments.

---

## Notes

- Variadic CRTP enables zero-cost mixin composition - all dispatch is compile-time.
- `using Bases::name...;` in using-declarations requires C++17.
- Fold expressions (`(expr, ...)`) require C++17 and are the cleanest way to apply an operation to each pack element.
- If two bases have the same method with the same signature, you'll get an ambiguity error - resolve with explicit qualification.
- Empty Base Optimization (EBO) means empty policy classes add zero size overhead to the composed type.
