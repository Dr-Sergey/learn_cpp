# Understand covariant return types in virtual functions

**Category:** Modern OOP Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/virtual>  

---

## Topic Overview

**Covariant return types** are a C++ language rule that allows a virtual function override to return a *more-derived* pointer or reference than the base class declares. This is the foundation of the virtual `clone()` idiom.

This file focuses on the **language rule** itself and its mechanics. (See the companion file on covariant returns with smart pointers for the `unique_ptr` limitation.)

### The Rule (formal)

If virtual `Base::f()` returns `B*` (or `B&`), then `Derived::f()` may return `D*` (or `D&`) provided:

1. `D` is **publicly** derived from `B` (or is `B` itself)
2. `D` is a **complete type** at the point of the override declaration
3. The cv-qualification of `D*` is the same or less than `B*`

The covariance rule is useful because it lets the caller get a pointer to the most-derived type when calling through a derived-typed variable, without needing a cast. When calling through a base pointer, the static return type is still `B*`, but virtual dispatch still runs the most-derived override at runtime.

```cpp
Base::f() -> B*        Derived::f() -> D*
            │                        │
            │  D publicly derives B  │
            └────────────────────────┘
                    COVARIANT - OK

Base::f() -> B*        Derived::f() -> X*
            │                        │
            │  X does NOT derive B   │
            └────────────────────────┘
                 COMPILE ERROR
```

---

## Self-Assessment

### Q1: Write a `clone()` virtual function that returns `Base*` in the base and `Derived*` in the derived class

Watch what happens at each level of the pointer type here. When you have an `ElectricCar*` and call `clone()`, you get back an `ElectricCar*` directly - the covariant return is visible at the call site. When you call through a `Car*` or `Vehicle*`, you get back the corresponding static type, but the actual virtual dispatch still runs `ElectricCar::clone()` every time.

```cpp
#include <iostream>
#include <string>

class Vehicle {
    std::string type_;
public:
    explicit Vehicle(std::string t) : type_(std::move(t)) {}
    virtual ~Vehicle() = default;

    // Base returns Vehicle*
    virtual Vehicle* clone() const {
        std::cout << "  Vehicle::clone()\n";
        return new Vehicle(*this);
    }

    virtual void describe() const {
        std::cout << "  Vehicle: " << type_ << "\n";
    }
};

class Car : public Vehicle {
    int horsepower_;
public:
    Car(std::string t, int hp) : Vehicle(std::move(t)), horsepower_(hp) {}

    // GOOD: Covariant: Car* overrides Vehicle*
    Car* clone() const override {
        std::cout << "  Car::clone()\n";
        return new Car(*this);
    }

    void describe() const override {
        std::cout << "  Car: " << horsepower_ << " HP\n";
    }

    int hp() const { return horsepower_; }
};

class ElectricCar : public Car {
    int range_km_;
public:
    ElectricCar(std::string t, int hp, int range)
        : Car(std::move(t), hp), range_km_(range) {}

    // GOOD: Covariant chain: ElectricCar* -> Car* -> Vehicle*
    ElectricCar* clone() const override {
        std::cout << "  ElectricCar::clone()\n";
        return new ElectricCar(*this);
    }

    void describe() const override {
        std::cout << "  ElectricCar: " << hp() << " HP, "
                  << range_km_ << " km range\n";
    }
};

int main() {
    ElectricCar tesla("Model S", 670, 600);

    // Through ElectricCar* - returns ElectricCar* (most derived)
    ElectricCar* copy1 = tesla.clone();
    copy1->describe();

    // Through Car* - returns Car* (covariant mid-level)
    Car* car_ptr = &tesla;
    Car* copy2 = car_ptr->clone();  // Actually calls ElectricCar::clone()
    copy2->describe();

    // Through Vehicle* - returns Vehicle* (base level)
    Vehicle* base_ptr = &tesla;
    Vehicle* copy3 = base_ptr->clone();  // Also calls ElectricCar::clone()
    copy3->describe();

    delete copy1;
    delete copy2;
    delete copy3;
}
// Expected output:
//   ElectricCar::clone()
//   ElectricCar: 670 HP, 600 km range
//   ElectricCar::clone()
//   ElectricCar: 670 HP, 600 km range
//   ElectricCar::clone()
//   ElectricCar: 670 HP, 600 km range
```

All three calls end up in `ElectricCar::clone()`. The covariance only affects which static type the compiler gives the return value at the call site, not which function actually runs.

---

### Q2: Explain the rule: covariant return type must be a pointer/reference to a more-derived type

Here are all the conditions that determine whether an override's return type qualifies as covariant. The table captures both the valid and invalid cases:

| Condition | Valid | Invalid |
| --- | --- | --- |
| `D*` where D derives from B | `Derived*` overrides `Base*` - Yes | `Unrelated*` overrides `Base*` - No |
| Reference works too | `Derived&` overrides `Base&` - Yes | `int&` overrides `Base&` - No |
| Same cv-qualification | `const D*` overrides `const B*` - Yes | `D*` overrides `const B*` - No |
| D must be complete | D fully defined before override - Yes | Forward-declared D - No |
| Private inheritance | `D*` where D privately inherits B - No | |

The cv-qualification rule catches a subtle case: you can't drop `const` in an override's return type. And the private inheritance case catches another: covariance requires a *public* IS-A relationship, not just any derivation.

```cpp
#include <iostream>

class Base {
public:
    virtual ~Base() = default;
    virtual Base* self() { return this; }
    virtual const Base& ref() const { return *this; }
};

class Derived : public Base {
public:
    // GOOD: Covariant pointer
    Derived* self() override { return this; }

    // GOOD: Covariant reference
    const Derived& ref() const override { return *this; }
};

// ERROR: Won't compile: private inheritance breaks covariance
class PrivateDerived : private Base {
    // Base::self returns Base*; this class privately inherits
    // PrivateDerived* self() override { return this; }  // ERROR
};

// ERROR: Won't compile: can't add const to override
class BadConst : public Base {
    // Const mismatch:
    // const Base* self() override { return this; }  // ERROR: different qualification
};

int main() {
    Derived d;
    Base* bp = &d;

    // Through Base* -> returns Base* (static type)
    Base* b_self = bp->self();
    std::cout << "Through Base*: " << b_self << "\n";

    // Through Derived* -> returns Derived* (covariant!)
    Derived* d_self = d.self();
    std::cout << "Through Derived*: " << d_self << "\n";
    std::cout << "Same object? " << (b_self == d_self ? "YES" : "NO") << "\n";
}
// Expected output:
//   Through Base*: 0x...
//   Through Derived*: 0x...
//   Same object? YES
```

Both pointers point to the same object - the covariant return type just changes the static type the compiler sees at the call site, not what the pointer actually points to.

---

### Q3: Show that covariant return with smart pointers does NOT work and present the workaround

The error you get when you try to return `unique_ptr<Button>` from an override of `unique_ptr<Widget>` is a compile-time type mismatch - the two `unique_ptr` instantiations are completely unrelated types as far as the language is concerned.

```cpp
#include <memory>

class Widget {
public:
    virtual ~Widget() = default;
    virtual std::unique_ptr<Widget> clone() const {
        return std::make_unique<Widget>(*this);
    }
};

class Button : public Widget {
public:
    // ERROR: unique_ptr<Button> is not covariant with unique_ptr<Widget>
    // std::unique_ptr<Button> clone() const override {
    //     return std::make_unique<Button>(*this);
    // }

    // GOOD: MUST use base type:
    std::unique_ptr<Widget> clone() const override {
        return std::make_unique<Button>(*this);
    }
};
```

**NVI Workaround:**

The solution is the same split used in the Prototype pattern file: an internal virtual function handles the covariant raw pointer (which works), and a public non-virtual wrapper converts it to a `unique_ptr` before the caller ever sees it.

```cpp
#include <iostream>
#include <memory>
#include <string>

class Widget {
protected:
    virtual Widget* clone_raw() const = 0;  // raw pointer: supports covariance

public:
    virtual ~Widget() = default;

    // Non-virtual public interface: returns smart pointer
    std::unique_ptr<Widget> clone() const {
        return std::unique_ptr<Widget>(clone_raw());
    }

    virtual void render() const = 0;
};

class Button : public Widget {
    std::string label_;
protected:
    // GOOD: Covariant raw pointer
    Button* clone_raw() const override {
        return new Button(*this);
    }

public:
    explicit Button(std::string l) : label_(std::move(l)) {}

    void render() const override {
        std::cout << "[Button: " << label_ << "]\n";
    }

    // Typed clone - available when you know it's a Button
    std::unique_ptr<Button> clone_button() const {
        return std::unique_ptr<Button>(clone_raw());
    }
};

class TextBox : public Widget {
    std::string text_;
protected:
    TextBox* clone_raw() const override {
        return new TextBox(*this);
    }

public:
    explicit TextBox(std::string t) : text_(std::move(t)) {}

    void render() const override {
        std::cout << "[TextBox: \"" << text_ << "\"]\n";
    }
};

int main() {
    Button btn("OK");
    TextBox txt("Hello");

    // Through base pointer - unique_ptr<Widget>
    Widget* w = &btn;
    auto w_clone = w->clone();
    w_clone->render();

    // Through derived type - unique_ptr<Button>!
    auto btn_clone = btn.clone_button();
    btn_clone->render();

    // Polymorphic collection
    auto txt_clone = txt.clone();
    txt_clone->render();
}
// Expected output:
//   [Button: OK]
//   [Button: OK]
//   [TextBox: "Hello"]
```

The raw pointer lives for exactly one line inside each `clone_raw()` - it's immediately wrapped. Callers interact entirely with `unique_ptr`, so there's no opportunity to accidentally leak or double-delete.

---

## Notes

- **Covariance has been in C++ since C++98** - it's not a modern feature, but it's underused.
- **The vtable mechanism:** When covariant returns are used, the compiler may adjust the returned pointer (thunk) to convert from `Derived*` to `Base*` at the vtable level. This has near-zero cost.
- **Multiple inheritance complicates covariance:** With virtual inheritance, pointer adjustments become more expensive (offset calculations at runtime).
- **Java and C#** also support covariant return types (Java since version 5). C++ has had it longer.
- **`std::clone_ptr` doesn't exist** in the standard library - it's a commonly wished-for utility. Libraries like `polymorphic_value` (P0201) propose it.
