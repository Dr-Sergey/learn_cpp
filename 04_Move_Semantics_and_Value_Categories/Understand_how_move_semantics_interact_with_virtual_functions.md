# Understand How Move Semantics Interact with Virtual Functions

**Category:** Move Semantics and Value Categories  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/move_constructor>  

---

## Topic Overview

### The Core Problem

Constructors (including move constructors) **cannot be virtual** in C++. This creates a tension with polymorphic types where we want to copy/move through a base pointer.

The reason constructors cannot be virtual is fundamental: virtual dispatch requires a vtable pointer, and the vtable pointer is set up *during* construction. While the constructor is running, the object does not yet have its final type identity - so there is nothing to dispatch through. The type must be fully known at construction time.

| Operation | Can be virtual? | Workaround |
| --- | :---: | --- |
| Destructor | Yes | (always make virtual for polymorphic bases) |
| Copy constructor | No | `virtual clone()` |
| Move constructor | No | `virtual clone()` with move semantics |
| Assignment operators | No (well, technically yes but almost never done) | Virtual `assign()` method |

### Key Interactions

- Declaring a **virtual destructor** suppresses implicit move operations (Rule of Five triggered)
- `= default` can restore move operations when a virtual destructor is declared
- Slicing occurs if you copy/move a Derived into a Base by value

---

## Self-Assessment

### Q1: Show that a virtual move constructor is not possible and explain the design implication

This example shows the language syntax that is illegal (in comments) and then demonstrates what you can do instead. The design implication is important: without virtual construction, you cannot polymorphically duplicate an object through a base pointer without extra machinery:

```cpp
#include <iostream>
#include <memory>
#include <string>

class Shape {
public:
    virtual ~Shape() = default;

    // Constructors CANNOT be virtual:
    // - The type must be known at construction time
    // - Virtual dispatch requires a vtable, which doesn't exist yet during construction
    // - C++ constructors build the vtable incrementally (base first, then derived)

    // virtual Shape(Shape&& other);     // ERROR: constructor can't be virtual
    // virtual Shape(const Shape& other); // ERROR: same reason

    virtual std::string name() const = 0;
    virtual double area() const = 0;
};

class Circle : public Shape {
    double radius_;
public:
    Circle(double r) : radius_(r) {}

    // Regular (non-virtual) move constructor -- works fine
    Circle(Circle&& other) noexcept : radius_(other.radius_) {
        std::cout << "  Circle move ctor\n";
    }

    std::string name() const override { return "Circle"; }
    double area() const override { return 3.14159 * radius_ * radius_; }
    double radius() const { return radius_; }
};

class Rectangle : public Shape {
    double w_, h_;
public:
    Rectangle(double w, double h) : w_(w), h_(h) {}

    std::string name() const override { return "Rectangle"; }
    double area() const override { return w_ * h_; }
};

// DESIGN IMPLICATION: Without virtual constructors, we can't do:
//   Shape* old = new Circle(5);
//   Shape* copy = new ???(*old);  // What type to construct? Can't know at compile time!

int main() {
    std::cout << "=== Virtual Move Constructor: Impossible ===\n\n";

    Circle c(5.0);
    std::cout << "1. Non-virtual move works for concrete types:\n";
    Circle c2 = std::move(c);  // Fine -- we know the type
    std::cout << "  c2: " << c2.name() << ", area=" << c2.area() << "\n";

    // But through a base pointer, we can't move:
    std::unique_ptr<Shape> sp = std::make_unique<Circle>(10.0);
    // Shape s2 = std::move(*sp);  // SLICING: only base part copied, Circle data lost!

    std::cout << "\n2. Slicing danger:\n";
    // Shape base_copy = *sp;   // Would compile but SLICES -- only Shape part
    std::cout << "  Can't polymorphically copy/move through base pointer\n";
    std::cout << "  Solution: virtual clone() method (see Q2)\n";

    return 0;
}
```

The slicing problem is what makes this tricky in practice. If you accidentally assign `*sp` into a `Shape` object, it compiles - but silently discards all the derived-class data. The solution is the `clone()` idiom shown in Q2.

### Q2: Implement a `clone()` virtual method as the idiomatic alternative to virtual copy/move

The `clone()` pattern gives you polymorphic copying through a base pointer. Each derived class overrides `clone()` and constructs a copy of itself using `std::make_unique<DerivedType>(*this)`. This way the correct constructor is always called because each derived class knows its own type:

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

class Animal {
public:
    virtual ~Animal() = default;

    // Virtual clone -- the idiomatic replacement for virtual copy constructor
    virtual std::unique_ptr<Animal> clone() const = 0;

    // Virtual "move-clone" -- optional, for when source can be consumed
    virtual std::unique_ptr<Animal> move_clone() {
        return clone();  // Default: just copy (safe fallback)
    }

    virtual std::string speak() const = 0;
    virtual std::string name() const = 0;
};

class Dog : public Animal {
    std::string name_;
    int tricks_;
public:
    Dog(std::string name, int tricks)
        : name_(std::move(name)), tricks_(tricks) {
        std::cout << "  [+] Dog(" << name_ << ", " << tricks_ << " tricks)\n";
    }

    Dog(const Dog& other) : name_(other.name_), tricks_(other.tricks_) {
        std::cout << "  [copy] Dog(" << name_ << ")\n";
    }

    Dog(Dog&& other) noexcept
        : name_(std::move(other.name_)), tricks_(other.tricks_) {
        std::cout << "  [move] Dog(" << name_ << ")\n";
    }

    // Clone: polymorphic deep copy
    std::unique_ptr<Animal> clone() const override {
        return std::make_unique<Dog>(*this);  // Uses copy ctor
    }

    // Move-clone: polymorphic move
    std::unique_ptr<Animal> move_clone() override {
        return std::make_unique<Dog>(std::move(*this));  // Uses move ctor
    }

    std::string speak() const override { return "Woof!"; }
    std::string name() const override { return name_; }
};

class Cat : public Animal {
    std::string name_;
public:
    Cat(std::string name) : name_(std::move(name)) {
        std::cout << "  [+] Cat(" << name_ << ")\n";
    }
    Cat(const Cat& other) : name_(other.name_) {
        std::cout << "  [copy] Cat(" << name_ << ")\n";
    }

    std::unique_ptr<Animal> clone() const override {
        return std::make_unique<Cat>(*this);
    }

    std::string speak() const override { return "Meow!"; }
    std::string name() const override { return name_; }
};

// Deep-copy a polymorphic collection
std::vector<std::unique_ptr<Animal>>
clone_collection(const std::vector<std::unique_ptr<Animal>>& src) {
    std::vector<std::unique_ptr<Animal>> result;
    result.reserve(src.size());
    for (const auto& a : src) {
        result.push_back(a->clone());  // Polymorphic deep copy!
    }
    return result;
}

int main() {
    std::cout << "=== Virtual clone() Pattern ===\n\n";

    std::vector<std::unique_ptr<Animal>> zoo;
    zoo.push_back(std::make_unique<Dog>("Rex", 5));
    zoo.push_back(std::make_unique<Cat>("Whiskers"));

    std::cout << "\nCloning the zoo:\n";
    auto zoo2 = clone_collection(zoo);

    std::cout << "\nOriginal zoo:\n";
    for (const auto& a : zoo)
        std::cout << "  " << a->name() << " says " << a->speak() << "\n";

    std::cout << "\nCloned zoo:\n";
    for (const auto& a : zoo2)
        std::cout << "  " << a->name() << " says " << a->speak() << "\n";

    std::cout << "\nMove-clone example:\n";
    auto moved_dog = zoo[0]->move_clone();
    std::cout << "  Moved: " << moved_dog->name() << "\n";

    return 0;
}
```

The `clone_collection` function is worth studying: it iterates through a `vector<unique_ptr<Animal>>` and calls `a->clone()` on each element. Even though the pointer type is `Animal*`, each call dispatches to the correct derived `clone()` override - that is virtual dispatch working exactly as intended.

### Q3: Explain why a polymorphic base class should have a virtual destructor and how that interacts with move

The virtual destructor requirement is about correctness. Without it, deleting a derived object through a base pointer is undefined behavior - the wrong destructor runs and resources leak. But there is a subtlety with move semantics: declaring *any* user-provided destructor (even `= default`) suppresses the implicitly generated move operations. You need to explicitly default them to get real moves back:

```cpp
#include <iostream>
#include <memory>
#include <type_traits>

// A polymorphic base WITHOUT virtual destructor -- DANGEROUS
struct BadBase {
    // ~BadBase() {}  // Non-virtual! (the default)
    virtual void f() {}
};

struct BadDerived : BadBase {
    int* data = new int[1000];
    ~BadDerived() {
        std::cout << "  ~BadDerived: freeing data\n";
        delete[] data;
    }
};

// A polymorphic base WITH virtual destructor -- CORRECT
struct GoodBase {
    virtual ~GoodBase() = default;  // Virtual destructor

    // PROBLEM: Declaring virtual destructor suppresses implicit move!
    // The compiler sees a user-declared destructor (even = default) and
    // does not generate move ctor/assignment (Rule of Five).
    //
    // SOLUTION: Explicitly default them:
    GoodBase() = default;
    GoodBase(const GoodBase&) = default;
    GoodBase& operator=(const GoodBase&) = default;
    GoodBase(GoodBase&&) = default;
    GoodBase& operator=(GoodBase&&) = default;
    virtual void f() {}
};

struct GoodDerived : GoodBase {
    std::string name = "derived";
    // Inherits move operations from GoodBase
};

int main() {
    std::cout << "=== Virtual Destructor and Move Interaction ===\n\n";

    // Without virtual destructor: UNDEFINED BEHAVIOR when deleting through base
    std::cout << "1. Without virtual destructor (BAD):\n";
    {
        BadBase* p = new BadDerived();
        delete p;  // UB: ~BadDerived not called -> memory leak!
        // Only ~BadBase runs (non-virtual dispatch)
        std::cout << "  (BadDerived destructor was NOT called -> LEAK)\n";
    }

    // With virtual destructor: correct polymorphic deletion
    std::cout << "\n2. With virtual destructor (GOOD):\n";
    {
        std::unique_ptr<GoodBase> p = std::make_unique<GoodDerived>();
        // ~GoodDerived correctly called through virtual dispatch
    }

    // Virtual destructor suppresses move -- verify with type traits
    std::cout << "\n3. Move suppression check:\n";

    struct NoVirtDtor {
        // No destructor declared -> move is implicit
        std::string data;
    };

    struct VirtDtorNoDefault {
        virtual ~VirtDtorNoDefault() {}  // User-provided -> suppresses move
        std::string data;
    };

    struct VirtDtorWithDefault {
        virtual ~VirtDtorWithDefault() = default;
        VirtDtorWithDefault() = default;
        VirtDtorWithDefault(VirtDtorWithDefault&&) = default;
        VirtDtorWithDefault& operator=(VirtDtorWithDefault&&) = default;
        VirtDtorWithDefault(const VirtDtorWithDefault&) = default;
        VirtDtorWithDefault& operator=(const VirtDtorWithDefault&) = default;
        std::string data;
    };

    std::cout << "  NoVirtDtor move:       "
              << std::is_move_constructible_v<NoVirtDtor> << "\n";          // 1
    std::cout << "  VirtDtorNoDefault move: "
              << std::is_move_constructible_v<VirtDtorNoDefault> << "\n";   // 1 (falls back to copy!)
    std::cout << "  VirtDtorWithDefault:    "
              << std::is_nothrow_move_constructible_v<VirtDtorWithDefault> << "\n";  // 1 (true move)

    return 0;
}
```

Notice the subtle point with `VirtDtorNoDefault`: `is_move_constructible` still returns true because the class falls back to copying when moved. The copy constructor acts as a move constructor in that case. But `is_nothrow_move_constructible` would return false, and the performance is copy-level rather than move-level. The `VirtDtorWithDefault` example with all five special members explicitly defaulted is the pattern to follow for any polymorphic base class.

---

## Notes

- C++ constructors cannot be virtual - use `virtual clone()` for polymorphic copying.
- Always declare a **virtual destructor** in polymorphic base classes.
- Virtual destructor **suppresses** implicit move generation - explicitly `= default` all five special members.
- Slicing occurs when copying a Derived object into a Base by value - always use pointers/references.
- Consider making polymorphic base classes non-copyable to prevent accidental slicing.
