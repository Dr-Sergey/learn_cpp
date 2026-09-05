# Understand virtual dispatch and its cost; prefer non-virtual interfaces

**Category:** Modern OOP Patterns  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/virtual>  

---

## Topic Overview

**Virtual dispatch** is how C++ calls the correct overridden function through a base pointer. It uses a **vtable** (virtual function table) - every polymorphic object contains a hidden `vptr` that points to its class's vtable.

Understanding the mechanism matters because virtual dispatch is not free, and in tight loops where it gets called millions of times, the cost adds up. On the other hand, for most program code (UI events, file I/O, business logic), the ~2 nanoseconds per call are genuinely irrelevant - don't over-optimize.

### Virtual Dispatch Mechanism

Here's what actually happens in hardware when you call a virtual function versus a plain function:

```cpp
Stack/Heap:                          Vtable (static, per-class):
┌────────────────────┐               ┌──────────────────────────┐
│ Dog object:         │               │ Dog vtable:               │
│   vptr ─────────────┼──────────────>│   [0] Dog::speak()        │
│   name_ = "Rex"     │               │   [1] Dog::eat()          │
│   breed_ = "Lab"    │               │   [2] Animal::sleep()     │
└────────────────────┘               └──────────────────────────┘

Virtual call: animal->speak()

  1. Load vptr from object             (memory read - cache miss possible)
  2. Load function pointer from vtable  (memory read - cache miss possible)
  3. Call through function pointer      (indirect branch - misprediction possible)

Non-virtual call: animal.bark()

  1. Call known address directly        (one instruction, fully predicted)
```

### Cost Summary

| Aspect | Virtual Call | Direct Call |
| --- | --- | --- |
| Instructions | ~3 memory loads + indirect call | 1 direct call |
| Branch prediction | Indirect branch - harder to predict | Always predicted |
| Inlining | **Cannot inline** (address unknown at compile time) | Fully inlinable |
| Cache pressure | Two extra cache lines (vptr + vtable) | None |
| Overhead per call | ~2-10 ns (depends on cache) | ~0.5-1 ns |

The inability to inline is often the bigger cost. When the compiler can inline a function, it can constant-fold, eliminate dead branches, and combine the call with surrounding code. A virtual call blocks all of that.

---

## Self-Assessment

### Q1: Benchmark virtual dispatch vs non-virtual call in a tight loop and quantify the overhead

This benchmark isolates the virtual call overhead by doing minimal work in the function itself. Real-world overhead will vary depending on how hot the vtable is in cache - if the same object type is called repeatedly, the branch predictor learns the pattern and the cost drops significantly.

```cpp
#include <iostream>
#include <chrono>
#include <memory>
#include <vector>

class Base {
public:
    virtual ~Base() = default;
    virtual int compute(int x) const { return x + 1; }

    // Non-virtual version for comparison
    int compute_nv(int x) const { return x + 1; }
};

class Derived : public Base {
public:
    int compute(int x) const override { return x + 1; }
};

template <typename Func>
double benchmark(const char* label, Func func, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    volatile int result = 0;
    for (int i = 0; i < iterations; ++i)
        result = func(i);
    auto end = std::chrono::high_resolution_clock::now();
    double ns = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count();
    std::cout << label << ": " << ns / iterations << " ns/call\n";
    return ns / iterations;
}

int main() {
    constexpr int N = 100'000'000;
    auto obj = std::make_unique<Derived>();
    Base* base_ptr = obj.get();

    // Virtual call through base pointer
    double virt_ns = benchmark("Virtual call", [&](int i) {
        return base_ptr->compute(i);  // indirect call via vtable
    }, N);

    // Non-virtual call
    double nv_ns = benchmark("Non-virtual call", [&](int i) {
        return base_ptr->compute_nv(i);  // direct call, inlinable
    }, N);

    // Direct call (known type)
    Derived d;
    double direct_ns = benchmark("Direct (known type)", [&](int i) {
        return d.compute(i);  // compiler may devirtualize this!
    }, N);

    std::cout << "\nOverhead ratio: " << virt_ns / nv_ns << "x\n";
}
// Typical output (varies by CPU/compiler):
//   Virtual call: ~2.5 ns/call
//   Non-virtual call: ~0.8 ns/call
//   Direct (known type): ~0.8 ns/call  (devirtualized!)
//   Overhead ratio: ~3x
```

Notice that the "direct (known type)" case matches the non-virtual cost. The compiler was able to devirtualize that call because it could prove at compile time that `d` is definitely a `Derived`, so there's no need for the vtable at all.

---

### Q2: Implement the Non-Virtual Interface (NVI) pattern and explain why it separates interface from customization

The NVI pattern is one of the most useful structural patterns in C++ class design. The idea is simple: public methods are non-virtual (they define the stable interface and can add pre/post-conditions), while the virtual methods are private or protected (they're the customization hooks for derived classes).

The reason this matters is that it lets the base class control the protocol around each operation without requiring every derived class to remember to call the right setup or teardown code.

```cpp
#include <iostream>
#include <string>
#include <chrono>
#include <iomanip>

// NVI: Public methods are NON-VIRTUAL (the interface).
//      Private/protected methods are VIRTUAL (the customization points).
class Document {
    // Public NON-VIRTUAL interface - controls the protocol
public:
    virtual ~Document() = default;

    // Users call this - it's the stable public API
    void save(const std::string& path) {
        // Pre-conditions / logging / locking happen HERE
        std::cout << "[Document] Validating...\n";
        validate();

        // Delegate to customizable implementation
        std::cout << "[Document] Saving to " << path << "...\n";
        do_save(path);

        // Post-conditions / cleanup happen HERE
        std::cout << "[Document] Recording timestamp...\n";
        last_saved_ = std::chrono::system_clock::now();
    }

    // Non-virtual: stable interface, can add behavior without touching derived
    void print_info() const {
        std::cout << "  Type: " << do_type_name() << "\n";
    }

private:
    std::chrono::system_clock::time_point last_saved_;

    // Private VIRTUAL - derived classes customize ONLY these
    virtual void validate() const {}  // default: no validation
    virtual void do_save(const std::string& path) const = 0;
    virtual std::string do_type_name() const = 0;
};

class TextDocument : public Document {
    std::string content_;
    void validate() const override {
        if (content_.empty())
            std::cout << "  [TextDocument] Warning: empty content\n";
    }
    void do_save(const std::string& path) const override {
        std::cout << "  [TextDocument] Writing " << content_.size()
                  << " chars to " << path << "\n";
    }
    std::string do_type_name() const override { return "TextDocument"; }

public:
    explicit TextDocument(std::string c) : content_(std::move(c)) {}
};

class ImageDocument : public Document {
    int width_, height_;
    void do_save(const std::string& path) const override {
        std::cout << "  [ImageDocument] Encoding " << width_ << "x" << height_
                  << " to " << path << "\n";
    }
    std::string do_type_name() const override { return "ImageDocument"; }

public:
    ImageDocument(int w, int h) : width_(w), height_(h) {}
};

int main() {
    TextDocument txt("Hello, World!");
    ImageDocument img(1920, 1080);

    txt.print_info();
    txt.save("/tmp/document.txt");

    std::cout << "\n";
    img.print_info();
    img.save("/tmp/image.png");
}
// Expected output:
//   Type: TextDocument
//   [Document] Validating...
//   [Document] Saving to /tmp/document.txt...
//     [TextDocument] Writing 13 chars to /tmp/document.txt
//   [Document] Recording timestamp...
//
//   Type: ImageDocument
//   [Document] Validating...
//   [Document] Saving to /tmp/image.png...
//     [ImageDocument] Encoding 1920x1080 to /tmp/image.png
//   [Document] Recording timestamp...
```

**Why NVI separates interface from customization:**

| Aspect | Traditional Virtual | NVI Pattern |
| --- | --- | --- |
| Pre/post conditions | Each derived class must add them manually | Base class enforces them |
| Adding logging | Must modify every derived class | Add once in base |
| Interface stability | Users call virtual methods directly | Public API is non-virtual = stable |
| Customization | Override public virtual methods | Override private/protected virtuals |
| Open/Closed | Must change interface to add behavior | Add behavior in base without touching derived |

---

### Q3: Show how devirtualization optimization can eliminate virtual call overhead in simple cases

The compiler is allowed to eliminate a virtual call whenever it can prove which function will actually be called. The most common case is a local variable of a known concrete type. The `final` keyword is the most reliable way to give the compiler the information it needs.

```cpp
#include <iostream>
#include <memory>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
};

class Circle : public Shape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return 3.14159 * r_ * r_; }
};

// Case 1: Compiler KNOWS the concrete type -> devirtualizes
double case_devirtualized() {
    Circle c(5.0);
    return c.area();  // Compiler knows it's Circle -> direct call, inlined
    // Generated assembly: just computes 3.14159 * 25.0 = 78.54 (constant folded!)
}

// Case 2: Compiler CANNOT devirtualize -> virtual call
double case_virtual(Shape& s) {
    return s.area();  // Unknown type -> must use vtable
}

// Case 3: make_unique with local scope -> can devirtualize
double case_local_unique_ptr() {
    auto c = std::make_unique<Circle>(5.0);
    return c->area();  // Compiler may devirtualize (knows type from make_unique)
}

// Case 4: Global/external pointer -> CANNOT devirtualize
double case_external(std::unique_ptr<Shape>& s) {
    return s->area();  // Type unknown -> virtual call
}

int main() {
    std::cout << "Devirtualized: " << case_devirtualized() << "\n";

    Circle c(5.0);
    std::cout << "Virtual: " << case_virtual(c) << "\n";
    std::cout << "Local unique_ptr: " << case_local_unique_ptr() << "\n";

    auto sp = std::make_unique<Circle>(5.0);
    std::unique_ptr<Shape> base_sp = std::make_unique<Circle>(5.0);
    std::cout << "External: " << case_external(base_sp) << "\n";
}
// Expected output:
//   Devirtualized: 78.5398
//   Virtual: 78.5398
//   Local unique_ptr: 78.5398
//   External: 78.5398
```

**When compilers can devirtualize:**

| Scenario | Devirtualizable? | Why |
| --- | --- | --- |
| `Circle c; c.area()` | **Yes** | Type is known statically |
| `auto p = make_unique<Circle>(); p->area()` | **Usually yes** | Type flows from construction |
| `Shape& s = c; s.area()` | **Sometimes** | Depends on optimizer (LTO helps) |
| `Shape* s = get_shape(); s->area()` | **No** | Type unknown at compile time |
| `final` class or method | **Yes** | Compiler knows no further override exists |

Using `final` is the most explicit signal you can give the compiler:

```cpp
// Using 'final' to help devirtualization:
class Square final : public Shape {
    double s_;
public:
    explicit Square(double s) : s_(s) {}
    double area() const override { return s_ * s_; }
};

// Even through Shape*, if compiler proves it's Square (e.g., LTO),
// it can devirtualize because 'final' guarantees no further overrides.
```

---

## Notes

- **`final` keyword** is the most reliable way to enable devirtualization - mark classes and methods `final` when you don't intend further derivation.
- **LTO (Link-Time Optimization)** significantly improves devirtualization by seeing all translation units at once.
- **Herb Sutter's NVI guideline:** "Make non-leaf non-virtual functions be public. Make virtual functions be private."
- **Benchmark tip:** Use `benchmark::DoNotOptimize()` (Google Benchmark) or `volatile` to prevent the compiler from optimizing away the entire loop.
- **Hot path rule:** Virtual dispatch overhead matters only in tight loops (millions of calls). For occasional calls (UI events, file I/O), the ~2ns overhead is irrelevant.
- **Alternatives to virtual dispatch:** CRTP (compile-time polymorphism), `std::variant` + `std::visit`, type erasure with SBO - all avoid vtable overhead.
