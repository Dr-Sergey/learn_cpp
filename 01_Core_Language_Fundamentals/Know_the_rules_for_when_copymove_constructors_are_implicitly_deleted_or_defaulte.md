# Know the rules for when copy/move constructors are implicitly deleted or defaulted

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/copy_constructor>  

---

## Topic Overview

C++ automatically generates up to **five special member functions**. Whether each one is **implicitly declared as defaulted**, **implicitly deleted**, or **not declared at all** depends on what other special members you have user-declared. Getting this wrong is how you accidentally make a type non-copyable or non-movable.

### The Five Special Members

| # | Member | Purpose |
| --- | --- | --- |
| 1 | Destructor | Cleanup |
| 2 | Copy constructor | `T(const T&)` |
| 3 | Copy assignment | `T& operator=(const T&)` |
| 4 | Move constructor | `T(T&&)` |
| 5 | Move assignment | `T& operator=(T&&)` |

### The Compatibility Matrix

The rows show what the compiler will implicitly generate; the columns show what you've already user-declared. Read it as: "If I declare X, what happens to Y?"

| You user-declare -> | Dtor | Copy Ctor | Copy Assign | Move Ctor | Move Assign |
| --- | --- | --- | --- | --- | --- |
| **Implicit Copy Ctor** | defaulted† | not declared | defaulted | **deleted** | **deleted** |
| **Implicit Copy Assign** | defaulted† | defaulted | not declared | **deleted** | **deleted** |
| **Implicit Move Ctor** | **not declared** | **not declared** | **not declared** | not declared | **not declared** |
| **Implicit Move Assign** | **not declared** | **not declared** | **not declared** | **not declared** | not declared |
| **Implicit Dtor** | not declared | defaulted | defaulted | defaulted | defaulted |

† = deprecated behavior (C++11); future standards may delete instead.

### Key Rules

1. **Declaring any move operation** -> implicit copy operations are **deleted** (not just suppressed).
2. **Declaring any copy operation or destructor** -> implicit move operations are **not declared** (the class falls back to copy).
3. `= default` can **restore** a suppressed or deleted implicit member - but only if the defaulted body would be well-formed.
4. `= delete` counts as user-declared - it triggers the same suppression rules.

### The Rule of Zero / Five

The practical takeaway is two patterns that cover almost every case. If you remember only one thing from this topic, let it be these:

```cpp
// Rule of Zero: let the compiler do everything
struct Good {
    std::string name;
    std::vector<int> data;
    // All five members are implicitly defaulted - correct and efficient
};

// Rule of Five: if you need one, declare all five
class ResourceOwner {
    int* ptr;
public:
    ResourceOwner() : ptr(new int(0)) {}
    ~ResourceOwner() { delete ptr; }

    ResourceOwner(const ResourceOwner& o) : ptr(new int(*o.ptr)) {}
    ResourceOwner& operator=(const ResourceOwner& o) {
        if (this != &o) { *ptr = *o.ptr; }
        return *this;
    }

    ResourceOwner(ResourceOwner&& o) noexcept : ptr(o.ptr) { o.ptr = nullptr; }
    ResourceOwner& operator=(ResourceOwner&& o) noexcept {
        delete ptr;
        ptr = o.ptr;
        o.ptr = nullptr;
        return *this;
    }
};
```

---

## Self-Assessment

### Q1: Show that declaring a move constructor suppresses implicit copy constructor generation

Watch what happens when you declare just the move constructor - the compiler deletes your copy operations:

```cpp
#include <iostream>

struct Widget {
    int value = 0;
    Widget() = default;
    Widget(Widget&& other) noexcept : value(other.value) {   // user-declared move ctor
        other.value = -1;
        std::cout << "Move ctor\n";
    }
    // Implicit copy ctor is now DELETED
    // Implicit copy assignment is now DELETED
    // Implicit move assignment is NOT DECLARED
};

int main() {
    Widget a;
    a.value = 42;

    Widget b = std::move(a);   // OK - uses move ctor
    std::cout << "b.value = " << b.value << "\n";   // 42

    // Widget c = b;            // ERROR: copy ctor is deleted
    // Widget d; d = b;         // ERROR: copy assignment is deleted
}
```

Note the asymmetry: move assignment is "not declared" (absent), while copy is "deleted" (present but unusable). If you try `Widget c = b;` the compiler actively rejects it.

How it works:

- By declaring `Widget(Widget&&)`, the compiler **deletes** the implicit copy constructor and copy assignment operator.
- Attempting `Widget c = b;` (copy) produces a compile error: `use of deleted function`.
- The move assignment operator is **not declared** (not deleted - just absent) because we only declared a move constructor, not a move assignment.

### Q2: Explain the `= default` escape hatch and when it restores a deleted implicit member

`= default` is how you say "yes, I want to trigger the suppression rules, but also please give me back the operations I still need":

```cpp
struct Gadget {
    std::string name;

    // User-declared move constructor -> copy operations deleted
    Gadget(Gadget&&) noexcept = default;

    // Restore copy operations with = default
    Gadget(const Gadget&) = default;            // restored!
    Gadget& operator=(const Gadget&) = default; // restored!

    // Also restore move assignment
    Gadget& operator=(Gadget&&) noexcept = default;

    Gadget() = default;
};

int main() {
    Gadget a;
    a.name = "test";

    Gadget b = a;              // copy ctor - works because we defaulted it
    Gadget c = std::move(a);   // move ctor - works
    b = c;                     // copy assign - works
}
```

There's a limit to what `= default` can restore. If the defaulted body is inherently ill-formed, you still get a deleted function - just at a different stage:

```cpp
struct Bad {
    const int x;                    // const member -> no assignment possible
    Bad() : x(0) {}
    Bad& operator=(const Bad&) = default;   // = default compiles but is DELETED
    // because the defaulted body is ill-formed (can't assign to const member)
};
```

`= default` on a special member whose defaulted body would be ill-formed (e.g., assigning to a `const` or reference member) results in a **deleted** function.

### Q3: Compatibility matrix for all five special members and their mutual suppression rules

The complete picture, in code format with a legend - useful to keep as a reference:

```cpp
                           User-declared:
                     Nothing  Dtor  CopyCtor  CopyAsgn  MoveCtor  MoveAsgn
Implicit Dtor        default  ----  default   default   default   default
Implicit CopyCtor    default  dflt† --------  default   DELETED   DELETED
Implicit CopyAsgn   default  dflt† default   --------  DELETED   DELETED
Implicit MoveCtor    default  none  none      none      --------  none
Implicit MoveAsgn   default  none  none      none      none      --------

Legend:
  default  = implicitly declared as defaulted
  dflt†    = defaulted but DEPRECATED (may become deleted in future)
  none     = not declared at all (does not participate in overload resolution)
  DELETED  = implicitly declared as deleted (participates but fails if selected)
  --------  = not applicable (you declared it yourself)
```

You can verify each column at compile time using the type traits:

```cpp
// Demonstrate each column:
#include <type_traits>
#include <iostream>

struct A {};                                     // nothing declared
struct B { ~B() {} };                            // user-declared dtor
struct C { C(const C&) = default; C() = default; }; // user-declared copy ctor
struct D { D(D&&) noexcept; D() = default; };    // user-declared move ctor

int main() {
    std::cout << std::boolalpha;

    // A: all five are implicitly defaulted
    std::cout << "A copyable: " << std::is_copy_constructible_v<A> << "\n";    // true
    std::cout << "A movable:  " << std::is_move_constructible_v<A> << "\n";    // true

    // B: dtor declared -> move ops not declared, copy ops deprecated-defaulted
    std::cout << "B copyable: " << std::is_copy_constructible_v<B> << "\n";    // true
    std::cout << "B movable:  " << std::is_move_constructible_v<B> << "\n";    // true (falls back to copy)

    // C: copy ctor declared -> move ops not declared
    std::cout << "C copyable: " << std::is_copy_constructible_v<C> << "\n";    // true
    std::cout << "C movable:  " << std::is_move_constructible_v<C> << "\n";    // true (falls back to copy)

    // D: move ctor declared -> copy ops DELETED
    std::cout << "D copyable: " << std::is_copy_constructible_v<D> << "\n";    // false!
    std::cout << "D movable:  " << std::is_move_constructible_v<D> << "\n";    // true
}
```

Notice `D` - declaring the move constructor made the class non-copyable. That's exactly what `std::unique_ptr` relies on.

---

## Notes

- The asymmetry (move suppresses copy as **deleted**, but copy/dtor suppresses move as **not declared**) is intentional - "not declared" allows fallback to copy, while "deleted" blocks copy entirely.
- `std::is_move_constructible_v<T>` returns `true` if move OR copy works (because `const T&` binds to rvalues). Use `std::is_nothrow_move_constructible_v` to confirm a true move exists.
- Class template `std::unique_ptr` is the canonical example: it declares a move constructor and move assignment, so its copy operations are deleted.
- Always prefer the **Rule of Zero** - let RAII wrappers manage resources so you declare none of the five.
