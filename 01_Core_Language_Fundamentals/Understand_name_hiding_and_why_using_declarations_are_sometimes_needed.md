# Understand name hiding and why using declarations are sometimes needed

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/using_declaration>  

---

## Topic Overview

**Name hiding** occurs when a name declared in an inner scope (e.g., derived class) hides all entities with the same name in an outer scope (e.g., base class). This is a fundamental C++ lookup rule that frequently surprises developers.

### The Problem: Derived Class Hides Base Overloads

Here's the scenario that catches everyone off guard. Adding a single function to a derived class silently hides every overload of the same name in the base:

```cpp
struct Base {
    void foo(int)    { std::cout << "Base::foo(int)\n"; }
    void foo(double) { std::cout << "Base::foo(double)\n"; }
};

struct Derived : Base {
    void foo(std::string) { std::cout << "Derived::foo(string)\n"; }
    // ALL Base::foo overloads are now HIDDEN!
};

Derived d;
d.foo("hello");   // OK - Derived::foo(string)
d.foo(42);        // ERROR! Base::foo(int) is hidden
d.foo(3.14);      // ERROR! Base::foo(double) is hidden
```

### The Fix: `using` Declaration

One line is all it takes to bring all the base class overloads back into scope:

```cpp
struct Derived : Base {
    using Base::foo;  // Bring ALL Base::foo overloads into Derived's scope
    void foo(std::string) { std::cout << "Derived::foo(string)\n"; }
};

Derived d;
d.foo("hello");   // OK - Derived::foo(string)
d.foo(42);        // OK - Base::foo(int), now visible
d.foo(3.14);      // OK - Base::foo(double), now visible
```

### Why Name Hiding Exists

Name hiding is **intentional** in C++:

1. **Safety:** Adding a function to a base class shouldn't silently change the meaning of code in derived classes.
2. **Locality:** The derived class author controls which names are visible - preventing accidental calls to base functions.
3. **Predictability:** Without hiding, adding `Base::foo(int)` later could redirect all `d.foo(42)` calls away from the derived class version.

### Name Hiding vs. Overriding vs. Overloading

If the table feels like a lot, the key distinction is this: hiding is a lookup-time effect (scope-based), overriding is a runtime effect (virtual table), and overloading is when multiple functions share a name in the same scope.

| Concept | Same signature? | `virtual`? | Effect |
| --- | --- | --- | --- |
| **Hiding** | Any (same or different) | No | Base name becomes invisible in derived scope |
| **Overriding** | Must match | Yes | Derived provides new implementation for virtual function |
| **Overloading** | Different parameters | N/A | Multiple functions with same name in **same** scope |

---

## Self-Assessment

### Q1: Show that a derived class method with the same name but different signature hides all base overloads

Notice that `Derived::process` takes a `std::string`, which is completely different from any of the base overloads. That doesn't matter - name hiding works on the name alone, not the signature.

```cpp
#include <iostream>
#include <string>

struct Base {
    void process(int x)    { std::cout << "Base::process(int): " << x << "\n"; }
    void process(double x) { std::cout << "Base::process(double): " << x << "\n"; }
    void process(char c)   { std::cout << "Base::process(char): " << c << "\n"; }
};

struct Derived : Base {
    // This ONE function hides ALL THREE Base::process overloads
    void process(const std::string& s) {
        std::cout << "Derived::process(string): " << s << "\n";
    }
};

int main() {
    Derived d;
    d.process("hello");    // OK - Derived::process(string)

    // ALL of these fail to compile - Base overloads are hidden:
    // d.process(42);      // error: no matching function
    // d.process(3.14);    // error: no matching function
    // d.process('A');     // error: no matching function

    // Workaround 1: explicit qualification
    d.Base::process(42);   // OK - explicitly calls Base::process(int)

    // Workaround 2: use `using` declaration (see Q2)
}
```

**How this works:**

- `Derived::process(const std::string&)` has a different signature from all base overloads, yet it hides **all of them**.
- Name hiding happens based on the **name** alone - signatures don't matter.
- When the compiler finds `process` in `Derived`'s scope, it stops looking in `Base`.
- Explicit qualification `d.Base::process(42)` bypasses hiding but is verbose.

### Q2: Use `using Base::method;` to bring hidden base overloads back into scope

After adding `using Base::process;`, the compiler treats all four overloads as if they were declared in the same scope and picks the best match normally.

```cpp
#include <iostream>
#include <string>

struct Base {
    void process(int x)    { std::cout << "Base::process(int): " << x << "\n"; }
    void process(double x) { std::cout << "Base::process(double): " << x << "\n"; }
    void process(char c)   { std::cout << "Base::process(char): " << c << "\n"; }
};

struct Derived : Base {
    using Base::process;  // <-- Bring ALL Base::process overloads into scope

    void process(const std::string& s) {
        std::cout << "Derived::process(string): " << s << "\n";
    }
};

int main() {
    Derived d;

    d.process("hello");   // Derived::process(string)
    d.process(42);        // Base::process(int) - now visible!
    d.process(3.14);      // Base::process(double) - now visible!
    d.process('A');       // Base::process(char) - now visible!

    // Overload resolution works across ALL four overloads
    // as if they were all declared in the same scope
}
```

**How this works:**

- `using Base::process;` introduces **all** `Base::process` overloads into `Derived`'s scope.
- Now overload resolution considers both `Base::process(int/double/char)` and `Derived::process(string)`.
- If `Derived` also had `process(int)`, it would **override** the `using`-imported `Base::process(int)` (Derived version wins).

### Q3: Explain why name hiding is intentional and how it interacts with virtual dispatch

**Answer:**

**Why intentional:**

Name hiding protects derived classes from base class changes. The scenario below shows why you want this protection:

```cpp
struct Base {
    // Version 1: no foo(int)
};

struct Derived : Base {
    void foo(long x) { /* uses x as long */ }
};

Derived d;
d.foo(42);  // Calls Derived::foo(long) - 42 converts to long

// Later, Base adds:
// void foo(int x) { /* something different */ }
// Without name hiding, d.foo(42) would silently call Base::foo(int) instead!
// Name hiding prevents this - Derived::foo(long) keeps being called.
```

**Interaction with virtual dispatch:**

Here's the subtlety: name hiding is a compile-time lookup rule, but virtual dispatch is a runtime mechanism. They operate at different stages, and you can be caught by both at once:

```cpp
#include <iostream>

struct Base {
    virtual void print(int x) { std::cout << "Base::print(int): " << x << "\n"; }
    virtual void print(double x) { std::cout << "Base::print(double): " << x << "\n"; }
};

struct Derived : Base {
    // Overrides print(int) but HIDES print(double)!
    void print(int x) override { std::cout << "Derived::print(int): " << x << "\n"; }
};

int main() {
    Derived d;
    Base* bp = &d;

    bp->print(42);    // Virtual dispatch -> Derived::print(int) (works fine via pointer)
    bp->print(3.14);  // Virtual dispatch -> Base::print(double) (works fine via pointer)

    d.print(42);      // Derived::print(int)
    // d.print(3.14); // ERROR! Derived hides Base::print(double)
    // Even though it's virtual, name hiding still applies at compile time!
}
```

**Key insight:** Name hiding is a **compile-time** lookup rule. Virtual dispatch is a **runtime** mechanism. When called through a base pointer/reference, virtual dispatch works normally. But when called directly on a derived object, name hiding prevents finding the base overload.

---

## Notes

- Name hiding applies to **all names**: functions, types, variables, enums - not just member functions.
- A `using` declaration in a derived class does not affect `private` base members - access control still applies.
- `using Base::operator=;` is commonly needed when a derived class defines its own assignment operator but wants to keep the base version available.
- In templates, name hiding interacts with **two-phase lookup**: dependent names in base classes require `this->` or `using` to be found.
