# Use std::unique_ptr as the Default Ownership Smart Pointer

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/unique_ptr>  

---

## Topic Overview

### Why `unique_ptr` Is the Default Choice

The short answer is that `unique_ptr` gives you all the safety of automatic cleanup with zero runtime cost compared to a raw pointer. If the table below feels like a lot, remember the rule: reach for `unique_ptr` first and only upgrade to `shared_ptr` if you genuinely need multiple independent owners.

| Feature | `unique_ptr` | `shared_ptr` | Raw pointer |
| --- | --- | --- | --- |
| **Overhead** | Zero (same as raw ptr) | Ref count + control block | Zero |
| **Ownership** | Exclusive | Shared | Unclear |
| **Automatic cleanup** | Yes | Yes | No |
| **Copyable** | No (move-only) | Yes | Yes |
| **Size** | `sizeof(T*)` | `2 x sizeof(T*)` | `sizeof(T*)` |

### Core Usage

The three things you most need to know are how to create one, how to transfer it, and what happens when it goes out of scope:

```cpp
// Creation
auto p = std::make_unique<Widget>(42);    // Preferred
std::unique_ptr<Widget> p2(new Widget);   // OK but less safe

// Transfer ownership
auto p3 = std::move(p);   // p is now nullptr
// auto p4 = p3;           // ERROR: can't copy

// Automatic cleanup at scope exit
{
    auto temp = std::make_unique<Resource>();
}  // Resource destroyed here
```

---

## Self-Assessment

### Q1: Replace every raw `new`/`delete` pair in a class with `unique_ptr` and verify the destructor becomes defaultable

The "Before" and "After" classes here do the same job, but the `unique_ptr` version eliminates the hand-written destructor and gets move semantics for free. That is the Rule of Zero in action.

```cpp
#include <iostream>
#include <memory>
#include <string>

// BEFORE: Raw pointers - manual memory management
class WidgetOld {
    int* data_;
    std::string* name_;
public:
    WidgetOld(int val, std::string name)
        : data_(new int(val)), name_(new std::string(std::move(name))) {}

    // Must write destructor - leak if forgotten!
    ~WidgetOld() {
        delete data_;
        delete name_;
    }

    // Must write copy/move or delete them - Rule of Three/Five
    WidgetOld(const WidgetOld&) = delete;
    WidgetOld& operator=(const WidgetOld&) = delete;

    void print() const {
        std::cout << "  " << *name_ << " = " << *data_ << "\n";
    }
};

// AFTER: unique_ptr - automatic cleanup, defaultable destructor
class WidgetNew {
    std::unique_ptr<int> data_;
    std::unique_ptr<std::string> name_;
public:
    WidgetNew(int val, std::string name)
        : data_(std::make_unique<int>(val))
        , name_(std::make_unique<std::string>(std::move(name))) {}

    // Destructor is DEFAULTABLE - unique_ptr cleans up automatically
    ~WidgetNew() = default;

    // Move operations are auto-generated (unique_ptr is movable)
    WidgetNew(WidgetNew&&) = default;
    WidgetNew& operator=(WidgetNew&&) = default;

    void print() const {
        std::cout << "  " << *name_ << " = " << *data_ << "\n";
    }
};

int main() {
    std::cout << "=== Before (raw pointers) ===\n";
    {
        WidgetOld w(42, "answer");
        w.print();
    }  // Manual delete in destructor

    std::cout << "\n=== After (unique_ptr) ===\n";
    {
        WidgetNew w(42, "answer");
        w.print();

        // Move works automatically
        WidgetNew w2 = std::move(w);
        w2.print();
    }  // Automatic cleanup - no manual delete needed

    std::cout << "\nBenefits:\n";
    std::cout << "  - No manual destructor needed\n";
    std::cout << "  - Exception-safe (no leak if constructor throws)\n";
    std::cout << "  - Move semantics for free\n";
    std::cout << "  - Follows Rule of Zero\n";

    return 0;
}
```

`WidgetNew` will never leak even if the second `make_unique` in its constructor throws - the first `unique_ptr` cleans itself up during stack unwinding. That is why `make_unique` is preferred over `new`.

### Q2: Explain why `unique_ptr` cannot be copied but can be moved, and show transfer of ownership

The reason copying is deleted is fundamental: `unique_ptr` expresses *exclusive* ownership. Two `unique_ptr` objects over the same resource would both try to delete it - that is a double-free. Moving transfers the ownership, leaving the source as `nullptr`.

```cpp
#include <iostream>
#include <memory>
#include <vector>

struct Resource {
    int id;
    Resource(int i) : id(i) { std::cout << "  [+] Resource " << id << "\n"; }
    ~Resource() { std::cout << "  [-] Resource " << id << "\n"; }
};

// unique_ptr represents EXCLUSIVE ownership:
// - Only ONE unique_ptr can own a given resource at a time
// - Copying would create TWO owners -> both would delete -> double-free!
// - Moving TRANSFERS ownership: source becomes nullptr, dest takes over

void take_ownership(std::unique_ptr<Resource> r) {
    std::cout << "  Received resource " << r->id << "\n";
    // r destroyed at function exit
}

std::unique_ptr<Resource> create(int id) {
    return std::make_unique<Resource>(id);  // Moves out via RVO
}

int main() {
    std::cout << "=== unique_ptr: move-only ownership ===\n\n";

    // Create
    auto r1 = std::make_unique<Resource>(1);

    // Copy is deleted:
    // auto r2 = r1;  // ERROR: use of deleted function

    // Move transfers ownership
    std::cout << "Before move: r1 owns " << r1->id << "\n";
    auto r2 = std::move(r1);
    std::cout << "After move: r1 is " << (r1 ? "valid" : "nullptr") << "\n";
    std::cout << "After move: r2 owns " << r2->id << "\n\n";

    // Transfer to function
    std::cout << "Transferring to function:\n";
    take_ownership(std::move(r2));
    std::cout << "r2 is " << (r2 ? "valid" : "nullptr") << "\n\n";

    // Factory function - move out
    std::cout << "Factory function:\n";
    auto r3 = create(3);
    std::cout << "r3 owns " << r3->id << "\n\n";

    // Store in vector (requires move)
    std::cout << "Vector of unique_ptr:\n";
    std::vector<std::unique_ptr<Resource>> vec;
    vec.push_back(std::make_unique<Resource>(10));
    vec.push_back(std::make_unique<Resource>(20));
    vec.push_back(std::move(r3));  // Move r3 into vector

    std::cout << "Vector has " << vec.size() << " resources\n\n";
    std::cout << "Destroying vector:\n";
    vec.clear();

    return 0;
}
```

After the move, `r1` is `nullptr` - it no longer owns anything. That is the "exclusive" part: there is always exactly zero or one owner.

### Q3: Write a factory function returning `unique_ptr<Base>` that correctly destroys derived objects

This pattern matters whenever you combine polymorphism with `unique_ptr`. The virtual destructor in the base class ensures that when `unique_ptr<Shape>` is destroyed it calls the derived destructor, not just `~Shape`.

```cpp
#include <iostream>
#include <memory>
#include <string>

class Shape {
public:
    // CRITICAL: virtual destructor for correct polymorphic deletion
    virtual ~Shape() { std::cout << "  ~Shape\n"; }
    virtual void draw() const = 0;
    virtual double area() const = 0;
};

class Circle : public Shape {
    double radius_;
public:
    Circle(double r) : radius_(r) {
        std::cout << "  Circle(r=" << radius_ << ") created\n";
    }
    ~Circle() override { std::cout << "  ~Circle(r=" << radius_ << ")\n"; }
    void draw() const override {
        std::cout << "  Drawing circle r=" << radius_ << "\n";
    }
    double area() const override { return 3.14159 * radius_ * radius_; }
};

class Rectangle : public Shape {
    double w_, h_;
public:
    Rectangle(double w, double h) : w_(w), h_(h) {
        std::cout << "  Rectangle(" << w_ << "x" << h_ << ") created\n";
    }
    ~Rectangle() override { std::cout << "  ~Rectangle(" << w_ << "x" << h_ << ")\n"; }
    void draw() const override {
        std::cout << "  Drawing rect " << w_ << "x" << h_ << "\n";
    }
    double area() const override { return w_ * h_; }
};

// Factory function - returns unique_ptr<Base>
// Derived destructor called correctly thanks to virtual ~Shape()
std::unique_ptr<Shape> create_shape(const std::string& type) {
    if (type == "circle")    return std::make_unique<Circle>(5.0);
    if (type == "rectangle") return std::make_unique<Rectangle>(3.0, 4.0);
    return nullptr;
}

int main() {
    std::cout << "=== Factory with unique_ptr<Base> ===\n\n";

    auto s1 = create_shape("circle");
    auto s2 = create_shape("rectangle");

    std::cout << "\nUsing shapes polymorphically:\n";
    for (const auto& s : {s1.get(), s2.get()}) {
        if (s) {
            s->draw();
            std::cout << "  Area: " << s->area() << "\n";
        }
    }

    std::cout << "\nDestruction (virtual dtor ensures correct cleanup):\n";
    s1.reset();  // ~Circle then ~Shape
    s2.reset();  // ~Rectangle then ~Shape

    // Without virtual destructor: ONLY ~Shape would be called -> UB/leak

    return 0;
}
// Expected output:
//   Circle(r=5) created
//   Rectangle(3x4) created
//
// Using shapes polymorphically:
//   Drawing circle r=5
//   Area: 78.5397
//   Drawing rect 3x4
//   Area: 12
//
// Destruction (virtual dtor ensures correct cleanup):
//   ~Circle(r=5)
//   ~Shape
//   ~Rectangle(3x4)
//   ~Shape
```

Without the virtual destructor, `delete` through a `Shape*` would only invoke `~Shape` and skip the derived class cleanup entirely - that is undefined behavior and a resource leak for anything the derived class owns.

---

## Notes

- **Default to `unique_ptr`** - it has zero overhead compared to raw pointers.
- Use `shared_ptr` only when you genuinely need shared ownership (multiple independent owners).
- Always ensure base classes have a **virtual destructor** when using `unique_ptr<Base>`.
- `unique_ptr` enables the Rule of Zero - classes with only `unique_ptr` members need no custom destructor.
- Use `make_unique` (C++14) for exception safety and to avoid writing `new`.
