# Apply the type-erasure pattern using virtual tables and concepts

**Category:** Modern OOP Patterns  
**Standard:** C++20  
**Reference:** <https://www.youtube.com/watch?v=QGcVXgEVMJg>  

---

## Topic Overview

**Type erasure** allows storing objects of *any type* that satisfies a concept (behavioral contract) inside a single, non-template wrapper - without requiring those types to inherit from a common base class. It's the technique behind `std::function`, `std::any`, and Sean Parent's approach from *"Inheritance is the Base Class of Evil"*.

The reason this is tricky to understand at first is that C++ normally requires shared type identity to store objects polymorphically. Either they share a base class (virtual dispatch) or they share a template parameter (static dispatch). Type erasure is a third way: hide the concrete type inside the wrapper using an internal virtual interface that the user never sees.

### How It Works

There are two layers. The outer layer is what users interact with - a plain, non-template class. The inner layer is where the type information lives:

```cpp
External Interface (non-template):     Internal Implementation:
┌──────────────────────────┐           ┌──────────────────────────────┐
│ class Drawable {          │           │ struct Concept {             │
│   Drawable(Circle c);     │ wraps    │   virtual void draw_() = 0; │
│   Drawable(Square s);     │ any T ->│   virtual unique_ptr clone()│
│   void draw();            │           │ };                          │
│ };                        │           │                             │
│                           │           │ template <typename T>       │
│ vector<Drawable> shapes;  │           │ struct Model : Concept {    │
│ // ONE type in the vector │           │   T data_;                  │
│                           │           │   void draw_() { data_.draw(); } │
│                           │           │ };                          │
└──────────────────────────┘           └──────────────────────────────┘

Key: Concept is the internal virtual interface (hidden from users)
     Model<T> wraps any T that has .draw() - no inheritance needed on T
```

---

## Self-Assessment

### Q1: Implement a drawable object that holds any type satisfying a `Drawable` concept without virtual inheritance

The beauty of Sean Parent's approach is that `Circle`, `Square`, and `Text` don't know about `Drawable` at all. They don't inherit from anything. `Drawable` finds them through duck typing - it just needs a `.draw()` method to exist.

**Solution - Sean Parent-Style Type Erasure:**

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>

// External types - NO shared base class, NO virtual functions!
struct Circle {
    double radius;
    void draw() const {
        std::cout << "  Circle(r=" << radius << ")\n";
    }
};

struct Square {
    double side;
    void draw() const {
        std::cout << "  Square(s=" << side << ")\n";
    }
};

struct Text {
    std::string content;
    void draw() const {
        std::cout << "  Text(\"" << content << "\")\n";
    }
};

// Type-erasing wrapper
class Drawable {
    // Internal virtual interface (hidden from users)
    struct Concept {
        virtual ~Concept() = default;
        virtual void draw_() const = 0;
        virtual std::unique_ptr<Concept> clone_() const = 0;
    };

    // Model wraps any T that has .draw()
    template <typename T>
    struct Model final : Concept {
        T data_;
        Model(T d) : data_(std::move(d)) {}
        void draw_() const override { data_.draw(); }
        std::unique_ptr<Concept> clone_() const override {
            return std::make_unique<Model>(data_);
        }
    };

    std::unique_ptr<Concept> pimpl_;

public:
    // Constructor: accepts ANY type with .draw()
    template <typename T>
    Drawable(T x) : pimpl_(std::make_unique<Model<T>>(std::move(x))) {}

    // Copy (deep clone)
    Drawable(const Drawable& other) : pimpl_(other.pimpl_->clone_()) {}
    Drawable& operator=(const Drawable& other) {
        pimpl_ = other.pimpl_->clone_();
        return *this;
    }

    // Move
    Drawable(Drawable&&) noexcept = default;
    Drawable& operator=(Drawable&&) noexcept = default;

    // Public interface
    void draw() const { pimpl_->draw_(); }
};

int main() {
    // Store DIFFERENT types in ONE container - no shared base class!
    std::vector<Drawable> shapes;
    shapes.push_back(Circle{5.0});
    shapes.push_back(Square{3.0});
    shapes.push_back(Text{"Hello"});

    std::cout << "Drawing all shapes:\n";
    for (const auto& shape : shapes)
        shape.draw();

    // Copy works (deep clone)
    auto shapes_copy = shapes;
    std::cout << "\nDrawing copied shapes:\n";
    for (const auto& shape : shapes_copy)
        shape.draw();
}
// Expected output:
//   Drawing all shapes:
//     Circle(r=5)
//     Square(s=3)
//     Text("Hello")
//
//   Drawing copied shapes:
//     Circle(r=5)
//     Square(s=3)
//     Text("Hello")
```

The `clone_` virtual method is what makes copying work. Since `Model<T>` knows its concrete type `T`, it can construct a new copy. The outer `Drawable` just calls `clone_()` through the `Concept*` pointer without needing to know `T`.

---

### Q2: Compare Sean Parent's approach with `std::any`

`std::any` stores any type too, but it gives you no interface to call on the stored object without first casting it back to its concrete type. Type erasure gives you a *meaningful* interface.

**Comparison:**

| Aspect | Type Erasure (Concept/Model) | `std::any` |
| --- | --- | --- |
| **Type safety** | Compile-time: only types with `.draw()` accepted | Runtime: `any_cast` can fail at runtime |
| **Interface** | Custom: `draw()`, `area()`, etc. | Generic: just stores/retrieves |
| **Operations without cast** | Yes - `drawable.draw()` | No - must `any_cast<T>` first |
| **Overhead** | One virtual call per operation | `any_cast` + optional allocation |
| **Use case** | Polymorphism without inheritance | General-purpose type-safe `void*` |

```cpp
#include <any>
#include <iostream>
#include <vector>

// std::any approach - LESS type-safe:
void any_approach() {
    std::vector<std::any> things;
    things.push_back(Circle{5.0});
    things.push_back(Square{3.0});
    things.push_back(42);         // <- oops, int doesn't have .draw()!

    for (const auto& thing : things) {
        // Must try-cast to every possible type!
        if (auto* c = std::any_cast<Circle>(&thing)) {
            c->draw();
        } else if (auto* s = std::any_cast<Square>(&thing)) {
            s->draw();
        }
        // Forgot to check int -> silently skipped
    }
}

// Type erasure approach - TYPE-SAFE at construction:
void erasure_approach() {
    std::vector<Drawable> shapes;
    shapes.push_back(Circle{5.0});
    shapes.push_back(Square{3.0});
    // shapes.push_back(42);  // ERROR: int has no .draw()

    for (const auto& shape : shapes)
        shape.draw();  // just works - no casting needed
}
```

The `std::any` approach forces you to enumerate every type at the call site - it scales poorly as types multiply. The type-erasure approach handles any number of types with a single `shape.draw()` call.

---

### Q3: Show the trade-off between dynamic allocation and small buffer optimization in a type-erased container

Every `Drawable` above does a heap allocation via `make_unique`. For small objects, that heap trip is wasteful - you pay for a full `new`/`delete` when the object could fit inline. Small Buffer Optimization (SBO) avoids this for objects small enough to fit in a pre-allocated buffer.

**Solution - SBO (Small Buffer Optimization):**

```cpp
#include <iostream>
#include <memory>
#include <cstring>
#include <new>
#include <type_traits>

// Type-erased wrapper WITH small buffer optimization
class AnyDrawable {
    struct Concept {
        virtual ~Concept() = default;
        virtual void draw() const = 0;
        virtual void clone_to(void* buf) const = 0;
        virtual void move_to(void* buf) noexcept = 0;
    };

    template <typename T>
    struct Model final : Concept {
        T data_;
        Model(T d) : data_(std::move(d)) {}
        void draw() const override { data_.draw(); }
        void clone_to(void* buf) const override {
            new (buf) Model(data_);  // placement new
        }
        void move_to(void* buf) noexcept override {
            new (buf) Model(std::move(data_));
        }
    };

    // Small buffer: avoid heap for small types
    static constexpr size_t SBO_SIZE = 64;
    alignas(std::max_align_t) char buffer_[SBO_SIZE];
    bool uses_heap_ = false;

    Concept* concept_() noexcept {
        return uses_heap_
            ? *reinterpret_cast<Concept**>(buffer_)  // pointer stored in buffer
            : reinterpret_cast<Concept*>(buffer_);   // object lives in buffer
    }

    const Concept* concept_() const noexcept {
        return uses_heap_
            ? *reinterpret_cast<Concept* const*>(buffer_)
            : reinterpret_cast<const Concept*>(buffer_);
    }

    void destroy() {
        if (uses_heap_) {
            delete concept_();
        } else {
            concept_()->~Concept();
        }
    }

public:
    template <typename T>
    AnyDrawable(T x) {
        using M = Model<T>;
        if constexpr (sizeof(M) <= SBO_SIZE && std::is_nothrow_move_constructible_v<M>) {
            // SBO: no heap allocation
            new (buffer_) M(std::move(x));
            uses_heap_ = false;
        } else {
            // Heap allocation for large types
            auto* p = new M(std::move(x));
            std::memcpy(buffer_, &p, sizeof(p));
            uses_heap_ = true;
        }
    }

    ~AnyDrawable() { destroy(); }

    void draw() const { concept_()->draw(); }
};

// Small type - fits in SBO buffer
struct Dot {
    int x, y;
    void draw() const { std::cout << "  Dot(" << x << "," << y << ")\n"; }
};

// Large type - goes to heap
struct BigShape {
    char data[256];
    void draw() const { std::cout << "  BigShape (256 bytes)\n"; }
};

int main() {
    std::cout << "SBO buffer size: " << 64 << " bytes\n";
    std::cout << "sizeof(Model<Dot>): " << sizeof(Dot) + 8 << " bytes (approx)\n";
    std::cout << "sizeof(Model<BigShape>): " << sizeof(BigShape) + 8 << " bytes (approx)\n\n";

    AnyDrawable small_obj(Dot{10, 20});     // <- SBO: no heap allocation
    AnyDrawable big_obj(BigShape{});         // <- heap allocated

    small_obj.draw();
    big_obj.draw();
}
// Expected output:
//   SBO buffer size: 64 bytes
//   sizeof(Model<Dot>): ~16 bytes (approx)  <- fits in SBO
//   sizeof(Model<BigShape>): ~264 bytes (approx)  <- heap
//
//   Dot(10,20)
//   BigShape (256 bytes)
```

The `if constexpr` branch at construction time is the key: it asks whether the `Model<T>` fits and can be safely moved without throwing. If both are true, placement-new into `buffer_` avoids the heap entirely. Otherwise, `new` is called and the resulting pointer is stuffed into the buffer instead.

**SBO Trade-offs:**

| Aspect | Heap Only | With SBO |
| --- | --- | --- |
| Small object allocation | `new` each time | No allocation - in-place |
| Large object allocation | `new` each time | `new` (same as heap only) |
| Buffer overhead | 0 (just a pointer) | 64 bytes per wrapper (even if empty) |
| Cache performance | Poor (pointer chase) | Good (data inline) |
| Move performance | Pointer swap | May need to move data |
| Implementation complexity | Simple | Significantly more complex |

---

## Notes

- **`std::function` uses this pattern** - small lambdas (up to 16-32 bytes typically) are stored in an internal buffer; large ones are heap-allocated.
- **`std::any`** also uses SBO internally (implementations vary: libstdc++ 8 bytes, libc++ 24 bytes, MSVC 64 bytes).
- **`std::move_only_function` (C++23)** allows type-erased move-only callables.
- **Concepts (C++20) as documentation:** You can define a concept to document what operations the erased types must provide, even though the actual enforcement is in the `Model` template.
- **`dyno` library** (Louis Dionne) and **Proxy library** (Microsoft) provide library-based type erasure with less boilerplate.
- **Performance tip:** If you know all types at compile time, `std::variant` + `std::visit` is faster than type erasure (no virtual dispatch needed).
