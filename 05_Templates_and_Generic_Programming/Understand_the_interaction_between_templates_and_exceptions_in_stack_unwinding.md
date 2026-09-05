# Understand the Interaction Between Templates and Exceptions in Stack Unwinding

**Category:** Templates and Generic Programming  
**Standard:** C++11 and later (not C++20-specific)  
**Reference:** <https://en.cppreference.com/w/cpp/language/destructor>  

---

## Topic Overview

### Templates and Stack Unwinding

When an exception is thrown, the runtime performs **stack unwinding**: it destroys all local objects between the `throw` and the matching `catch`, calling their destructors in reverse order of construction. This is how C++ guarantees cleanup even when errors occur.

For **template-based objects**, the destructor is itself a template instantiation. This creates several interactions that are worth understanding, especially if you use RAII wrappers or smart pointers:

| Concept | Implication |
| --- | --- |
| Destructor instantiation | `~MyClass<T>()` is instantiated on demand - it must be valid for the specific `T` |
| Stack unwinding order | Template objects are destroyed just like any other locals - reverse construction order |
| Incomplete types | `std::unique_ptr<T>` destructor calls `delete` on `T` - `T` must be complete |
| `noexcept` in destructors | Destructors are implicitly `noexcept`; throwing during unwinding calls `std::terminate` |
| Pimpl pattern | Forward-declared types + `unique_ptr` require destructor defined where `T` is complete |

### Why This Matters

Here's the sequence of events when an exception is thrown through a stack frame that contains template objects:

```cpp
    throw exception
         |
         v
    +------------------+
    | destroy local #3  |  ~Guard<FileHandle>()   <- template destructor
    | destroy local #2  |  ~unique_ptr<Impl>()    <- needs Impl complete!
    | destroy local #1  |  ~string()              <- standard destructor
    +------------------+
         |
         v
    catch block reached
```

If any destructor is ill-formed (e.g., operates on an incomplete type), the compiler rejects the code - and the error message often appears far from the real cause.

---

## Self-Assessment

### Q1: Show that a destructor instantiated from a template is called during stack unwinding

The key thing to observe here is the reverse-order destruction. Three `Guard` objects are constructed in order `g1`, `g2`, `g3` - and they are destroyed in the reverse order `g3`, `g2`, `g1` as the exception unwinds the stack. Each destructor is a separate template instantiation for its particular `T`:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

// A template whose destructor logs its invocation
template <typename T>
class Guard {
    std::string label_;
    T value_;
public:
    Guard(std::string label, T val)
        : label_(std::move(label)), value_(std::move(val)) {
        std::cout << "  Constructed Guard<" << label_ << "> with " << value_ << "\n";
    }

    ~Guard() {
        // This destructor is instantiated for each T used.
        // It IS called during stack unwinding, just like non-template destructors.
        std::cout << "  ~Guard<" << label_ << "> destroying " << value_ << "\n";
    }
};

void risky_operation() {
    Guard<int> g1("int", 42);
    Guard<std::string> g2("string", "hello");
    Guard<double> g3("double", 3.14);

    std::cout << "  About to throw...\n";
    throw std::runtime_error("something went wrong");
    // g3, g2, g1 destructors called in REVERSE order during unwinding
}

int main() {
    std::cout << "=== Stack unwinding with template destructors ===\n";
    try {
        risky_operation();
    } catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
    }

    return 0;
}
// Expected output:
//   Constructed Guard<int> with 42
//   Constructed Guard<string> with hello
//   Constructed Guard<double> with 3.14
//   About to throw...
//   ~Guard<double> destroying 3.14
//   ~Guard<string> destroying hello
//   ~Guard<int> destroying 42
//   Caught: something went wrong
```

The three destructors for `int`, `string`, and `double` are separate instantiations of `~Guard<T>()`, and all three are still triggered correctly during unwinding.

### Q2: Explain why `~unique_ptr<T>` requires `T` to be complete at the point of instantiation

This is the single most common pitfall in the Pimpl pattern. The reason is that `unique_ptr`'s destructor needs to call `delete` on a raw pointer - and `delete` requires the compiler to know the size of `T` and the address of `T`'s destructor. Both of those require a complete type definition.

`std::unique_ptr<T>`'s destructor calls `delete` on the raw pointer it holds:

```cpp
~unique_ptr() {
    // Simplified internal implementation:
    if (ptr_) {
        get_deleter()(ptr_);  // default_delete<T>::operator()(T* p) calls delete p;
    }
}
```

`delete p` requires the compiler to:

1. Know the **size** of `T` (to deallocate the right amount of memory)
2. Call `T`'s **destructor** (which must be visible)

Both require `T` to be a **complete type** - its definition must be visible.

The reason this trips people up is that the *destructor of `Widget` is what triggers the instantiation* - not the declaration of the `unique_ptr` member. If you don't declare `~Widget()`, the compiler generates it inline in the header where `Impl` is still an incomplete type:

```cpp
// widget.h
class Impl;   // Forward declaration — INCOMPLETE type

class Widget {
    std::unique_ptr<Impl> pimpl_;
public:
    Widget();
    // If destructor is not declared here, the compiler generates:
    //   ~Widget() = default;
    // ...which instantiates ~unique_ptr<Impl>() RIGHT HERE, where Impl is incomplete!
    // This is undefined behavior, often producing a warning or error.

    ~Widget();   // <-- declare destructor, define it in widget.cpp
};

// widget.cpp
#include "widget.h"
#include "impl.h"  // NOW Impl is complete

Widget::~Widget() = default;   // ~unique_ptr<Impl>() instantiated HERE — Impl is complete
```

| Approach | `T` Complete? | Result |
| --- | :---: | --- |
| Implicit destructor in header | No | Undefined behavior / compile error |
| Declared destructor, defined in `.cpp` | Yes | Correct |
| `std::shared_ptr<T>` | No | Works (type-erased deleter) |

`shared_ptr` differs because it captures the deleter at construction time using type erasure, so its destructor doesn't need `T` to be complete later.

### Q3: Show a compile error from a `unique_ptr<T>` to an incomplete T in a destructor

The comments in this example show what each compiler reports. The correct version demonstrates the fix - declare the destructor in the header, define it in the `.cpp` where `Engine` is complete, even if the definition is just `= default`:

```cpp
// === This code demonstrates the error — DO NOT use this pattern ===

// Step 1: Forward declare
class Engine;

// Step 2: Class with unique_ptr to incomplete type
class Car {
    std::unique_ptr<Engine> engine_;
public:
    Car();
    // No declared destructor!
    // Compiler generates ~Car() = default HERE
    // -> instantiates ~unique_ptr<Engine>()
    // -> calls delete on Engine*
    // -> Engine is incomplete -> ERROR or UNDEFINED BEHAVIOR
};

// Typical compiler error messages:
// GCC:   "invalid application of 'sizeof' to an incomplete type 'Engine'"
// Clang: "delete called on pointer to incomplete type 'Engine'"  (warning)
// MSVC:  "use of undefined type 'Engine'"

// === CORRECT version ===
#include <iostream>
#include <memory>
#include <string>
#include <utility>

// Forward declare
class Engine;

class Car {
    std::unique_ptr<Engine> engine_;
public:
    Car();
    ~Car();  // Declared here, defined in .cpp where Engine is complete
    void describe() const;
};

// In a real project, this would be in car.cpp:
class Engine {
    std::string type_;
    int horsepower_;
public:
    Engine(std::string type, int hp) : type_(std::move(type)), horsepower_(hp) {}
    ~Engine() { std::cout << "  ~Engine(" << type_ << ")\n"; }
    std::string info() const { return type_ + " " + std::to_string(horsepower_) + "hp"; }
};

// Define destructor WHERE Engine is complete
Car::Car() : engine_(std::make_unique<Engine>("V8", 450)) {
    std::cout << "  Car constructed\n";
}

Car::~Car() = default;  // ~unique_ptr<Engine> instantiated here — Engine is complete

void Car::describe() const {
    std::cout << "  Car with engine: " << engine_->info() << "\n";
}

int main() {
    std::cout << "=== Correct pimpl with unique_ptr ===\n";
    {
        Car car;
        car.describe();
        std::cout << "  Leaving scope...\n";
    }
    std::cout << "  Car destroyed successfully\n";

    return 0;
}
// Expected output:
//   Car constructed
//   Car with engine: V8 450hp
//   Leaving scope...
//   ~Engine(V8)
//   Car destroyed successfully
```

---

## Notes

- **Stack unwinding** destroys template objects the same as non-template objects - reverse construction order.
- Template destructors are **instantiated on demand**: if a destructor is never called, it's never instantiated.
- `unique_ptr<T>` destructor requires `T` to be complete - this is the #1 pitfall with the **Pimpl pattern**.
- Fix: declare the destructor in the header, define it (even as `= default`) in the `.cpp` where `T` is complete.
- `shared_ptr<T>` does NOT require `T` to be complete at destruction - it stores a type-erased deleter.
- **Never throw from a destructor** - if the destructor is called during stack unwinding and throws, `std::terminate` is called.
