# Understand the four pillars of OOP - encapsulation, inheritance, polymorphism, abstraction

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

The four pillars of OOP in C++ are not merely academic concepts - they are design tools with specific trade-offs. Senior engineers must know when each pillar *hurts* as much as when it helps.

```cpp
┌──────────────────────────────────────────────────┐
│                 OOP Pillars                       │
├──────────────┬──────────────┬──────────────┬──────┤
│ Encapsulation│ Inheritance  │ Polymorphism │ Abstr│
│              │              │              │action│
│ Hide state   │ Reuse via    │ One interface│ Hide │
│ behind API   │ is-a         │ many impls   │ impl │
│              │              │              │detail│
│ Control      │ Extend       │ Dispatch     │ Defin│
│ invariants   │ behavior     │ at runtime   │seams │
│              │              │ or compile   │      │
└──────────────┴──────────────┴──────────────┴──────┘
```

### When Each Pillar Applies (and When It Doesn't)

| Pillar | Use when... | Avoid when... |
| --- | --- | --- |
| **Encapsulation** | You have invariants to protect | Aggregate data with no invariants (use struct) |
| **Inheritance** | True is-a relationship with shared interface | Code reuse only (use composition) |
| **Polymorphism** | Set of types varies at runtime, open for extension | Closed set of types (use `std::variant`) |
| **Abstraction** | Hiding platform/vendor/implementation details | Adding abstraction layers with only one implementation |

---

## Self-Assessment

### Q1: Design a class hierarchy that demonstrates all four pillars working together

Here is a sensor system where each pillar does its job. Notice how abstraction defines the interface, encapsulation hides the calibration internals, inheritance specializes the behavior, and polymorphism lets `SensorArray` treat every sensor the same way regardless of its concrete type.

```cpp
#include <memory>
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>
#include <numeric>

// ABSTRACTION: Define the interface, hide implementation
class Sensor {
public:
    virtual ~Sensor() = default;
    virtual double read() const = 0;                     // Pure interface
    virtual std::string name() const = 0;
    virtual bool calibrate(double reference) = 0;

    // Non-virtual interface - common filtering logic
    double read_filtered(int samples) const {
        std::vector<double> readings(samples);
        for (auto& r : readings) r = read();
        std::sort(readings.begin(), readings.end());
        // Trim outliers: drop top/bottom 10%
        int trim = samples / 10;
        double sum = std::accumulate(
            readings.begin() + trim, readings.end() - trim, 0.0);
        return sum / (samples - 2 * trim);
    }
};

// ENCAPSULATION: Hide calibration state, enforce invariants
class TemperatureSensor : public Sensor {
    double offset_ = 0.0;      // private: calibration offset
    double gain_ = 1.0;        // private: calibration gain
    double raw_read() const {
        return 25.0 + (std::rand() % 100) / 100.0;  // Simulated hardware read
    }

public:
    // INVARIANT: gain_ > 0 always
    double read() const override {
        return raw_read() * gain_ + offset_;
    }

    std::string name() const override { return "TempSensor"; }

    bool calibrate(double reference) override {
        double raw = raw_read();
        if (raw == 0.0) return false;   // Guard invariant
        gain_ = reference / raw;
        offset_ = 0.0;
        return true;
    }
};

// INHERITANCE: Extend with specialized behavior
class PressureSensor : public Sensor {
    double zero_point_;
    double scale_;

public:
    explicit PressureSensor(double scale = 1.0)
        : zero_point_(0.0), scale_(scale) {}

    double read() const override {
        return (1013.25 + (std::rand() % 100) / 10.0 - 5.0) * scale_ + zero_point_;
    }

    std::string name() const override { return "PressureSensor"; }

    bool calibrate(double reference) override {
        zero_point_ = reference - read();
        return true;
    }
};

// POLYMORPHISM: Uniform processing of different sensor types
class SensorArray {
    std::vector<std::unique_ptr<Sensor>> sensors_;

public:
    void add(std::unique_ptr<Sensor> s) {
        sensors_.push_back(std::move(s));
    }

    void read_all() const {
        for (const auto& s : sensors_) {
            // Same code works for ANY sensor type
            std::cout << s->name() << ": " << s->read_filtered(10) << "\n";
        }
    }

    void calibrate_all(double reference) {
        for (auto& s : sensors_) {
            if (!s->calibrate(reference))
                std::cerr << s->name() << ": calibration failed!\n";
        }
    }
};

int main() {
    SensorArray array;
    array.add(std::make_unique<TemperatureSensor>());
    array.add(std::make_unique<PressureSensor>(1.5));
    array.read_all();
    return 0;
}
```

### Q2: Explain the key trade-offs and common violations of each pillar

Each pillar has a characteristic way that developers misuse it. Here are the most common violations - and what to do instead.

**Encapsulation violations:**

The classic encapsulation mistake is writing a getter and a setter for every private field. That is not encapsulation - it is just private storage with a public interface that lets anyone do anything. Real encapsulation means exposing *operations*, not raw state.

```cpp
// BAD: getters/setters for everything = no encapsulation
class BadAccount {
    double balance_;
public:
    double getBalance() { return balance_; }
    void setBalance(double b) { balance_ = b; }  // Anyone can set negative balance!
};

// GOOD: Enforce invariants through operations
class Account {
    double balance_;
public:
    explicit Account(double initial) : balance_(initial) {
        if (initial < 0) throw std::invalid_argument("Negative balance");
    }
    double balance() const { return balance_; }
    void deposit(double amount) {
        if (amount <= 0) throw std::invalid_argument("Non-positive deposit");
        balance_ += amount;
    }
    bool withdraw(double amount) {
        if (amount <= 0 || amount > balance_) return false;
        balance_ -= amount;
        return true;
    }
};
```

**Inheritance misuse:**

The most famous inheritance misuse is the Square/Rectangle problem, which illustrates a violation of the Liskov Substitution Principle. The reason this trips people up: geometrically, a square *is* a rectangle. But in code, a `Square` cannot honor the behavioral contract of `Rectangle`, because setting width and height independently is a core part of that contract. When your derived class has to break the base class's invariants to make sense, inheritance is the wrong tool.

```cpp
// BAD: Square IS-NOT-A Rectangle (LSP violation)
class Rectangle {
public:
    virtual void setWidth(int w) { width_ = w; }
    virtual void setHeight(int h) { height_ = h; }
    int area() const { return width_ * height_; }
protected:
    int width_ = 0, height_ = 0;
};

class Square : public Rectangle {  // BROKEN: violates LSP
    void setWidth(int w) override { width_ = height_ = w; }
    void setHeight(int h) override { width_ = height_ = h; }
    // Client code: r.setWidth(5); r.setHeight(3); assert(r.area() == 15); // FAILS for Square!
};

// GOOD: Separate types or use composition
struct Rect { int w, h; int area() const { return w * h; } };
struct Square { int side; int area() const { return side * side; } };
```

**Abstraction overhead:**

Not every class needs an interface. If there is only one implementation now and no realistic prospect of a second one, the extra layer of indirection is pure cost: more files, more cognitive overhead, more virtual dispatch. Extract the interface *when you actually need it*, not speculatively.

```cpp
// BAD: Abstraction for one implementation = pointless indirection
class ILogger { public: virtual void log(std::string_view) = 0; virtual ~ILogger() = default; };
class ConsoleLogger : public ILogger { /* only impl that will ever exist */ };

// GOOD: Just use the concrete class until you actually need a second implementation
class Logger {
public:
    void log(std::string_view msg) { std::cout << msg << "\n"; }
};
// Extract interface LATER when you need FileLogger, NetworkLogger, etc.
```

### Q3: Show how modern C++ often replaces classical OOP with lighter alternatives

Virtual dispatch and heap allocation are not the only tools for polymorphism. Modern C++ frequently uses `std::variant` for closed type sets, which gives you value semantics, stack allocation, and compile-time exhaustiveness checking. Here is the same shape problem done both ways so you can see the contrast.

```cpp
#include <variant>
#include <vector>
#include <iostream>
#include <cmath>
#include <type_traits>
#include <numbers>

// Classical OOP (virtual dispatch)
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual void draw() const = 0;
};
class Circle : public Shape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return std::numbers::pi * r_ * r_; }
    void draw() const override { std::cout << "Circle(r=" << r_ << ")\n"; }
};

// Modern C++: std::variant (no virtual, no heap)
struct CircleV  { double r; };
struct RectV    { double w, h; };
struct TriangleV { double base, height; };

using ShapeV = std::variant<CircleV, RectV, TriangleV>;

double area(const ShapeV& s) {
    return std::visit([](const auto& shape) -> double {
        using T = std::decay_t<decltype(shape)>;
        if constexpr (std::is_same_v<T, CircleV>)
            return std::numbers::pi * shape.r * shape.r;
        else if constexpr (std::is_same_v<T, RectV>)
            return shape.w * shape.h;
        else
            return 0.5 * shape.base * shape.height;
    }, s);
}

// When to use which:
// Virtual: open set of types (plugins, user-defined shapes)
// Variant: closed set, better cache locality, no heap alloc, exhaustive matching
// CRTP:    compile-time polymorphism, zero overhead, no late binding

int main() {
    std::vector<ShapeV> shapes = { CircleV{5}, RectV{3, 4}, TriangleV{6, 8} };
    for (const auto& s : shapes)
        std::cout << "Area: " << area(s) << "\n";
    // Area: 78.5398
    // Area: 12
    // Area: 24
    return 0;
}
```

---

## Notes

- **Encapsulation is not getters/setters.** It means protecting invariants through a meaningful API.
- **Inheritance is the tightest coupling.** Prefer composition; extract an interface only when more than one implementation exists.
- **Polymorphism has three flavors in C++:** runtime (virtual), compile-time (CRTP/templates), value-based (variant). Choose by use case.
- **"Program to an interface"** means depend on abstractions - in C++ that is an abstract base class or a concept, not just `class I*`.
- Most C++ experts agree: **prefer value semantics by default**, use reference semantics (virtual + heap) only when necessary.
