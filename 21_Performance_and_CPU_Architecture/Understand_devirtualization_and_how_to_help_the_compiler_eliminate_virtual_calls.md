# Understand devirtualization and how to help the compiler eliminate virtual calls

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/final>  

---

## Topic Overview

Every virtual call goes through the vtable: load the vtable pointer from the object, look up the function pointer at the right offset, then call through that pointer. That indirect call costs roughly 5 ns per invocation and also blocks inlining. In a tight loop over a polymorphic collection, this adds up fast.

Devirtualization is when the compiler replaces that indirect call with a direct call (or inlines the body entirely), dropping the overhead to zero. The table below shows the four main ways it can happen:

| Devirt technique | When it works | Compiler flag |
| --- | --- | --- |
| Local type visible | `Derived d; d.func()` | `-O2` (default) |
| `final` class | Class marked `final` | `-O2` (default) |
| `final` method | Method marked `final` | `-O2` (default) |
| LTO whole-program | Only one derived class exists | `-flto` |
| PGO speculative | Profile shows one dominant type | `-fprofile-use` |

Here is what the assembly difference looks like. The virtual call version has to chase two pointers before it can even issue the call:

```asm
Virtual call (indirect):             Devirtualized (direct):
  mov rax, [rdi]       ; load vptr     call Derived::func  ; direct call
  mov rax, [rax + 16]  ; load func ptr (or inlined entirely)
  call rax             ; indirect call
  ~5ns overhead         ~0ns overhead
```

---

## Self-Assessment

### Q1: Devirtualization when dynamic type is locally visible

When the compiler can see where an object was created in the same scope - for example, `Derived d;` on the stack - it knows the exact type and can prove the vtable lookup would always return `Derived::compute`. So it skips the lookup entirely.

```cpp
#include <iostream>

struct Base {
    virtual int compute(int x) const { return x; }
    virtual ~Base() = default;
};

struct Derived : Base {
    int compute(int x) const override { return x * 2; }
};

struct Derived2 : Base {
    int compute(int x) const override { return x * 3; }
};

int main() {
    // CASE 1: Local object - compiler KNOWS the type
    Derived d;
    Base& ref = d;
    std::cout << ref.compute(5) << '\n';  // 10
    // Compiler sees: ref is actually Derived
    // Assembly: call Derived::compute  (direct, possibly inlined!)
    // No vtable lookup needed.

    // CASE 2: Stack-allocated, type visible
    Derived2 d2;
    Base* ptr = &d2;
    std::cout << ptr->compute(5) << '\n';  // 15
    // Compiler sees ptr = &d2, type is Derived2
    // Assembly: direct call or inlined

    // CASE 3: Dynamic allocation - compiler CAN devirtualize
    auto p = std::make_unique<Derived>();
    std::cout << p->compute(5) << '\n';  // 10
    // Compiler may still devirtualize: sees new Derived() in same scope

    // CASE 4: Factory function from another TU - compiler CANNOT devirtualize
    // Base* unknown = create_something();  // type unknown!
    // unknown->compute(5);  // must use vtable lookup

    // Verify with Compiler Explorer (godbolt.org):
    //   g++ -O2: Cases 1-3 show direct call, Case 4 shows indirect
}
```

Case 4 is the hard one. If the object comes from another translation unit (a factory function, a dependency-injected pointer, etc.), the compiler has no idea what derived type it is at call time - so the vtable lookup is unavoidable unless you use LTO or PGO.

### Q2: `final` enables devirtualization

The `final` keyword tells the compiler that no further subclass can override a method (or that a class can have no subclasses at all). That single guarantee is enough for the compiler to devirtualize any call through a pointer or reference to that type.

```cpp
#include <iostream>
#include <memory>
#include <chrono>
#include <vector>

struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

// Without final: compiler cannot devirtualize through pointers
struct CircleNoFinal : Shape {
    double radius;
    CircleNoFinal(double r) : radius(r) {}
    double area() const override { return 3.14159265 * radius * radius; }
};

// With final: compiler KNOWS no further derived classes exist
struct CircleFinal final : Shape {
    double radius;
    CircleFinal(double r) : radius(r) {}
    double area() const override { return 3.14159265 * radius * radius; }
};

// Can also mark individual methods as final:
struct Square : Shape {
    double side;
    Square(double s) : side(s) {}
    double area() const final { return side * side; }  // no override of area() possible
};

template<typename T>
[[gnu::noinline]] double sum_areas(const std::vector<T*>& shapes) {
    double total = 0;
    for (auto* s : shapes) {
        total += s->area();  // devirtualized if T is final!
    }
    return total;
}

int main() {
    constexpr int N = 1'000'000;
    std::vector<CircleNoFinal*> no_final(N);
    std::vector<CircleFinal*> with_final(N);

    for (int i = 0; i < N; ++i) {
        no_final[i] = new CircleNoFinal(1.0);
        with_final[i] = new CircleFinal(1.0);
    }

    auto bench = [](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile double r = 0;
        for (int i = 0; i < 100; ++i) r = fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench([&]{ return sum_areas(no_final); },   "Without final");
    bench([&]{ return sum_areas(with_final); }, "With final   ");
    // With final: direct call, possibly inlined -> 2-3x faster in tight loop

    for (auto* p : no_final) delete p;
    for (auto* p : with_final) delete p;
}
```

The `[[gnu::noinline]]` attribute on `sum_areas` is there to prevent the whole function from being inlined into `main`, which would allow the compiler to devirtualize even the non-final version. The benchmark is measuring a realistic call-through-pointer scenario.

### Q3: Profile-Guided Optimization (PGO) speculative devirtualization

When the compiler cannot prove the exact type at compile time, PGO offers a third option: collect real data about which concrete type actually shows up at runtime, then generate a type-checked fast path for that dominant type. If 95% of calls go through `ConsoleLogger`, the compiler emits a type guard and a direct call for that case, falling back to the vtable only for the other 5%.

```cpp
#include <iostream>
#include <memory>
#include <vector>

struct Logger {
    virtual void log(const char* msg) = 0;
    virtual ~Logger() = default;
};

struct ConsoleLogger : Logger {
    void log(const char* msg) override {
        // In production: std::cout << msg << '\n';
    }
};

struct FileLogger : Logger {
    void log(const char* msg) override {
        // In production: write to file
    }
};

void process(Logger* logger, int n) {
    for (int i = 0; i < n; ++i) {
        logger->log("tick");  // virtual call
    }
}

// PGO speculative devirtualization works as follows:
//
// Step 1: Instrument build
//   g++ -O2 -fprofile-generate -o app app.cpp
//   ./app   # run with representative workload
//   # produces .gcda profile data files
//
// Step 2: Optimized build using profile
//   g++ -O2 -fprofile-use -o app app.cpp
//
// What the compiler does with profile data:
//   Profile shows: logger->log() calls ConsoleLogger::log 95% of the time.
//
//   Generated code becomes:
//     if (vtable_of(logger) == vtable_of(ConsoleLogger)) {
//         ConsoleLogger::log(logger, msg);  // DIRECT call (inlinable!)
//     } else {
//         logger->log(msg);  // fallback: indirect virtual call
//     }
//
// The speculative guard (type check) is very cheap (~1 cycle).
// For the 95% common case: direct call + possible inlining.
// For the 5% rare case: normal virtual call (no worse than before).
//
// Clang: -fprofile-instr-generate / -fprofile-instr-use
// MSVC:  /GL + /LTCG with profile-guided optimization

int main() {
    ConsoleLogger console;
    process(&console, 1000000);  // PGO profile captures this

    std::cout << "PGO speculative devirt:\n";
    std::cout << "  1. Profile: collect vtable usage statistics\n";
    std::cout << "  2. Compile: insert type guard for dominant type\n";
    std::cout << "  3. Runtime: dominant path is direct call\n";
}
```

The reason this trips people up is that PGO devirtualization is not guaranteed - it depends on the profile reflecting typical production traffic. If the real workload is different from the profiling run, the speculative guard pays its ~1 cycle cost on every call without the benefit. Make sure your profiling workload is representative.

---

## Notes

- `final` is the easiest devirtualization technique - zero runtime cost, just a type annotation.
- `-Rpass=devirt` (Clang) or `-fdump-ipa-devirt` (GCC) shows which calls were devirtualized.
- CRTP (Curiously Recurring Template Pattern) avoids virtual dispatch entirely at compile time, at the cost of templates.
- In tight loops, virtual call overhead can be 5-10x compared to a direct call; devirtualization plus inlining removes it completely.
- LTO (`-flto`) enables cross-TU devirtualization for the whole-program case.
