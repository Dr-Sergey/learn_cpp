# Know the Curiously Recurring Template Pattern (CRTP) and Its Uses

**Category:** Templates and Generic Programming  
**Standard:** C++11, C++23 (CRTP)  
**Reference:** <https://en.cppreference.com/w/cpp/language/crtp>  

---

## Topic Overview

### What Is CRTP

The **Curiously Recurring Template Pattern** is when a class derives from a template instantiated with itself. That sounds like a tongue-twister, so here is what it looks like in code:

```cpp
template <typename Derived>
class Base { /* can call static_cast<Derived&>(*this) */ };

class MyClass : public Base<MyClass> { /* ... */ };
```

The key insight is that the base class "knows" the derived type at compile time. That means it can call derived methods through a `static_cast` without any virtual dispatch. This is what people mean by **static polymorphism** - all the type resolution happens at compile time, so there is zero overhead compared to virtual functions.

### Classic CRTP Use Cases

Here is a quick map of where CRTP shows up in practice:

| Use Case | Description |
| --- | --- |
| **Static polymorphism** | Call derived methods via `static_cast<Derived&>(*this)` - zero overhead |
| **Mixin injection** | Add operators, logging, serialization to any derived class |
| **Counting instances** | Each derived class gets its own static counter |
| **Enable clone** | Automate `clone()` in polymorphic hierarchies |
| **Comparisons** | Derive `!=`, `<=`, `>=`, `>` from `==` and `<` |

### CRTP vs Virtual Functions vs C++23 Deducing-This

If you are weighing your options, this table summarizes the trade-offs:

| Feature | Virtual dispatch | CRTP | Deducing-this (C++23) |
| --- | :---: | :---: | :---: |
| Runtime polymorphism | Yes | No | No |
| Zero-overhead calls | No (vtable) | **Yes** | **Yes** |
| Boilerplate | Low | Medium | **Low** |
| Works with base pointers | Yes | No | No |
| Readability | Good | Fair | **Good** |
| Available since | C++98 | C++98 | **C++23** |

---

## Self-Assessment

### Q1: Implement a CRTP base that adds `operator==` and `operator!=` derived from a single `equals()` method

The idea here is that every class that needs equality operators only has to provide one method - `equals()`. The CRTP base handles both `==` and `!=` automatically. Watch how both `Point` and `CaseInsensitiveString` get their operators for free just by inheriting from `EqualityComparable<Derived>`:

```cpp
#include <iostream>
#include <string>
#include <cmath>

// CRTP base: derives == and != from a single equals() in Derived
template <typename Derived>
class EqualityComparable {
public:
    friend bool operator==(const Derived& lhs, const Derived& rhs) {
        return lhs.equals(rhs);
    }
    friend bool operator!=(const Derived& lhs, const Derived& rhs) {
        return !lhs.equals(rhs);
    }
};

// Derived class: only needs to implement equals()
class Point : public EqualityComparable<Point> {
    double x_, y_;
public:
    Point(double x, double y) : x_(x), y_(y) {}

    // Only this method needs to be written
    bool equals(const Point& other) const {
        constexpr double eps = 1e-9;
        return std::abs(x_ - other.x_) < eps &&
               std::abs(y_ - other.y_) < eps;
    }

    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        return os << "(" << p.x_ << ", " << p.y_ << ")";
    }
};

// Another derived class reusing the same CRTP base
class CaseInsensitiveString : public EqualityComparable<CaseInsensitiveString> {
    std::string data_;
public:
    CaseInsensitiveString(std::string s) : data_(std::move(s)) {}

    bool equals(const CaseInsensitiveString& other) const {
        if (data_.size() != other.data_.size()) return false;
        for (size_t i = 0; i < data_.size(); ++i) {
            if (std::tolower(data_[i]) != std::tolower(other.data_[i]))
                return false;
        }
        return true;
    }
};

int main() {
    Point a(1.0, 2.0), b(1.0, 2.0), c(3.0, 4.0);
    std::cout << a << " == " << b << " : " << (a == b) << "\n";  // 1
    std::cout << a << " != " << c << " : " << (a != c) << "\n";  // 1
    std::cout << a << " == " << c << " : " << (a == c) << "\n";  // 0

    CaseInsensitiveString s1("Hello"), s2("hELLO"), s3("World");
    std::cout << "Hello == hELLO : " << (s1 == s2) << "\n";  // 1
    std::cout << "Hello != World : " << (s1 != s3) << "\n";  // 1

    return 0;
}
```

Both classes get working `==` and `!=` without writing those operators themselves - the CRTP base wires them up from `equals()`. That is the mixin injection pattern in action.

**Expected output:**

```text
(1, 2) == (1, 2) : 1
(1, 2) != (3, 4) : 1
(1, 2) == (3, 4) : 0
Hello == hELLO : 1
Hello != World : 1
```

### Q2: Show how CRTP achieves static polymorphism without virtual dispatch overhead

The key difference from virtual functions is that `ShapeBase<Circle>` is a distinct type from `ShapeBase<Square>`. When `area()` calls `area_impl()`, the compiler knows the derived type at compile time and can inline the call directly. The benchmark at the end makes this concrete:

```cpp
#include <iostream>
#include <chrono>
#include <vector>
#include <memory>

// --- CRTP: Static Polymorphism (no vtable) ---
template <typename Derived>
class ShapeBase {
public:
    double area() const {
        // At compile time, Derived is known -> direct function call
        return static_cast<const Derived*>(this)->area_impl();
    }

    void describe() const {
        std::cout << "Area = " << area() << "\n";
    }
};

class Circle : public ShapeBase<Circle> {
    double r_;
public:
    Circle(double r) : r_(r) {}
    double area_impl() const { return 3.14159265 * r_ * r_; }
};

class Square : public ShapeBase<Square> {
    double s_;
public:
    Square(double s) : s_(s) {}
    double area_impl() const { return s_ * s_; }
};

// --- Virtual: Dynamic Polymorphism (vtable) ---
class VShape {
public:
    virtual ~VShape() = default;
    virtual double area() const = 0;
};

class VCircle : public VShape {
    double r_;
public:
    VCircle(double r) : r_(r) {}
    double area() const override { return 3.14159265 * r_ * r_; }
};

class VSquare : public VShape {
    double r_;
public:
    VSquare(double r) : r_(r) {}
    double area() const override { return r_ * r_; }
};

// CRTP: works great when the type is known at compile time
template <typename Shape>
double total_area(const std::vector<Shape>& shapes) {
    double sum = 0;
    for (const auto& s : shapes) sum += s.area();
    return sum;
}

int main() {
    // CRTP usage - no virtual dispatch, calls are inlined
    Circle c(5.0);
    Square s(4.0);
    c.describe();  // Area = 78.5398
    s.describe();  // Area = 16

    // Each container holds ONE concrete type -> CRTP works
    std::vector<Circle> circles(1000, Circle(3.0));
    std::vector<Square> squares(1000, Square(2.0));

    // Virtual usage - requires base pointer, indirect call
    std::vector<std::unique_ptr<VShape>> vshapes;
    for (int i = 0; i < 1000; ++i) vshapes.push_back(std::make_unique<VCircle>(3.0));

    // Benchmark
    constexpr int N = 1'000'000;
    volatile double sink;

    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) sink = total_area(circles);
    auto t2 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) {
        double sum = 0;
        for (const auto& p : vshapes) sum += p->area();
        sink = sum;
    }
    auto t3 = std::chrono::high_resolution_clock::now();

    auto ms1 = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    auto ms2 = std::chrono::duration_cast<std::chrono::milliseconds>(t3 - t2).count();
    std::cout << "\nCRTP (static):  " << ms1 << " ms\n";
    std::cout << "Virtual (dynamic): " << ms2 << " ms\n";
    std::cout << "(CRTP is faster - calls are inlined, no indirect branch)\n";

    return 0;
}
```

Notice that the CRTP version stores homogeneous containers (`vector<Circle>`, `vector<Square>`). That is the trade-off: CRTP is faster, but you cannot mix derived types in a single collection the way you can with virtual functions.

### Q3: Compare CRTP with C++23 deducing-this for the same use case

In C++23, you can write `void print(this const auto& self)` and the compiler automatically deduces the most-derived type without any `static_cast` or template parameter on the base class. The side-by-side comment block in this example makes the reduction in boilerplate clear:

```cpp
#include <iostream>
#include <string>

// === CRTP approach (C++11) ===
template <typename Derived>
class Printable_CRTP {
public:
    void print() const {
        // Must static_cast to access derived members
        const auto& self = static_cast<const Derived&>(*this);
        std::cout << "CRTP: " << self.to_string() << "\n";
    }
};

class Widget_CRTP : public Printable_CRTP<Widget_CRTP> {
    int id_;
public:
    Widget_CRTP(int id) : id_(id) {}
    std::string to_string() const { return "Widget#" + std::to_string(id_); }
};

// === C++23 deducing-this approach ===
// No CRTP needed! The base class deduces the derived type from 'this'
class Printable_DT {
public:
    // 'this auto&& self' deduces the most-derived type
    void print(this const auto& self) {
        std::cout << "Deducing-this: " << self.to_string() << "\n";
    }
};

class Widget_DT : public Printable_DT {
    int id_;
public:
    Widget_DT(int id) : id_(id) {}
    std::string to_string() const { return "Widget#" + std::to_string(id_); }
};

// === Side-by-side comparison ===
/*
 CRTP:                                    Deducing-this (C++23):
 ─────                                    ──────────────────────
 template <typename Derived>              class Printable {
 class Printable {                        public:
 public:                                      void print(this const auto& self) {
     void print() const {                         cout << self.to_string();
         auto& self =                         }
           static_cast<const Derived&>    };
           (*this);                       
         cout << self.to_string();        class Widget : public Printable {
     }                                        string to_string() const;
 };                                       };
                                          
 class Widget :                           // Simpler! No template param on base.
   public Printable<Widget> {             // No static_cast.
     string to_string() const;            // Reads like normal inheritance.
 };

 Comparison table:
 ┌────────────────────────┬─────────────┬─────────────────┐
 │ Feature                │ CRTP        │ Deducing-this   │
 ├────────────────────────┼─────────────┼─────────────────┤
 │ Boilerplate            │ High        │ Low             │
 │ static_cast needed     │ Yes         │ No              │
 │ Template on base       │ Yes         │ No              │
 │ Chainable methods      │ Easy        │ Easier          │
 │ Works pre-C++23        │ Yes         │ No              │
 │ Performance            │ Same        │ Same            │
 └────────────────────────┴─────────────┴─────────────────┘
*/

int main() {
    Widget_CRTP w1(42);
    w1.print();  // CRTP: Widget#42

    Widget_DT w2(99);
    w2.print();  // Deducing-this: Widget#99

    std::cout << "\nVerdict: Prefer deducing-this in C++23+.\n";
    std::cout << "Use CRTP only for pre-C++23 codebases.\n";

    return 0;
}
```

Both approaches produce identical runtime behavior - the performance is the same. The deducing-this version just removes a lot of ceremony.

**Expected output:**

```text
CRTP: Widget#42
Deducing-this: Widget#99

Verdict: Prefer deducing-this in C++23+.
Use CRTP only for pre-C++23 codebases.
```

---

## Notes

- CRTP = a class derives from `Base<Derived>` - the base knows the derived type at compile time.
- Primary uses: static polymorphism, mixin injection (operators, serialization), instance counting.
- Zero-overhead: CRTP calls are resolved and often inlined at compile time - no vtable.
- Limitation: cannot mix derived types in a single container (unlike virtual polymorphism).
- C++23 deducing-this (`this auto& self`) replaces most CRTP use cases more cleanly.
- CRTP remains relevant for pre-C++23 code and for complex multi-level mixin hierarchies.
