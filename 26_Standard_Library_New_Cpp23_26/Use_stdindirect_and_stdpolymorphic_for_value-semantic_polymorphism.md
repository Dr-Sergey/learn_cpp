# Use std::indirect and std::polymorphic for value-semantic polymorphism

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3019r11.html>  

---

## Topic Overview

This file focuses on the **practical usage patterns** and **migration from clone-based idioms** to `std::indirect<T>` and `std::polymorphic<Base>`. (See also file #576 for core API overview and comparisons.)

### The Clone Problem

Before C++26, making a polymorphic type copyable required a manual `clone()` pattern. It worked, but it was fragile: every new derived class had to remember to implement `clone()`, return the right type, and keep it in sync with the copy constructor. Forgetting any of that was a silent runtime bug.

```cpp
// OLD: Manual clone() idiom — error-prone, boilerplate
struct Shape {
    virtual ~Shape() = default;
    virtual std::unique_ptr<Shape> clone() const = 0;  // Must implement in EVERY derived class
    virtual double area() const = 0;
};
struct Circle : Shape {
    double r;
    std::unique_ptr<Shape> clone() const override { return std::make_unique<Circle>(*this); }
    double area() const override { return 3.14159 * r * r; }
};
// Every new derived class must remember to implement clone() — easy to forget

// NEW: std::polymorphic — automatic deep copy, zero boilerplate
// std::polymorphic<Shape> s = Circle{5.0};
// auto s2 = s;  // Deep copies the Circle automatically!
```

`std::polymorphic` bakes the clone mechanism into the type itself, using internal type erasure. You never write `clone()` again.

---

## Self-Assessment

### Q1: Use std::indirect<T> to store a heap-allocated T with value semantics (copyable, movable)

Here is a practical scenario: a `Config` struct that holds a large payload. You want heap allocation to keep the struct small, but you also want copy and assign to work naturally without writing a custom copy constructor. `std::indirect<BigData>` gives you exactly that - the `Config` struct does not need any special member functions at all.

**Answer:**

<!-- compile: needs `indirect` (not in the CI standard library yet) -->
```cpp
#include <indirect>
#include <iostream>
#include <string>
#include <vector>

// Use case: a struct with a large member that benefits from heap allocation
// but still needs value semantics (copy, assign, compare)

struct BigData {
    std::string payload;
    BigData(std::string s) : payload(std::move(s)) {}
    auto operator<=>(const BigData&) const = default;
};

struct Config {
    std::string name;
    std::indirect<BigData> data;  // Heap-allocated but copyable!

    // No need to write copy ctor, move ctor, assignment operators
    // std::indirect handles everything automatically
};

int main() {
    Config c1{"prod", std::indirect<BigData>{BigData("large payload data...")}};

    // Copy: deep copies the BigData
    Config c2 = c1;
    c2.name = "staging";
    c2.data.value().payload = "modified payload";

    std::cout << c1.name << ": " << c1.data.value().payload << '\n';
    // prod: large payload data...  <- unchanged!
    std::cout << c2.name << ": " << c2.data.value().payload << '\n';
    // staging: modified payload

    // Works in containers with full value semantics
    std::vector<Config> configs;
    configs.push_back(c1);
    configs.push_back(c2);
    auto backup = configs;  // Deep copy of vector of Configs

    std::cout << "Backup has " << backup.size() << " configs\n";
}
```

Notice that `Config` has no user-written special members at all - the compiler generates them, and they do the right thing because `std::indirect` itself is copyable, movable, and assignable with deep-copy semantics.

### Q2: Use std::polymorphic<Base> to store any type derived from Base with deep-copy value semantics

This example builds a document model with multiple element types (`Text`, `Image`, `Link`). The key thing to notice is what is missing: there is no `clone()` method on `Element`, no manual copy loop, and the `Document` type is a plain struct with no special members. Deep copying the entire document is one line.

**Answer:**

<!-- compile: needs `polymorphic` (not in the CI standard library yet) -->
```cpp
#include <polymorphic>
#include <indirect>
#include <iostream>
#include <vector>
#include <string>

// Domain: Document elements
struct Element {
    virtual ~Element() = default;
    virtual std::string render() const = 0;
    // NOTE: No clone() method needed!
};

struct Text : Element {
    std::string content;
    Text(std::string s) : content(std::move(s)) {}
    std::string render() const override { return content; }
};

struct Image : Element {
    std::string url;
    int width, height;
    Image(std::string u, int w, int h) : url(std::move(u)), width(w), height(h) {}
    std::string render() const override {
        return "<img src='" + url + "' " + std::to_string(width) + "x" + std::to_string(height) + ">";
    }
};

struct Link : Element {
    std::string href, text;
    Link(std::string h, std::string t) : href(std::move(h)), text(std::move(t)) {}
    std::string render() const override {
        return "<a href='" + href + "'>" + text + "</a>";
    }
};

// Document: value-semantic container of polymorphic elements
struct Document {
    std::string title;
    std::vector<std::polymorphic<Element>> elements;
    // Fully copyable! No clone() boilerplate!
};

int main() {
    Document doc;
    doc.title = "My Page";
    doc.elements.push_back(Text{"Hello, world!"});
    doc.elements.push_back(Image{"photo.jpg", 800, 600});
    doc.elements.push_back(Link{"https://example.com", "Click here"});

    // Deep copy the entire document
    Document doc_copy = doc;
    doc_copy.title = "Copy of My Page";

    // Modify copy — original is unaffected
    doc_copy.elements.push_back(Text{"Added to copy only"});

    std::cout << doc.title << " (" << doc.elements.size() << " elements):\n";
    for (const auto& elem : doc.elements)
        std::cout << "  " << elem.value().render() << '\n';

    std::cout << doc_copy.title << " (" << doc_copy.elements.size() << " elements):\n";
    for (const auto& elem : doc_copy.elements)
        std::cout << "  " << elem.value().render() << '\n';

    // Output:
    // My Page (3 elements):
    //   Hello, world!
    //   <img src='photo.jpg' 800x600>
    //   <a href='https://example.com'>Click here</a>
    // Copy of My Page (4 elements):
    //   Hello, world!
    //   <img src='photo.jpg' 800x600>
    //   <a href='https://example.com'>Click here</a>
    //   Added to copy only
}
```

Adding a new element type like `Video` or `Table` in the future requires zero changes to `Document` or any existing element type. There is no `clone()` method to forget, and no base class to update.

### Q3: Compare std::polymorphic<Base> with unique_ptr<Base> + clone() and explain the ergonomic improvement

The old pattern was not wrong - it worked - but it accumulated boilerplate at every new derived class and created a category of bug (mismatched or forgotten `clone()`) that simply does not exist with `std::polymorphic`. The table inside the code block quantifies exactly what is eliminated.

**Answer:**

<!-- compile: needs C++26 library support for `std::polymorphic` (not in the CI standard library yet) -->
```cpp
#include <memory>
#include <vector>
#include <iostream>
#include <map>

// OLD: unique_ptr + clone()
struct ShapeOld {
    virtual ~ShapeOld() = default;
    virtual std::unique_ptr<ShapeOld> clone() const = 0;  // Boilerplate #1
    virtual double area() const = 0;
};

struct CircleOld : ShapeOld {
    double r;
    CircleOld(double r) : r(r) {}
    std::unique_ptr<ShapeOld> clone() const override {     // Boilerplate #2
        return std::make_unique<CircleOld>(*this);         // Must match derived type!
    }
    double area() const override { return 3.14159 * r * r; }
};

// Copy a vector of shapes — manual loop required:
std::vector<std::unique_ptr<ShapeOld>> copy_shapes(
    const std::vector<std::unique_ptr<ShapeOld>>& src) {
    std::vector<std::unique_ptr<ShapeOld>> dst;
    dst.reserve(src.size());
    for (const auto& s : src)
        dst.push_back(s->clone());  // Boilerplate #3
    return dst;
}

// NEW: std::polymorphic (C++26)
struct ShapeNew {
    virtual ~ShapeNew() = default;
    virtual double area() const = 0;
    // No clone() needed!
};

struct CircleNew : ShapeNew {
    double r;
    CircleNew(double r) : r(r) {}
    double area() const override { return 3.14159 * r * r; }
    // No clone() needed!
};

int main() {
    // NEW: just copy the vector — std::polymorphic handles deep copy
    std::vector<std::polymorphic<ShapeNew>> shapes;
    shapes.push_back(CircleNew{5.0});
    auto shapes_copy = shapes;  // One line! Deep copies everything!

    std::cout << shapes_copy[0].value().area() << '\n';  // 78.5398

    /*
    Ergonomic improvements:
    ┌───────────────────────────────┬────────────────────┬─────────────────────┐
    │ Task                          │ unique_ptr + clone  │ std::polymorphic    │
    ├───────────────────────────────┼────────────────────┼─────────────────────┤
    │ Copy a shape                  │ s->clone()         │ auto s2 = s;        │
    │ Copy a vector of shapes       │ Manual loop        │ auto v2 = v;        │
    │ Add new derived class         │ Must add clone()   │ Just add class      │
    │ Forget clone() in derived     │ Runtime bug        │ N/A — handled auto  │
    │ clone() returns wrong type    │ Subtle bug         │ N/A — impossible    │
    │ Use in std::map values        │ Cannot (not copy)  │ Works naturally     │
    │ Pass by value to function     │ Cannot             │ Works naturally     │
    │ Sort by value                 │ Cannot             │ With operator<=>    │
    └───────────────────────────────┴────────────────────┴─────────────────────┘

    Lines of boilerplate eliminated per derived class: ~4-5 lines
    Bug surface area: clone() mismatch bugs eliminated entirely
    */
}
```

The reason `clone() returns wrong type` is listed as a subtle bug is that a derived class can accidentally return `std::make_unique<Base>(*this)` instead of `std::make_unique<Derived>(*this)` - it compiles, runs, and silently loses the derived data. `std::polymorphic` eliminates this entire class of bug because the framework, not the user, performs the copy.

---

## Notes

- `std::polymorphic<Base>` internally stores a type-erased copy function alongside the object - no user `clone()` needed.
- `std::indirect<T>` is for **non-polymorphic** heap allocation with value semantics (analogous to how `std::string` wraps a heap `char*`).
- Both guarantee non-null: they always hold a valid object (no `nullptr` state unlike `unique_ptr`).
- Small buffer optimization is implementation-defined but encouraged for `std::indirect`.
- Migration path: replace `unique_ptr<Base> + clone()` with `polymorphic<Base>` for an immediate ergonomic improvement.
