# Use the visitor pattern with std::variant instead of double dispatch

**Category:** Modern OOP Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant/visit>  

---

## Topic Overview

The classic **Visitor pattern** uses double dispatch through virtual `accept(Visitor&)` methods - complex, fragile, and hard to maintain. With `std::variant` + `std::visit`, you get the same type-safe dispatch **without** virtual functions, accept methods, or forward declarations.

The reason the classic approach is so painful is that adding a new operation requires modifying the `Visitor` base class and every single concrete shape class to add a new `visit` overload. Adding a new type requires the reverse: adding an `accept` method to the new type and an overload to every existing visitor. With `std::variant`, operations are just free functions containing a `std::visit` call - adding a new operation is one new function, no modifications to existing code.

### Classic vs Variant Visitor

```cpp
CLASSIC DOUBLE DISPATCH:                VARIANT VISITOR:

struct Visitor;                         using Shape = std::variant<
struct Shape {                              Circle, Rect, Triangle>;
    virtual void accept(Visitor&) = 0;
};                                      double area(const Shape& s) {
struct Visitor {                            return std::visit(overloaded{
    virtual void visit(Circle&) = 0;           [](const Circle& c) { ... },
    virtual void visit(Rect&) = 0;             [](const Rect& r)   { ... },
    virtual void visit(Triangle&) = 0;         [](const Triangle& t){ ... }
};                                         }, s);
                                        }
// Each shape needs accept() {          // No accept(), no Visitor base,
//   visitor.visit(*this);              // no forward declarations!
// }
```

### Expression Problem

| Want to add... | Variant (closed set) | Virtual (open set) |
| --- | --- | --- |
| **New operation** | Easy - write new visitor | Hard - add to Visitor + all shapes |
| **New type** | Hard - update all visitors | Easy - add new derived class |

---

## Self-Assessment

### Q1: Replace a virtual `accept(Visitor&)` double dispatch hierarchy with `std::variant` + `std::visit`

The shapes themselves are plain structs with no virtual functions whatsoever. The `overloaded` helper is a small utility that lets you build a visitor inline by combining lambdas. Each operation is just a function that calls `std::visit`:

```cpp
#include <iostream>
#include <variant>
#include <vector>
#include <cmath>
#include <string>

// --- Types (simple structs, no virtual functions!) ---
struct Circle    { double radius; };
struct Rectangle { double width, height; };
struct Triangle  { double base, height; };

using Shape = std::variant<Circle, Rectangle, Triangle>;

// --- overloaded helper (C++17) ---
template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;  // deduction guide

// --- Operation 1: area ---
double area(const Shape& shape) {
    return std::visit(overloaded{
        [](const Circle& c)    { return 3.14159 * c.radius * c.radius; },
        [](const Rectangle& r) { return r.width * r.height; },
        [](const Triangle& t)  { return 0.5 * t.base * t.height; }
    }, shape);
}

// --- Operation 2: describe ---
std::string describe(const Shape& shape) {
    return std::visit(overloaded{
        [](const Circle& c)    { return "Circle(r=" + std::to_string(c.radius) + ")"; },
        [](const Rectangle& r) { return "Rect(" + std::to_string(r.width) + "x" +
                                         std::to_string(r.height) + ")"; },
        [](const Triangle& t)  { return "Triangle(b=" + std::to_string(t.base) + ")"; }
    }, shape);
}

int main() {
    std::vector<Shape> shapes = {
        Circle{5.0},
        Rectangle{3.0, 4.0},
        Triangle{6.0, 3.0}
    };

    for (const auto& s : shapes)
        std::cout << describe(s) << " -> area = " << area(s) << "\n";
}
// Expected output:
//   Circle(r=5.000000) -> area = 78.5398
//   Rect(3.000000x4.000000) -> area = 12
//   Triangle(b=6.000000) -> area = 9
```

**No `accept()` method, no `Visitor` base class, no virtual functions at all!**

---

### Q2: Compare the closed (variant) vs open (virtual) visitor for extensibility in each dimension

Here are both approaches applied to an animal hierarchy. The comments explain where each approach excels and where it struggles:

```cpp
#include <iostream>
#include <variant>
#include <memory>
#include <vector>

// === CLOSED SET (variant): Easy to add operations ===
struct Dog  { std::string name; };
struct Cat  { std::string name; };
struct Bird { std::string name; };

using Animal = std::variant<Dog, Cat, Bird>;

template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

// Operation 1: speak -- trivial to add
void speak(const Animal& a) {
    std::visit(overloaded{
        [](const Dog& d)  { std::cout << d.name << ": Woof!\n"; },
        [](const Cat& c)  { std::cout << c.name << ": Meow!\n"; },
        [](const Bird& b) { std::cout << b.name << ": Tweet!\n"; }
    }, a);
}

// Operation 2: feed -- also trivial
void feed(const Animal& a) {
    std::visit(overloaded{
        [](const Dog&)  { std::cout << "  Giving kibble.\n"; },
        [](const Cat&)  { std::cout << "  Giving fish.\n"; },
        [](const Bird&) { std::cout << "  Giving seeds.\n"; }
    }, a);
}

// Adding Operation 3 is easy -- just write another std::visit.
// But adding a new animal (Fish) requires updating ALL visitors!

// === OPEN SET (virtual): Easy to add types ===
struct AnimalV {
    virtual void speak() const = 0;
    virtual ~AnimalV() = default;
};

struct DogV : AnimalV {
    void speak() const override { std::cout << "Woof!\n"; }
};

struct CatV : AnimalV {
    void speak() const override { std::cout << "Meow!\n"; }
};

// Adding FishV is easy -- just derive from AnimalV.
// But adding a new operation (feed) requires changing AnimalV + all derived!

int main() {
    // Closed set -- variant
    std::vector<Animal> zoo = {Dog{"Rex"}, Cat{"Whiskers"}, Bird{"Tweety"}};
    for (const auto& a : zoo) {
        speak(a);
        feed(a);
    }

    std::cout << "---\n";

    // Open set -- virtual
    std::vector<std::unique_ptr<AnimalV>> zoo2;
    zoo2.push_back(std::make_unique<DogV>());
    zoo2.push_back(std::make_unique<CatV>());
    for (const auto& a : zoo2)
        a->speak();
}
// Expected output:
//   Rex: Woof!
//     Giving kibble.
//   Whiskers: Meow!
//     Giving fish.
//   Tweety: Tweet!
//     Giving seeds.
//   ---
//   Woof!
//   Meow!
```

**Summary table:**

| Dimension | Variant (closed types) | Virtual (open types) |
| --- | --- | --- |
| Add new operation | **Easy** - new `std::visit` | Hard - modify base + all derived |
| Add new type | Hard - update all visitors | **Easy** - new derived class |
| Heap allocation | No | Yes (usually `unique_ptr`) |
| Type at compile time | Known (variant alternatives) | Unknown (runtime polymorphism) |
| Performance | Better (no indirection, inlinable) | Worse (vtable lookup) |

---

### Q3: Show how adding a new variant type forces all visitors to handle it (compile-time exhaustiveness)

This is the flip side of the expression problem. Adding a type to the variant immediately breaks every visitor that does not handle it - which is exactly what you want, because it forces you to consciously update the code instead of silently missing a case:

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <vector>

struct TextNode   { std::string text; };
struct ImageNode  { std::string url; int width; };
// struct VideoNode { std::string url; int duration; };  // <- UNCOMMENT TO SEE ERRORS!

using Node = std::variant<TextNode, ImageNode  /*, VideoNode */>;

template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

// Visitor with EXPLICIT handlers (no auto catch-all)
std::string render(const Node& node) {
    return std::visit(overloaded{
        [](const TextNode& t)  { return "<p>" + t.text + "</p>"; },
        [](const ImageNode& i) { return "<img src=\"" + i.url + "\" width=" +
                                         std::to_string(i.width) + ">"; }
        // If VideoNode is added to the variant but NOT handled here:
        // COMPILER ERROR:
        //   error: no match for call to '(overloaded<...>)(const VideoNode&)'
        //   note: no known conversion for argument from 'const VideoNode' to ...
        //
        // The compiler FORCES you to add:
        // , [](const VideoNode& v) { return "<video src=\"" + v.url + "\">"; }
    }, node);
}

int main() {
    std::vector<Node> page = {
        TextNode{"Hello, World!"},
        ImageNode{"logo.png", 200},
    };

    for (const auto& n : page)
        std::cout << render(n) << "\n";

    // TRY IT: Uncomment VideoNode in the variant above.
    // The render() function will fail to compile until you add a VideoNode handler.
    // This is COMPILE-TIME exhaustiveness -- impossible with virtual dispatch!
}
// Expected output:
//   <p>Hello, World!</p>
//   <img src="logo.png" width=200>
```

**The exhaustiveness guarantee:**

1. `std::visit` must handle ALL variant alternatives
2. If you use explicit lambda overloads (no `auto` catch-all), each type must match
3. Adding `VideoNode` to `using Node = variant<..., VideoNode>` immediately breaks every visitor missing it
4. This acts like a sealed class + pattern matching in Rust/Kotlin - unmatched cases are compile errors

---

## Notes

- **`overloaded` helper:** Required in C++17. In C++20 you can omit the deduction guide. Some codebases define it once in a utility header.
- **`auto` catch-all:** Adding `[](auto&&) { ... }` to the visitor disables exhaustiveness checking - use with caution.
- **Performance:** `std::visit` typically compiles to a jump table or `if`-chain. With small variants (2-3 types), it can be fully inlined.
- **Variant size:** `sizeof(variant<A,B,C>)` = `max(sizeof(A), sizeof(B), sizeof(C))` + discriminant (usually 1-4 bytes).
- **Related pattern:** Use `std::variant` for state machines (see State pattern topic) and algebraic data types.
- **When to prefer virtual:** When types are defined in separate translation units, libraries, or plugins (open set of types).
