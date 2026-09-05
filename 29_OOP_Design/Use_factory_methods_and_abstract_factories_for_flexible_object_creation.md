# Use factory methods and abstract factories for flexible object creation

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

When you need to create objects but don't want the caller to worry about which concrete type gets constructed - or you want to swap implementations later without changing call sites - factory patterns are your go-to tool. There are three main flavors, each solving a slightly different problem:

| Pattern | Purpose | Returns | Extensibility |
| --- | --- | --- | --- |
| **Factory method** | Create one product | `unique_ptr<Base>` | Override in derived class |
| **Static factory** | Named constructors | Value or `unique_ptr` | Single class |
| **Abstract factory** | Create family of related products | Multiple `unique_ptr<Base>` | New families via new factory |

The key intuition is this: a constructor can only do one thing - initialize the object. A factory function can make a decision first, then create the right object. That decision-making power is what makes factories useful. You move the "which type should I build?" logic out of the constructor and into a dedicated place that is easy to change, test, and extend.

### When to Use

Here's a quick guide for picking the right flavor:

```cpp
Factory method:      Type determined by subclass or parameter
Static factory:      Multiple construction modes for one type
Abstract factory:    Families of related objects that must match
                     (e.g., UI widgets for different platforms)
```

---

## Self-Assessment

### Q1: Implement static factory methods and a parameterized factory

Static factory methods are just named constructors - instead of a single `Shape(args)` constructor that tries to do everything, you give callers descriptive entry points like `Circle::from_area()` or `Rect::square()`. The parameterized factory goes a step further: it takes a string or enum key and dispatches to the right concrete type at runtime. Here is what that looks like in practice:

**Answer:**

```cpp
#include <memory>
#include <string>
#include <cmath>
#include <stdexcept>
#include <iostream>
#include <numbers>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual std::string name() const = 0;
};

class Circle : public Shape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return std::numbers::pi * r_ * r_; }
    std::string name() const override { return "Circle"; }

    // Static factory: named constructors
    static Circle unit() { return Circle(1.0); }
    static Circle from_area(double a) { return Circle(std::sqrt(a / std::numbers::pi)); }
};

class Rect : public Shape {
    double w_, h_;
public:
    Rect(double w, double h) : w_(w), h_(h) {}
    double area() const override { return w_ * h_; }
    std::string name() const override { return "Rect"; }

    static Rect square(double side) { return Rect(side, side); }
};

// Parameterized factory function
std::unique_ptr<Shape> create_shape(const std::string& type, double a, double b = 0) {
    if (type == "circle") return std::make_unique<Circle>(a);
    if (type == "rect")   return std::make_unique<Rect>(a, b);
    throw std::invalid_argument("Unknown shape: " + type);
}

int main() {
    auto c = Circle::from_area(100.0);  // Named constructor
    auto s = Rect::square(5.0);         // Named constructor
    auto shape = create_shape("circle", 3.0);  // Parameterized factory
    std::cout << shape->name() << ": " << shape->area() << "\n";
    return 0;
}
```

Notice how `from_area` makes the intent crystal clear at the call site - you'd never guess from `Circle(5.64)` that the radius was computed from an area, but `Circle::from_area(100.0)` tells you exactly what's happening. The `create_shape` function shows the next level: given a runtime string, it picks the right concrete type and hands you back a `unique_ptr<Shape>`. The caller doesn't know and doesn't care which subclass it got.

### Q2: Implement an Abstract Factory for cross-platform UI

The abstract factory pattern solves a trickier problem: not just "create one thing," but "create a consistent *family* of things." If you mix a Windows button with a Linux text box, your UI looks wrong and behaves inconsistently. The abstract factory prevents that mismatch by packaging related creators together - one factory object is responsible for every widget in a platform's family. Take a look:

**Answer:**

```cpp
#include <memory>
#include <string>
#include <iostream>

// Abstract products
class Button {
public:
    virtual ~Button() = default;
    virtual void render() const = 0;
};

class TextBox {
public:
    virtual ~TextBox() = default;
    virtual void render() const = 0;
};

// Concrete products: Windows
class WinButton : public Button {
public:
    void render() const override { std::cout << "[Win Button]\n"; }
};
class WinTextBox : public TextBox {
public:
    void render() const override { std::cout << "[Win TextBox]\n"; }
};

// Concrete products: Linux
class LinuxButton : public Button {
public:
    void render() const override { std::cout << "[Linux Button]\n"; }
};
class LinuxTextBox : public TextBox {
public:
    void render() const override { std::cout << "[Linux TextBox]\n"; }
};

// Abstract Factory
class UIFactory {
public:
    virtual ~UIFactory() = default;
    virtual std::unique_ptr<Button> create_button() const = 0;
    virtual std::unique_ptr<TextBox> create_textbox() const = 0;
};

class WinFactory : public UIFactory {
public:
    std::unique_ptr<Button> create_button() const override {
        return std::make_unique<WinButton>();
    }
    std::unique_ptr<TextBox> create_textbox() const override {
        return std::make_unique<WinTextBox>();
    }
};

class LinuxFactory : public UIFactory {
public:
    std::unique_ptr<Button> create_button() const override {
        return std::make_unique<LinuxButton>();
    }
    std::unique_ptr<TextBox> create_textbox() const override {
        return std::make_unique<LinuxTextBox>();
    }
};

// Client code works with any factory
void build_ui(const UIFactory& factory) {
    auto btn = factory.create_button();
    auto txt = factory.create_textbox();
    btn->render();
    txt->render();
}

int main() {
    WinFactory wf;
    LinuxFactory lf;
    build_ui(wf);  // Consistent Windows UI
    build_ui(lf);  // Consistent Linux UI
    return 0;
}
```

The beauty here is that `build_ui` doesn't know or care which platform it's running on. You pass in a `WinFactory` and everything it creates is guaranteed to be Windows-style. Add a `MacFactory` later and `build_ui` works with it immediately - no changes required. The abstract factory is the glue that keeps the family consistent.

### Q3: Show a self-registering factory using a map of creators

The previous examples have a central switch or a hardcoded set of factory classes. Self-registering factories eliminate that central list entirely: each plugin type announces itself at static initialization time. This is especially useful in plugin architectures where new types are added without touching existing code. The trick is using a static registry map and a helper template that registers the type before `main` runs:

**Answer:**

```cpp
#include <memory>
#include <string>
#include <unordered_map>
#include <functional>
#include <stdexcept>
#include <iostream>

class Plugin {
public:
    virtual ~Plugin() = default;
    virtual void execute() = 0;
};

// Self-registering factory
class PluginFactory {
public:
    using Creator = std::function<std::unique_ptr<Plugin>()>;

    static PluginFactory& instance() {
        static PluginFactory f;
        return f;
    }

    void register_plugin(const std::string& name, Creator creator) {
        creators_[name] = std::move(creator);
    }

    std::unique_ptr<Plugin> create(const std::string& name) const {
        auto it = creators_.find(name);
        if (it == creators_.end())
            throw std::runtime_error("Unknown plugin: " + name);
        return it->second();
    }

private:
    PluginFactory() = default;
    std::unordered_map<std::string, Creator> creators_;
};

// Auto-registration helper
template<typename T>
struct RegisterPlugin {
    explicit RegisterPlugin(const std::string& name) {
        PluginFactory::instance().register_plugin(
            name, [] { return std::make_unique<T>(); });
    }
};

// Plugins self-register at static init time
class Compressor : public Plugin {
    static RegisterPlugin<Compressor> reg_;
public:
    void execute() override { std::cout << "Compressing...\n"; }
};
RegisterPlugin<Compressor> Compressor::reg_{"compressor"};

class Encryptor : public Plugin {
    static RegisterPlugin<Encryptor> reg_;
public:
    void execute() override { std::cout << "Encrypting...\n"; }
};
RegisterPlugin<Encryptor> Encryptor::reg_{"encryptor"};

int main() {
    auto p = PluginFactory::instance().create("compressor");
    p->execute();  // "Compressing..."
    return 0;
}
```

The static `reg_` member in each class triggers registration before `main` even starts. You can drop a new plugin `.cpp` file into the build and it just works - no modification to the factory or any other file needed. That is the whole point: new types are added by addition, not by modification.

---

## Notes

- **Static factory methods** are the simplest - use them for named constructors (`from_string()`, `default_config()`) when a plain constructor would leave the intent unclear.
- Parameterized factories use a string or enum key and are easily made extensible by plugging into a registration map.
- Abstract factories enforce **consistency** across product families - you can't accidentally mix products from different families.
- Self-registering factories eliminate the central switch statement, so new types add themselves without touching existing code.
- Always return `unique_ptr` from factories - the caller gets clear ownership and you avoid raw pointer confusion.
- Prefer free factory functions or static methods over the GoF virtual factory method pattern in modern C++ - the virtual approach is rarely needed.
