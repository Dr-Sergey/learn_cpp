# Understand covariant return types and their limitations with smart pointers

**Category:** Modern OOP Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/virtual>  

---

## Topic Overview

**Covariant return types** allow an overriding virtual function to return a pointer/reference to a *more-derived* type than the base function declares. This works with raw pointers and references. It does **NOT** work with smart pointers (`unique_ptr`, `shared_ptr`) because they are unrelated template instantiations, not pointer types.

This file focuses on the **smart pointer limitation** and production-quality workarounds.

### Covariant Rule

The rule in a nutshell: if the base returns `T*`, the override may return `U*` provided `U` derives from `T`. This is well-defined because `U*` is genuinely a subtype of `T*`. The same logic does not extend to templates.

```cpp
Rule: If Base::f() returns T*, then Derived::f() may return U*
      where U publicly inherits from T.

Works:
  Base*    -> Derived*       raw pointer
  Base&    -> Derived&       reference

Does NOT work:
  unique_ptr<Base> -> unique_ptr<Derived>  different template types!
  shared_ptr<Base> -> shared_ptr<Derived>  same reason
```

### The Problem with Smart Pointers

The reason smart pointers can't be covariant is straightforward: they are ordinary class templates, and template instantiations with different type arguments have no inheritance relationship whatsoever, even if those type arguments do inherit from each other.

```cpp
unique_ptr<Base>   and   unique_ptr<Derived>

These are DIFFERENT, UNRELATED types:

  - unique_ptr<Base>    == unique_ptr<Base, default_delete<Base>>
  - unique_ptr<Derived> == unique_ptr<Derived, default_delete<Derived>>

C++ covariance only applies to raw pointers/references.
Template instantiations have no inheritance relationship.
```

---

## Self-Assessment

### Q1: Write a virtual `Base* clone()` in Base and override with `Derived* clone()` in Derived

The covariant raw pointer works cleanly. When you call `clone()` through a `Dog*`, you get back a `Dog*` directly - no cast needed. When you call through an `Animal*`, you get back an `Animal*` that actually points to a `Dog`. The virtual dispatch runs correctly in both cases.

```cpp
#include <iostream>
#include <string>

class Animal {
    std::string species_;
public:
    explicit Animal(std::string s) : species_(std::move(s)) {}
    virtual ~Animal() = default;

    // Virtual clone returning raw pointer
    virtual Animal* clone() const {
        std::cout << "  Animal::clone()\n";
        return new Animal(*this);
    }

    const std::string& species() const { return species_; }
    virtual void speak() const { std::cout << "  [" << species_ << "]\n"; }
};

class Dog : public Animal {
    std::string trick_;
public:
    Dog(std::string s, std::string trick)
        : Animal(std::move(s)), trick_(std::move(trick)) {}

    // GOOD: Covariant return: Dog* overrides Animal*
    Dog* clone() const override {
        std::cout << "  Dog::clone()\n";
        return new Dog(*this);
    }

    void speak() const override {
        std::cout << "  Dog[" << species() << "] knows: " << trick_ << "\n";
    }

    const std::string& trick() const { return trick_; }
};

int main() {
    Dog rex("Shepherd", "shake");

    // Through derived pointer - returns Dog* (covariant)
    Dog* rex_copy = rex.clone();  // Dog* directly, no cast needed
    rex_copy->speak();
    std::cout << "  Trick: " << rex_copy->trick() << "\n";

    // Through base pointer - returns Animal* (but actually Dog*)
    Animal* base = &rex;
    Animal* base_copy = base->clone();  // Returns Animal* (static type)
    base_copy->speak();  // Still calls Dog::speak() (dynamic dispatch)

    delete rex_copy;
    delete base_copy;
}
// Expected output:
//   Dog::clone()
//   Dog[Shepherd] knows: shake
//   Trick: shake
//   Dog::clone()
//   Dog[Shepherd] knows: shake
```

Both calls end up in `Dog::clone()` because the virtual dispatch always finds the most-derived override. The difference is only in the static return type you see at the call site.

---

### Q2: Explain why covariant return types work for raw pointers but not for smart pointers

The reason trips people up because `unique_ptr<Dog>` *can* convert to `unique_ptr<Animal>` (via a move), so it feels like there should be some relationship. But conversion and covariance are not the same thing. Covariance requires an actual inheritance hierarchy between the return types - and `unique_ptr<Dog>` does not inherit from `unique_ptr<Animal>`.

```cpp
Animal*  <- Dog*    Dog publicly inherits Animal
                       The compiler knows Dog* IS-A Animal*

unique_ptr<Animal>  <- unique_ptr<Dog>
  These are DIFFERENT template instantiations:

  - unique_ptr<Animal> is a class
  - unique_ptr<Dog>    is a DIFFERENT class
  - unique_ptr<Dog> does NOT inherit from unique_ptr<Animal>
  - Therefore: NOT covariant
```

**Why it can't be fixed in the language:**

| Reason | Explanation |
| --- | --- |
| **Templates are not covariant** | `Wrapper<Derived>` has no inheritance relationship with `Wrapper<Base>` |
| **Implicit conversion is not covariance** | `unique_ptr<Dog>` can convert TO `unique_ptr<Animal>` (move), but that's a lossy conversion |
| **Ownership semantics differ** | `unique_ptr<Dog>::~unique_ptr()` calls `delete dog_ptr`, while `unique_ptr<Animal>::~unique_ptr()` calls `delete animal_ptr` - different deleters! |
| **Smart pointers are user types** | The compiler has no special knowledge of `unique_ptr` to enable covariance |

Here's the compiler error you'll hit if you try it:

```cpp
#include <memory>
#include <iostream>

class Base {
public:
    virtual ~Base() = default;
    virtual std::unique_ptr<Base> clone() const {
        return std::make_unique<Base>(*this);
    }
};

class Derived : public Base {
public:
    // ERROR: This will NOT compile:
    // std::unique_ptr<Derived> clone() const override { ... }
    // Error: return type is not covariant with Base::clone()

    // GOOD: Must return unique_ptr<Base>:
    std::unique_ptr<Base> clone() const override {
        return std::make_unique<Derived>(*this);
    }
};
```

---

### Q3: Show the workaround: a public non-virtual `clone_ptr()` calls a virtual `do_clone()` returning raw pointer

The NVI (Non-Virtual Interface) pattern solves this cleanly. The internal virtual function returns a raw covariant pointer, which the language does support. The public-facing wrapper immediately wraps it in a `unique_ptr`, so callers never see the raw pointer at all.

```cpp
#include <iostream>
#include <memory>
#include <string>

class Shape {
protected:
    // Private virtual: returns raw covariant pointer
    virtual Shape* do_clone() const = 0;

public:
    virtual ~Shape() = default;

    // Public non-virtual: wraps in unique_ptr
    std::unique_ptr<Shape> clone() const {
        return std::unique_ptr<Shape>(do_clone());
    }

    virtual void draw() const = 0;
};

class Circle : public Shape {
    double radius_;
protected:
    // GOOD: Covariant: Circle* overrides Shape* (raw pointer)
    Circle* do_clone() const override {
        return new Circle(*this);
    }

public:
    explicit Circle(double r) : radius_(r) {}
    void draw() const override {
        std::cout << "Circle(r=" << radius_ << ")\n";
    }

    // Bonus: typed clone for when you KNOW it's a Circle
    std::unique_ptr<Circle> clone_as_circle() const {
        return std::unique_ptr<Circle>(do_clone());
    }
};

class Square : public Shape {
    double side_;
protected:
    Square* do_clone() const override {
        return new Square(*this);
    }

public:
    explicit Square(double s) : side_(s) {}
    void draw() const override {
        std::cout << "Square(s=" << side_ << ")\n";
    }

    std::unique_ptr<Square> clone_as_square() const {
        return std::unique_ptr<Square>(do_clone());
    }
};

int main() {
    // Through base pointer - returns unique_ptr<Shape>
    std::unique_ptr<Shape> shape = std::make_unique<Circle>(5.0);
    auto shape_copy = shape->clone();  // unique_ptr<Shape>
    shape_copy->draw();

    // Through derived type - returns unique_ptr<Circle>!
    Circle c(3.0);
    auto circle_copy = c.clone_as_circle();  // unique_ptr<Circle>
    circle_copy->draw();

    // CRTP alternative: eliminate boilerplate
    std::cout << "\nBoth approaches produce correct clones.\n";
}
// Expected output:
//   Circle(r=5)
//   Circle(r=3)
//
//   Both approaches produce correct clones.
```

**Summary of the NVI workaround:**

```cpp
User calls:           clone()          [non-virtual, returns unique_ptr<Base>]
                         │
                         v
Internally calls:     do_clone()       [virtual, returns Derived* - covariant!]
                         │
                         v
Wrapped:              unique_ptr<Base>(raw Derived*)
```

The key insight is that the raw pointer is only alive for an instant - it's immediately transferred into a `unique_ptr`, so there's no window where it can leak.

---

## Notes

- **This is the standard idiom** for polymorphic cloning in modern C++. Used in Google's Abseil library, LLVM, and many others.
- **CRTP can automate the `do_clone()` override** - see the Prototype pattern file for the `Cloneable<Base, Derived>` CRTP helper.
- **`shared_ptr` has the same limitation** - no covariant returns for `shared_ptr<Derived>` overriding `shared_ptr<Base>`.
- **Virtual destructor required:** Without it, `unique_ptr<Base>` deleting a `Derived` object is undefined behavior.
- **C++ proposal P0847 (deducing this)** in C++23 can simplify some clone patterns but doesn't fully solve the covariance issue.
- **Rust's `Clone` trait** avoids this problem entirely by being a value-based concept - no pointer covariance needed.
