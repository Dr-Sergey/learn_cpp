# Use final keyword strategically to enable devirtualization

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

The `final` specifier tells the compiler that a class or virtual function cannot be overridden further, enabling **devirtualization** - replacing indirect vtable calls with direct calls or inlining. The reason this matters is that every virtual call normally requires the CPU to look up a function pointer in the vtable at runtime, which prevents inlining and can cause branch mispredictions in tight loops. When the compiler can prove the exact concrete type, it can bypass that indirection entirely.

```cpp
Without final:   obj->draw()  ->  vtable lookup ->  indirect call (branch misprediction)
With final:      obj->draw()  ->  direct call ->  possibly inlined (zero overhead)
```

The reason this trips people up is that `final` is not just a design hint - it is an optimization contract. You are telling the compiler "there will never be a subclass that overrides this," and the compiler uses that guarantee to generate faster code.

| Usage | Syntax | Effect |
| --- | --- | --- |
| Final class | `class Leaf final : public Base {}` | No further derivation |
| Final method | `void draw() const final {}` | No further override |
| Both | Class final implies all methods final | Maximum optimization |

---

## Self-Assessment

### Q1: Show devirtualization with and without final

The compiler can only skip the vtable when it can prove at compile time what the actual type is. A `final` class guarantees no subclass can exist, so whenever the compiler sees a concrete `CircleFinal` object it knows for certain which `area()` will be called. Watch how the template function takes advantage of this:

**Answer:**

```cpp
#include <memory>
#include <vector>
#include <numeric>
#include <iostream>
#include <chrono>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
};

// Without final: compiler can't devirtualize
class CircleOpen : public Shape {
    double r_;
public:
    explicit CircleOpen(double r) : r_(r) {}
    double area() const override { return 3.14159265 * r_ * r_; }
};

// With final: compiler KNOWS no further overrides exist
class CircleFinal final : public Shape {
    double r_;
public:
    explicit CircleFinal(double r) : r_(r) {}
    double area() const override { return 3.14159265 * r_ * r_; }
};

// When the compiler can prove the dynamic type, it devirtualizes
template<typename T>
double sum_areas(const std::vector<T>& shapes, size_t n) {
    double total = 0;
    for (size_t i = 0; i < n; ++i)
        total += shapes[i].area();  // Devirtualized if T is final!
    return total;
}

int main() {
    // Devirtualization happens with concrete final type
    std::vector<CircleFinal> finals(10000, CircleFinal(1.0));
    auto result = sum_areas(finals, finals.size());  // Direct call, possibly inlined
    std::cout << "Sum: " << result << "\n";

    // Through pointer: compiler may still devirtualize with final
    auto p = std::make_unique<CircleFinal>(5.0);
    std::cout << p->area() << "\n";  // Compiler knows it's CircleFinal
    return 0;
}
```

The template instantiation with `CircleFinal` gives the compiler full type information at the call site. With `CircleOpen`, a subclass could always override `area()`, so the compiler has to play it safe and use the vtable. With `CircleFinal`, no such subclass is possible, so the compiler can call `area()` directly and even inline the multiplication into the loop body.

### Q2: Use final to seal a class hierarchy at the right level

You don't have to mark the whole hierarchy final - just the classes that are genuinely meant to be leaves. Here, `Logger` and `FileLogger` stay open for extension, but `RotatingFileLogger` is finished - nobody should subclass it. `BufferedLogger` shows yet another option: sealing a single method while leaving the class itself extensible. Notice the deliberate mix of open and sealed in the same hierarchy:

**Answer:**

```cpp
#include <iostream>

// Open for extension
class Logger {
public:
    virtual ~Logger() = default;
    virtual void log(const char* msg) = 0;
    virtual void flush() { /* default: no-op */ }
};

// Intermediate: can be extended further
class FileLogger : public Logger {
protected:
    void write_to_file(const char* msg) {
        std::cout << "[FILE] " << msg << "\n";
    }
public:
    void log(const char* msg) override {
        write_to_file(msg);
    }
};

// FINAL: this is the leaf - no further extension
class RotatingFileLogger final : public FileLogger {
    int max_lines_ = 1000;
    int current_line_ = 0;
public:
    void log(const char* msg) override {
        if (++current_line_ > max_lines_) {
            // rotate logic
            current_line_ = 0;
        }
        write_to_file(msg);
    }
    void flush() override { std::cout << "[FLUSH]\n"; }
};

// Final method without final class
class BufferedLogger : public Logger {
public:
    // Subclasses MUST NOT override log - but CAN override flush
    void log(const char* msg) final {
        buffer_ += msg;
        buffer_ += '\n';
        if (buffer_.size() > 4096) flush();
    }
    void flush() override {
        std::cout << buffer_;
        buffer_.clear();
    }
private:
    std::string buffer_;
};

int main() {
    RotatingFileLogger logger;
    logger.log("Hello");
    logger.flush();
    return 0;
}
```

The `final` on `BufferedLogger::log` is a design statement: the buffering strategy is locked in, but how you flush is still up to subclasses. This lets you share the buffering logic without anyone accidentally breaking it by overriding `log`. It is a precise scalpel rather than a sledgehammer - you seal exactly what needs sealing and nothing more.

### Q3: Demonstrate the performance impact with a benchmark

This benchmark compares a template function that calls through the concrete final type (devirtualizable) against a function that takes a `Base&` reference (must go through the vtable). Both compute the same result - the only difference is whether the compiler can see through the abstraction. Run this with optimization enabled to see the real effect:

**Answer:**

```cpp
#include <vector>
#include <memory>
#include <chrono>
#include <iostream>
#include <cmath>

class Base {
public:
    virtual ~Base() = default;
    virtual double compute(double x) const = 0;
};

class OpenImpl : public Base {
public:
    double compute(double x) const override {
        return std::sin(x) * std::cos(x);
    }
};

class FinalImpl final : public Base {
public:
    double compute(double x) const override {
        return std::sin(x) * std::cos(x);
    }
};

template<typename T>
double bench_concrete(const T& obj, int iterations) {
    double sum = 0;
    for (int i = 0; i < iterations; ++i)
        sum += obj.compute(static_cast<double>(i));  // Devirtualized!
    return sum;
}

double bench_virtual(const Base& obj, int iterations) {
    double sum = 0;
    for (int i = 0; i < iterations; ++i)
        sum += obj.compute(static_cast<double>(i));  // Virtual call
    return sum;
}

int main() {
    constexpr int N = 10'000'000;
    FinalImpl fi;
    OpenImpl oi;

    auto t1 = std::chrono::high_resolution_clock::now();
    volatile double r1 = bench_concrete(fi, N);  // Template: devirtualized
    auto t2 = std::chrono::high_resolution_clock::now();
    volatile double r2 = bench_virtual(oi, N);    // Virtual dispatch
    auto t3 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::milliseconds;
    std::cout << "Devirtualized: " << std::chrono::duration_cast<ms>(t2-t1).count() << "ms\n";
    std::cout << "Virtual:       " << std::chrono::duration_cast<ms>(t3-t2).count() << "ms\n";
    return 0;
}
```

With `-O2` or `-O3`, the devirtualized version can often inline the `compute` call entirely and then vectorize the loop. The virtual version has to call through the vtable on every iteration, which stalls the CPU's instruction pipeline. In tight numerical loops, this difference can be dramatic - sometimes several times faster, because you gain both inlining and the ability for the auto-vectorizer to process multiple iterations in a single CPU instruction.

---

## Notes

- `final` enables **devirtualization**: the compiler replaces vtable lookups with direct or inlined calls, which also opens the door to auto-vectorization.
- A good default is to mark leaf classes `final` - you can always remove it later if you genuinely need to extend the class, but it's harder to add after the fact when clients are depending on the open hierarchy.
- `final` on a method prevents further overriding in subclasses without preventing the class itself from being subclassed - useful when you want to lock down part of the behavior while leaving the rest extensible.
- GCC and Clang with `-O2` are quite aggressive about devirtualization when final types are visible in the same translation unit.
- LTO (Link-Time Optimization) can devirtualize even without `final` if it can prove at link time that no override exists - but `final` makes that proof local and reliable.
- The performance impact is most significant in tight loops: you eliminate branch mispredictions and enable inlining, both of which matter a lot when the function body is small.
