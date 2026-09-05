# Know all the ways a type can be incomplete and the rules around using incomplete types

**Category:** Type System and Deduction  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/type#Incomplete_type>  

---

## Topic Overview

An **incomplete type** is a type that has been declared but not yet defined. The compiler knows the name exists but doesn't know the type's size or layout. This comes up more often than you might expect, and getting the rules wrong usually produces confusing linker errors or silent undefined behavior.

### Ways a Type Can Be Incomplete

There are five common ways you end up with an incomplete type. The code below shows each one and what the compiler accepts or rejects:

```cpp
// 1. Forward-declared class/struct
class Widget;               // Incomplete - no definition yet
struct Node;                // Incomplete

// 2. Forward-declared enum (C++11, with underlying type)
enum class Color : int;     // Incomplete - enumerators unknown

// 3. void - always incomplete
// void x;                  // ERROR: cannot create object of type void

// 4. Arrays of unknown bound
extern int arr[];           // Incomplete - size unknown

// 5. Class being defined (inside its own body)
struct Self {
    // 'Self' is incomplete here - can't have Self member
    // Self s;              // ERROR: incomplete type
    Self* next;             // OK: pointer to incomplete type
};
```

Notice that inside a struct's own body, the struct itself is still incomplete - which is why you can have a pointer to `Self` but not a value member of type `Self`.

### What You CAN Do With Incomplete Types

The key insight is that pointers and references have a fixed, known size regardless of what they point to, so the compiler doesn't need the full type layout to work with them:

```cpp
class Widget;  // Forward declaration only

Widget* ptr;                    // OK: declare pointer
Widget& ref = *ptr;             // OK: declare reference
Widget* make_widget();          // OK: return type in function declaration
void process(Widget* w);        // OK: parameter type (pointer/reference)
extern Widget global_widget;    // OK: extern declaration
using WidgetPtr = Widget*;      // OK: type alias involving pointer
std::unique_ptr<Widget> up;     // OK: unique_ptr can hold incomplete type
```

### What You CANNOT Do With Incomplete Types

Once you try to actually use the type - create an object, measure its size, access its members - the compiler needs the full definition:

```cpp
class Widget;  // Forward declaration only

// Widget w;                   // ERROR: create instance (need size)
// sizeof(Widget);             // ERROR: get size
// Widget arr[10];             // ERROR: create array
// std::vector<Widget> v;      // ERROR: most containers need complete type
// ptr->method();              // ERROR: access members (layout unknown)
// delete ptr;                 // UB without complete type (no dtor call)
// dynamic_cast<Derived*>(ptr) // ERROR: needs complete type info
```

### std::unique_ptr and Incomplete Types

`std::unique_ptr<T>` only requires `T` to be complete at the **point of destruction** (where the deleter is called). This is what makes the Pimpl idiom possible:

```cpp
// widget.h
class Widget;  // Forward declaration
class Container {
    std::unique_ptr<Widget> pImpl;  // OK: Widget is incomplete here
public:
    Container();
    ~Container();  // Declared but NOT defined in header
};

// widget.cpp
#include "widget.h"
class Widget { /* full definition */ };
Container::Container() = default;
Container::~Container() = default;  // Widget is complete here - deleter works
```

The reason you must declare `~Container()` in the header and define it in the `.cpp` is that if the compiler generates the destructor inline in the header, it will try to instantiate `unique_ptr<Widget>::~unique_ptr()` where `Widget` is still incomplete - and that's undefined behavior or a compile error.

---

## Self-Assessment

### Q1: Show that forward-declared types can be used in pointers/references but not instantiated

The goal here is to see both sides: what works with a forward declaration, and what requires the full definition. Watch how the code at the top uses `Widget*` freely, then everything that needs the full type only works after the `class Widget { ... }` definition appears:

```cpp
#include <iostream>
#include <memory>

// Forward declaration - Widget is INCOMPLETE from this point
class Widget;

// OK: Functions with pointers/references to incomplete types
Widget* create_widget();
void destroy_widget(Widget* w);
void process(Widget& w);

// OK: Class using pointer to incomplete type
class Manager {
    Widget* widget_ = nullptr;        // OK: pointer
    // Widget widget_;                 // ERROR: incomplete type
    // std::vector<Widget> widgets_;   // ERROR: needs complete type
public:
    void init();
    void cleanup();
};

// --- Now define Widget (becomes complete) ---
class Widget {
    int id_;
    std::string name_;
public:
    Widget(int id, std::string name) : id_(id), name_(std::move(name)) {}
    void print() const { std::cout << "Widget(" << id_ << ", " << name_ << ")\n"; }
    int id() const { return id_; }
};

// All operations now work because Widget is complete:
Widget* create_widget() { return new Widget(1, "Test"); }
void destroy_widget(Widget* w) { delete w; }
void process(Widget& w) { w.print(); }

void Manager::init() { widget_ = create_widget(); }
void Manager::cleanup() { destroy_widget(widget_); widget_ = nullptr; }

int main() {
    Widget w(42, "Direct");  // Now complete - can instantiate
    w.print();               // Can access members

    Widget* ptr = create_widget();
    process(*ptr);           // Dereference and use
    destroy_widget(ptr);

    Manager m;
    m.init();
    m.cleanup();
}
```

Forward declaration gives the compiler just the name - enough for pointers and references (fixed size: 8 bytes on 64-bit). The compiler needs the full definition to know the size (for stack allocation), the layout (for member access), and the destructor (for deletion).

### Q2: Explain why `std::unique_ptr<T>` can work with an incomplete `T`

`unique_ptr<T>` stores a pointer (`T*`) - which has a known size regardless of whether `T` is complete. The critical operation that needs `T` to be complete is **deletion** (calling the destructor), which only happens inside `unique_ptr`'s destructor. Here's the pattern in action:

```cpp
#include <iostream>
#include <memory>

// header.h - Widget is incomplete
class Widget;

class Pimpl {
    std::unique_ptr<Widget> impl_;  // OK: only stores T*
public:
    Pimpl();
    ~Pimpl();                        // Declare - do NOT define in header!
    // If we let the compiler generate ~Pimpl() here, it would instantiate
    // unique_ptr<Widget>::~unique_ptr(), which calls delete on Widget.
    // But Widget is incomplete here - UB (no destructor call) or compiler error.

    Pimpl(Pimpl&&);                  // Same issue - must declare, define in .cpp
    Pimpl& operator=(Pimpl&&);

    void use();
};

// source.cpp - Widget is complete
class Widget {
public:
    int value = 42;
    ~Widget() { std::cout << "Widget destroyed\n"; }
};

Pimpl::Pimpl() : impl_(std::make_unique<Widget>()) {}
Pimpl::~Pimpl() = default;          // OK: Widget is complete here
Pimpl::Pimpl(Pimpl&&) = default;
Pimpl& Pimpl::operator=(Pimpl&&) = default;

void Pimpl::use() {
    std::cout << "Widget value: " << impl_->value << "\n";
}

int main() {
    Pimpl p;
    p.use();
}  // "Widget destroyed" - destructor properly called
```

**Key rule:** `std::unique_ptr<T>` declaration = incomplete OK. `std::unique_ptr<T>` destruction = needs complete `T`.

**Contrast with `std::shared_ptr<T>`:** `shared_ptr` captures the deleter at *construction* time (type-erased), so even `~shared_ptr()` doesn't need `T` to be complete. However, the *constructor* does need complete `T`.

### Q3: Use incomplete types to break circular header dependencies

This is the most practical reason you'll reach for forward declarations in real projects. When `Engine` needs to know about `Window` and `Window` needs to know about `Engine`, a naive mutual `#include` creates a circular dependency that won't compile. A forward declaration in each header breaks the cycle:

```cpp
#include <iostream>
#include <memory>
#include <string>

// Problem: Engine needs Window*, Window needs Engine*
// If both headers #include each other - circular dependency!

// ---- engine.h ----
class Window;  // Forward declaration breaks the cycle!

class Engine {
    Window* main_window_ = nullptr;  // OK: pointer to incomplete type
public:
    void set_window(Window* w) { main_window_ = w; }
    void render();  // Defined in .cpp where Window is complete
};

// ---- window.h ----
class Engine;  // Forward declaration

class Window {
    Engine* engine_ = nullptr;       // OK: pointer to incomplete type
    std::string title_;
public:
    Window(std::string title) : title_(std::move(title)) {}
    void set_engine(Engine* e) { engine_ = e; }
    void show();    // Defined in .cpp where Engine is complete
    const std::string& title() const { return title_; }
};

// ---- engine.cpp ----
// #include "engine.h"
// #include "window.h"  // Full definition now available
void Engine::render() {
    if (main_window_) {
        std::cout << "Rendering in window: " << main_window_->title() << "\n";
    }
}

// ---- window.cpp ----
// #include "window.h"
// #include "engine.h"  // Full definition now available
void Window::show() {
    std::cout << "Showing window: " << title_ << "\n";
    if (engine_) {
        engine_->render();
    }
}

int main() {
    Engine engine;
    Window window("Main Window");

    engine.set_window(&window);
    window.set_engine(&engine);

    window.show();
    // Output:
    // Showing window: Main Window
    // Rendering in window: Main Window
}
```

The pattern is straightforward: in headers, use forward declarations for types only needed via pointer or reference; in `.cpp` files, `#include` the full definition header so the type becomes complete where you actually need to use it.

---

## Notes

- Forward declare classes with `class Foo;` - it works for both `class` and `struct` (the keyword doesn't need to match the definition).
- `std::shared_ptr<T>` can also hold incomplete types but needs `T` complete at the point of construction.
- The Pimpl idiom (Pointer to IMPLementation) is the canonical use of incomplete types - it hides implementation details and reduces compile-time dependencies.
- `static_assert(sizeof(T) > 0)` or `static_assert(!std::is_void_v<T>)` can be used to catch accidental use of incomplete types.
- You cannot inherit from an incomplete type, `static_cast` through an incomplete hierarchy, or use `typeid` on an incomplete type.
