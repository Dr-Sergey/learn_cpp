# Understand Guaranteed Copy Elision (prvalue materialization, C++17)

**Category:** Move Semantics and Value Categories  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/copy_elision>  

---

## Topic Overview

### What Changed in C++17

Before C++17, copy/move elision was an optimization the compiler was *permitted* to perform. In C++17, certain cases are **mandatory** - the standard redefines prvalues so that no copy/move ever exists in the first place.

The key mental shift is this: in C++17, a prvalue like `T{args}` is not an object - it is an *initialization recipe*. It tells the compiler how to construct the target object directly, without creating a temporary along the way. No temporary means no copy or move is needed. That is not an optimization; it is a change in what the language says is happening.

| Concept | Pre-C++17 | C++17+ |
| --- | --- | --- |
| `T x = T{args}` | Conceptually copy/move, may be elided | **No copy/move exists** - prvalue directly initializes `x` |
| `return T{args}` (URVO) | May be elided | **Guaranteed** - prvalue materializes at call site |
| `return named_var` (NRVO) | May be elided | **Still optional** (not guaranteed) |
| Non-movable types | Can't be returned by value | **CAN** be returned by value (if prvalue) |

### Key Terminology

- **Prvalue materialization**: A prvalue doesn't create a temporary - it's an *initialization recipe* that directly constructs the target object.
- **URVO** (Unnamed Return Value Optimization): `return T{...}` - guaranteed in C++17.
- **NRVO** (Named Return Value Optimization): `return named` - still optional, but nearly universal.

---

## Self-Assessment

### Q1: Show that `Widget f() { return Widget{42}; }` is guaranteed to not call any copy/move constructor in C++17

The constructors in this example are instrumented so you can see exactly what gets called. In C++17, you will only ever see the regular constructor - never the copy or move constructor - regardless of how many times a prvalue passes through function boundaries:

```cpp
#include <iostream>

class Widget {
    int value_;
public:
    Widget(int v) : value_(v) {
        std::cout << "  Constructor(" << v << ")\n";
    }

    // Explicitly track copy and move
    Widget(const Widget& other) : value_(other.value_) {
        std::cout << "  COPY constructor\n";
    }
    Widget(Widget&& other) noexcept : value_(other.value_) {
        std::cout << "  MOVE constructor\n";
    }

    int value() const { return value_; }
};

// Return prvalue -- guaranteed no copy/move in C++17
Widget make_widget(int v) {
    return Widget{v};  // Prvalue: directly constructs at call site
}

// Chaining prvalues -- still no copy/move!
Widget make_double(int v) {
    return make_widget(v * 2);  // Prvalue -> prvalue -> direct initialization
}

int main() {
    std::cout << "=== Guaranteed Copy Elision (C++17) ===\n\n";

    std::cout << "1. Direct initialization from prvalue:\n";
    Widget w1 = Widget{42};
    // Output: only "Constructor(42)" -- no copy or move

    std::cout << "\n2. Function returning prvalue:\n";
    Widget w2 = make_widget(100);
    // Output: only "Constructor(100)" -- guaranteed, not just optimized

    std::cout << "\n3. Chained prvalue functions:\n";
    Widget w3 = make_double(50);
    // Output: only "Constructor(100)" -- still no copy/move

    std::cout << "\n4. Prvalue as function argument:\n";
    auto show = [](Widget w) {
        std::cout << "  Received: " << w.value() << "\n";
    };
    show(Widget{77});
    // C++17: Widget{77} directly constructs the parameter

    std::cout << "\nResults: " << w1.value() << ", "
              << w2.value() << ", " << w3.value() << "\n";

    return 0;
}
// Expected output (C++17, all compilers):
//   1. Direct initialization from prvalue:
//     Constructor(42)
//   2. Function returning prvalue:
//     Constructor(100)
//   3. Chained prvalue functions:
//     Constructor(100)
//   4. Prvalue as function argument:
//     Constructor(77)
//     Received: 77
//   Results: 42, 100, 100
```

The chained case is especially worth noticing: `make_double` calls `make_widget`, which returns a prvalue, and `make_double` in turn returns that prvalue. The recipe gets passed along the chain, and the actual construction happens only once - directly into `w3`.

### Q2: Explain why NRVO is not guaranteed while URVO is mandatory in C++17

**URVO (mandatory):** When returning a prvalue (`return T{...}`), the standard says prvalues are not objects - they are recipes for initialization. The prvalue directly initializes the caller's target. No temporary exists, so no copy/move is needed.

**NRVO (optional):** When returning a named variable (`return local_var`), the variable **is** an object with identity (an lvalue). The compiler must logically create it, then copy/move it to the return slot. Compilers **usually** optimize this away, but the standard can't mandate it because:

1. Multiple return paths may return different named variables
2. The named variable may need to exist at a specific address
3. Exception handling may need the local to exist separately

The fundamental difference is that a named variable has identity - it lives somewhere specific in memory. A prvalue has no identity; it is just a description of how to make a value, so there is nothing to copy or move.

```cpp
#include <iostream>

class Heavy {
    int id_;
public:
    Heavy(int id) : id_(id) { std::cout << "  Construct #" << id_ << "\n"; }
    Heavy(const Heavy& o) : id_(o.id_) { std::cout << "  COPY #" << id_ << "\n"; }
    Heavy(Heavy&& o) noexcept : id_(o.id_) { std::cout << "  MOVE #" << id_ << "\n"; }
    int id() const { return id_; }
};

// URVO: return prvalue -> GUARANTEED no copy/move (C++17)
Heavy urvo() {
    return Heavy{1};  // Prvalue -- directly initializes at call site
}

// NRVO: return named variable -> compiler USUALLY elides, but NOT guaranteed
Heavy nrvo() {
    Heavy h{2};       // Named local -- has identity (lvalue)
    return h;         // NRVO likely applied, but not mandated
}

// Case where NRVO is difficult/impossible
Heavy conditional_return(bool flag) {
    Heavy a{10};
    Heavy b{20};
    if (flag) return a;  // Which variable to construct in return slot?
    return b;            // Compiler can't know at compile time -> NRVO may fail
}

int main() {
    std::cout << "=== URVO (guaranteed) ===\n";
    Heavy h1 = urvo();           // One Constructor, zero copies

    std::cout << "\n=== NRVO (optional, usually applied) ===\n";
    Heavy h2 = nrvo();           // One Constructor, maybe zero copies

    std::cout << "\n=== Conditional return (NRVO harder) ===\n";
    Heavy h3 = conditional_return(true);  // May see a MOVE here

    std::cout << "\nIDs: " << h1.id() << ", " << h2.id() << ", " << h3.id() << "\n";
    return 0;
}
// Typical output:
//   === URVO (guaranteed) ===
//     Construct #1                  <- ALWAYS just one construction
//   === NRVO (optional, usually applied) ===
//     Construct #2                  <- Usually just one (NRVO applied)
//   === Conditional return (NRVO harder) ===
//     Construct #10
//     Construct #20
//     MOVE #10                      <- May see move (NRVO not applied)
```

### Q3: Demonstrate a class with deleted copy/move that can still be returned by value in C++17

This was simply impossible before C++17. If a type had no copy or move constructor, you could not return it by value - full stop. C++17 changes that entirely for the prvalue case, because no copy or move is ever attempted. This opens the door for factory functions returning non-movable resources:

```cpp
#include <iostream>
#include <memory>

// A non-copyable, non-movable type
class Unique {
    int value_;
public:
    Unique(int v) : value_(v) {
        std::cout << "  Constructed Unique(" << v << ")\n";
    }

    // DELETED copy and move -- in C++14 this can't be returned by value
    Unique(const Unique&) = delete;
    Unique(Unique&&) = delete;
    Unique& operator=(const Unique&) = delete;
    Unique& operator=(Unique&&) = delete;

    int value() const { return value_; }
};

// C++17: Legal! Prvalue doesn't need copy or move
Unique make_unique_val(int v) {
    return Unique{v};    // Prvalue -> materializes directly at call site
}

// Chaining also works -- still no copy/move
Unique make_doubled(int v) {
    return make_unique_val(v * 2);
}

// Works as function parameter too!
void consume(Unique u) {
    std::cout << "  Consumed: " << u.value() << "\n";
}

// Real-world use case: factory returning non-movable mutex-guarded resource
class LockedResource {
    int resource_;
    // Imagine this has a mutex -- can't copy or move
public:
    LockedResource(int r) : resource_(r) {
        std::cout << "  LockedResource(" << r << ") created\n";
    }
    LockedResource(const LockedResource&) = delete;
    LockedResource(LockedResource&&) = delete;

    int get() const { return resource_; }
};

LockedResource create_locked(int id) {
    return LockedResource{id};  // C++17: OK! Prvalue materialization
}

int main() {
    std::cout << "=== Non-movable type returned by value (C++17) ===\n\n";

    std::cout << "1. Basic return:\n";
    Unique u = make_unique_val(42);
    std::cout << "  Value: " << u.value() << "\n";

    std::cout << "\n2. Chained return:\n";
    Unique u2 = make_doubled(21);
    std::cout << "  Value: " << u2.value() << "\n";

    std::cout << "\n3. Prvalue as function argument:\n";
    consume(Unique{99});

    std::cout << "\n4. Real-world: non-movable factory:\n";
    LockedResource lr = create_locked(7);
    std::cout << "  Resource: " << lr.get() << "\n";

    // What does NOT work:
    // Unique u3 = u;             // Error: copy deleted
    // Unique u4 = std::move(u);  // Error: move deleted
    // Unique named{5}; return named;  // Error in NRVO context: needs move

    return 0;
}
// Expected output:
//   1. Basic return:
//     Constructed Unique(42)
//     Value: 42
//   2. Chained return:
//     Constructed Unique(42)
//     Value: 42
//   3. Prvalue as function argument:
//     Constructed Unique(99)
//     Consumed: 99
//   4. Real-world: non-movable factory:
//     LockedResource(7) created
//     Resource: 7
```

The last comment block in the code is worth paying attention to: you still cannot return a *named* `Unique` because NRVO is not guaranteed and the fallback is a move - which is deleted. Guaranteed copy elision only helps when you return a prvalue directly.

---

## Notes

- C++17 prvalues are **not objects** - they are "initialization recipes" that construct directly into the target.
- Guaranteed copy elision makes it possible to return **non-movable types** by value from factory functions.
- NRVO is still optional but nearly every compiler applies it - write `noexcept` move ctors as fallback.
- Don't use `std::move` in `return` statements when NRVO would apply - it **prevents** NRVO.
- `auto x = T{args}` in C++17 creates exactly one object (no temporary + move).
