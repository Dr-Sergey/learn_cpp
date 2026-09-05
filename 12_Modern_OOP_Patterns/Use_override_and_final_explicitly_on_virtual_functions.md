# Use override and final explicitly on virtual functions

**Category:** Modern OOP Patterns  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/override>  

---

## Topic Overview

`override` and `final` are contextual keywords (C++11) that catch virtual function signature mismatches at compile time and control inheritance hierarchies. They cost nothing at runtime - they are purely a safety net that the compiler enforces during compilation.

### Quick Reference

| Keyword | Where | Effect |
| --- | --- | --- |
| `override` | On a virtual function in derived | Compiler error if base has no matching virtual |
| `final` | On a virtual function | Prevents further override in grandchildren |
| `final` | On a class | Prevents any class from inheriting from it |

### The Problem Without `override`

This is the silent bug that `override` was designed to prevent. A one-character difference in signature creates a brand-new virtual function instead of overriding the base one, and the compiler says nothing:

```cpp
Base:       virtual void process(int x);
Derived:    virtual void process(int x) const;   // <- SILENT NEW FUNCTION!
                                                  //   Different signature (const)
                                                  //   Never overrides Base::process
                                                  //   Compiles without warning!
```

With `override`:

```cpp
Derived:    void process(int x) const override;   // <- COMPILER ERROR!
            // error: 'process' marked override but does not override
```

The moment you add `override`, the compiler immediately tells you exactly what went wrong. Without it, the bug lives silently in production.

---

## Self-Assessment

### Q1: Show a silent bug where a derived class virtual function signature differs by one `const` from the base

This is the most common form of the bug. The derived class adds `const` to the method and the programmer believes they have overridden the base class, but they have not. Notice the surprising output:

```cpp
#include <iostream>
#include <memory>

struct Document {
    virtual std::string render() { return "Document::render"; }
    virtual ~Document() = default;
};

struct HtmlDocument : Document {
    // BUG: added 'const' -- this does NOT override base!
    // It creates a COMPLETELY NEW virtual function.
    virtual std::string render() const { return "HtmlDocument::render"; }
};

int main() {
    std::unique_ptr<Document> doc = std::make_unique<HtmlDocument>();

    // Calls Document::render() -- NOT HtmlDocument::render() const!
    std::cout << doc->render() << "\n";  // Surprise!

    // Direct call to the derived class works:
    HtmlDocument html;
    std::cout << html.render() << "\n";  // calls non-const -> Document::render!

    // Only const reference calls the derived version:
    const HtmlDocument& chtml = html;
    std::cout << chtml.render() << "\n";  // calls const -> HtmlDocument::render
}
// Expected output:
//   Document::render
//   Document::render
//   HtmlDocument::render
```

**Why this is dangerous:** The code compiles, runs, and produces wrong results silently. In production, `render()` through a base pointer always calls the base version. The programmer thinks they overrode it but actually declared a new unrelated virtual function.

**Other common signature mismatches:**

| Base | Derived (BUG) | Difference |
| --- | --- | --- |
| `void f(int)` | `void f(long)` | Parameter type |
| `void f(int*)` | `void f(const int*)` | Pointer constness |
| `void f()` | `void f() const` | Method constness |
| `void f(string)` | `void f(string&)` | Value vs reference |

---

### Q2: Add `override` to fix the bug and explain how the compiler catches the mismatch

Adding `override` forces the compiler to verify that a matching virtual function exists in the base class. If the signatures don't align exactly, you get a clear compile-time error instead of a silent runtime surprise:

```cpp
#include <iostream>
#include <memory>

struct Document {
    virtual std::string render() { return "Document::render"; }
    virtual ~Document() = default;
};

// FIX: Use override -- compiler immediately catches the error
struct HtmlDocument : Document {
    // std::string render() const override { ... }
    // ───────────────────────────────────────────
    // error: 'render' marked 'override' but does not override
    // any base class member function
    //   note: candidate is 'virtual std::string Document::render()'
    //   note: type mismatch: 'const' qualification differs

    // CORRECT: match base signature exactly
    std::string render() override { return "HtmlDocument::render"; }
};

// BEST PRACTICE: Always use override on EVERY overriding function
struct MarkdownDocument : Document {
    std::string render() override { return "# MarkdownDocument"; }
};

int main() {
    std::unique_ptr<Document> docs[] = {
        std::make_unique<HtmlDocument>(),
        std::make_unique<MarkdownDocument>(),
    };

    for (const auto& doc : docs)
        std::cout << doc->render() << "\n";
}
// Expected output:
//   HtmlDocument::render
//   # MarkdownDocument
```

The virtual dispatch now works correctly. Both types report through the base pointer as intended.

**Rules for `override`:**

1. Can only be used on virtual member functions in derived classes
2. The base class function must be `virtual` (or itself an override)
3. Signature must match **exactly**: return type, parameters, const/volatile, ref-qualifiers
4. `override` does **not** make a function virtual - it requires an existing virtual in base
5. Compiler checks at compile time - zero runtime cost

**Compiler flags that help further:**

```cpp
GCC/Clang: -Wsuggest-override     Warns when override is missing
MSVC:      /W4                     Partial detection
clang-tidy: modernize-use-override Auto-fixes missing override
```

---

### Q3: Use `final` on a class to prevent further inheritance and explain a use case

`final` has two forms. On a single virtual function it stops that particular function from being overridden further down the hierarchy. On a class it prevents the class from being subclassed at all. The latter also enables a compiler optimization called devirtualization:

```cpp
#include <iostream>
#include <memory>
#include <vector>

// --- final on a virtual function ---
struct Shape {
    virtual double area() const = 0;
    virtual std::string name() const = 0;
    virtual ~Shape() = default;
};

struct Circle : Shape {
    double radius;
    Circle(double r) : radius(r) {}
    double area() const override { return 3.14159 * radius * radius; }

    // final: NO derived class can override name() further
    std::string name() const override final { return "Circle"; }
};

// struct FancyCircle : Circle {
//     std::string name() const override { return "FancyCircle"; }
//     // ERROR: cannot override 'final' function 'Circle::name'
// };

// --- final on a class ---
struct Square final : Shape {
    double side;
    Square(double s) : side(s) {}
    double area() const override { return side * side; }
    std::string name() const override { return "Square"; }
};

// struct RoundedSquare : Square { };
// ERROR: cannot derive from 'final' class 'Square'

// --- Use case: devirtualization optimization ---
// When a class is final, the compiler KNOWS the exact type.
// It can replace virtual dispatch with a direct call -> inlining possible.
void print_area(const Square& s) {
    // Compiler can devirtualize: s.area() -> direct call, no vtable lookup
    std::cout << s.name() << ": " << s.area() << "\n";
}

int main() {
    Circle c(5.0);
    Square s(4.0);

    // Polymorphic use
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(3.0));
    shapes.push_back(std::make_unique<Square>(2.0));

    for (const auto& shape : shapes)
        std::cout << shape->name() << " area = " << shape->area() << "\n";

    // Devirtualized use
    print_area(s);
}
// Expected output:
//   Circle area = 28.2743
//   Square area = 4
//   Square: 16
```

When `Square` is `final`, the compiler can see `print_area` taking a `const Square&` and know that `s.area()` will always dispatch to `Square::area` - never to any subclass. It can then call it directly and potentially inline it.

**When to use `final`:**

| Use Case | Explanation |
| --- | --- |
| **Security classes** | Prevent subclassing of crypto/auth classes that have invariants |
| **Performance** | Enable devirtualization - compiler eliminates vtable lookup |
| **Framework leaf classes** | Signal "this class is not designed for extension" |
| **Singleton/RAII guards** | Classes that must not be subclassed to work correctly |
| **Value semantics** | Types that use CRTP or expect exact type match |

---

## Notes

- **Always use `override`** - it costs nothing and prevents an entire class of bugs. Enable `-Wsuggest-override` in CI.
- `override` and `final` are **contextual keywords** - they're only keywords in these positions. You can still have variables named `override` or `final` (though you shouldn't).
- A function can be both `override` and `final`: `void f() override final;` - "I override the base AND nobody overrides me."
- The `virtual` keyword on derived overrides is **redundant** when using `override` - prefer `void f() override` over `virtual void f() override`.
- `final` on destructors is allowed but rare: `~Derived() override final = default;`
