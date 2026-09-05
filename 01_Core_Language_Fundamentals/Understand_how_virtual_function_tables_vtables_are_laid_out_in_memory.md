# Understand how virtual function tables (vtables) are laid out in memory

**Category:** Core Language Fundamentals  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/virtual>  

---

## Topic Overview

A **vtable** (virtual function table) is the mechanism most compilers use to implement virtual function dispatch. Each class with virtual functions has a vtable - a static array of function pointers. Each object of that class contains a hidden **vptr** (vtable pointer) that points to its class's vtable.

### How It Works

Each object carries one pointer to its class's vtable. Overridden slots in the derived vtable replace the base's pointer; non-overridden slots inherit it:

```cpp
                     vtable for Base          vtable for Derived
                    +--------------+         +--------------+
                    | &Base::~Base |         | &Derived::~D |
                    | &Base::foo   |         | &Derived::foo|  <- overridden
                    | &Base::bar   |         | &Base::bar   |  <- inherited
                    +--------------+         +--------------+

Base object:                              Derived object:
+--------+                                +--------+
| vptr --+--> vtable for Base             | vptr --+--> vtable for Derived
| data   |                                | Base   |
+--------+                                | data   |
                                          | Derived|
                                          | data   |
                                          +--------+
```

### Virtual Call Mechanism

A virtual call compiles down to a pointer indirection through the vtable - roughly three machine operations:

```cpp
Base* p = new Derived();
p->foo();
// Compiler generates:
//   1. Load vptr from *p          (p->vptr)
//   2. Index into vtable           (vptr[1] for foo)
//   3. Call the function pointer    (calls Derived::foo)
```

This adds one pointer indirection compared to a non-virtual call.

### Key Facts

| Aspect | Detail |
| --- | --- |
| vtable storage | One per class, in the read-only data segment |
| vptr per object | One hidden member (typically at offset 0), size = `sizeof(void*)` |
| Override | Derived's vtable replaces the slot with Derived's function pointer |
| Non-overridden | Derived's vtable keeps Base's function pointer in that slot |
| Order of entries | Typically matches order of first virtual declaration in the class hierarchy |

### Virtual Destructor in the vtable

Under the Itanium ABI, the destructor actually occupies **two slots** - the compiler needs separate entries for "destroy object in place" and "destroy object and free memory":

```cpp
struct Base {
    virtual ~Base() = default;      // vtable slot 0 (Itanium ABI: two entries for dtor)
    virtual void foo() {}           // vtable slot 1 (or 2)
    virtual void bar() {}           // vtable slot 2 (or 3)
};
```

Under the Itanium ABI, the destructor occupies **two slots**: one for the "complete object destructor" and one for the "deleting destructor" (which also frees memory).

### ABI Impact: Adding Virtual Functions

Adding a virtual function to the **middle** of a class shifts the vtable indices of all subsequent virtual functions. Any code compiled against the old layout will call the wrong function.

---

## Self-Assessment

### Q1: Draw the vtable layout for a class with two virtual functions and one virtual destructor

**Answer:**

Start with the class definition and trace how the compiler fills in each vtable:

```cpp
struct Base {
    virtual ~Base() {}        // dtor
    virtual void draw() {}    // func 1
    virtual void update() {}  // func 2
};

struct Derived : Base {
    void draw() override {}   // overrides draw
    // update is inherited from Base
};
```

**vtable for Base (Itanium ABI):**

```cpp
Index | Entry                        | Notes
------+------------------------------+---------------------------
  0   | Base::~Base() [complete]     | complete object destructor
  1   | Base::~Base() [deleting]     | deleting destructor
  2   | Base::draw()                 | virtual function #1
  3   | Base::update()               | virtual function #2
```

**vtable for Derived:**

```cpp
Index | Entry                        | Notes
------+------------------------------+---------------------------
  0   | Derived::~Derived() [compl]  | overrides Base dtor
  1   | Derived::~Derived() [del]    | overrides Base dtor
  2   | Derived::draw()              | OVERRIDDEN -- points to Derived::draw
  3   | Base::update()               | INHERITED -- still points to Base::update
```

**Object layout (simplified):**

```cpp
sizeof(Base)    = sizeof(vptr) + 0 data = 8 bytes (on 64-bit)
sizeof(Derived) = sizeof(vptr) + 0 data = 8 bytes (on 64-bit)

Base obj:     [vptr -> Base_vtable]
Derived obj:  [vptr -> Derived_vtable]
```

### Q2: Show how adding a new virtual function to the middle of a base class breaks ABI

Watch what happens to `resize()` - the client compiled against v1 will keep calling slot 3, but in v2 slot 3 is now `animate()`:

```cpp
// === Version 1: Library ships with this ===
struct Widget_v1 {
    virtual ~Widget_v1() = default;     // slot 0-1
    virtual void draw() {}              // slot 2
    virtual void resize() {}            // slot 3
};

// Client code compiled against v1:
// widget->draw()   -> vptr[2]
// widget->resize() -> vptr[3]

// === Version 2: Library adds a function IN THE MIDDLE ===
struct Widget_v2 {
    virtual ~Widget_v2() = default;     // slot 0-1
    virtual void draw() {}              // slot 2
    virtual void animate() {}           // slot 3  <- NEW!
    virtual void resize() {}            // slot 4  <- SHIFTED from 3 to 4!
};

// Client code compiled against v1 still calls vptr[3] for resize()
// But in v2, vptr[3] is animate() -> WRONG FUNCTION CALLED!

// This is an ABI break. The client must be recompiled.
```

**How to avoid ABI breaks:**

1. **Always add new virtual functions at the END** of the class.
2. Use the **Pimpl idiom** to hide the vtable from the public header.
3. Use **`final`** on classes that shouldn't be inherited from - ABI-compatible additions are then possible in the implementation.
4. In library design, prefer **non-virtual interfaces (NVI)** where possible.

### Q3: Explain why calling a virtual function in a constructor does not dispatch to the overrider

**Answer:**

During construction, the vptr is set to the **current class's vtable**, not the most-derived class's vtable. This happens step by step, and you can see it in the output:

```cpp
#include <iostream>

struct Base {
    Base() {
        // At this point, vptr -> Base's vtable
        who();   // calls Base::who(), NOT Derived::who()
    }
    virtual void who() { std::cout << "Base\n"; }
    virtual ~Base() = default;
};

struct Derived : Base {
    int data;
    Derived() : Base(), data(42) {
        // At this point, vptr -> Derived's vtable
        who();   // calls Derived::who()
    }
    void who() override { std::cout << "Derived (data=" << data << ")\n"; }
};

int main() {
    Derived d;
    // Output:
    //   Base           <- called during Base constructor
    //   Derived (data=42)  <- called during Derived constructor
}
```

**Why:** When `Base::Base()` runs, `Derived`'s members haven't been initialized yet. If virtual dispatch went to `Derived::who()`, it could access `data` before it's initialized - undefined behavior. The standard prevents this by mandating that the vptr points to the currently-constructing class during each constructor phase.

**Construction order:**

1. Base constructor runs -> vptr = Base::vtable
2. Derived members initialized
3. Derived constructor body runs -> vptr = Derived::vtable

**Destruction follows the reverse:** During `~Derived()`, vptr = Derived::vtable. During `~Base()`, vptr = Base::vtable. So virtual calls in destructors also don't dispatch to the (already-destroyed) derived class.

---

## Notes

- The vtable is an **implementation detail** - C++ doesn't mandate it. But every major compiler (GCC, Clang, MSVC) uses vtables.
- `sizeof` increases by one pointer (`8 bytes` on 64-bit) per vtable pointer. Classes with no virtual functions have no vptr.
- `final` on a virtual call allows the compiler to **devirtualize** - calling the function directly without the vtable indirection.
- Use `-fdump-class-hierarchy` (GCC) or `clang -cc1 -fdump-record-layouts` to inspect vtable layouts.
