# Implement the Prototype pattern using virtual clone methods

**Category:** Modern OOP Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/virtual>  

---

## Topic Overview

The **Prototype pattern** creates new objects by copying an existing *prototype* instance. In C++, this typically means a `virtual clone()` method that returns a `unique_ptr<Base>`, letting you deep-copy a polymorphic object through a base pointer without knowing its concrete type.

### Why Clone is Needed

The core problem is that you often hold a `Base*` pointing to some derived object, and you need a copy of whatever it actually is. A plain copy constructor won't help you here because C++ resolves copy construction statically - it sees a `Base` and copies a `Base`, slicing off the derived part in the process.

Here's the situation that forces the issue:

```cpp
Base* b = get_shape();     // could be Circle, Square, Triangle...
Base* copy = ???;          // Can't just do *b - that SLICES!

// copy constructor doesn't work polymorphically:
// BAD: SLICES to Base
Base copy2 = *b;

// We need:
// GOOD: returns unique_ptr<Base> holding the actual derived type
auto copy3 = b->clone();
```

By delegating the copy to a virtual function, each derived class copies itself and hands you back the result wrapped in a smart pointer.

### Virtual Clone Structure

The shape of the solution is simple: a pure virtual `clone()` in the base, one override per derived class, each calling its own copy constructor.

```cpp
┌─────────────────────────────────┐
│         Shape (abstract)         │
│  virtual ~Shape() = default;     │
│  virtual unique_ptr<Shape>       │
│    clone() const = 0;            │
└──────────┬──────────────────────┘
           │
    ┌──────┴──────┐
    v             v
  Circle        Square
  clone() {     clone() {
    return        return
    make_unique   make_unique
    <Circle>      <Square>
    (*this);      (*this);
  }             }
```

---

## Self-Assessment

### Q1: Write a virtual `clone() -> unique_ptr<Base>` in a base class and override in each derived class

Each derived class implements `clone()` by invoking its own copy constructor inside `make_unique`. That single line is all you need - the copy constructor does the deep copy, and `make_unique` wraps the result in a `unique_ptr<Shape>`.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

class Shape {
public:
    virtual ~Shape() = default;
    virtual std::unique_ptr<Shape> clone() const = 0;
    virtual void draw() const = 0;
    virtual std::string name() const = 0;
};

class Circle : public Shape {
    double radius_;
public:
    explicit Circle(double r) : radius_(r) {}

    std::unique_ptr<Shape> clone() const override {
        return std::make_unique<Circle>(*this);  // copy constructor
    }

    void draw() const override {
        std::cout << "Circle(r=" << radius_ << ")\n";
    }
    std::string name() const override { return "Circle"; }
};

class Rectangle : public Shape {
    double w_, h_;
public:
    Rectangle(double w, double h) : w_(w), h_(h) {}

    std::unique_ptr<Shape> clone() const override {
        return std::make_unique<Rectangle>(*this);
    }

    void draw() const override {
        std::cout << "Rectangle(" << w_ << "x" << h_ << ")\n";
    }
    std::string name() const override { return "Rectangle"; }
};

class Triangle : public Shape {
    double base_, height_;
public:
    Triangle(double b, double h) : base_(b), height_(h) {}

    std::unique_ptr<Shape> clone() const override {
        return std::make_unique<Triangle>(*this);
    }

    void draw() const override {
        std::cout << "Triangle(b=" << base_ << ", h=" << height_ << ")\n";
    }
    std::string name() const override { return "Triangle"; }
};

// Deep-copy an entire collection through base pointers
std::vector<std::unique_ptr<Shape>> clone_all(
    const std::vector<std::unique_ptr<Shape>>& shapes)
{
    std::vector<std::unique_ptr<Shape>> copies;
    copies.reserve(shapes.size());
    for (const auto& s : shapes)
        copies.push_back(s->clone());
    return copies;
}

int main() {
    std::vector<std::unique_ptr<Shape>> originals;
    originals.push_back(std::make_unique<Circle>(5.0));
    originals.push_back(std::make_unique<Rectangle>(3.0, 4.0));
    originals.push_back(std::make_unique<Triangle>(6.0, 2.0));

    std::cout << "Originals:\n";
    for (const auto& s : originals) s->draw();

    // Clone the entire collection
    auto copies = clone_all(originals);

    std::cout << "\nClones (independent copies):\n";
    for (const auto& s : copies) s->draw();

    // Verify independence: addresses differ
    std::cout << "\nOriginal[0] at: " << originals[0].get()
              << "\nClone[0] at:    " << copies[0].get() << "\n";
}
// Expected output:
//   Originals:
//   Circle(r=5)
//   Rectangle(3x4)
//   Triangle(b=6, h=2)
//
//   Clones (independent copies):
//   Circle(r=5)
//   Rectangle(3x4)
//   Triangle(b=6, h=2)
//
//   Original[0] at: 0x1234
//   Clone[0] at:    0x5678   <- different address, independent copy
```

Notice that `clone_all` doesn't know about `Circle`, `Rectangle`, or `Triangle` at all - it just calls `clone()` on each base pointer and gets back the right type every time. That's the whole point of virtual dispatch working for you here.

---

### Q2: Show that covariant return types (`unique_ptr<Derived>`) don't work directly and explain the workaround

This is a genuinely tricky corner of the language, so it's worth slowing down on it. The reason covariant returns don't work with smart pointers comes down to one fact: `unique_ptr<Derived>` and `unique_ptr<Base>` are completely unrelated types. The C++ covariance rule only applies to raw pointer types where there's an actual inheritance relationship.

```cpp
// Covariant return types work with RAW pointers:
class Base {
public:
    virtual Base* clone_raw() const { return new Base(*this); }
};

class Derived : public Base {
public:
    // GOOD: Derived* is covariant with Base*
    Derived* clone_raw() const override { return new Derived(*this); }
};

// But NOT with smart pointers:
class Base2 {
public:
    virtual std::unique_ptr<Base2> clone() const = 0;
};

class Derived2 : public Base2 {
public:
    // ERROR: unique_ptr<Derived2> is NOT covariant with unique_ptr<Base2>
    // (they are unrelated types - covariance only works for pointers/references)
    // std::unique_ptr<Derived2> clone() const override { ... }

    // GOOD: MUST return unique_ptr<Base2>:
    std::unique_ptr<Base2> clone() const override {
        return std::make_unique<Derived2>(*this);
    }
};
```

**Workaround - Non-Virtual Interface (NVI) for Covariant Clone:**

The trick is to split the job in two: a private virtual function returns a raw covariant pointer (which the language does support), and a public non-virtual wrapper converts that raw pointer into a safe `unique_ptr`. The caller only ever sees the smart pointer version.

```cpp
#include <iostream>
#include <memory>

class Shape {
protected:
    // Private virtual implementation
    virtual Shape* clone_impl() const = 0;

public:
    virtual ~Shape() = default;

    // Public non-virtual wrapper returns unique_ptr<Shape>
    std::unique_ptr<Shape> clone() const {
        return std::unique_ptr<Shape>(clone_impl());
    }
};

class Circle : public Shape {
    double radius_;
protected:
    // Covariant raw pointer - works!
    Circle* clone_impl() const override {
        return new Circle(*this);
    }

public:
    explicit Circle(double r) : radius_(r) {}

    // Non-virtual: returns the derived type for KNOWN-type usage
    std::unique_ptr<Circle> clone_circle() const {
        return std::unique_ptr<Circle>(clone_impl());
    }

    void draw() const { std::cout << "Circle(r=" << radius_ << ")\n"; }
};

int main() {
    Circle c(5.0);

    // Through base pointer - returns unique_ptr<Shape>
    Shape* base = &c;
    auto copy1 = base->clone();  // unique_ptr<Shape>

    // Through derived type - returns unique_ptr<Circle>
    auto copy2 = c.clone_circle();  // unique_ptr<Circle>!
    copy2->draw();

    std::cout << "copy1 type: Shape* (but holds Circle)\n";
    std::cout << "copy2 type: Circle* (directly)\n";
}
// Expected output:
//   Circle(r=5)
//   copy1 type: Shape* (but holds Circle)
//   copy2 type: Circle* (directly)
```

The beauty of this pattern is that the caller never touches a raw pointer - the raw pointer is entirely hidden inside the class. When you need a typed clone (`unique_ptr<Circle>`), `clone_circle()` gives you one; when you only have a base pointer, `clone()` gives you a `unique_ptr<Shape>` that still holds the real object.

---

### Q3: Compare virtual clone with a template-based copy factory for the same hierarchy

You'll sometimes see people reach for a template `copy_factory` function instead of a virtual `clone()`. Each approach has a real place - the table below captures when to prefer which.

| Aspect | Virtual `clone()` | Template Copy Factory |
| --- | --- | --- |
| **Requires virtual?** | Yes | No |
| **Works through base pointer?** | Yes | No - needs concrete type |
| **Boilerplate** | `clone()` override in every derived class | One template function |
| **Runtime cost** | Virtual dispatch | Zero (inlined) |
| **Open for extension?** | Yes - just add new derived class | Yes |
| **Forgettable?** | Yes - can forget to override (=0 helps) | No - compile-time |

The CRTP `Cloneable` helper shown below is the best of both worlds: the virtual dispatch stays, but you don't have to write the `clone()` override by hand in every single derived class.

```cpp
#include <iostream>
#include <memory>
#include <type_traits>

// Template-based copy factory - no virtual functions needed
template <typename T>
std::unique_ptr<T> copy_factory(const T& obj) {
    static_assert(std::is_copy_constructible_v<T>,
                  "T must be copy constructible");
    return std::make_unique<T>(obj);
}

// CRTP approach to eliminate clone() boilerplate:
template <typename Base, typename Derived>
class Cloneable : public Base {
public:
    using Base::Base;

    std::unique_ptr<Base> clone() const override {
        return std::make_unique<Derived>(static_cast<const Derived&>(*this));
    }
};

class Animal {
public:
    virtual ~Animal() = default;
    virtual std::unique_ptr<Animal> clone() const = 0;
    virtual void speak() const = 0;
};

// No clone() boilerplate - CRTP handles it!
class Dog : public Cloneable<Animal, Dog> {
    std::string name_;
public:
    explicit Dog(std::string n) : name_(std::move(n)) {}
    void speak() const override {
        std::cout << name_ << " says: Woof!\n";
    }
};

class Cat : public Cloneable<Animal, Cat> {
    std::string name_;
public:
    explicit Cat(std::string n) : name_(std::move(n)) {}
    void speak() const override {
        std::cout << name_ << " says: Meow!\n";
    }
};

int main() {
    // Virtual clone via CRTP - zero boilerplate per derived class
    std::unique_ptr<Animal> a = std::make_unique<Dog>("Rex");
    auto b = a->clone();
    a->speak();
    b->speak();

    // Template copy factory - only works when type is known
    Dog d("Buddy");
    auto d_copy = copy_factory(d);
    d_copy->speak();
}
// Expected output:
//   Rex says: Woof!
//   Rex says: Woof!
//   Buddy says: Woof!
```

Notice that `Dog` and `Cat` have no `clone()` implementation at all - `Cloneable` writes it for them using the `static_cast<const Derived&>(*this)` trick to access the most-derived type. Adding a new animal to this hierarchy is truly just inheriting from `Cloneable<Animal, NewAnimal>`.

---

## Notes

- **Always make `clone()` pure virtual** (`= 0`) in the base class - this forces every derived class to implement it. Forgetting `clone` in one derived class is a common bug.
- **CRTP Cloneable** eliminates the boilerplate: each derived class just inherits `Cloneable<Base, Derived>` instead of writing `clone()` manually.
- **`clone()` should call the copy constructor**, not default-construct + assign. This preserves invariants established in the constructor.
- **Deep vs shallow clone:** `clone()` using the copy constructor will deep-copy value members but shallow-copy raw pointers. Use smart pointers or implement custom copy constructors for proper deep cloning.
- **In C++23,** `std::copyable_function` and deducing-this further reduce the need for clone boilerplate in type-erased contexts.
