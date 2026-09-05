# Use std::indirect and std::polymorphic (C++26) for value-semantic polymorphism

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/memory/indirect>  

---

## Topic Overview

C++26 introduces `std::indirect<T>` and `std::polymorphic<Base>` to solve a long-standing problem: **owning heap-allocated objects with value semantics** - copy, move, compare, and assign as if they were plain values.

### The Problem They Solve

Before these types existed, every option came with a painful compromise. Here is the landscape:

```cpp
unique_ptr<Shape>  -> Owns, movable, but NOT copyable. No value semantics.
shared_ptr<Shape>  -> Shared ownership, not exclusive. Copy = shallow alias.
Shape* raw         -> No ownership at all.
Shape by value     -> Slicing! Derived data lost.

std::indirect<Shape>     -> Owns a SINGLE concrete Shape. Deep copy. Value semantics.
std::polymorphic<Shape>  -> Owns ANY derived Shape. Deep copy. Polymorphic dispatch.
```

The reason this gap existed so long is that deep copy for a polymorphic type requires knowing the concrete type at copy time - something raw pointers cannot express. The new types encode that information internally so you do not have to.

### Key Distinction

| Type                        | What it wraps          | Copyable? | Polymorphic? |
| --- | --- | --- | --- |
| `std::indirect<T>`          | Exactly type `T`       | Yes (deep) | No          |
| `std::polymorphic<Base>`    | `Base` or any derived  | Yes (deep) | Yes (virtual)|

---

## Self-Assessment

### Q1: Store a polymorphic object by value using std::indirect<Base> and call virtual methods without pointer syntax

This example stores `Circle` and `Rect` objects inside `std::polymorphic<Shape>` wrappers and puts them in a `std::vector`. Notice that the vector itself is fully copyable - `auto shapes_copy = shapes;` just works, doing a deep copy of every polymorphic shape. That single line would be impossible with `unique_ptr`.

**Answer:**

<!-- compile: needs `indirect` (not in the CI standard library yet) -->
```cpp
// C++26 — requires <indirect> / <polymorphic> support
#include <indirect>
#include <polymorphic>
#include <iostream>
#include <string>
#include <vector>

struct Shape {
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual std::string name() const = 0;
};

struct Circle : Shape {
    double radius;
    Circle(double r) : radius(r) {}
    double area() const override { return 3.14159 * radius * radius; }
    std::string name() const override { return "Circle"; }
};

struct Rect : Shape {
    double w, h;
    Rect(double w, double h) : w(w), h(h) {}
    double area() const override { return w * h; }
    std::string name() const override { return "Rect"; }
};

int main() {
    // std::polymorphic<Shape> — stores any derived type with value semantics
    std::polymorphic<Shape> s1 = Circle{5.0};
    std::polymorphic<Shape> s2 = Rect{3.0, 4.0};

    // Call virtual methods — NO pointer syntax (no -> needed)
    std::cout << s1.value().name() << ": " << s1.value().area() << '\n';
    // Circle: 78.5398
    std::cout << s2.value().name() << ": " << s2.value().area() << '\n';
    // Rect: 12

    // Deep copy — s3 is an independent Circle
    auto s3 = s1;  // Copies the Circle, not just the pointer
    // Modifying s3 does NOT affect s1

    // Store in a vector — fully copyable container of polymorphic objects!
    std::vector<std::polymorphic<Shape>> shapes;
    shapes.push_back(Circle{2.0});
    shapes.push_back(Rect{5.0, 6.0});

    // Copy the entire vector (deep copies every shape)
    auto shapes_copy = shapes;

    for (const auto& s : shapes_copy)
        std::cout << s.value().name() << " area=" << s.value().area() << '\n';
}
```

You access the stored object through `.value()`, which returns a reference to the underlying `Shape`. Virtual dispatch works normally from there. You get all the polymorphic behavior without any pointer arithmetic or manual memory management.

### Q2: Explain how indirect provides value semantics (copy, move, comparison) for heap-allocated objects

`std::indirect<T>` is essentially a "deep-copying `unique_ptr`" - it owns a heap-allocated `T`, but unlike `unique_ptr`, it is copyable. Copying creates a new independent heap allocation. Comparison is forwarded to `T`'s own comparison operators. This example walks through each of those behaviors explicitly.

**Answer:**

<!-- compile: needs `indirect` (not in the CI standard library yet) -->
```cpp
#include <indirect>
#include <iostream>
#include <string>
#include <compare>

struct Document {
    std::string title;
    std::string content;

    auto operator<=>(const Document&) const = default;
    bool operator==(const Document&) const = default;
};

int main() {
    // std::indirect<T> = heap-allocated T with value semantics
    std::indirect<Document> doc1{Document{"Report", "Hello world"}};

    // Copy: deep copy of the heap object
    auto doc2 = doc1;  // Allocates NEW Document, copies content
    doc2.value().title = "Modified Copy";

    std::cout << doc1.value().title << '\n';  // "Report" — unchanged!
    std::cout << doc2.value().title << '\n';  // "Modified Copy"
    // Unlike shared_ptr: modifying doc2 does NOT affect doc1

    // Move: transfers ownership (no allocation)
    auto doc3 = std::move(doc1);
    // doc1 is now in a moved-from state
    std::cout << doc3.value().title << '\n';  // "Report"

    // Comparison: delegates to T's operators
    std::indirect<Document> a{Document{"A", "content"}};
    std::indirect<Document> b{Document{"B", "content"}};
    std::indirect<Document> a2{Document{"A", "content"}};

    std::cout << (a == a2) << '\n';  // 1 (compares Document values!)
    std::cout << (a < b) << '\n';    // 1 ("A" < "B")
    // Compare operations are forwarded to Document::operator<=>

    // Assignment: deep copy assignment
    b = a;  // Copies a's Document into b's storage
    std::cout << (a == b) << '\n';  // 1

    /*
    Memory model:
      doc1:  [indirect] --> [heap: Document{"Report", "..."}]
      doc2:  [indirect] --> [heap: Document{"Report", "..."}]  <- independent copy!

    vs unique_ptr:
      up1:   [unique_ptr] --> [heap: Document]
      up2 = up1;  // ERROR: deleted copy constructor

    vs shared_ptr:
      sp1:   [shared_ptr] --> [heap: Document] <-- sp2  (aliased, same object)
    */
}
```

The memory model comment at the bottom is the clearest way to see the distinction. With `shared_ptr`, two variables point to the same heap object. With `indirect`, two variables point to two separate heap objects that happen to have equal contents. Modifying one never affects the other.

### Q3: Compare std::indirect with unique_ptr<Base> for expressing ownership vs polymorphism

The table below captures the design space. The code shows the concrete ergonomic difference when you want to sort a vector of heap-allocated objects by value.

**Answer:**

| Feature                    | `std::indirect<T>`         | `std::unique_ptr<T>`        | `std::polymorphic<Base>` |
| --- | --- | --- | --- |
| **Ownership**              | Exclusive                   | Exclusive                    | Exclusive                |
| **Copyable**               | Yes (deep copy)            | No                           | Yes (deep copy + clone)  |
| **Movable**                | Yes                         | Yes                          | Yes                      |
| **Polymorphic**            | No (exact type `T`)        | Yes (via `Base*`)            | Yes (stored derived type)|
| **Value comparison**       | Yes (delegates to `T`)     | No (compares pointers)       | Not by default           |
| **Syntax to access**       | `.value()` or `*`          | `->` or `*`                  | `.value()` or `*`        |
| **Container-friendly**     | Fully (copy, sort, etc.)   | Move-only containers         | Fully                    |
| **Overhead**               | 1 heap alloc per object    | 1 heap alloc per object      | 1 heap alloc + type info |
| **Use when...**            | Need copyable heap `T`     | Move-only ownership          | Copyable polymorphism    |

```cpp
#include <memory>
#include <vector>
#include <algorithm>
#include <iostream>

struct Widget {
    int priority;
    std::string name;
    auto operator<=>(const Widget&) const = default;
};

void demonstrate_difference() {
    // unique_ptr: cannot copy, cannot sort by value
    std::vector<std::unique_ptr<Widget>> v1;
    v1.push_back(std::make_unique<Widget>(3, "C"));
    v1.push_back(std::make_unique<Widget>(1, "A"));
    // auto v2 = v1;  // ERROR: unique_ptr is not copyable
    // std::sort requires copying for swap -> works but compares pointers!

    // std::indirect: copyable + value-comparable
    std::vector<std::indirect<Widget>> v3;
    v3.emplace_back(Widget{3, "C"});
    v3.emplace_back(Widget{1, "A"});

    auto v4 = v3;  // Deep copy of entire vector — works!
    std::sort(v3.begin(), v3.end());  // Sorts by Widget::operator<=>
    // v3[0] = Widget{1, "A"}, v3[1] = Widget{3, "C"}

    // Rule of thumb:
    // - Need a non-copyable unique owner?     -> unique_ptr
    // - Need a copyable value on the heap?    -> std::indirect
    // - Need copyable + polymorphic dispatch? -> std::polymorphic
}
```

The rule of thumb at the bottom of the code is worth memorizing. `unique_ptr` remains the right choice when you explicitly do not want copying. `std::indirect` fills the gap that always existed between "I want heap allocation" and "I want value semantics" - a gap that previously had no clean standard answer.

---

## Notes

- `std::indirect` and `std::polymorphic` are C++26 (P3019); available in reference implementations.
- `std::indirect` is essentially a "deep-copying `unique_ptr`" with value comparison forwarding to the stored type.
- `std::polymorphic` uses internal type erasure to store and clone any derived object - no user-written `clone()` method needed.
- Both always hold a value (no null state) - unlike `unique_ptr` which can be `nullptr`.
- For nullable semantics, use `std::optional<std::indirect<T>>`.
