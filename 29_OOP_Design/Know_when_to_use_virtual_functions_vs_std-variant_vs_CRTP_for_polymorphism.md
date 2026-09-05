# Know when to use virtual functions vs std::variant vs CRTP for polymorphism

**Category:** OOP Design  
**Standard:** C++17  

---

## Topic Overview

C++ is unusual among mainstream languages in offering *three* distinct polymorphism mechanisms with radically different trade-offs. Most languages give you one. C++ gives you all three, each suited to a different situation. Understanding the differences is what separates decent C++ from excellent C++.

Here is the comparison at a glance:

| Aspect | Virtual Functions | std::variant | CRTP |
| --- | :---: | :---: | :---: |
| Dispatch time | Runtime | Runtime (visit) | **Compile-time** |
| Open/Closed set | **Open** (add types freely) | Closed (fixed type list) | Open (templates) |
| Heap required | Yes (polymorphic ptr) | **No** (inline storage) | **No** |
| Heterogeneous container | `vector<unique_ptr<Base>>` | `vector<variant>` | **No** (different types) |
| Adding new types | **Easy** (new derived class) | Recompile all visitors | **Easy** |
| Adding new operations | Hard (modify base) | **Easy** (new visitor) | Moderate |
| vtable overhead | ~8 bytes/object + indirection | Size = max alternative + tag | **Zero** |

### Decision Flowchart

Work through this when you don't know which to reach for:

```cpp
Is the set of types known at compile time?
├─ YES -> Do you need a container of mixed types?
│        ├─ YES -> std::variant + std::visit
│        └─ NO  -> CRTP or simple templates
└─ NO  -> Virtual functions (open set)
```

---

## Self-Assessment

### Q1: Implement the same problem with all three approaches

The best way to internalize the differences is to solve the same problem three ways and then look at what you ended up with. Here is the classic "compute the area of a shape" problem done with each mechanism. Pay attention to what you can and cannot do with each.

```cpp
#include <variant>
#include <vector>
#include <memory>
#include <iostream>
#include <cmath>
#include <type_traits>
#include <numbers>

// ---- APPROACH 1: Virtual functions ----
namespace virt {
    class Shape {
    public:
        virtual ~Shape() = default;
        virtual double area() const = 0;
    };
    class Circle : public Shape {
        double r_;
    public:
        explicit Circle(double r) : r_(r) {}
        double area() const override { return std::numbers::pi * r_ * r_; }
    };
    class Rect : public Shape {
        double w_, h_;
    public:
        Rect(double w, double h) : w_(w), h_(h) {}
        double area() const override { return w_ * h_; }
    };
    double total_area(const std::vector<std::unique_ptr<Shape>>& shapes) {
        double sum = 0;
        for (const auto& s : shapes) sum += s->area();
        return sum;
    }
}

// ---- APPROACH 2: std::variant ----
namespace var {
    struct Circle { double r; };
    struct Rect { double w, h; };
    using Shape = std::variant<Circle, Rect>;

    double area(const Shape& s) {
        return std::visit([](const auto& shape) -> double {
            using T = std::decay_t<decltype(shape)>;
            if constexpr (std::is_same_v<T, Circle>)
                return std::numbers::pi * shape.r * shape.r;
            else
                return shape.w * shape.h;
        }, s);
    }
    double total_area(const std::vector<Shape>& shapes) {
        double sum = 0;
        for (const auto& s : shapes) sum += area(s);
        return sum;
    }
}

// ---- APPROACH 3: CRTP ----
namespace crtp {
    template<typename Derived>
    class Shape {
    public:
        double area() const {
            return static_cast<const Derived*>(this)->area_impl();
        }
    };
    class Circle : public Shape<Circle> {
        double r_;
        friend class Shape<Circle>;
        double area_impl() const { return std::numbers::pi * r_ * r_; }
    public:
        explicit Circle(double r) : r_(r) {}
    };
    // Can't store Circle and Rect in same container!
    template<typename ShapeT>
    double process(const Shape<ShapeT>& s) {
        return s.area();  // Zero-overhead dispatch
    }
}

int main() {
    // Virtual: heap-allocated, open set
    std::vector<std::unique_ptr<virt::Shape>> v1;
    v1.push_back(std::make_unique<virt::Circle>(5));
    v1.push_back(std::make_unique<virt::Rect>(3, 4));
    std::cout << "Virtual: " << virt::total_area(v1) << "\n";

    // Variant: value semantics, cache friendly, closed set
    std::vector<var::Shape> v2 = { var::Circle{5}, var::Rect{3, 4} };
    std::cout << "Variant: " << var::total_area(v2) << "\n";

    // CRTP: zero overhead, no mixed containers
    crtp::Circle c(5);
    std::cout << "CRTP: " << crtp::process(c) << "\n";
    return 0;
}
```

The CRTP version calls `area()` with no indirection at all - the compiler sees the exact type at compile time and inlines everything. The trade-off is that you cannot put a `Circle` and a `Rect` into the same vector.

### Q2: Show where each approach is the clear winner

Rather than memorizing the trade-off table, it helps to anchor each approach to a real-world scenario where it is obviously the right choice.

```cpp
// VIRTUAL wins: Plugin system (open set, loaded at runtime)
class AudioPlugin {
public:
    virtual ~AudioPlugin() = default;
    virtual void process(float* buf, size_t n) = 0;
};
// New plugins via dlopen() - set is OPEN

// VARIANT wins: Compiler AST (closed set, many operations)
using Expr = std::variant<IntLit, FloatLit, BinOp, UnaryOp, Ident>;
std::string to_string(const Expr& e) { return std::visit(/*...*/); }
int evaluate(const Expr& e) { return std::visit(/*...*/); }
// Easy to add operations; compiler checks exhaustiveness

// CRTP wins: Math library (zero-overhead, hot inner loop)
template<typename D> class VecBase {
public:
    auto dot(const D& o) const {
        const auto& s = static_cast<const D&>(*this);
        return s.x()*o.x() + s.y()*o.y() + s.z()*o.z();  // Inlined!
    }
};
// Millions of calls per frame - no vtable indirection
```

The reason variant is so good for an AST is the **exhaustiveness check**. If you add a new `Expr` alternative and forget to handle it in `evaluate`, the compiler tells you. With virtual functions, forgetting to override a method silently falls back to the base class - which is the worst kind of bug because it compiles fine and does the wrong thing.

### Q3: Show a hybrid variant + virtual approach

Sometimes you have a mostly-closed set of types but need an escape hatch for external extensibility. The hybrid approach handles the common cases with fast variant dispatch and falls back to virtual for the rest.

```cpp
#include <variant>
#include <memory>
#include <vector>
#include <iostream>
#include <numbers>

// Known types get fast variant dispatch
struct Circle { double r; };
struct Rect { double w, h; };

// Unknown types use virtual (escape hatch)
class CustomShape {
public:
    virtual ~CustomShape() = default;
    virtual double area() const = 0;
};

using Shape = std::variant<Circle, Rect, std::unique_ptr<CustomShape>>;

double area(const Shape& s) {
    return std::visit([](const auto& shape) -> double {
        using T = std::decay_t<decltype(shape)>;
        if constexpr (std::is_same_v<T, Circle>)
            return std::numbers::pi * shape.r * shape.r;
        else if constexpr (std::is_same_v<T, Rect>)
            return shape.w * shape.h;
        else
            return shape->area();  // Virtual only for custom types
    }, s);
}
// Best of both: fast for known types, extensible for custom
```

The nice part about this hybrid is that `Circle` and `Rect` still get value semantics and cache-friendly storage, while a third-party `CustomShape` can still plug into the system through the virtual interface.

---

## Notes

- Default choice for most code: virtual functions. They are well-understood and flexible.
- Use variant for closed sets (AST, message types, state machines) with value semantics.
- Use CRTP only for measurably hot paths where vtable overhead matters - and even then, profile first.
- Variant visitors get compile-time exhaustiveness checking - forgetting a case is a compile error, not a silent bug.
- The **Expression Problem**: virtual = easy to add types, hard to add operations; variant = the opposite. Knowing this makes the choice obvious for your use case.
