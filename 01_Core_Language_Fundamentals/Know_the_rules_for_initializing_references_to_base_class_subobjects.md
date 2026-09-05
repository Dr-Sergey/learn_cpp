# Know the rules for initializing references to base class subobjects

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/reference_initialization>  

---

## Topic Overview

When a class `Derived` publicly inherits from `Base`, every `Derived` object contains a **base-class subobject** of type `Base`. A reference (or pointer) to `Base` can bind directly to that subobject - this is the foundation of runtime polymorphism in C++.

### Key Rules

| Expression | Effect |
| --- | --- |
| `Base& b = derived;` | `b` refers to the `Base` subobject inside `derived` - **no copy, no slicing** |
| `Base b = derived;` | **Object slicing** - copies only the `Base` part into a brand-new `Base` object |
| `const Base& b = Derived{};` | Lifetime of the temporary is extended to match `b` |
| `Base&& b = Derived{};` | Also extends the temporary's lifetime (rvalue ref) |

### Why References Avoid Slicing

A reference is just an alias for an existing object. When you write `Base& b = d;`, the compiler adjusts the address to point to the `Base` subobject inside `d` (pointer adjustment may occur with multiple or virtual inheritance). No constructor is called, no data is lost.

By contrast, `Base b = d;` invokes the `Base` **copy constructor**, which copies only the `Base` members - any `Derived`-specific data is discarded, and the vtable pointer is that of `Base`.

### Pointer Adjustment

With non-virtual single inheritance the base subobject typically sits at offset 0, so the pointer/reference value is identical. With **multiple inheritance**, the compiler inserts an implicit offset. You don't usually see it, but it matters:

```cpp
struct A { int a; };
struct B { int b; };
struct C : A, B { int c; };

C obj;
A& ra = obj;   // points to offset 0
B& rb = obj;   // points to offset sizeof(A) - pointer adjusted
```

`&ra` and `&rb` have different values even though both "refer to `obj`". The compiler handles this transparently.

### Virtual Dispatch Through Base References

Because the reference still points to the original `Derived` object, virtual function calls dispatch to the `Derived` override. That's what makes polymorphism work:

```cpp
#include <iostream>

struct Animal {
    virtual std::string speak() const { return "..."; }
    virtual ~Animal() = default;
};

struct Dog : Animal {
    std::string speak() const override { return "Woof!"; }
};

void greet(const Animal& a) {
    std::cout << a.speak() << "\n";    // virtual dispatch
}

int main() {
    Dog d;
    greet(d);   // prints "Woof!" - not "..."
}
```

`greet` doesn't know or care what the concrete type is - it just uses the reference and lets the vtable sort it out.

---

## Self-Assessment

### Q1: Explain why `Derived d; Base& b = d;` is well-formed but `Base b = d;` slices the object

**Answer:**

`Base& b = d;` binds the reference directly to `d`'s base-class subobject. No constructor is called; `b` is merely an alias. The original `Derived` object remains intact, virtual dispatch still reaches `Derived` overrides, and all `Derived` data members are preserved.

`Base b = d;` calls `Base`'s copy constructor with the `Base` subobject of `d` as the source. A **new** `Base` object is created on the stack containing only `Base`'s members. All `Derived`-specific members are lost ("sliced off"), and the vtable pointer of `b` is `Base`'s, so virtual calls go to `Base`'s implementations.

You can see both behaviors side by side here:

```cpp
#include <iostream>

struct Base {
    virtual void who() const { std::cout << "Base\n"; }
    virtual ~Base() = default;
};

struct Derived : Base {
    int extra = 42;
    void who() const override { std::cout << "Derived (extra=" << extra << ")\n"; }
};

int main() {
    Derived d;

    Base& ref = d;       // reference - no slicing
    ref.who();           // "Derived (extra=42)"

    Base copy = d;       // slicing copy
    copy.who();          // "Base"
    // copy has no 'extra' member at all
}
```

The reference dispatches to `Derived::who`; the sliced copy dispatches to `Base::who`. Same source object, completely different behavior.

### Q2: Show that binding a reference to a base class subobject does not create a new object

If you're ever unsure whether a reference is an alias or a copy, check the address:

```cpp
#include <iostream>

struct Base { int x = 10; virtual ~Base() = default; };
struct Derived : Base { int y = 20; };

int main() {
    Derived d;
    Base& b = d;

    // 1) Same address - no new object
    std::cout << "Address of d:        " << static_cast<void*>(&d) << "\n";
    std::cout << "Address through b:   " << static_cast<void*>(&b) << "\n";
    std::cout << "Same object? " << (&b == static_cast<Base*>(&d) ? "yes" : "no") << "\n";

    // 2) Mutation through reference is visible on original
    b.x = 99;
    std::cout << "d.x after b.x=99: " << d.x << "\n";  // 99

    // 3) sizeof does not change
    std::cout << "sizeof(Base)=    " << sizeof(Base) << "\n";
    std::cout << "sizeof(Derived)= " << sizeof(Derived) << "\n";
    // b is still a Derived object in memory, just viewed as Base&
}
```

The address comparison and the mutation test both confirm: there's one object in memory, and both `d` and `b` name it.

How it works:

- `&b` and `static_cast<Base*>(&d)` point to the same address - no copy was made.
- Mutating through `b` changes `d`'s member directly.
- The reference is merely an alias, not a new independent object.

### Q3: Demonstrate using a base class reference to call a virtual function and verify dispatch

Polymorphic containers are one of the most common uses of base class references. Here the `print_area` function knows nothing about circles or rectangles - it just calls `area()` through the reference:

```cpp
#include <iostream>
#include <vector>
#include <memory>

struct Shape {
    virtual double area() const = 0;
    virtual std::string name() const = 0;
    virtual ~Shape() = default;
};

struct Circle : Shape {
    double r;
    explicit Circle(double radius) : r(radius) {}
    double area() const override { return 3.14159265 * r * r; }
    std::string name() const override { return "Circle"; }
};

struct Rectangle : Shape {
    double w, h;
    Rectangle(double width, double height) : w(width), h(height) {}
    double area() const override { return w * h; }
    std::string name() const override { return "Rectangle"; }
};

void print_area(const Shape& s) {   // base reference
    std::cout << s.name() << " area = " << s.area() << "\n";
}

int main() {
    Circle c(5.0);
    Rectangle r(3.0, 4.0);

    print_area(c);   // "Circle area = 78.5398"
    print_area(r);   // "Rectangle area = 12"

    // Polymorphic container
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(2.0));
    shapes.push_back(std::make_unique<Rectangle>(6.0, 7.0));

    for (const auto& s : shapes) {
        print_area(*s);   // dispatches to correct override
    }
}
```

`print_area` is written once and works correctly for any type that inherits from `Shape` - now or in the future. That's the power of polymorphism through base class references.

How it works:

- `print_area` takes a `const Shape&` - it never knows the concrete type at compile time.
- The vtable mechanism ensures `s.area()` and `s.name()` call the correct `Derived` override.
- This works identically for references and pointers to base.

---

## Notes

- Always pass polymorphic objects by reference or pointer - passing by value slices.
- `dynamic_cast<Derived&>(base_ref)` can recover the original type (throws `std::bad_cast` on failure).
- With **virtual inheritance**, pointer adjustment can involve runtime vtable lookups - more expensive than simple offsets.
- In C++23, `std::print` can replace `std::cout` for formatted output; the reference rules remain unchanged.
