# Know the Difference Between `operator new` and `::operator new`

**Category:** Memory and Ownership  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/memory/new/operator_new>  

---

## Topic Overview

### The `new` Expression vs `operator new`

When you write `new T(args)`, the compiler does two things:

1. **Calls `operator new(sizeof(T))`** to allocate raw memory
2. **Calls `T`'s constructor** on that memory

`operator new` is the **allocation function** - it only allocates bytes, it doesn't construct objects. This distinction matters because you can replace or override the allocation step independently of the construction step.

### Class-Scope vs Global `operator new`

The reason this trips people up is that `operator new` is not one thing - it can be defined at class scope (affecting only that class) or at global scope (affecting everything). The lookup rules determine which one gets called.

| Feature | `T::operator new` (class scope) | `::operator new` (global scope) |
| --- | --- | --- |
| Scope | Only for class `T` (and derived unless overridden) | All types, fallback |
| Lookup | Found first via ADL | Used when no class version exists |
| Typical use | Pool allocators for specific types | Profiler tracking all allocations |
| Syntax to force global | `::new T(...)` bypasses class version | - |
| Can override | Yes, as static member | Yes, replaceable globally |

### All Forms of `operator new`

There are several overloads, each serving a different purpose. Placement `new` is non-replaceable - you cannot override it.

```cpp
void* operator new  (size_t);                    // single-object
void* operator new[](size_t);                    // array
void* operator new  (size_t, void* p);           // placement (non-replaceable)
void* operator new  (size_t, std::nothrow_t);    // non-throwing
void* operator new  (size_t, std::align_val_t);  // C++17 aligned
```

### Key Rules

1. **Class `operator new` is `static`** (even without the keyword) - it runs before the object exists
2. **Derived classes inherit** the base's `operator new` unless they define their own
3. **`::new`** explicitly calls global `operator new`, bypassing class-scope versions
4. **Always pair**: if you override `operator new`, also override `operator delete`

---

## Self-Assessment

### Q1: Explain that `operator new` can be overloaded per class while `::operator new` is global

This example shows all three cases side by side: a class with its own allocator, a call that forces the global allocator, and a type with no override at all.

```cpp
#include <iostream>
#include <cstdlib>
#include <cstddef>
#include <new>

class Widget {
public:
    int value;

    // Class-scope operator new - only used for Widget allocations
    static void* operator new(std::size_t size) {
        std::cout << "[Widget::operator new] allocating " << size << " bytes\n";
        void* p = std::malloc(size);
        if (!p) throw std::bad_alloc();
        return p;
    }

    static void operator delete(void* p) noexcept {
        std::cout << "[Widget::operator delete] freeing\n";
        std::free(p);
    }

    Widget(int v) : value(v) {
        std::cout << "[Widget ctor] value=" << v << "\n";
    }
    ~Widget() {
        std::cout << "[Widget dtor]\n";
    }
};

int main() {
    std::cout << "--- new Widget(42) uses Widget::operator new ---\n";
    Widget* w = new Widget(42);
    delete w;

    std::cout << "\n--- ::new Widget(99) forces global ::operator new ---\n";
    Widget* w2 = ::new Widget(99);
    ::delete w2;

    std::cout << "\n--- new int(7) uses global ::operator new (no class override) ---\n";
    int* pi = new int(7);
    std::cout << "int value: " << *pi << "\n";
    delete pi;

    return 0;
}
```

**Output:**

```text
--- new Widget(42) uses Widget::operator new ---
[Widget::operator new] allocating 4 bytes
[Widget ctor] value=42
[Widget dtor]
[Widget::operator delete] freeing

--- ::new Widget(99) forces global ::operator new ---
[Widget ctor] value=99
[Widget dtor]

--- new int(7) uses global ::operator new (no class override) ---
int value: 7
```

Notice that the `::new Widget(99)` case shows no allocation message - the global allocator has no logging. The class-specific one is completely bypassed when you use `::new`.

**Summary:**

- `new Widget(42)` -> looks up `Widget::operator new` first (found) -> uses it
- `::new Widget(99)` -> explicitly uses global `::operator new`, bypassing class version
- `new int(7)` -> no class-scope `operator new` for `int` -> uses global `::operator new`

### Q2: Show how to override global `::operator new` to track all allocations for a profiler

Replacing the global `operator new` is a blunt instrument - it captures every allocation in the program. That is exactly what a profiler needs.

```cpp
#include <iostream>
#include <cstdlib>
#include <cstddef>
#include <new>

// Global allocation counters
static size_t g_total_allocated = 0;
static size_t g_num_allocations = 0;
static size_t g_num_deallocations = 0;

// Replace global operator new
void* operator new(std::size_t size) {
    g_total_allocated += size;
    ++g_num_allocations;
    std::cout << "[global new] " << size << " bytes (total: "
              << g_total_allocated << " bytes, #" << g_num_allocations << ")\n";
    void* p = std::malloc(size);
    if (!p) throw std::bad_alloc();
    return p;
}

// Replace global operator delete
void operator delete(void* p) noexcept {
    ++g_num_deallocations;
    std::cout << "[global delete] #" << g_num_deallocations << "\n";
    std::free(p);
}

void operator delete(void* p, std::size_t size) noexcept {
    ++g_num_deallocations;
    std::cout << "[global delete] " << size << " bytes, #" << g_num_deallocations << "\n";
    std::free(p);
}

struct Point {
    double x, y, z;
    Point(double a, double b, double c) : x(a), y(b), z(c) {}
};

void print_stats() {
    std::cout << "\n=== Allocation Stats ===\n";
    std::cout << "  Total allocated:  " << g_total_allocated << " bytes\n";
    std::cout << "  Num allocations:  " << g_num_allocations << "\n";
    std::cout << "  Num deallocations:" << g_num_deallocations << "\n";
}

int main() {
    std::cout << "--- Allocating objects ---\n";
    int* a = new int(42);
    double* b = new double(3.14);
    Point* p = new Point(1, 2, 3);

    std::cout << "\n--- Freeing objects ---\n";
    delete a;
    delete b;
    delete p;

    print_stats();
    return 0;
}
```

**Output (typical):**

```text
--- Allocating objects ---
[global new] 4 bytes (total: 4 bytes, #1)
[global new] 8 bytes (total: 12 bytes, #2)
[global new] 24 bytes (total: 36 bytes, #3)

--- Freeing objects ---
[global delete] 4 bytes, #1
[global delete] 8 bytes, #2
[global delete] 24 bytes, #3

=== Allocation Stats ===
  Total allocated:  36 bytes
  Num allocations:  3
  Num deallocations:3
```

**Caution:** Replacing global `operator new` affects ALL allocations in the program, including standard library internals. Use with care in production.

### Q3: Demonstrate that overriding `operator new` in a derived class does NOT affect other classes

This shows the scope rule in action. `Derived` has its own `operator new`, `DerivedChild` inherits it, and everything else falls back to the global allocator.

```cpp
#include <iostream>
#include <cstdlib>
#include <cstddef>
#include <new>

class Base {
public:
    int x = 0;
    virtual ~Base() { std::cout << "  ~Base()\n"; }
};

class Derived : public Base {
public:
    int y = 0;

    // Only Derived (and its subclasses) use this operator new
    static void* operator new(std::size_t size) {
        std::cout << "[Derived::operator new] " << size << " bytes\n";
        void* p = std::malloc(size);
        if (!p) throw std::bad_alloc();
        return p;
    }

    static void operator delete(void* p) noexcept {
        std::cout << "[Derived::operator delete]\n";
        std::free(p);
    }

    ~Derived() override { std::cout << "  ~Derived()\n"; }
};

class DerivedChild : public Derived {
public:
    int z = 0;
    // Inherits Derived::operator new - will use it
    ~DerivedChild() override { std::cout << "  ~DerivedChild()\n"; }
};

class Unrelated {
public:
    double data = 0;
    ~Unrelated() { std::cout << "  ~Unrelated()\n"; }
};

int main() {
    std::cout << "--- new Base ---\n";
    Base* b = new Base;         // Uses global ::operator new
    delete b;

    std::cout << "\n--- new Derived ---\n";
    Derived* d = new Derived;   // Uses Derived::operator new
    delete d;

    std::cout << "\n--- new DerivedChild ---\n";
    DerivedChild* dc = new DerivedChild;  // Inherits Derived::operator new
    delete dc;

    std::cout << "\n--- new Unrelated ---\n";
    Unrelated* u = new Unrelated;  // Uses global ::operator new
    delete u;

    std::cout << "\n--- Summary ---\n";
    std::cout << "Base:         global ::operator new\n";
    std::cout << "Derived:      Derived::operator new  (class-scope)\n";
    std::cout << "DerivedChild: Derived::operator new  (inherited)\n";
    std::cout << "Unrelated:    global ::operator new\n";

    return 0;
}
```

**Output:**

```text
--- new Base ---
  ~Base()

--- new Derived ---
[Derived::operator new] 16 bytes
  ~Derived()
  ~Base()
[Derived::operator delete]

--- new DerivedChild ---
[Derived::operator new] 24 bytes
  ~DerivedChild()
  ~Derived()
  ~Base()
[Derived::operator delete]

--- new Unrelated ---
  ~Unrelated()

--- Summary ---
Base:         global ::operator new
Derived:      Derived::operator new  (class-scope)
DerivedChild: Derived::operator new  (inherited)
Unrelated:    global ::operator new
```

`Derived::operator new` only affects `Derived` and its subclasses. `Base` and `Unrelated` are completely unaffected - their allocations go through the global path as normal.

---

## Notes

- **`new T`** = allocate + construct. **`operator new`** = allocate only.
- If class defines `operator new`, `new T(...)` uses it. Use `::new T(...)` to bypass.
- Always override both `operator new` and `operator delete` as a pair.
- `operator new[]` is separate from `operator new` - override both if needed.
- In C++17, `operator new` with `std::align_val_t` is called for over-aligned types automatically.
