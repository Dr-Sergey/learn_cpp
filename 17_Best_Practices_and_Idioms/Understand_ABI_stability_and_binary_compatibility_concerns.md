# Understand ABI stability and binary compatibility concerns

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/abi_tag>  

---

## Topic Overview

**ABI (Application Binary Interface)** defines how compiled code interacts at the binary level: object layout, name mangling, calling conventions, vtable layout. The reason this matters is that when you ship a compiled library, users link against the binary - not the source. If you change something that alters the binary layout and users do not recompile their code, the mismatch causes silent corruption or crashes, with no compiler warning to help you.

### ABI-Breaking vs ABI-Safe Changes

Not every code change breaks ABI. The rule of thumb is: anything that affects how objects are laid out in memory or how functions are identified by name will break it.

| Change | ABI breaks? | Why |
| --- | --- | --- |
| Add data member | Yes | Changes `sizeof(T)` |
| Reorder data members | Yes | Changes member offsets |
| Add virtual function | Yes | Changes vtable layout |
| Reorder virtual functions | Yes | Changes vtable indices |
| Change function signature | Yes | Changes mangled name |
| Add non-virtual member function | No | Not in class layout |
| Add static member function | No | No instance layout change |
| Add enum value at end | No | Enum value, not object layout |
| Change function body (implementation) | No | Layout unchanged |

---

## Self-Assessment

### Q1: Explain what changes break ABI

The danger here is invisible at link time. Old object files compiled against v1 contain hardcoded offsets for accessing members. If v2 rearranges or adds members, those offsets are now wrong - but nothing tells you that.

```cpp
#include <iostream>

// Version 1.0 of your library:
class Widget_v1 {
    int x_;          // offset 0
    int y_;          // offset 4
public:
    virtual void draw();  // vtable slot 0
    int x() const { return x_; }
    int y() const { return y_; }
};
// sizeof(Widget_v1) = 16 (vtable ptr + 2 ints, with padding)

// Version 2.0 - ABI-BREAKING changes:
class Widget_v2 {
    int x_;          // offset 0
    double z_;       // ADDED: shifts y_ offset!
    int y_;          // was at offset 4, now at offset 12
public:
    virtual void draw();    // vtable slot 0
    virtual void resize();  // ADDED: changes vtable layout!
    int x() const { return x_; }
    int y() const { return y_; }
};
// sizeof(Widget_v2) = 32 (different!)

// Code compiled against v1 that calls widget->y() will read
// from offset 4, but in v2 that's now part of z_.
// Result: WRONG VALUE or CRASH!

int main() {
    std::cout << "v1 size: " << sizeof(Widget_v1) << '\n';
    std::cout << "v2 size: " << sizeof(Widget_v2) << '\n';
    std::cout << "Sizes differ = ABI break!\n";
}
// Expected output:
// v1 size: 16
// v2 size: 32
// Sizes differ = ABI break!
```

The size change is the observable symptom. The actual danger is that old compiled code still uses the old offsets, now pointing into the wrong fields of the new layout.

### Q2: Show how Pimpl maintains ABI stability across library versions

The Pimpl ("pointer to implementation") idiom solves ABI stability by hiding all the data behind a pointer. The public class becomes a stable shell - its size is always `sizeof(unique_ptr)`, regardless of how much you change the implementation.

```cpp
#include <iostream>
#include <memory>

// PUBLIC HEADER (stable ABI):
class Widget {
public:
    Widget();
    ~Widget();
    void draw() const;
    int value() const;

private:
    struct Impl;                    // forward declaration only
    std::unique_ptr<Impl> pimpl_;   // always sizeof(unique_ptr) = 8
};
// sizeof(Widget) NEVER CHANGES regardless of what's in Impl

// IMPLEMENTATION FILE (can change freely):
struct Widget::Impl {
    int x_ = 10;
    int y_ = 20;
    // VERSION 2: can add fields without breaking ABI!
    // double z_ = 30;
    // std::string name_ = "widget";
};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;

void Widget::draw() const {
    std::cout << "Drawing at (" << pimpl_->x_ << ", " << pimpl_->y_ << ")\n";
}

int Widget::value() const {
    return pimpl_->x_ + pimpl_->y_;
}

int main() {
    Widget w;
    w.draw();
    std::cout << "Value: " << w.value() << '\n';
    std::cout << "sizeof(Widget): " << sizeof(Widget) << '\n';
    // sizeof stays the same even if Impl changes!
}
// Expected output:
// Drawing at (10, 20)
// Value: 30
// sizeof(Widget): 8
```

The tradeoff is one extra heap allocation per object and one extra pointer indirection per method call. For most code that is acceptable - and it is far better than forcing all your users to recompile whenever you add a private field.

### Q3: Describe the difference between API compatibility and ABI compatibility

The reason this distinction trips people up is that both sound like "compatibility." API is source-level - the compiler checks it. ABI is binary-level - nothing checks it at link time, which is what makes ABI breaks so dangerous.

**API (Application Programming Interface) compatibility:**

- Source-level: can you compile code written against the old version with the new version?
- Checked by the compiler.

**ABI (Application Binary Interface) compatibility:**

- Binary-level: can you link/load code compiled against the old version with the new library binary?
- NOT checked by any tool at link time - mismatches cause silent corruption.

The four combinations are worth understanding explicitly:

```cpp
API compatible, ABI compatible:
  Old code compiles AND runs with new library binary.
  Example: adding a new free function to the library.

API compatible, ABI INCOMPATIBLE:
  Old code compiles with new headers but crashes with new binary.
  Example: adding a data member (old binary expects old sizeof).

API INCOMPATIBLE, ABI INCOMPATIBLE:
  Old code doesn't compile AND doesn't run.
  Example: renaming a function.

API INCOMPATIBLE, ABI compatible:
  Old code doesn't compile but old binary still works.
  Example: making a function [[nodiscard]] (source warning, same binary).
```

The most treacherous case is "API compatible, ABI INCOMPATIBLE" - your code compiles clean, so you ship it, and then things crash at runtime in ways that are very hard to diagnose.

**Comparison:**

| Aspect | API compatibility | ABI compatibility |
| --- | --- | --- |
| Level | Source code | Binary (object code) |
| Detected by | Compiler | Nothing (crashes at runtime) |
| Example break | Rename function | Add data member |
| Fix | Recompile | Recompile ALL dependents |
| Maintained by | Header design | Object layout stability |

---

## Notes

- The C++ standard does not define ABI. Each platform has its own (Itanium C++ ABI on Linux, MSVC ABI on Windows).
- libstdc++ and libc++ have different ABIs - you can't mix them.
- Techniques for ABI stability: Pimpl, abstract interfaces (pure virtual), C-style `extern "C"` functions.
- `inline namespace` can be used for ABI versioning (e.g., `namespace v2 { ... }`).
- COM (Windows) and D-Bus (Linux) are ABI-stable interface technologies.
