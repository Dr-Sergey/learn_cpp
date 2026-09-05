# Understand Module Reachability and Visibility Rules

**Category:** Modules and Build Cpp20  
**Standard:** C++20  
**Reference:** [cppreference - Modules](https://en.cppreference.com/w/cpp/language/modules)  

---

## Topic Overview

C++20 modules introduce two distinct concepts controlling how importers interact with declarations: **visibility** and **reachability**. These replace the header-era model where `#include` makes everything textually visible. Understanding the distinction is essential for designing clean module interfaces and avoiding surprising compilation errors.

**Visibility** means a name can be found by name lookup. A declaration is visible to an importer only if it is **exported** from the module interface. **Reachability** means the semantic properties of a declaration (its type, members, base classes) are available to the compiler even if the name cannot be looked up. A declaration is reachable if it appears in any module interface unit that is transitively imported, regardless of whether it was exported.

The reason this trips people up is that visibility and reachability can come apart: a type can be completely usable (you can create values of it, access its members, call its methods) while being completely unnamed (you cannot write its name in your source code). Think of it as the type existing and being understood by the compiler, but having no spell-able name from where you are.

| Property | Condition | Effect |
| --- | --- | --- |
| Visible | `export`-ed and in an interface unit | Name lookup finds it. Can name the type directly. |
| Reachable but not visible | In interface unit, not `export`-ed | Compiler knows the type's layout/members. Cannot name it. |
| Not reachable | In implementation unit | Compiler has no information. Completely hidden. |

Transitive imports add another layer. `export import M2;` inside module `M1` makes all of `M2`'s exported declarations **visible** (and reachable) to importers of `M1`. A non-exported `import M2;` inside `M1`'s interface unit makes `M2`'s exported declarations **reachable** (but not visible) to importers of `M1`. This is the mechanism by which types "leak" through return types, parameter types, and base classes without being directly nameable.

```cpp
+---------------------------------------------------------+
|               Importer: main.cpp                        |
|  import M1;                                             |
|                                                         |
|  +-------------+    +------------------------------+    |
|  |  Visible     |    |  Reachable (not visible)     |    |
|  |  M1 exports  |    |  M2 exports (via non-export  |    |
|  |              |    |  import in M1 interface)      |    |
|  +-------------+    +------------------------------+    |
|                                                         |
|  +--------------------------------------------------+   |
|  |  NOT reachable                                    |   |
|  |  M1 implementation-unit declarations              |   |
|  |  M2 non-exported declarations                     |   |
|  +--------------------------------------------------+   |
+---------------------------------------------------------+
```

The practical consequence: you can return a non-exported type from an exported function, and the caller can use `auto` to capture and operate on it - but cannot spell its name. This is the "Voldemort type" pattern, now formalized through reachability.

---

## Self-Assessment

### Q1: Which types can `main.cpp` name directly, and which can it only use via `auto`

This example has three types to track. `Token` is exported. `SourceLoc` is in the interface unit but not exported. `AnnotatedToken` is exported and happens to contain a `SourceLoc` member. Watch what the consumer can and cannot do:

```cpp
// ---------- detail.cppm ----------
export module detail;

export struct Token {
    int id;
    const char* text;
};

struct SourceLoc {               // not exported
    int line, col;
};

export struct AnnotatedToken {
    Token tok;
    SourceLoc loc;               // SourceLoc reachable (in interface), not visible
};

export AnnotatedToken next_token();

// ---------- main.cpp ----------
import detail;

int main() {
    Token t{1, "hello"};              // OK: Token is visible (exported)

    AnnotatedToken at = next_token(); // OK: AnnotatedToken is visible
    int line = at.loc.line;           // OK: SourceLoc is reachable, members accessible

    // SourceLoc sl{10, 5};           // ERROR: SourceLoc is NOT visible
    auto loc = at.loc;                // OK: auto deduces reachable type
    // decltype(at.loc) loc2{};       // OK per standard (reachable), but some
                                      // compilers may warn
}
```

`Token` and `AnnotatedToken` are **visible** - exported, directly nameable. `SourceLoc` is **reachable** - its members and layout are known (you can access `.line`), but you cannot name the type `SourceLoc` in `main.cpp`. The `auto` keyword or `decltype` are required to work with it.

---

### Q2: How do transitive imports affect visibility and reachability

This is where the rules get subtle. Whether a type ends up visible or merely reachable in your code depends on how the chain of imports is set up - not just whether the type was exported in its home module. Here's an example with `Widget` flowing through two different import chains:

```cpp
// ---------- base.cppm ----------
export module base;

export class Widget {
public:
    virtual void draw() const = 0;
    virtual ~Widget() = default;
};

// ---------- toolkit.cppm ----------
export module toolkit;

export import base;              // re-export: Widget becomes VISIBLE to toolkit importers

export class Button : public Widget {
public:
    void draw() const override;
};

// ---------- app_framework.cppm ----------
export module app_framework;

import base;                     // non-export import: Widget is REACHABLE only

export class App {
public:
    virtual void on_start();
    Widget* root_widget();       // return type uses reachable type
};

// ---------- main.cpp ----------
import toolkit;
import app_framework;

int main() {
    // Via toolkit (export import base):
    Widget* w = nullptr;              // OK: Widget is VISIBLE through toolkit
    Button b;
    b.draw();

    // Via app_framework (import base, no re-export):
    App app;
    auto* root = app.root_widget();   // OK: Widget* is reachable
    root->draw();                     // OK: can call through reachable type

    // If we had ONLY imported app_framework (without toolkit):
    // Widget* w2;                    // would be ERROR: Widget not visible
    // auto* w3 = app.root_widget(); // would be OK: still reachable
}
```

The table makes the rule concrete - notice that when both imports are present, the strongest property wins:

| In `toolkit.cppm` | In `app_framework.cppm` | `Widget` in `main.cpp` |
| --- | --- | --- |
| `export import base;` | - | Visible + reachable |
| - | `import base;` | Reachable only |
| Both imported | Both imported | Visible (strongest wins) |

---

### Q3: What are the practical consequences of reachability for template instantiations and ADL

This is the trickiest question in the set. When a non-exported type is reachable, ADL (Argument-Dependent Lookup) can still find friend functions associated with it. Without this rule, reachable-but-not-visible types returned from exported functions would be nearly impossible to use - you couldn't even compare two values for equality.

```cpp
// ---------- geo.cppm ----------
export module geo;

struct Point {                   // not exported
    double x, y;
    friend bool operator==(Point a, Point b) {   // found by ADL
        return a.x == b.x && a.y == b.y;
    }
};

export Point origin() { return {0.0, 0.0}; }
export Point translate(Point p, double dx, double dy) {
    return {p.x + dx, p.y + dy};
}

// ---------- algo.cppm ----------
export module algo;

import geo;                      // non-export import

export template<typename T>
bool is_same_position(T a, T b) {
    return a == b;               // relies on operator== being reachable
}

// ---------- main.cpp ----------
import geo;
import algo;

int main() {
    auto p1 = origin();
    auto p2 = translate(p1, 1.0, 0.0);

    // Point is reachable: ADL finds operator== through the associated module
    bool same = (p1 == p2);             // OK: ADL finds friend operator==

    bool check = is_same_position(p1, p2); // OK: template instantiated,
                                            // operator== found via ADL

    // Point p3{0, 0};                  // ERROR: Point not visible (can't name it)
}
```

The critical insight: ADL considers the **owning module** of a type's associated entities. Even though `Point` is not exported, ADL can find `operator==` declared as a friend because the type is reachable and ADL searches the associated module. This is by-design - without this rule, non-exported types returned from exported functions would be largely unusable.

---

## Notes

- **Visibility is a subset of Reachability**: every visible declaration is reachable, but not vice versa.
- `export import M;` propagates both visibility and reachability. Plain `import M;` in an interface unit propagates only reachability.
- `import M;` in an **implementation unit** propagates nothing to the module's importers.
- The "Voldemort type" pattern (reachable-but-not-visible) is intentional - it enables encapsulation while allowing rich return types.
- ADL is module-aware: it searches the owning module of associated types, finding non-visible but reachable friend functions.
- Reachability applies to class completeness: if a class is reachable, the compiler knows its full definition (size, members, bases), enabling `sizeof`, member access, and inheritance.
- `decltype` on a reachable-but-not-visible type is valid - the type exists, it simply has no visible name in the importing TU.
- When designing module interfaces, deliberately control which types are exported; use reachability for implementation types that appear in signatures but should not be part of the public API surface.
- Compilers encode reachability in the BMI (Binary Module Interface) - all declarations from interface units are included, not just exported ones.
