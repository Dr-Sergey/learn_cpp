# Prevent object slicing with a clone pattern in polymorphic hierarchies

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**Object slicing** is one of those bugs that makes C++ feel hostile until you understand what is happening. It occurs when you copy a derived class object into a base class object - and in doing so, you silently discard everything the derived class added. The "derived part" is literally sliced off and thrown away.

The reason this trips people up is that it compiles cleanly and produces no warnings by default. Your code runs, but it runs *wrong*. There is no runtime crash, no exception, no diagnostic - just quietly incorrect behavior, which is the hardest kind of bug to track down.

```cpp
Derived d;          Memory: [Base part | Derived part]
Base b = d;         Memory: [Base part]  <- Derived part GONE!
b.virtual_fn();     Calls Base::virtual_fn, NOT Derived::virtual_fn
```

Here is the quick reference for which situations cause slicing and how to avoid them:

| Scenario | Slicing? | Fix |
| --- | :---: | --- |
| `Base b = derived;` | Yes | Use pointer/reference |
| `void f(Base b)` | Yes | Take `const Base&` |
| `vector<Base>` | Yes | Use `vector<unique_ptr<Base>>` |
| `Base& r = derived;` | No | Safe |
| `unique_ptr<Base> p = make_unique<Derived>()` | No | Safe |

---

## Self-Assessment

### Q1: Demonstrate object slicing and its consequences

Watch how `bad_speak` silently loses the `Dog` and always calls the base class method - even though you passed a `Dog`. Then watch `good_speak` take a reference and get polymorphism for free.

```cpp
#include <iostream>
#include <vector>
#include <memory>

class Animal {
public:
    virtual ~Animal() = default;
    virtual std::string speak() const { return "..."; }
    std::string name;
};

class Dog : public Animal {
public:
    std::string speak() const override { return "Woof!"; }
    std::string breed;  // Lost on slicing!
};

// BUG: takes by value -> slices!
void bad_speak(Animal a) {
    std::cout << a.speak() << "\n";  // Always calls Animal::speak!
}

// FIX: takes by reference
void good_speak(const Animal& a) {
    std::cout << a.speak() << "\n";  // Calls Dog::speak for dogs
}

int main() {
    Dog d;
    d.name = "Rex";
    d.breed = "Labrador";

    bad_speak(d);   // "..."  <- SLICED! Dog part gone
    good_speak(d);  // "Woof!" <- Correct, polymorphic

    // Vector slicing trap
    std::vector<Animal> bad_zoo;   // Stores copies of BASE only
    bad_zoo.push_back(d);          // Dog sliced to Animal!
    std::cout << bad_zoo[0].speak() << "\n";  // "..." WRONG!

    // Correct: store pointers
    std::vector<std::unique_ptr<Animal>> good_zoo;
    good_zoo.push_back(std::make_unique<Dog>(d));
    std::cout << good_zoo[0]->speak() << "\n";  // "Woof!" Correct!
    return 0;
}
```

The vector example is especially important. A `vector<Animal>` stores `Animal` objects by value - it has no concept of "I'm actually holding a Dog." Every element is exactly `sizeof(Animal)` bytes, so any derived-class data gets chopped off the moment you push.

### Q2: Show compile-time prevention of slicing

The best fix for slicing is to make it a compile error rather than a runtime surprise. You can do this by deleting the copy operations in the base class. Once you do that, the compiler will reject any code that tries to copy a `Shape` by value - which is exactly what you want.

```cpp
#include <memory>
#include <iostream>

// Strategy 1: Delete copy operations in base class
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;

    // Prevent slicing at compile time
    Shape(const Shape&) = delete;
    Shape& operator=(const Shape&) = delete;

protected:
    Shape() = default;  // Only derived classes can construct
    Shape(Shape&&) = default;
};

class Circle : public Shape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return 3.14159 * r_ * r_; }

    // Clone for when you DO need a copy
    std::unique_ptr<Shape> clone() const {
        return std::make_unique<Circle>(*this);
    }
private:
    Circle(const Circle&) = default;  // Only clone() can copy
};

// Strategy 2: Make base class abstract (can't instantiate)
class AbstractBase {
public:
    virtual ~AbstractBase() = default;
    virtual void process() = 0;  // Pure virtual -> can't create Base object
};

// Strategy 3: Protected constructor
class ProtectedBase {
protected:
    ProtectedBase() = default;  // Can't create standalone Base
public:
    virtual ~ProtectedBase() = default;
    virtual int value() const { return 0; }
};

int main() {
    Circle c(5.0);
    // Shape s = c;  // COMPILE ERROR: copy deleted!

    // Must use pointers
    std::unique_ptr<Shape> p = std::make_unique<Circle>(5.0);
    std::cout << p->area() << "\n";
    return 0;
}
```

The three strategies here form a useful hierarchy. An abstract base class prevents direct instantiation, which already blocks one common form of accidental slicing. Deleting copy operations is the most direct protection. A protected constructor is a softer version - it prevents external code from constructing the base directly, but does not fully close the door on slicing through assignment.

### Q3: Implement a clone pattern to safely copy polymorphic objects

When you have deleted copy operations (as above), how do you deep-copy a polymorphic object? The answer is the **clone pattern**: a virtual `clone()` method that each derived class overrides to create a proper copy of itself. This is the standard replacement for copy constructors in polymorphic hierarchies.

```cpp
#include <memory>
#include <vector>
#include <iostream>

class Document {
public:
    virtual ~Document() = default;
    virtual std::unique_ptr<Document> clone() const = 0;
    virtual void print() const = 0;

    Document(const Document&) = delete;  // Prevent slicing
    Document& operator=(const Document&) = delete;
protected:
    Document() = default;
};

class TextDoc : public Document {
    std::string text_;
public:
    explicit TextDoc(std::string t) : text_(std::move(t)) {}

    std::unique_ptr<Document> clone() const override {
        return std::make_unique<TextDoc>(text_);
    }
    void print() const override { std::cout << "Text: " << text_ << "\n"; }
};

class SpreadSheet : public Document {
    int rows_, cols_;
public:
    SpreadSheet(int r, int c) : rows_(r), cols_(c) {}

    std::unique_ptr<Document> clone() const override {
        return std::make_unique<SpreadSheet>(rows_, cols_);
    }
    void print() const override {
        std::cout << "Sheet: " << rows_ << "x" << cols_ << "\n";
    }
};

// Deep-copy a whole collection safely
std::vector<std::unique_ptr<Document>>
clone_all(const std::vector<std::unique_ptr<Document>>& docs) {
    std::vector<std::unique_ptr<Document>> result;
    result.reserve(docs.size());
    for (const auto& d : docs)
        result.push_back(d->clone());
    return result;
}

int main() {
    std::vector<std::unique_ptr<Document>> docs;
    docs.push_back(std::make_unique<TextDoc>("Hello"));
    docs.push_back(std::make_unique<SpreadSheet>(10, 5));

    auto copies = clone_all(docs);
    for (const auto& d : copies) d->print();
    return 0;
}
```

Each derived class knows its own type, so it can allocate the right concrete type and copy the right data. The caller sees only `unique_ptr<Document>` - it doesn't need to know what the actual type is, and the copy is always correct and complete.

---

## Notes

- Slicing is silent - no compiler warning by default; code compiles and runs with wrong behavior.
- Prevention tiers: (1) pure virtual base, (2) deleted copy, (3) protected constructor.
- Always use `unique_ptr<Base>` or references for polymorphic collections.
- The clone pattern replaces copy constructors for polymorphic hierarchies.
- Functions taking polymorphic types should accept `const Base&` or `Base&`, never `Base`.
- Clang-Tidy has a slicing check you can enable: `-checks=bugprone-slicing`.
