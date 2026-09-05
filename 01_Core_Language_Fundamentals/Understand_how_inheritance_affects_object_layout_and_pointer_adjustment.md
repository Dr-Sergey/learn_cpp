# Understand how inheritance affects object layout and pointer adjustment

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/derived_class>  

---

## Topic Overview

When a class inherits from one or more base classes, the base-class subobject(s) are embedded in the derived object's memory layout. **Pointer adjustment** is the compiler's process of adjusting a pointer value when casting between base and derived types, especially with multiple inheritance.

### Single Inheritance Layout

With single inheritance the base subobject sits at the very beginning of the derived object, so both pointers point to the same address:

```cpp
Derived object:
+----------------+  <- address of Derived AND Base (same)
|  Base part     |
|  (members)     |
+----------------+
|  Derived part  |
|  (members)     |
+----------------+
```

With single inheritance, `Base*` and `Derived*` have the **same address**.

### Multiple Inheritance Layout

With multiple inheritance each base subobject is placed at a different offset, and the compiler must adjust pointer values when you cast between them:

```cpp
struct A { int a; };          // 4 bytes
struct B { int b; };          // 4 bytes
struct C : A, B { int c; };   // A at offset 0, B at offset 4, c at offset 8
```

```cpp
C object:
+----------------+  offset 0  <- address of C and A
|  A::a          |
+----------------+  offset 4  <- address of B (adjusted!)
|  B::b          |
+----------------+  offset 8
|  C::c          |
+----------------+
```

Casting `C*` to `B*` requires **adding an offset** to the pointer.

### static_cast vs reinterpret_cast

`static_cast` knows about the hierarchy and performs the offset adjustment. `reinterpret_cast` does nothing but reinterpret the raw bits - when the offset is non-zero that gives you a pointer to the wrong memory:

```cpp
C obj;
B* bp = static_cast<B*>(&obj);       // compiler adjusts pointer by sizeof(A)
B* bad = reinterpret_cast<B*>(&obj); // NO adjustment -- wrong address! UB!
```

---

## Self-Assessment

### Q1: Show that a `Derived*` and `Base*` point to the same address for single inheritance

The numbers confirm what the layout diagram predicts - with single inheritance the base subobject is at offset zero, so both pointers are identical:

```cpp
#include <iostream>

struct Base { int x = 1; };
struct Derived : Base { int y = 2; };

int main() {
    Derived d;
    Base* bp = &d;       // implicit upcast
    Derived* dp = &d;

    std::cout << "Derived* = " << static_cast<void*>(dp) << "\n";
    std::cout << "Base*    = " << static_cast<void*>(bp) << "\n";
    std::cout << "Same?    " << (static_cast<void*>(dp) == static_cast<void*>(bp)
                                 ? "yes" : "no") << "\n";
    // Output: Same? yes

    // Verify: base subobject is at the beginning of derived
    std::cout << "offsetof(Derived, x) = " << offsetof(Derived, x) << "\n";  // 0
    std::cout << "offsetof(Derived, y) = " << offsetof(Derived, y) << "\n";  // 4
}
```

**How it works:**

- With single inheritance, the base-class subobject is placed at offset 0 in the derived object.
- Therefore `static_cast<Base*>(&d)` yields the same numeric address as `&d`.
- No pointer adjustment is needed.

### Q2: Show that for multiple inheritance, casting to a non-first base adjusts the pointer value

Here you can see the adjustment in action - `A*` matches `C*` exactly, but `B*` is shifted by `sizeof(A)` bytes:

```cpp
#include <iostream>
#include <cstdint>

struct A { int a = 1; };
struct B { int b = 2; };
struct C : A, B { int c = 3; };

int main() {
    C obj;

    A* ap = &obj;  // first base -- no adjustment
    B* bp = &obj;  // second base -- pointer adjusted!
    C* cp = &obj;

    std::cout << "C* = " << static_cast<void*>(cp) << "\n";
    std::cout << "A* = " << static_cast<void*>(ap) << "\n";
    std::cout << "B* = " << static_cast<void*>(bp) << "\n";

    auto addr_c = reinterpret_cast<uintptr_t>(cp);
    auto addr_a = reinterpret_cast<uintptr_t>(ap);
    auto addr_b = reinterpret_cast<uintptr_t>(bp);

    std::cout << "A offset from C: " << (addr_a - addr_c) << " bytes\n";  // 0
    std::cout << "B offset from C: " << (addr_b - addr_c) << " bytes\n";  // 4 (sizeof A)

    // bp != cp as raw addresses!
    std::cout << "B* == C* (void*)? "
              << (static_cast<void*>(bp) == static_cast<void*>(cp) ? "yes" : "no")
              << "\n";  // no!

    // But comparison through base pointer works correctly:
    std::cout << "bp->b = " << bp->b << "\n";  // 2 -- correct!
}
```

**How it works:**

- `A` is at offset 0 (first base), so `A*` = `C*`.
- `B` is at offset `sizeof(A)` (second base), so `B*` = `C*` + 4 bytes.
- The compiler inserts this adjustment automatically with `static_cast` or implicit conversions.

### Q3: Explain why `reinterpret_cast` between base and derived is UB while `static_cast` is not

**Answer:**

```cpp
#include <iostream>

struct A { int a = 10; };
struct B { int b = 20; };
struct C : A, B { int c = 30; };

int main() {
    C obj;

    // static_cast: compiler knows the layout and adjusts the pointer
    B* good = static_cast<B*>(&obj);
    std::cout << "static_cast B* -> b = " << good->b << "\n";  // 20 - correct

    // reinterpret_cast: just reinterprets the bit pattern -- NO adjustment
    B* bad = reinterpret_cast<B*>(&obj);
    std::cout << "reinterpret_cast B* -> b = " << bad->b << "\n";  // 10 - reads A::a!
    // This is UNDEFINED BEHAVIOR -- we're reading the wrong memory

    // Why the difference:
    // static_cast knows that B subobject is at offset sizeof(A) and adds it.
    // reinterpret_cast blindly treats &obj as B* -- it points to A's data.

    // For the FIRST base class, the addresses happen to match:
    A* a1 = static_cast<A*>(&obj);
    A* a2 = reinterpret_cast<A*>(&obj);
    // a1 == a2 -- coincidence, not a guarantee. Still UB to use reinterpret_cast.
}
```

**Key points:**

1. `static_cast` uses the class hierarchy information to compute the correct offset - it's always correct for valid casts.
2. `reinterpret_cast` performs no offset adjustment - it just reinterprets the pointer bits.
3. With single inheritance, the offset is often 0 (coincidentally correct), which hides the bug.
4. With multiple inheritance, `reinterpret_cast` to a non-first base produces a pointer to the wrong subobject - accessing it is UB.
5. **Rule:** Always use `static_cast` for base-to-derived or derived-to-base conversions.

---

## Notes

- Virtual inheritance adds a **vbase offset** stored in the vtable - pointer adjustment becomes a runtime lookup.
- `dynamic_cast` is the only cast that can safely downcast when the exact type is unknown at compile time.
- The Itanium ABI (used by GCC/Clang on Linux) specifies exact layout rules; MSVC uses a different ABI.
- Use `-fdump-class-hierarchy` (GCC) or `/d1reportAllClassLayout` (MSVC) to inspect actual object layouts.
