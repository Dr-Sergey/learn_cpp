# Know when to use inheritance vs composition and prefer composition

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**"Prefer composition over inheritance"** is the single most important OOP design guideline, and the reason is coupling. When you inherit from a class, your derived class depends on every `protected` and `public` member of the base - including implementation details you may not even know exist. Composition, on the other hand, ties you only to a well-defined interface. The base class can change its internals and you never notice.

The rule of thumb is simple: use inheritance when you truly have an "is-a" relationship and need runtime polymorphism. Use composition for everything else.

### Decision Matrix

If the table feels like a lot, the key insight is this: inheritance is for *open* polymorphic hierarchies; composition is for *reuse and flexibility*.

| Criterion | Use Inheritance | Use Composition |
| --- | :---: | :---: |
| True **is-a** relationship | Yes | |
| Need virtual dispatch | Yes | |
| Open set of types (plugins) | Yes | |
| Code reuse only | | Yes |
| **has-a** relationship | | Yes |
| Need to change behavior at runtime | | Yes |
| Want independent testing | | Yes |
| Multiple "inheritance" of behavior | | Yes (multiple members) |

### The Coupling Problem

This diagram captures why inheritance creates so much coupling compared to composition. When a base class changes its protected members, every single derived class must be reviewed and possibly updated. With composition, only the class that owns the component cares.

```cpp
Inheritance:                        Composition:
  Base class changes break ALL         Interface changes break ONLY
  derived classes.                     classes that USE that interface.

  class Derived : public Base {        class Widget {
    // Sees ALL of Base:               //  Sees ONLY Engine's public API:
    // - protected members             Engine engine_;
    // - implementation details        public:
    // - virtual dispatch table        void run() { engine_.start(); }
  };                                   };
```

---

## Self-Assessment

### Q1: Refactor an inheritance hierarchy to use composition

This example shows the classic problem with inheritance: a `Duck` that flies *and* swims breaks the single-parent hierarchy. The composition approach solves this by building animals from interchangeable behavior components.

```cpp
#include <memory>
#include <iostream>
#include <string>
#include <utility>
#include <vector>

// BAD: Deep inheritance for code reuse
class Animal {
protected:
    std::string name_;
    int energy_ = 100;
public:
    explicit Animal(std::string n) : name_(std::move(n)) {}
    virtual void eat() { energy_ += 20; }
    virtual void sleep() { energy_ += 50; }
    virtual std::string describe() = 0;
    virtual ~Animal() = default;
};

class FlyingAnimal : public Animal {
protected:
    int altitude_ = 0;
public:
    using Animal::Animal;
    virtual void fly() { altitude_ += 100; energy_ -= 10; }
};

class SwimmingAnimal : public Animal {
protected:
    int depth_ = 0;
public:
    using Animal::Animal;
    virtual void swim() { depth_ += 10; energy_ -= 5; }
};

// PROBLEM: Duck flies AND swims - diamond inheritance!
// class Duck : public FlyingAnimal, public SwimmingAnimal { ... };  // Messy!

// GOOD: Composition with behavior components

// Behavior interfaces
class IMovement {
public:
    virtual ~IMovement() = default;
    virtual void move() = 0;
    virtual int energy_cost() const = 0;
    virtual std::string description() const = 0;
};

class Flying : public IMovement {
    int altitude_ = 0;
public:
    void move() override { altitude_ += 100; }
    int energy_cost() const override { return 10; }
    std::string description() const override {
        return "flying at " + std::to_string(altitude_) + "m";
    }
};

class Swimming : public IMovement {
    int depth_ = 0;
public:
    void move() override { depth_ += 10; }
    int energy_cost() const override { return 5; }
    std::string description() const override {
        return "swimming at " + std::to_string(depth_) + "m depth";
    }
};

class Walking : public IMovement {
    int distance_ = 0;
public:
    void move() override { distance_ += 50; }
    int energy_cost() const override { return 3; }
    std::string description() const override {
        return "walked " + std::to_string(distance_) + "m";
    }
};

// Composed animal - any combination of behaviors
class Animal2 {
    std::string name_;
    int energy_ = 100;
    std::vector<std::unique_ptr<IMovement>> movements_;

public:
    explicit Animal2(std::string name) : name_(std::move(name)) {}

    template<typename M, typename... Args>
    Animal2& add_movement(Args&&... args) {
        movements_.push_back(std::make_unique<M>(std::forward<Args>(args)...));
        return *this;
    }

    void perform_all() {
        for (auto& m : movements_) {
            m->move();
            energy_ -= m->energy_cost();
            std::cout << name_ << " is " << m->description()
                      << " (energy: " << energy_ << ")\n";
        }
    }
};

int main() {
    Animal2 duck("Duck");
    duck.add_movement<Flying>()
        .add_movement<Swimming>()
        .add_movement<Walking>();
    duck.perform_all();
    // Duck is flying at 100m (energy: 90)
    // Duck is swimming at 10m depth (energy: 85)
    // Duck is walked 50m (energy: 82)

    Animal2 penguin("Penguin");
    penguin.add_movement<Swimming>()
           .add_movement<Walking>();
    penguin.perform_all();
    return 0;
}
```

With composition, a `Penguin` is just a duck without the `Flying` component. You never have to touch the class hierarchy again - you just choose which behavior objects to plug in.

### Q2: Explain the trade-offs - when inheritance IS the right choice

Inheritance is not always wrong. The key is that all three conditions below should hold. If even one is missing, reach for composition instead.

```cpp
// Condition 1: True is-a relationship (Liskov Substitution holds)
// Condition 2: You need virtual dispatch (runtime polymorphism)
// Condition 3: The set of types is OPEN (others will add derived classes)

// GOOD use of inheritance: widget toolkit
class Widget {
public:
    virtual ~Widget() = default;
    virtual void paint(Canvas& c) = 0;          // Must be overridden
    virtual Size preferred_size() const = 0;
    virtual void handle_event(const Event& e) {}  // Optional override

    // NVI: framework controls the layout algorithm
    void layout(const Rect& bounds) {
        validate_bounds(bounds);  // pre-condition
        do_layout(bounds);        // customization point
        mark_dirty();             // post-condition
    }
private:
    virtual void do_layout(const Rect& bounds) = 0;
};

// Users extend the OPEN set:
class Button : public Widget { /* ... */ };
class Slider : public Widget { /* ... */ };
class CustomPlot : public Widget { /* user-defined */ };

// BAD use of inheritance: just for code reuse
class BadStack : public std::vector<int> {  // WRONG
    // Inherits push_back, insert, erase, operator[] - all violate stack semantics!
};

// GOOD: Composition
class Stack {
    std::vector<int> data_;  // has-a vector, not is-a
public:
    void push(int v) { data_.push_back(v); }
    int pop() { int v = data_.back(); data_.pop_back(); return v; }
    bool empty() const { return data_.empty(); }
    // Only stack operations exposed - invariants preserved
};
```

The `BadStack` example is a classic mistake. Inheriting from `std::vector` just to get its storage means your "stack" suddenly has `push_back`, `insert`, `erase`, and all the other vector operations exposed to callers - none of which belong on a stack. Composition fixes this cleanly.

**Inheritance vs Composition trade-off summary:**

| Aspect | Inheritance | Composition |
| --- | --- | --- |
| Coupling | Tight (sees protected) | Loose (sees public only) |
| Flexibility | Fixed at compile time | Changeable at runtime |
| Testing | Harder (base pulled in) | Easy (mock components) |
| Diamond problem | Yes | No |
| Code reuse | Implicit (inherit all) | Explicit (delegate calls) |
| Boilerplate | Less (inherited) | More (forwarding methods) |

### Q3: Show the Strategy pattern as the composition-based alternative to inheritance

The reason this trips people up is the *hierarchy explosion* problem. If you try to model all the combinations of compression and encryption as inherited classes, you end up with `ZlibAesFile`, `ZlibChaChaFile`, `LzmaAesFile`, and so on - the number of classes grows as N times M. Composition lets you write N + M classes and mix them freely.

```cpp
#include <memory>
#include <iostream>
#include <functional>

// Strategy via composition - replace inheritance hierarchies

// Instead of: CompressedFile, EncryptedFile, CompressedEncryptedFile (hierarchy explosion)
// Use: File + CompressionStrategy + EncryptionStrategy

class CompressionStrategy {
public:
    virtual ~CompressionStrategy() = default;
    virtual std::vector<uint8_t> compress(std::span<const uint8_t> data) = 0;
    virtual std::vector<uint8_t> decompress(std::span<const uint8_t> data) = 0;
};

class NoCompression : public CompressionStrategy {
public:
    std::vector<uint8_t> compress(std::span<const uint8_t> d) override {
        return {d.begin(), d.end()};
    }
    std::vector<uint8_t> decompress(std::span<const uint8_t> d) override {
        return {d.begin(), d.end()};
    }
};

class ZlibCompression : public CompressionStrategy {
    int level_;
public:
    explicit ZlibCompression(int level = 6) : level_(level) {}
    std::vector<uint8_t> compress(std::span<const uint8_t> d) override {
        // zlib compress...
        std::cout << "Compressing " << d.size() << " bytes (level " << level_ << ")\n";
        return {d.begin(), d.end()};  // simplified
    }
    std::vector<uint8_t> decompress(std::span<const uint8_t> d) override {
        return {d.begin(), d.end()};
    }
};

class EncryptionStrategy {
public:
    virtual ~EncryptionStrategy() = default;
    virtual std::vector<uint8_t> encrypt(std::span<const uint8_t> data) = 0;
    virtual std::vector<uint8_t> decrypt(std::span<const uint8_t> data) = 0;
};

// File class: COMPOSED of strategies, not inheriting behaviors
class SecureFile {
    std::unique_ptr<CompressionStrategy> compressor_;
    std::unique_ptr<EncryptionStrategy> encryptor_;
    std::vector<uint8_t> data_;

public:
    SecureFile(std::unique_ptr<CompressionStrategy> comp,
               std::unique_ptr<EncryptionStrategy> enc)
        : compressor_(std::move(comp)), encryptor_(std::move(enc)) {}

    // Can change strategy at runtime!
    void set_compression(std::unique_ptr<CompressionStrategy> comp) {
        compressor_ = std::move(comp);
    }

    void write(std::span<const uint8_t> data) {
        auto compressed = compressor_->compress(data);
        data_ = encryptor_->encrypt(compressed);
    }

    std::vector<uint8_t> read() {
        auto decrypted = encryptor_->decrypt(data_);
        return compressor_->decompress(decrypted);
    }
};

// Without composition, you'd need:
// File, CompressedFile, EncryptedFile, CompressedEncryptedFile,
// ZlibFile, LzmaFile, AesFile, ChaChaFile, ZlibAesFile... (exponential!)
// With composition: File + N compressors + M encryptors = N + M classes
```

Notice `set_compression` - you can swap the strategy at runtime, something a class hierarchy could never do without reconstructing the entire object.

---

## Notes

- Litmus test: "Can I replace the base class pointer with a derived class pointer in ALL scenarios?" If no, don't inherit.
- `std::string`, `std::vector` - don't inherit from STL containers. They lack virtual destructors. Use composition.
- Composition enables runtime flexibility - swap strategies and components without recompiling.
- In embedded systems, composition with interfaces enables hardware abstraction layers (HAL) that can be mocked for testing.
- Use private inheritance only as an implementation detail (rarely), and always prefer composition first.
