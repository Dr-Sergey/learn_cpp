# Write Type-Erased Wrappers Using Templates

**Category:** Templates and Generic Programming  
**Standard:** C++11, C++17 (basic)  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/function>  

---

## Topic Overview

### What Is Type Erasure

**Type erasure** hides concrete types behind a uniform interface. The caller interacts with a non-template type, while the implementation uses templates internally to handle any concrete type.

The problem it solves is that you sometimes want to store objects of different types in the same container, or pass them around without the caller knowing what the concrete type is. Inheritance can do this, but it requires every type to derive from a base class - which you can't always control. Type erasure solves it without that constraint.

```cpp
// User sees:
Function<int(int, int)> f = [](int a, int b) { return a + b; };
f(3, 4);  // calls the lambda - type of lambda is erased

// Internally: a virtual base class stores the concrete callable
```

### The Type Erasure Pattern

The standard recipe has three parts. The **Concept** is an abstract base class with the interface you want to expose. The **Model** is a templated class that inherits from Concept and wraps a concrete object. The **Wrapper** is what the user actually holds - a non-template class that owns a `unique_ptr<Concept>` pointing at a `Model<T>`.

```cpp
┌─────────────────────────────────────────────────┐
│ Type-Erased Wrapper (e.g., Function<Sig>)       │
│                                                  │
│  ┌──────────────────────────────┐               │
│  │ Concept (abstract base)      │               │
│  │  virtual R call(Args...) = 0 │               │
│  │  virtual clone() = 0         │               │
│  └───────────┬──────────────────┘               │
│              │                                   │
│  ┌───────────┴──────────────────┐               │
│  │ Model<T> (concrete impl)     │               │
│  │  T obj_;                     │               │
│  │  R call(Args...) override    │               │
│  │    { return obj_(args...); } │               │
│  └──────────────────────────────┘               │
│                                                  │
│  unique_ptr<Concept> pimpl_;                     │
└─────────────────────────────────────────────────┘
```

### Standard Library Type-Erased Types

| Type | Erases | Provides |
| --- | --- | --- |
| `std::function<R(Args...)>` | Any callable with signature `R(Args...)` | `operator()` |
| `std::any` | Any copyable type | `std::any_cast<T>()` |
| `std::shared_ptr<void>` | Any type (via custom deleter) | Pointer access |
| `std::move_only_function` (C++23) | Non-copyable callables | `operator()` |

---

## Self-Assessment

### Q1: Implement a simplified `std::function` using type erasure with virtual dispatch internally

The partial specialization on `R(Args...)` is what lets you write `Function<int(int,int)>` and have the compiler split that into a return type `R = int` and argument types `Args... = int, int`. The `clone()` virtual function is needed to make copies work - since `pimpl_` is a `unique_ptr`, copying the wrapper means calling `clone()` to get a fresh heap object of the right type.

```cpp
#include <iostream>
#include <memory>
#include <utility>
#include <string>
#include <functional>

// === Simplified std::function using type erasure ===

// Primary template (undefined) - only the partial specialization below is used
template <typename Signature>
class Function;

// Partial specialization for function signature R(Args...)
template <typename R, typename... Args>
class Function<R(Args...)> {
    // === Concept: abstract interface for any callable ===
    struct Concept {
        virtual ~Concept() = default;
        virtual R invoke(Args... args) = 0;
        virtual std::unique_ptr<Concept> clone() const = 0;
    };

    // === Model: wraps a specific callable type ===
    template <typename F>
    struct Model : Concept {
        F func_;

        explicit Model(F f) : func_(std::move(f)) {}

        R invoke(Args... args) override {
            return func_(std::forward<Args>(args)...);
        }

        std::unique_ptr<Concept> clone() const override {
            return std::make_unique<Model>(func_);
        }
    };

    std::unique_ptr<Concept> pimpl_;

public:
    // Default: empty function
    Function() = default;

    // Construct from any callable
    template <typename F>
    Function(F f) : pimpl_(std::make_unique<Model<F>>(std::move(f))) {}

    // Copy
    Function(const Function& other)
        : pimpl_(other.pimpl_ ? other.pimpl_->clone() : nullptr) {}

    Function& operator=(const Function& other) {
        if (this != &other)
            pimpl_ = other.pimpl_ ? other.pimpl_->clone() : nullptr;
        return *this;
    }

    // Move
    Function(Function&&) noexcept = default;
    Function& operator=(Function&&) noexcept = default;

    // Invoke
    R operator()(Args... args) {
        if (!pimpl_) throw std::bad_function_call();
        return pimpl_->invoke(std::forward<Args>(args)...);
    }

    explicit operator bool() const { return pimpl_ != nullptr; }
};

// === Test with different callable types ===
int add(int a, int b) { return a + b; }

struct Multiplier {
    int factor;
    int operator()(int x) const { return x * factor; }
};

int main() {
    // Lambda
    Function<int(int, int)> f1 = [](int a, int b) { return a + b; };
    std::cout << "Lambda:   f1(3, 4) = " << f1(3, 4) << "\n";  // 7

    // Function pointer
    Function<int(int, int)> f2 = &add;
    std::cout << "FuncPtr:  f2(3, 4) = " << f2(3, 4) << "\n";  // 7

    // Functor
    Function<int(int)> f3 = Multiplier{5};
    std::cout << "Functor:  f3(6) = " << f3(6) << "\n";  // 30

    // Copy works
    auto f4 = f1;
    std::cout << "Copy:     f4(10, 20) = " << f4(10, 20) << "\n";  // 30

    // Stateful lambda
    int counter = 0;
    Function<void()> f5 = [&counter]() { ++counter; };
    f5(); f5(); f5();
    std::cout << "Stateful: counter = " << counter << "\n";  // 3

    // Boolean test
    Function<void()> empty;
    std::cout << "empty: " << (empty ? "has value" : "empty") << "\n";

    return 0;
}
```

Notice that `Function` itself is not a template in the way users see it - it takes a signature string like `int(int,int)`. That signature is split by the partial specialization, and the Model template handles the actual callable type internally.

**Expected output:**

```text
Lambda:   f1(3, 4) = 7
FuncPtr:  f2(3, 4) = 7
Functor:  f3(6) = 30
Copy:     f4(10, 20) = 30
Stateful: counter = 3
empty: empty
```

### Q2: Explain why `std::any` uses type erasure and what the Small Buffer Optimization achieves

`std::any` needs to hold any copyable type without knowing at compile time what that type will be. That is the defining use case for type erasure. The interesting implementation detail is the Small Buffer Optimization (SBO): instead of always allocating on the heap, the implementation stores small objects directly inside the `std::any` object itself. This eliminates heap allocation for common small types like `int` and `double`.

```cpp
#include <iostream>
#include <any>
#include <string>
#include <typeinfo>

int main() {
    std::cout << "=== std::any and Type Erasure ===\n\n";

    // std::any can hold ANY copyable type:
    std::any a = 42;
    std::cout << "int: " << std::any_cast<int>(a) << "\n";

    a = std::string("hello");
    std::cout << "string: " << std::any_cast<std::string>(a) << "\n";

    a = 3.14;
    std::cout << "double: " << std::any_cast<double>(a) << "\n";

    // Type check:
    std::cout << "type: " << a.type().name() << "\n";
    std::cout << "has_value: " << a.has_value() << "\n";

    // Wrong cast throws:
    try {
        std::any_cast<int>(a);  // a holds double, not int
    } catch (const std::bad_any_cast& e) {
        std::cout << "Bad cast: " << e.what() << "\n";
    }

    std::cout << "\n=== Why Type Erasure? ===\n";
    std::cout << "std::any must store ANY copyable type in a single,\n";
    std::cout << "non-template class. Internally it uses:\n\n";
    std::cout << "  struct Manager {        // Concept (type-erased ops)\n";
    std::cout << "    void (*destroy)(storage&);\n";
    std::cout << "    void (*copy)(const storage&, storage&);\n";
    std::cout << "    const type_info& (*type)();\n";
    std::cout << "  };\n\n";

    std::cout << "=== Small Buffer Optimization (SBO) ===\n\n";
    std::cout << "Problem: heap allocation for every std::any is slow.\n";
    std::cout << "Solution: SBO stores small objects inline (no heap).\n\n";

    std::cout << "  struct any {\n";
    std::cout << "    union Storage {\n";
    std::cout << "      void* heap_ptr;           // for large objects\n";
    std::cout << "      alignas(8) char buf[N];   // for small objects (SBO)\n";
    std::cout << "    };\n";
    std::cout << "    Manager* mgr;\n";
    std::cout << "    Storage data;\n";
    std::cout << "  };\n\n";

    std::cout << "  sizeof(any) = " << sizeof(std::any) << " bytes\n";
    std::cout << "  Typical SBO buffer: 16-32 bytes\n\n";

    std::cout << "  Small types (int, double, ptr): stored in-place -> no allocation\n";
    std::cout << "  Large types (string, vector):   stored on heap -> one allocation\n\n";

    std::cout << "Benefits of SBO:\n";
    std::cout << "  1. No heap allocation for small types (int, double, pointers)\n";
    std::cout << "  2. Better cache locality\n";
    std::cout << "  3. Reduced memory fragmentation\n";
    std::cout << "  4. Faster construction and destruction\n";

    return 0;
}
```

### Q3: Compare virtual dispatch type erasure with template-based type erasure (CRTP) in terms of call overhead

The fundamental trade-off is heterogeneity versus performance. Virtual dispatch lets you store objects of different types in one container - that requires a runtime vtable lookup per call. CRTP eliminates the indirection entirely, but every type is statically distinct, so you can't mix them in a single container.

```cpp
#include <iostream>
#include <memory>
#include <chrono>
#include <vector>

// === Approach 1: Virtual Dispatch Type Erasure ===
class Drawable_Virtual {
    struct Concept {
        virtual ~Concept() = default;
        virtual void draw() const = 0;
        virtual std::unique_ptr<Concept> clone() const = 0;
    };
    template <typename T>
    struct Model : Concept {
        T obj_;
        Model(T o) : obj_(std::move(o)) {}
        void draw() const override { obj_.draw(); }
        std::unique_ptr<Concept> clone() const override {
            return std::make_unique<Model>(*this);
        }
    };
    std::unique_ptr<Concept> pimpl_;
public:
    template <typename T>
    Drawable_Virtual(T obj) : pimpl_(std::make_unique<Model<T>>(std::move(obj))) {}
    Drawable_Virtual(const Drawable_Virtual& o) : pimpl_(o.pimpl_->clone()) {}
    void draw() const { pimpl_->draw(); }
};

// === Approach 2: CRTP (Static Polymorphism) - no type erasure ===
template <typename Derived>
class Drawable_CRTP {
public:
    void draw() const {
        static_cast<const Derived*>(this)->draw_impl();
    }
};

// === Concrete types ===
struct Circle {
    void draw() const { /* renders circle */ }
};

struct Square {
    void draw() const { /* renders square */ }
};

struct Circle_CRTP : Drawable_CRTP<Circle_CRTP> {
    void draw_impl() const { /* renders circle */ }
};

struct Square_CRTP : Drawable_CRTP<Square_CRTP> {
    void draw_impl() const { /* renders square */ }
};

int main() {
    std::cout << "=== Virtual Dispatch vs CRTP Comparison ===\n\n";

    std::cout << "Feature              Virtual Dispatch         CRTP (Static)\n";
    std::cout << "-------------------  -----------------------  -------------\n";
    std::cout << "Call overhead        1 vtable indirection     Zero (inlined)\n";
    std::cout << "Memory overhead      vtable ptr + heap alloc  No overhead\n";
    std::cout << "Heterogeneous coll.  Yes (value semantic)     No (diff types)\n";
    std::cout << "Runtime flexibility  Yes                      No\n";
    std::cout << "Compile time         Fast                     Slower (more templates)\n";
    std::cout << "Can inline?          Not usually              Yes, always\n";
    std::cout << "Cache friendly?      No (pointer chasing)     Yes (contiguous data)\n\n";

    // Virtual dispatch: can store different types in one container
    std::vector<Drawable_Virtual> shapes;
    shapes.push_back(Drawable_Virtual(Circle{}));
    shapes.push_back(Drawable_Virtual(Square{}));
    for (const auto& s : shapes) {
        s.draw();  // virtual call -> indirect (vtable lookup)
    }
    std::cout << "Virtual: drew " << shapes.size() << " shapes (heterogeneous)\n";

    // CRTP: each type is separate - no heterogeneous container
    std::vector<Circle_CRTP> circles(100);
    for (const auto& c : circles) {
        c.draw();  // static call -> inlined, no overhead
    }
    std::cout << "CRTP: drew " << circles.size() << " circles (homogeneous)\n";

    std::cout << "\n=== When to Use Each ===\n";
    std::cout << "Virtual dispatch type erasure:\n";
    std::cout << "  - Need heterogeneous containers (different types in one collection)\n";
    std::cout << "  - Types determined at runtime\n";
    std::cout << "  - Value semantics required (std::function, std::any)\n";
    std::cout << "\nCRTP (static polymorphism):\n";
    std::cout << "  - All types known at compile time\n";
    std::cout << "  - Performance-critical inner loops\n";
    std::cout << "  - Homogeneous containers only\n";
    std::cout << "  - Numeric/scientific computing\n";

    return 0;
}
```

---

## Notes

- Type erasure = templates (internal) + virtual dispatch (hidden from user) + value semantics (external).
- The three core components: **Concept** (abstract interface), **Model** (templated implementation), **Wrapper** (user-facing non-template).
- `std::function` has overhead from heap allocation + virtual dispatch - avoid in hot loops.
- **SBO** typically stores objects up to about 32 bytes inline (implementation-dependent).
- `std::move_only_function` (C++23) avoids the copy requirement, enabling unique_ptr captures.
- Sean Parent's "Inheritance Is The Base Class of Evil" talk popularized this pattern.
- For maximum performance, consider `std::function_ref` (C++26) - no ownership, no allocation.
