# Understand the cost model of virtual dispatch in tight loops

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/final>  

---

## Topic Overview

Virtual dispatch has a real price. Each virtual call requires loading the vtable pointer out of the object, loading the function pointer from the vtable, and then executing an indirect branch - a branch whose target the CPU cannot know until runtime. In a tight loop that runs millions of times, that overhead stops being negligible and starts dominating your profile.

Here is what the CPU actually does at the instruction level:

```cpp
Virtual call sequence:           Direct call:
  mov rax, [rdi]       1 cycle    call Circle::area   0 overhead
  mov rax, [rax+16]    4 cycles   (possibly inlined!)
  call rax             ~5 cycles
  (branch mispredict)  15 cycles  <- if mixed types!
  Total: ~10-25 cycles             Total: 0-3 cycles
```

The ~25-cycle worst case only fires when the branch target predictor (BTB) misses - which happens whenever you process a mix of types in alternating order. The CPU predicts "next call will go to the same target as last time," and when it doesn't, you pay a 15-cycle penalty per iteration.

| Cost component | Cycles | Notes |
| --- | --- | --- |
| Load vptr from object | 4 | L1 cache hit |
| Load function ptr from vtable | 4 | L1 cache hit |
| Indirect branch | 1-15 | BTB hit: 1, miss: 15 |
| Inlining blocked | varies | Virtual calls can't be inlined |

---

## Self-Assessment

### Q1: Benchmark virtual vs direct call in a 10M iteration loop

To make the cost concrete, here is a benchmark that puts virtual dispatch, compiler-devirtualized dispatch, and a plain non-virtual call side by side. The key thing to watch is the `DerivedFinal` case - marking a class `final` tells the compiler no further derivation is possible, so it can devirtualize the call even when the pointer type is `Base*`.

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <chrono>

struct Base {
    virtual double compute(double x) const = 0;
    virtual ~Base() = default;
};

struct Derived : Base {
    double compute(double x) const override { return x * x + 1.0; }
};

struct DerivedFinal final : Base {
    double compute(double x) const override { return x * x + 1.0; }
};

// Non-virtual equivalent:
struct Direct {
    double compute(double x) const { return x * x + 1.0; }
};

int main() {
    constexpr int N = 10'000'000;
    Derived d;
    DerivedFinal df;
    Direct direct;

    auto bench = [&](auto fn, const char* label) {
        volatile double sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i)
            sum += fn(static_cast<double>(i));
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    Base* bp = &d;
    Base* bpf = &df;

    bench([&](double x){ return bp->compute(x); },     "Virtual (Base*)     ");
    bench([&](double x){ return bpf->compute(x); },    "Virtual (final*)    ");
    bench([&](double x){ return d.compute(x); },       "Devirt (known type) ");
    bench([&](double x){ return direct.compute(x); },  "Direct (non-virtual)");

    // Typical results (g++ -O2):
    // Virtual (Base*):      ~35 ms  (vtable lookup every iteration)
    // Virtual (final*):     ~12 ms  (devirtualized by compiler!)
    // Devirt (known type):  ~12 ms  (compiler sees Derived type)
    // Direct (non-virtual): ~10 ms  (inlined!)
}
```

Notice that `DerivedFinal` through a `Base*` pointer achieves nearly the same time as calling the concrete type directly. That is devirtualization: the compiler looked at the declaration, saw `final`, and replaced the indirect call with a direct one. No runtime change, just a compile-time annotation.

### Q2: Contiguous storage enables devirtualization via type homogeneity

The pointer-array pattern (`vector<unique_ptr<Shape>>`) is the classic way to write polymorphic code in C++, but it has two performance strikes against it. First, every `area()` call is a virtual dispatch. Second, following a `unique_ptr` to get to the object is a pointer dereference - you're jumping to a random heap address for each element.

This example compares three approaches with the same data and measures the gap:

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <chrono>
#include <variant>

struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

struct Circle : Shape {
    double r;
    Circle(double r) : r(r) {}
    double area() const override { return 3.14159265 * r * r; }
};

struct Rect : Shape {
    double w, h;
    Rect(double w, double h) : w(w), h(h) {}
    double area() const override { return w * h; }
};

// Approach 1: Pointer array (virtual dispatch, cache-unfriendly)
double sum_virtual(const std::vector<std::unique_ptr<Shape>>& shapes) {
    double total = 0;
    for (auto& s : shapes)
        total += s->area();  // indirect call + pointer chase
    return total;
}

// Approach 2: std::variant (no virtual dispatch, contiguous memory)
using ShapeVar = std::variant<Circle, Rect>;

double sum_variant(const std::vector<ShapeVar>& shapes) {
    double total = 0;
    for (auto& s : shapes) {
        total += std::visit([](auto& shape) { return shape.area(); }, s);
        // No vtable! std::visit uses compile-time dispatch.
        // Objects stored contiguously in vector.
    }
    return total;
}

// Approach 3: Separate homogeneous arrays (best performance)
double sum_homogeneous(const std::vector<Circle>& circles,
                       const std::vector<Rect>& rects) {
    double total = 0;
    for (auto& c : circles) total += c.area();  // direct, inlined!
    for (auto& r : rects) total += r.area();    // direct, inlined!
    return total;
}

int main() {
    constexpr int N = 1'000'000;

    // Setup
    std::vector<std::unique_ptr<Shape>> ptrs;
    std::vector<ShapeVar> variants;
    std::vector<Circle> circles;
    std::vector<Rect> rects;

    for (int i = 0; i < N; ++i) {
        if (i % 2 == 0) {
            ptrs.push_back(std::make_unique<Circle>(1.0));
            variants.emplace_back(Circle(1.0));
            circles.emplace_back(1.0);
        } else {
            ptrs.push_back(std::make_unique<Rect>(1.0, 2.0));
            variants.emplace_back(Rect(1.0, 2.0));
            rects.emplace_back(1.0, 2.0);
        }
    }

    auto bench = [](auto fn, const char* label) {
        volatile double r = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < 100; ++i) r = fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench([&]{ return sum_virtual(ptrs); },             "Virtual (ptr array)  ");
    bench([&]{ return sum_variant(variants); },         "Variant (contiguous) ");
    bench([&]{ return sum_homogeneous(circles, rects); }, "Homogeneous (split)  ");
    // Virtual:     ~250ms (indirect calls + pointer chasing)
    // Variant:     ~100ms (no vtable, contiguous, but visit overhead)
    // Homogeneous: ~30ms  (direct+inlined, perfect cache locality)
}
```

The 8x gap between virtual and homogeneous is not unusual. Homogeneous arrays win on two fronts simultaneously: the calls are direct (possibly inlined), and the memory access is a straight linear scan that the hardware prefetcher handles perfectly.

### Q3: `final` keyword enables devirtualization

The `final` keyword is the cheapest optimization in this topic - it costs nothing at runtime and only requires a one-word annotation at the class declaration. When the compiler sees `Cat final`, it knows no subclass can override `speak()`, so a call through a `Cat*` or a call where the type is provably `Cat` can be turned into a direct branch.

```cpp
#include <iostream>
#include <chrono>

struct Animal {
    virtual int speak() const = 0;
    virtual ~Animal() = default;
};

// Without final: compiler cannot assume no further derivations
struct Dog : Animal {
    int speak() const override { return 1; }
};

// With final: compiler KNOWS no more derived classes
struct Cat final : Animal {
    int speak() const override { return 2; }
};

template<typename T>
[[gnu::noinline]] long long sum_calls(const T* obj, int n) {
    long long total = 0;
    for (int i = 0; i < n; ++i) {
        total += obj->speak();
    }
    return total;
}

int main() {
    constexpr int N = 100'000'000;
    Dog dog;
    Cat cat;

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile auto r = fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    // Through Base pointer:
    Animal* ap_dog = &dog;
    Animal* ap_cat = &cat;
    bench([&]{ return sum_calls(ap_dog, N); }, "Dog via Animal*  (virtual) ");
    bench([&]{ return sum_calls(ap_cat, N); }, "Cat via Animal*  (virtual) ");

    // Through concrete type (devirtualized):
    bench([&]{ return sum_calls(&dog, N); },   "Dog direct       (devirt)  ");
    bench([&]{ return sum_calls(&cat, N); },   "Cat final direct (devirt)  ");

    // Verify devirtualization:
    //   g++ -O2 -S test.cpp | grep 'call.*speak'
    //   Dog via Animal*: callq *(%rax)    <- indirect
    //   Cat final:       callq Cat::speak <- direct (or inlined!)
}
```

You can verify what the compiler actually produced by inspecting the assembly output. An indirect call looks like `callq *(%rax)` - it reads the target from a register. A devirtualized call looks like `callq Cat::speak` - the target is baked in at compile time, and the CPU's branch predictor handles it easily.

---

## Notes

- Virtual dispatch costs ~5-25 cycles depending on BTB accuracy.
- In tight loops with mixed types (Circle, Rect alternating), BTB misses cause 15-cycle penalties.
- `final` is the cheapest optimization: zero runtime cost, just a compile-time annotation.
- `std::variant` + `std::visit` eliminates vtable overhead with value semantics.
- For performance-critical code: prefer templates (CRTP) or separate homogeneous containers over polymorphism.
