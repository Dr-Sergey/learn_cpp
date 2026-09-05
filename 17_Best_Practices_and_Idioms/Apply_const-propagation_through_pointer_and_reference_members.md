# Apply const-propagation through pointer and reference members

**Category:** Best Practices and Idioms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/experimental/propagate_const>  

---

## Topic Overview

In C++ a `const` member function makes `this` a pointer-to-const, so **member pointers become `T* const`** (the pointer itself is const), not `const T*` (what it points to is const). This means a `const` method can still mutate the object through a stored pointer.

The reason this trips people up is that `const` on a method sounds like a blanket promise - "this object won't change." But C++'s const model is *shallow*: it freezes the directly stored bytes of the object, not anything those bytes point to. A pointer member stores an address, and making the address read-only says nothing about the memory at that address.

### The Shallow-Const Problem

Here's the problem made concrete. The `const` qualifier on `modify()` doesn't do what you might hope:

```cpp
struct Widget {
    int* ptr;                // raw pointer member
    void modify() const {    // const method
        *ptr = 42;           // COMPILES! ptr is int* const, not const int*
    }
};

┌───────────────────┐   ptr (int* const)   ┌─────────┐
│  const Widget      │ ────────────▶ │ int     │  <- mutable!
│  (this is const)   │  can't change ptr  │ *ptr=42 │  <- allowed!
└───────────────────┘  but CAN write *ptr └─────────┘
```

The pointer itself is frozen (you can't make `ptr` point somewhere else), but the integer it points to is completely writable. That's shallow const in a nutshell.

### Solutions

There are a few ways to get the deep-const behaviour you probably wanted:

| Approach | How | Pros | Cons |
| --- | --- | --- | --- |
| Manual `const`/non-`const` overloads | Provide both accessors | Full control | Boilerplate |
| `std::experimental::propagate_const<T*>` | Wraps pointer | Automatic deep-const | Experimental |
| Custom wrapper | Write your own propagate_const | Standard-only | More code |

---

## Self-Assessment

### Q1: Show that a `const T*` member in a const method allows calling const methods on `*ptr`

The key distinction here is between `CarBroken` (which stores a plain `Engine*`) and `CarFixed` (which manually re-casts to `const Engine*` inside const methods). Watch how the compiler reacts differently to each:

```cpp
#include <iostream>
#include <string>
#include <memory>

class Engine {
    int power_ = 100;
public:
    int power() const { return power_; }          // const method
    void boost() { power_ += 50; }                // non-const method
};

// Problem: shallow const
class CarBroken {
    Engine* engine_;  // raw pointer
public:
    CarBroken(Engine* e) : engine_(e) {}

    void inspect() const {
        // In a const method, engine_ becomes Engine* const
        // (the pointer is const, NOT what it points to)
        std::cout << "Power: " << engine_->power() << '\n';  // OK: const method
        engine_->boost();  // COMPILES! But violates logical const-ness!
    }
};

// Solution: use const Engine* for read-only access
class CarFixed {
    Engine* engine_;
public:
    CarFixed(Engine* e) : engine_(e) {}

    void inspect() const {
        const Engine* e = engine_;  // explicitly propagate const
        std::cout << "Power: " << e->power() << '\n';  // OK
        // e->boost();  // ERROR: 'boost' is not const
    }

    void tune() {  // non-const: can modify
        engine_->boost();  // OK
    }
};

int main() {
    Engine e;
    CarFixed car(&e);
    car.inspect();  // read-only access
    car.tune();     // permitted: non-const method
    car.inspect();
}
// Expected output:
// Power: 100
// Power: 150
```

The fix is straightforward: inside the `const` method, bind a `const Engine*` local variable from the raw pointer. That local variable carries the const-ness forward, so the compiler will refuse any non-const calls through it.

### Q2: Explain why const correctness on members is not automatically transitive through pointers

**The C++ const model is "shallow":**

When you have a `const` object, all its **directly stored** members become const. But for pointer/reference members, only **the pointer itself** becomes const, not what it points to:

```cpp
struct S {
    int value;       // in const S: const int
    int* ptr;        // in const S: int* const (NOT const int*)
    int& ref;        // in const S: int& (references can't be rebound anyway)
    std::unique_ptr<int> uptr;  // in const S: const unique_ptr<int>
                                // BUT: *uptr is still mutable!
};
```

**Why?** The C++ object model treats a pointer member as containing an **address** (an integer). Making an object `const` freezes the address (you can't reseat the pointer), but doesn't affect the separate object at that address.

The same problem affects `unique_ptr` and `shared_ptr` - they're smart wrappers around a pointer, so `const unique_ptr<T>` is analogous to `T* const`: you can't change which object it owns, but you can still mutate the owned object:

```cpp
struct Widget {
    std::unique_ptr<int> data = std::make_unique<int>(0);

    void mutate() const {
        *data = 42;  // COMPILES in const method!
        // data = std::make_unique<int>(1);  // ERROR: can't reseat
    }
};
```

This is a common source of surprise: people expect smart pointers to be "safer" but they inherit the same shallow-const behaviour as raw pointers when stored as members.

### Q3: Use `propagate_const` to make const propagate through a pointer member

The cleanest solution is a wrapper that forwards const-ness to the pointee. When the wrapper is accessed through a const context it hands back a `const T*`; through a non-const context it hands back a plain `T*`. Here's a minimal implementation to make that concrete:

```cpp
#include <iostream>
#include <memory>

// Simple propagate_const implementation (works in standard C++17+)
template<typename T>
class propagate_const {
    T ptr_;
public:
    propagate_const(T p) : ptr_(std::move(p)) {}

    // Non-const access
    auto& operator*() { return *ptr_; }
    auto* operator->() { return &*ptr_; }
    auto* get() { return &*ptr_; }

    // Const access: propagates const to pointee
    const auto& operator*() const { return *ptr_; }
    const auto* operator->() const { return &*ptr_; }
    const auto* get() const { return &*ptr_; }

    explicit operator bool() const { return static_cast<bool>(ptr_); }
};

class Engine {
    int power_ = 100;
public:
    int power() const { return power_; }
    void boost() { power_ += 50; }
};

// Deep-const through propagate_const
class Car {
    propagate_const<std::unique_ptr<Engine>> engine_;
public:
    Car() : engine_(std::make_unique<Engine>()) {}

    void inspect() const {
        // engine_ in const context -> const Engine*
        std::cout << "Power: " << engine_->power() << '\n';
        // engine_->boost();  // ERROR: boost() is non-const!
    }

    void tune() {
        // engine_ in non-const context -> Engine*
        engine_->boost();  // OK
    }
};

int main() {
    Car car;
    car.inspect();
    car.tune();
    car.inspect();

    const Car& cref = car;
    cref.inspect();  // OK: const access
    // cref.tune();  // ERROR: tune() is non-const
}
// Expected output:
// Power: 100
// Power: 150
// Power: 150
```

Notice that `Car` itself hasn't changed much - the key is that `propagate_const` wraps the `unique_ptr` and its `const` overloads of `operator->` and `operator*` return `const Engine*` when called in a const context. The `inspect()` function now genuinely cannot call `boost()` - the protection is automatic and no extra code is needed at the call site.

---

## Notes

- `std::experimental::propagate_const` is in `<experimental/propagate_const>` (not yet standardized).
- Writing your own is ~20 lines and avoids experimental dependency.
- `propagate_const<unique_ptr<T>>` gives deep-const semantics for owned pointers.
- For non-owning pointers, consider separate `const T*` / `T*` accessors.
- The Pimpl idiom benefits greatly from `propagate_const` since the impl pointer should propagate const.
