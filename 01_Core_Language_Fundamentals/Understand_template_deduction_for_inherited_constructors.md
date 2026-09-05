# Understand template deduction for inherited constructors

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/using_declaration>  

---

## Topic Overview

Inherited constructors (`using Base::Base;`) bring base class constructors into the derived class. When combined with class template argument deduction (CTAD), some interesting interactions arise.

### Inheriting Constructors

The `using Base::Base;` declaration is a compact way to expose every base constructor in your derived class. Think of it as having the compiler write one thin forwarding constructor for each base constructor automatically.

```cpp
struct Base {
    Base(int x) { std::cout << "Base(int): " << x << "\n"; }
    Base(int x, double y) { std::cout << "Base(int,double): " << x << "," << y << "\n"; }
};

struct Derived : Base {
    using Base::Base;  // Inherit ALL Base constructors
    // Equivalent to:
    // Derived(int x) : Base(x) {}
    // Derived(int x, double y) : Base(x, y) {}
};

Derived d1(42);       // calls Base(int)
Derived d2(1, 3.14);  // calls Base(int, double)
```

Both constructors of `Base` are now usable on `Derived` without repeating them.

### CTAD and Inherited Constructors

With C++17, CTAD can deduce template arguments from constructor parameters. When constructors are inherited, the deduction guides from the base class are **not** automatically inherited - you may need explicit deduction guides.

```cpp
template<typename T>
struct Wrapper {
    T value;
    Wrapper(T v) : value(v) {}
};

// CTAD works: Wrapper w(42) -> Wrapper<int>

template<typename T>
struct DerivedWrapper : Wrapper<T> {
    using Wrapper<T>::Wrapper;  // Inherit constructors
};

// DerivedWrapper dw(42);  // May NOT work — CTAD guides aren't inherited
// Fix: add explicit deduction guide
template<typename T>
DerivedWrapper(T) -> DerivedWrapper<T>;
// Now: DerivedWrapper dw(42); -> DerivedWrapper<int>
```

The explicit deduction guide at the bottom is what gives CTAD the information it needs to deduce `T` for `DerivedWrapper`.

### Constructor Shadowing

If the derived class defines a constructor with the same signature as an inherited one, the derived version **wins** (shadows the inherited one).

```cpp
struct Base {
    Base(int x) { std::cout << "Base(int)\n"; }
};

struct Derived : Base {
    using Base::Base;  // Inherit Base(int)
    Derived(int x) : Base(x) { std::cout << "Derived(int)\n"; }  // Shadows!
};

Derived d(42);  // Calls Derived(int), NOT Base(int) directly
```

Shadowing applies per signature - if `Derived` only defines `Derived(int)`, the inherited `Base(double)` (if any) is still available.

---

## Self-Assessment

### Q1: Show that `using Base::Base;` inherits all base constructors into the derived class

Here the interesting part is that each inherited constructor initializes the base subobject, while derived members still pick up their default member initializers.

```cpp
#include <iostream>
#include <string>

struct Base {
    int x;
    Base() : x(0) { std::cout << "Base()\n"; }
    Base(int v) : x(v) { std::cout << "Base(int): " << v << "\n"; }
    Base(int v, const std::string& s) : x(v) {
        std::cout << "Base(int, string): " << v << ", " << s << "\n";
    }
};

struct Derived : Base {
    using Base::Base;  // Inherits ALL three constructors
    // Derived's own members are default-initialized
    double extra = 3.14;
};

int main() {
    Derived d1;              // Base() -> x=0, extra=3.14
    Derived d2(42);          // Base(int) -> x=42, extra=3.14
    Derived d3(1, "hello");  // Base(int, string) -> x=1, extra=3.14

    std::cout << "d1.x=" << d1.x << " extra=" << d1.extra << "\n";
    std::cout << "d2.x=" << d2.x << " extra=" << d2.extra << "\n";
    std::cout << "d3.x=" << d3.x << " extra=" << d3.extra << "\n";
}
```

Notice that `extra` always ends up as `3.14` regardless of which inherited constructor was called - it uses its own default member initializer because no inherited constructor touches derived members.

**How this works:**

- `using Base::Base;` injects all base constructors as if they were defined in `Derived`.
- Derived members use their default member initializers (`extra = 3.14`).
- Inherited constructors do NOT initialize derived members unless they have defaults - this is a common pitfall.

### Q2: Explain how CTAD interacts with inherited constructors

**Answer:**

CTAD (Class Template Argument Deduction) and inherited constructors have a tricky interaction:

1. Base class CTAD guides are NOT automatically inherited. Only explicitly declared or compiler-generated deduction guides for the derived class are used.
2. You must add explicit deduction guides for the derived class if you want CTAD to work.

The example below shows the problem and fix side by side - the deduction guide for `NamedContainer` is the only thing that makes the unqualified `NamedContainer nc(42)` compile.

```cpp
#include <iostream>

template<typename T>
struct Container {
    T value;
    Container(T v) : value(v) {}
};

// CTAD works: Container c(42) -> Container<int>

template<typename T>
struct NamedContainer : Container<T> {
    using Container<T>::Container;  // Inherit constructors
    std::string name = "default";
};

// Without deduction guide:
// NamedContainer nc(42);  // ERROR: CTAD can't deduce T

// With explicit deduction guide:
template<typename T>
NamedContainer(T) -> NamedContainer<T>;

// Now works:
// NamedContainer nc(42);  // -> NamedContainer<int>

int main() {
    NamedContainer nc(42);
    std::cout << "value: " << nc.value << "\n";  // 42
    std::cout << "name: " << nc.name << "\n";    // "default"

    NamedContainer ns(std::string("hello"));
    std::cout << "value: " << ns.value << "\n";  // "hello"
}
```

**Key rules:**

- C++17 CTAD only considers the class's own constructors and deduction guides.
- Inherited constructors are "transparent" to CTAD - the base's guides don't help.
- Always provide explicit deduction guides for template classes with inherited constructors if CTAD usage is needed.

### Q3: Show a case where an inherited constructor shadows a derived class constructor of the same signature

Watch what happens with `Derived d1(10)` - both `Base(int)` and `Derived(int)` exist, but only one wins.

```cpp
#include <iostream>

struct Base {
    int x;
    Base(int v) : x(v) {
        std::cout << "Base(int): " << v << "\n";
    }
    Base(double d) : x(static_cast<int>(d)) {
        std::cout << "Base(double): " << d << "\n";
    }
};

struct Derived : Base {
    using Base::Base;  // Inherits Base(int) and Base(double)

    // This SHADOWS the inherited Base(int)
    Derived(int v) : Base(v * 2) {
        std::cout << "Derived(int): " << v << " (doubled to " << x << ")\n";
    }
    // Base(double) is STILL inherited — not shadowed
};

int main() {
    Derived d1(10);
    // Output:
    //   Base(int): 20
    //   Derived(int): 10 (doubled to 20)
    // -> Derived(int) was called, NOT Base(int)

    Derived d2(3.14);
    // Output:
    //   Base(double): 3.14
    // -> Inherited Base(double), no Derived(double) defined

    std::cout << "d1.x = " << d1.x << "\n";  // 20 (doubled)
    std::cout << "d2.x = " << d2.x << "\n";  // 3
}
```

The derived constructor for `int` intercepts the call and doubles the value before passing it to the base. The `double` overload has no shadow, so it flows straight through to `Base`.

**How this works:**

- When `Derived` defines `Derived(int)`, it shadows the inherited `Base(int)`.
- `Base(double)` remains inherited because `Derived` doesn't define a `Derived(double)`.
- Shadowing is based on signature matching - same parameter types = shadow.
- The derived constructor can still delegate to the base via `Base(v * 2)`.

---

## Notes

- Inherited constructors are implicitly `inline` and have the same access specifier as in the base class.
- If the base has a constructor that the derived class's `using` would make accessible, but it's `private` in the base, the inherited constructor is also inaccessible.
- `using Base::Base;` does NOT inherit the default constructor if the derived class has any non-default-constructible members without default member initializers.
- In C++17, inherited constructors are treated as if the derived class had them directly - this changed subtly from C++11/14 where they were "forwarding" constructors.
