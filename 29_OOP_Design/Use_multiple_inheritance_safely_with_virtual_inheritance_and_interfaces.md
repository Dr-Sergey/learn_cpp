# Use multiple inheritance safely with virtual inheritance and interfaces

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

### The Diamond Problem

Multiple inheritance becomes tricky the moment two base classes share a common ancestor. The classic example is the diamond:

```cpp
         Animal
        /      \
    Mammal    Flyer
        \      /
        Bat        <- Two copies of Animal?
```

Without any special handling, a `Bat` would contain two separate `Animal` subobjects - one through `Mammal` and one through `Flyer`. Accessing `bat.name` becomes ambiguous and the compiler rejects it. Virtual inheritance fixes this by guaranteeing a single shared `Animal` subobject no matter how many paths lead to it.

The reason this trips people up is that the diamond only appears when your base classes have *shared state* (data members). If both bases are pure interfaces with no data, there is nothing to duplicate and no problem to solve. The table below shows at a glance which scenario you are in:

| MI Type | Use Case | Pitfalls |
| --- | --- | --- |
| Interface MI | Multiple abstract interfaces | None - always safe |
| Implementation MI | Reuse from multiple bases | Diamond, ambiguity |
| Virtual inheritance | Solve diamond problem | Performance cost, complexity |

### Safe Rules

Here are the four rules that keep multiple inheritance manageable:

```cpp
1. Multiple INTERFACES: always OK (pure virtual, no data)
2. Multiple bases with DATA: use virtual inheritance for diamond
3. Prefer composition over MI for code reuse
4. Never use MI just because you can - use it because you must
```

---

## Self-Assessment

### Q1: Show the diamond problem and fix it with virtual inheritance

The `broken` namespace shows what goes wrong without virtual inheritance - the `Bat` constructor would need to call `Animal` twice via two separate chains, and any access to `name` is ambiguous. The `fixed` namespace resolves this by declaring both `Mammal` and `Flyer` to inherit *virtually* from `Animal`, which tells the compiler to merge the two paths into one shared base subobject:

**Answer:**

```cpp
#include <iostream>
#include <string>

// WITHOUT virtual inheritance: diamond problem
namespace broken {
    class Animal {
    public:
        std::string name;
        Animal(std::string n) : name(std::move(n)) {
            std::cout << "Animal(" << name << ") constructed\n";
        }
    };
    class Mammal : public Animal {
    public:
        Mammal(std::string n) : Animal(std::move(n)) {}
    };
    class Flyer : public Animal {
    public:
        Flyer(std::string n) : Animal(std::move(n)) {}
    };
    // class Bat : public Mammal, public Flyer {
    //     Bat() : Mammal("bat-m"), Flyer("bat-f") {}
    //     // bat.name is AMBIGUOUS! Two Animal subobjects
    // };
}

// WITH virtual inheritance: single Animal subobject
namespace fixed {
    class Animal {
    public:
        std::string name;
        Animal(std::string n = "") : name(std::move(n)) {
            std::cout << "Animal(" << name << ") constructed\n";
        }
        virtual ~Animal() = default;
    };
    class Mammal : virtual public Animal {
    public:
        Mammal(std::string n) : Animal(std::move(n)) {
            std::cout << "Mammal constructed\n";
        }
    };
    class Flyer : virtual public Animal {
    public:
        Flyer(std::string n) : Animal(std::move(n)) {
            std::cout << "Flyer constructed\n";
        }
    };
    class Bat : public Mammal, public Flyer {
    public:
        // IMPORTANT: Most-derived class must initialize virtual base
        Bat(std::string n)
            : Animal(n)        // Must explicitly init virtual base
            , Mammal(n)
            , Flyer(n) {}
    };
}

int main() {
    fixed::Bat bat("Bruce");
    std::cout << bat.name << "\n";  // Unambiguous! Single Animal
    return 0;
}
```

The reason this trips people up: in the `fixed` version, `Bat` must directly initialize `Animal` in its own initializer list, even though `Mammal` and `Flyer` also have `Animal` initializations. With virtual inheritance, those intermediate initializations are silently ignored - the most-derived class always "wins" and initializes the shared base. Forgetting this rule leads to the virtual base being default-constructed even when you didn't intend that. If you run the fixed example you will see `Animal` constructed exactly once, confirming only one subobject exists.

### Q2: Use multiple pure interfaces safely (the recommended pattern)

Inheriting from multiple *pure* interfaces (no data members, only pure virtual functions) is always safe - there's no shared state to conflict, so no diamond problem can arise. This is the normal, everyday use of multiple inheritance in C++ and you should feel comfortable reaching for it. Here is what clean interface-based MI looks like:

**Answer:**

```cpp
#include <iostream>
#include <memory>

// Pure interfaces: zero data, only virtual methods
class ISerializable {
public:
    virtual ~ISerializable() = default;
    virtual std::string serialize() const = 0;
};

class IRenderable {
public:
    virtual ~IRenderable() = default;
    virtual void render() const = 0;
};

class IClickable {
public:
    virtual ~IClickable() = default;
    virtual void on_click(int x, int y) = 0;
};

// Multiple interface inheritance: ALWAYS SAFE
class Button : public IRenderable, public IClickable, public ISerializable {
    std::string label_;
public:
    explicit Button(std::string label) : label_(std::move(label)) {}

    void render() const override {
        std::cout << "[" << label_ << "]\n";
    }
    void on_click(int x, int y) override {
        std::cout << label_ << " clicked at (" << x << "," << y << ")\n";
    }
    std::string serialize() const override {
        return "Button:" + label_;
    }
};

// Polymorphic usage through any interface
void draw(const IRenderable& r) { r.render(); }
void save(const ISerializable& s) { std::cout << s.serialize() << "\n"; }

int main() {
    Button btn("Submit");
    draw(btn);           // Uses IRenderable
    save(btn);           // Uses ISerializable
    btn.on_click(10, 20); // Uses IClickable
    return 0;
}
```

Notice how `draw` only knows about `IRenderable` and `save` only knows about `ISerializable`. Neither needs to know that `Button` happens to also implement the other interfaces. This is exactly the kind of clean dependency boundary you want - each function depends only on what it actually uses.

### Q3: Show mixin classes using MI for reusable behavior

Template-based mixins are the modern C++ alternative to implementation-level multiple inheritance. Instead of inheriting from two concrete base classes and risking a diamond, you layer behaviors by chaining templates. `ThreadSafe<Timestamped<Document>>` reads like a recipe: take a `Document`, add timestamps, then wrap it with a mutex. Each mixin in the chain adds one responsibility and passes everything else through. The result is a linear chain with no diamond anywhere:

**Answer:**

```cpp
#include <iostream>
#include <chrono>
#include <mutex>
#include <string>

// Mixin: adds timestamp tracking to any class
template<typename Base>
class Timestamped : public Base {
    using Clock = std::chrono::system_clock;
    Clock::time_point created_ = Clock::now();
    Clock::time_point modified_ = Clock::now();
public:
    using Base::Base;  // Inherit constructors

    void touch() { modified_ = Clock::now(); }
    auto age() const {
        return std::chrono::duration_cast<std::chrono::seconds>(
            Clock::now() - created_).count();
    }
};

// Mixin: adds thread-safety to any class
template<typename Base>
class ThreadSafe : public Base {
    mutable std::mutex mtx_;
public:
    using Base::Base;

    template<typename F>
    auto with_lock(F&& f) {
        std::lock_guard lk(mtx_);
        return f(static_cast<Base&>(*this));
    }
    template<typename F>
    auto with_lock(F&& f) const {
        std::lock_guard lk(mtx_);
        return f(static_cast<const Base&>(*this));
    }
};

// Base class
class Document {
    std::string content_;
public:
    Document() = default;
    explicit Document(std::string c) : content_(std::move(c)) {}
    void set_content(std::string c) { content_ = std::move(c); }
    const std::string& content() const { return content_; }
};

// Compose mixins via inheritance chain
using SafeTimedDoc = ThreadSafe<Timestamped<Document>>;

int main() {
    SafeTimedDoc doc("Initial content");

    doc.with_lock([](Document& d) {
        d.set_content("Updated");
    });

    doc.touch();
    std::cout << "Age: " << doc.age() << "s\n";

    doc.with_lock([](const Document& d) {
        std::cout << d.content() << "\n";
    });
    return 0;
}
```

The mixin chain avoids any diamond problem because there's only one path from `SafeTimedDoc` up to `Document` - the chain is linear. You get composition-like flexibility with inheritance-like access to internal state, and zero runtime overhead compared to hand-rolled solutions. Adding a new mixin is as simple as wrapping the type with another template layer.

---

## Notes

- **Safe MI rule:** multiple *interfaces* (pure virtual, no data) is always fine - this is the everyday use case and carries no gotchas.
- Virtual inheritance adds roughly 8 bytes overhead per object and slightly slower dispatch due to the virtual base table pointer - not a dealbreaker, but worth knowing.
- The **most-derived class** must explicitly initialize virtual bases in its constructor, even though intermediate bases also list initializations for the same virtual base - those are ignored at runtime.
- Prefer **composition** or **CRTP mixins** over implementation-level multiple inheritance whenever you're reusing behavior rather than expressing genuine "is-a" relationships across multiple hierarchies.
- The mixin pattern via templates (`ThreadSafe<Timestamped<Base>>`) is the modern C++ approach: you get stacked behavior with a linear, diamond-free inheritance chain.
- If you find yourself reaching for diamond inheritance with data, that's usually a signal the design should be refactored - there's typically a cleaner way to express the relationships.
