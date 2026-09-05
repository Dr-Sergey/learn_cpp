# Understand pack expansion in base class lists and using declarations

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/pack_expansion>  

---

## Topic Overview

Pack expansion (`...`) can be used in base class lists and `using` declarations to inherit from or bring in names from a variadic number of bases. This is the foundation for the **overloaded pattern** and **mixin composition**.

### Pack Expansion in Base Class Lists

You can inherit from every type in a parameter pack with a single `...`. The result is a type that combines all of the bases' members:

```cpp
// Inherit from all types in the pack
template<typename... Bases>
struct Derived : Bases... {
    // Derived inherits from every type in Bases...
};

struct A { void a() {} };
struct B { void b() {} };
struct C { void c() {} };

Derived<A, B, C> obj;
obj.a();  // from A
obj.b();  // from B
obj.c();  // from C
```

### Pack Expansion in Using Declarations (C++17)

Inheriting from multiple bases is only half the story. If those bases all define the same function name, name hiding will leave only one visible - unless you expand the `using` declaration too:

```cpp
// Bring operator() from all bases into scope
template<typename... Bases>
struct Overloaded : Bases... {
    using Bases::operator()...;  // C++17 pack expansion in using-declaration
};

// The classic "overloaded" pattern for std::visit:
template<typename... Ts>
Overloaded(Ts...) -> Overloaded<Ts...>;  // CTAD guide

auto visitor = Overloaded{
    [](int x)    { std::cout << "int: " << x << "\n"; },
    [](double x) { std::cout << "double: " << x << "\n"; },
    [](auto x)   { std::cout << "other: " << x << "\n"; }
};
```

### Pack Expansion in Lambda Captures

Packs can also appear in lambda capture lists, giving you a closure that stores each pack element as a separate member:

```cpp
template<typename... Args>
auto make_tuple_printer(Args... args) {
    // Capture the entire pack by value
    return [args...]() {
        ((std::cout << args << " "), ...);
        std::cout << "\n";
    };
}

auto printer = make_tuple_printer(1, 2.5, "hello");
printer();  // Output: 1 2.5 hello
```

---

## Self-Assessment

### Q1: Expand a pack in a base class list: struct Derived : Bases... {} and explain the implications

The `Entity` template below lets you bolt on any combination of mixins at the call site. Each mixin contributes its methods to `Entity`, and you can use a subset just by changing the template arguments.

```cpp
#include <iostream>
#include <string>

struct Logger {
    void log(const std::string& msg) { std::cout << "[LOG] " << msg << "\n"; }
};

struct Serializer {
    void serialize() { std::cout << "[SERIALIZE]\n"; }
};

struct Validator {
    bool validate() { std::cout << "[VALIDATE]\n"; return true; }
};

// Pack expansion in base class list
template<typename... Mixins>
struct Entity : Mixins... {
    std::string name;
    Entity(const std::string& n) : name(n) {}
};

int main() {
    Entity<Logger, Serializer, Validator> e("Player");

    e.log("Spawned");     // from Logger
    e.serialize();        // from Serializer
    e.validate();         // from Validator

    // Can also use with subset:
    Entity<Logger> simple("NPC");
    simple.log("Created");
    // simple.serialize(); // ERROR - no Serializer base
}
```

**How this works:**

- `struct Entity : Mixins...` expands the parameter pack into a list of base classes.
- For `Entity<Logger, Serializer, Validator>`, this becomes `Entity : Logger, Serializer, Validator`.
- Each base contributes its members to the derived class.
- **Implications:** Empty Base Optimization (EBO) applies - empty bases add no size. If two bases have same-named members, name hiding/ambiguity rules apply. Construction order follows the pack order (left to right).

### Q2: Use `using Bases::operator()...;` to bring all operator() overloads into scope

This is the canonical `Overloaded` pattern that makes `std::visit` readable. Without the `using` expansion, name hiding would leave you with only one `operator()` visible - the whole trick depends on bringing all of them in at once.

```cpp
#include <iostream>
#include <string>
#include <variant>

// The overloaded pattern - THE canonical use of pack expansion in using-declarations
template<typename... Ts>
struct Overloaded : Ts... {
    using Ts::operator()...;  // C++17: bring ALL operator() into scope
};

// C++17 deduction guide
template<typename... Ts>
Overloaded(Ts...) -> Overloaded<Ts...>;

int main() {
    std::variant<int, double, std::string> v = "hello";

    std::visit(Overloaded{
        [](int x)              { std::cout << "int: " << x << "\n"; },
        [](double x)           { std::cout << "double: " << x << "\n"; },
        [](const std::string& s) { std::cout << "string: " << s << "\n"; }
    }, v);
    // Output: string: hello

    v = 42;
    std::visit(Overloaded{
        [](int x)              { std::cout << "int: " << x << "\n"; },
        [](double x)           { std::cout << "double: " << x << "\n"; },
        [](const std::string& s) { std::cout << "string: " << s << "\n"; }
    }, v);
    // Output: int: 42
}
```

**How this works:**

- Each lambda is a unique class type with its own `operator()`.
- `Overloaded` inherits from all of them via `Ts...`.
- Without `using Ts::operator()...;`, name hiding would make only one `operator()` visible.
- The pack expansion in the using-declaration brings **all** `operator()` overloads into scope simultaneously.

### Q3: Expand a pack in a lambda capture list and show the resulting closure layout

Think of `[args...]` as telling the compiler to create a separate data member in the closure struct for each element of the pack. The comment in the example makes the closure's internal layout explicit.

```cpp
#include <iostream>
#include <string>

template<typename... Args>
auto capture_all(Args... args) {
    // Pack expansion in lambda capture: each arg is captured by value
    return [args...]() {
        // Fold expression to print all captured values
        ((std::cout << args << " "), ...);
        std::cout << "\n";
    };
}

template<typename... Args>
auto capture_by_move(Args&&... args) {
    // C++20: init-capture with pack expansion
    return [...captured = std::forward<Args>(args)]() {
        ((std::cout << captured << " "), ...);
        std::cout << "\n";
    };
}

int main() {
    auto f = capture_all(1, 2.5, std::string("hello"));
    f();  // Output: 1 2.5 hello

    // The closure conceptually looks like:
    // struct Lambda {
    //     int    arg0 = 1;
    //     double arg1 = 2.5;
    //     string arg2 = "hello";
    //     void operator()() const { cout << arg0 << arg1 << arg2; }
    // };

    std::cout << "Closure size: " << sizeof(f) << " bytes\n";
    // Typically: 4 (int) + 8 (double) + 32 (string) + padding

    auto g = capture_by_move(10, 3.14, std::string("world"));
    g();  // Output: 10 3.14 world
}
```

**How this works:**

- `[args...]` captures each element of the pack as a separate member of the closure object.
- The closure's size equals the sum of all captured types (plus alignment padding).
- C++20 allows `[...x = std::forward<Args>(args)]` for perfect-forwarding capture (move semantics).

---

## Notes

- Pack expansion in using-declarations requires **C++17**. Base class list expansion works since C++11.
- The `Overloaded` pattern is the most common real-world use - essential for `std::visit` with `std::variant`.
- C++20 added init-capture pack expansion: `[...x = expr]` - enables move-capturing packs into lambdas.
- If two bases in the pack have the same member name, you'll get ambiguity errors unless resolved by `using`.
