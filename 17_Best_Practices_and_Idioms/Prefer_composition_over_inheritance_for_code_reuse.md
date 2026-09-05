# Prefer composition over inheritance for code reuse

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#c-inheritance-and-oo>  

---

## Topic Overview

**Composition** models "has-a" relationships; **inheritance** models "is-a" relationships. The distinction sounds academic, but it has real consequences. Using inheritance purely for code reuse - not for polymorphism - creates tight coupling and fragile hierarchies that are hard to test, hard to change, and hard to reason about.

### Inheritance vs Composition

The classic illustration is a `Car` and an `Engine`. A `Car` has an engine, it is not an engine - so inheritance is the wrong tool:

```cpp
Inheritance:                     Composition:
class Car : public Engine { }    class Car {
  // Car IS-AN Engine??             Engine engine_;   // Car HAS-AN Engine
  // Nonsensical!                };
  // Car inherits all Engine     // Car delegates to engine_
  // methods including            // Only exposes what makes sense
  // Engine::set_rpm()
```

When you inherit just for reuse, you expose everything the base class does - whether it makes sense for the derived class or not.

---

## Self-Assessment

### Q1: Refactor a deep inheritance hierarchy into flat composition

Three levels of inheritance just to share a `log` method is a classic sign that inheritance is being misused. Notice how the composition version keeps each class focused on a single responsibility and lets you inject dependencies rather than bake them in.

```cpp
#include <iostream>
#include <memory>
#include <string>

// BAD: deep inheritance for code reuse
class LoggerBase {
protected:
    void log(const std::string& msg) { std::cout << "[LOG] " << msg << '\n'; }
};

class SerializerBase : public LoggerBase {
protected:
    std::string serialize(int data) {
        log("serializing");
        return std::to_string(data);
    }
};

class ValidatorBase : public SerializerBase {
protected:
    bool validate(int data) {
        log("validating");
        return data >= 0;
    }
};

// 3 levels deep just for code reuse!
class ServiceBad : public ValidatorBase {
public:
    void process(int data) {
        if (validate(data))
            std::cout << "Result: " << serialize(data) << '\n';
    }
};

// GOOD: flat composition
class Logger {
public:
    void log(const std::string& msg) { std::cout << "[LOG] " << msg << '\n'; }
};

class Serializer {
    Logger& logger_;
public:
    Serializer(Logger& l) : logger_(l) {}
    std::string serialize(int data) {
        logger_.log("serializing");
        return std::to_string(data);
    }
};

class Validator {
    Logger& logger_;
public:
    Validator(Logger& l) : logger_(l) {}
    bool validate(int data) {
        logger_.log("validating");
        return data >= 0;
    }
};

class ServiceGood {
    Logger logger_;
    Serializer serializer_{logger_};
    Validator validator_{logger_};
public:
    void process(int data) {
        if (validator_.validate(data))
            std::cout << "Result: " << serializer_.serialize(data) << '\n';
    }
};

int main() {
    ServiceGood service;
    service.process(42);
}
// Expected output:
// [LOG] validating
// [LOG] serializing
// Result: 42
```

Each component in the composition version is independently testable and reusable. If you want to swap in a different serializer or a mock logger, you just change what you pass to the constructor.

### Q2: Explain why "has-a" relationships should use composition, not inheritance

The simplest test is to say the sentence out loud: "A Stack is-a Vector" sounds wrong because it is wrong. A `Stack` built on inheritance from `std::vector` exposes `push_back`, `insert`, `erase`, and everything else a vector can do - destroying the stack abstraction entirely.

**The key test: "Is-a" vs "Has-a"**

| Relationship | Model | Example |
| --- | --- | --- |
| Dog IS-A Animal | Inheritance | `class Dog : public Animal` |
| Car HAS-A Engine | Composition | `Engine engine_;` member |
| Stack IS-A Vector? | No! | Stack doesn't support all vector ops |
| Stack HAS-A Vector | Composition | `vector<T> data_;` member |

Why composition is better for "has-a":

1. **No unwanted interface exposure:** Inheritance exposes ALL base methods. Composition exposes only what you delegate.
2. **Flexibility:** Can swap implementations at runtime (strategy pattern).
3. **No diamond problem:** Multiple composition is trivial; multiple inheritance is complex.
4. **Decoupling:** Components can evolve independently.

Here is the stack example made concrete:

```cpp
// BAD: Stack inherits from vector
class StackBad : public std::vector<int> {
    // Users can call push_back(), insert(), erase()
    // Stack should only allow push/pop!
};

// GOOD: Stack composes vector
class StackGood {
    std::vector<int> data_;  // hidden!
public:
    void push(int v) { data_.push_back(v); }
    int pop() { int v = data_.back(); data_.pop_back(); return v; }
    bool empty() const { return data_.empty(); }
    // Only stack operations exposed
};
```

The `StackGood` class exposes exactly the interface a stack should have and nothing more. Users cannot accidentally misuse it.

### Q3: Show inheritance breaking LSP and how composition fixes it

The Liskov Substitution Principle says: wherever you use a base class, a derived class should work correctly too. The Square-Rectangle problem is the most famous LSP violation - it shows that even intuitively reasonable inheritance hierarchies can break this rule.

```cpp
#include <iostream>
#include <stdexcept>

// LSP violation: Square IS-A Rectangle?
class Rectangle {
protected:
    int width_, height_;
public:
    Rectangle(int w, int h) : width_(w), height_(h) {}
    virtual void set_width(int w) { width_ = w; }
    virtual void set_height(int h) { height_ = h; }
    int area() const { return width_ * height_; }
};

class Square : public Rectangle {
public:
    Square(int s) : Rectangle(s, s) {}
    void set_width(int w) override { width_ = height_ = w; }  // must keep square!
    void set_height(int h) override { width_ = height_ = h; }
};

// LSP test: code that works with Rectangle breaks with Square
void resize(Rectangle& r) {
    r.set_width(5);
    r.set_height(10);
    // Postcondition: area should be 50
    if (r.area() != 50)
        std::cout << "LSP VIOLATED! Area = " << r.area() << '\n';
    else
        std::cout << "OK: Area = " << r.area() << '\n';
}

// FIX: composition with a Shape concept
struct Dimensions {
    int width, height;
    int area() const { return width * height; }
};

class RectangleFixed {
    Dimensions dims_;
public:
    RectangleFixed(int w, int h) : dims_{w, h} {}
    void set_width(int w) { dims_.width = w; }
    void set_height(int h) { dims_.height = h; }
    int area() const { return dims_.area(); }
};

class SquareFixed {
    int side_;
public:
    SquareFixed(int s) : side_(s) {}
    void set_side(int s) { side_ = s; }
    int area() const { return side_ * side_; }
    // No set_width/set_height: squares don't have independent dimensions
};

int main() {
    Rectangle rect(3, 4);
    resize(rect);  // OK

    Square sq(3);
    resize(sq);    // LSP violation!
}
// Expected output:
// OK: Area = 50
// LSP VIOLATED! Area = 100
```

The reason this trips people up is that geometry tells us a square is a special case of a rectangle. That is true mathematically, but not true in software when objects are mutable. The composition fix avoids the whole problem by giving `SquareFixed` only the interface it can honestly support.

---

## Notes

- C++ Core Guideline C.120: "Use class hierarchies to represent concepts with inherent hierarchical structure (only)."
- Inheritance for polymorphism is fine; inheritance for code reuse alone is usually wrong.
- Strategy pattern (composition + polymorphism) gives the best of both worlds.
- Modern C++ favors static polymorphism (templates/concepts) over deep hierarchies.
