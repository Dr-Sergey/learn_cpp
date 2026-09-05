# Use static polymorphism via templates to avoid vtable overhead in hot paths

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/virtual>  

---

## Topic Overview

**Static polymorphism** uses templates to select behavior at compile time, eliminating virtual dispatch overhead. The compiler can inline the calls and optimize aggressively. This is most valuable in tight inner loops where each virtual call costs 3-5 cycles plus a potential instruction-cache miss.

The contrast with virtual dispatch comes down to when the decision is made:

```cpp
Virtual dispatch (runtime):         Static dispatch (compile-time):
  obj->render()                       renderAll<OpenGL>(renderer)
    |                                   |
    v                                   v
  vtable lookup -> jump                Direct call (inlined!)
    |                                   |
  ~3-5 cycles + cache miss            0 cycles overhead
```

---

## Self-Assessment

### Q1: Replace a virtual interface with template-based static polymorphism

This example times both approaches over a million iterations. The empty function bodies make the benchmark synthetic, but they demonstrate the structural difference - and in real code with small function bodies the template version often lets the compiler completely inline the call.

```cpp
#include <iostream>
#include <vector>
#include <chrono>

// === Approach 1: Virtual (runtime polymorphism) ===
class IRenderer {
public:
    virtual void drawPixel(int x, int y) = 0;
    virtual ~IRenderer() = default;
};

class OpenGLRenderer : public IRenderer {
public:
    void drawPixel(int x, int y) override {
        // In real code: glDrawPixel(x, y)
    }
};

void renderAll_virtual(IRenderer& renderer, int n) {
    for (int i = 0; i < n; ++i)
        renderer.drawPixel(i, i);  // virtual dispatch each iteration
}

// === Approach 2: Template (static polymorphism) ===
struct OpenGLRendererStatic {
    void drawPixel(int x, int y) {
        // Same implementation, no virtual
    }
};

struct VulkanRendererStatic {
    void drawPixel(int x, int y) {
        // Different backend
    }
};

template<typename Renderer>
void renderAll_static(Renderer& renderer, int n) {
    for (int i = 0; i < n; ++i)
        renderer.drawPixel(i, i);  // direct call, likely inlined!
}

int main() {
    constexpr int N = 1'000'000;

    // Virtual dispatch
    OpenGLRenderer vr;
    auto t1 = std::chrono::high_resolution_clock::now();
    renderAll_virtual(vr, N);
    auto t2 = std::chrono::high_resolution_clock::now();

    // Static dispatch
    OpenGLRendererStatic sr;
    auto t3 = std::chrono::high_resolution_clock::now();
    renderAll_static(sr, N);
    auto t4 = std::chrono::high_resolution_clock::now();

    auto virtual_ns = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();
    auto static_ns = std::chrono::duration_cast<std::chrono::microseconds>(t4 - t3).count();

    std::cout << "Virtual: " << virtual_ns << " us\n";
    std::cout << "Static:  " << static_ns << " us\n";
    std::cout << "(Static is typically 2-5x faster in hot loops)\n";
}
// Expected output (varies):
// Virtual: ~3000 us
// Static:  ~600 us
// (Static is typically 2-5x faster in hot loops)
```

The template version is faster primarily because the call can be inlined: the compiler sees the full body of `drawPixel` at the call site and can optimize across the loop boundary.

### Q2: CRTP as a static polymorphism pattern

The **Curiously Recurring Template Pattern (CRTP)** is the classic way to get polymorphic behavior through inheritance without a vtable. The trick is that the base class takes the derived class as a template parameter, which lets it call derived methods via a `static_cast` - resolved entirely at compile time.

```cpp
#include <iostream>

// CRTP: Curiously Recurring Template Pattern
template<typename Derived>
class Shape {
public:
    void draw() const {
        // Static dispatch: calls Derived::drawImpl()
        static_cast<const Derived*>(this)->drawImpl();
    }

    double area() const {
        return static_cast<const Derived*>(this)->areaImpl();
    }
};

class Circle : public Shape<Circle> {
    double radius_;
public:
    explicit Circle(double r) : radius_(r) {}
    void drawImpl() const { std::cout << "Drawing circle r=" << radius_ << '\n'; }
    double areaImpl() const { return 3.14159 * radius_ * radius_; }
};

class Square : public Shape<Square> {
    double side_;
public:
    explicit Square(double s) : side_(s) {}
    void drawImpl() const { std::cout << "Drawing square s=" << side_ << '\n'; }
    double areaImpl() const { return side_ * side_; }
};

// Works with ANY Shape<T> - no vtable needed
template<typename T>
void printInfo(const Shape<T>& shape) {
    shape.draw();
    std::cout << "Area: " << shape.area() << '\n';
}

int main() {
    Circle c(5.0);
    Square s(3.0);

    printInfo(c);
    printInfo(s);
}
// Expected output:
// Drawing circle r=5
// Area: 78.5398
// Drawing square s=3
// Area: 9
```

Notice that `printInfo` takes a `Shape<T>&` - it works for any concrete type that inherits from `Shape<T>`, but you get a separate instantiation for each `T`. The compiler sees the exact derived type and can inline everything. The tradeoff is that you cannot store a `Circle` and a `Square` in the same container through their base - for that, you need virtual dispatch (see Q3).

### Q3: When is runtime polymorphism still necessary

Static polymorphism is not a universal replacement. If you need to store objects of different concrete types together and operate on them uniformly at runtime, you need virtual dispatch. Templates cannot help you here because the type must be known at instantiation time.

```cpp
#include <iostream>
#include <memory>
#include <vector>

// Runtime polymorphism IS needed when:
// 1. Types are chosen at runtime (plugin systems, user config)
// 2. Heterogeneous collections are required
// 3. Separate compilation across shared library boundaries

// Example: heterogeneous collection requires virtual dispatch
class Animal {
public:
    virtual void speak() const = 0;
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void speak() const override { std::cout << "Woof!\n"; }
};

class Cat : public Animal {
public:
    void speak() const override { std::cout << "Meow!\n"; }
};

int main() {
    // CANNOT use templates here - types decided at runtime!
    std::vector<std::unique_ptr<Animal>> zoo;
    zoo.push_back(std::make_unique<Dog>());
    zoo.push_back(std::make_unique<Cat>());
    zoo.push_back(std::make_unique<Dog>());

    for (const auto& a : zoo)
        a->speak();  // must be virtual - type varies per element
}
// Expected output:
// Woof!
// Meow!
// Woof!
```

Use the right tool for the job. A quick decision guide:

**Decision Guide:**

| Scenario | Use |
| --- | --- |
| Hot loop, known types | Templates (static) |
| Plugin/extension system | Virtual (runtime) |
| Heterogeneous container | Virtual (runtime) |
| Compile-time policy selection | Templates (static) |
| Shared library API | Virtual (runtime) |
| Performance-critical math | Templates (static) |

---

## Notes

- `std::variant` + `std::visit` is a middle ground: type-safe, no vtable, but closed set of types.
- C++20 concepts provide compile-time interface checking for template-based polymorphism.
- CRTP is being partially replaced by "deducing this" (C++23): `void draw(this auto const& self)`.
- Profile first! Virtual dispatch is often negligible outside of tight inner loops.
