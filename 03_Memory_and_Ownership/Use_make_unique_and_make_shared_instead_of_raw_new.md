# Use make_unique and make_shared Instead of Raw new

**Category:** Memory and Ownership  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique>  

---

## Topic Overview

### Why Avoid Raw `new`

If you find yourself writing `new` directly in application code, there are usually four things working against you. The factory functions fix all of them.

| Issue | Raw `new` | `make_unique`/`make_shared` |
| --- | --- | --- |
| **Exception safety** | Can leak in expressions | Always safe |
| **Allocations** | 2 for `shared_ptr(new T)` | 1 for `make_shared` |
| **Code clarity** | Type repeated twice | Type written once |
| **Custom deleters** | Required with raw `new` | Not supported (use ctor) |

### Core Pattern

Just write `auto` and let the factory function handle the type. You never have a naked pointer in flight.

```cpp
// BAD:
std::unique_ptr<Widget> p1(new Widget(1, 2));
std::shared_ptr<Widget> p2(new Widget(3, 4));

// GOOD:
auto p1 = std::make_unique<Widget>(1, 2);   // C++14
auto p2 = std::make_shared<Widget>(3, 4);   // C++11
```

### When You Can't Use `make_shared`/`make_unique`

1. Custom deleters -> `unique_ptr<T, Deleter>(ptr, deleter)`
2. Weak pointers keeping control block alive -> `shared_ptr(new T)` releases object memory sooner
3. Aggregate/brace initialization (before C++20) -> `unique_ptr<T>(new T{1, 2})`

---

## Self-Assessment

### Q1: Explain why `f(shared_ptr<T>(new T), g())` could leak memory while `f(make_shared<T>(), g())` cannot

This is the classic exception-safety argument for `make_shared`, and it is genuinely subtle. The reason the raw-`new` version can leak is that function arguments may be evaluated in any order (pre-C++17), so the sequence `new T` -> `g() throws` -> `shared_ptr ctor never runs` is legal. Once the raw pointer exists but the `shared_ptr` has not yet taken ownership, nobody owns it. C++17 tightened the rules somewhat, but `make_shared` remains the recommended approach because there is simply no window of unowned memory.

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>

struct Widget {
    int id;
    Widget(int i) : id(i) { std::cout << "Widget(" << id << ") created\n"; }
    ~Widget() { std::cout << "Widget(" << id << ") destroyed\n"; }
};

int g() {
    throw std::runtime_error("g() failed!");
    return 42;
}

void f(std::shared_ptr<Widget> w, int val) {
    std::cout << "f() called with widget " << w->id << " and val " << val << "\n";
}

int main() {
    // PROBLEM: With raw new, evaluation order can leak:
    //
    // f(shared_ptr<Widget>(new Widget(1)), g());
    //
    // Possible evaluation order (pre-C++17):
    //   1. new Widget(1)          — allocates on heap
    //   2. g()                    — THROWS!
    //   3. shared_ptr constructor — NEVER REACHED
    //   -> Widget(1) is leaked! No one owns it.
    //
    // C++17 fixed this partially (function args fully evaluated
    // before interleaving), but make_shared is STILL preferred.

    // SAFE: make_shared — allocation + construction is one step
    //
    // f(make_shared<Widget>(1), g());
    //
    // Either make_shared completes (Widget is owned) and then g() throws
    //   -> shared_ptr destructor cleans up
    // Or g() runs first and throws
    //   -> Widget was never created

    try {
        // This is ALWAYS safe:
        // f(std::make_shared<Widget>(1), g());
        // (Uncomment to see — Widget is either fully managed or never created)

        std::cout << "Demo: the leak scenario\n";
        std::cout << "  Step 1: new Widget(1)  -> allocated\n";
        std::cout << "  Step 2: g()            -> throws!\n";
        std::cout << "  Step 3: shared_ptr()   -> never runs\n";
        std::cout << "  Result: MEMORY LEAK\n\n";

        std::cout << "Fix: make_shared bundles allocation + ownership\n";
        std::cout << "  -> no intermediate naked pointer exists\n";
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    // Correct usage:
    auto w = std::make_shared<Widget>(2);
    std::cout << "Safe: widget " << w->id << " fully managed\n";

    return 0;
}
```

### Q2: Show that `make_shared` performs a single allocation while `shared_ptr<T>(new T)` does two

`make_shared` packs the object and its control block into a single memory block. Two allocations become one, and the object and its reference count end up side by side in memory. The example below makes this concrete by overriding global `operator new` to count calls.

```cpp
#include <iostream>
#include <memory>
#include <cstdlib>
#include <cstddef>

// Track allocations by overriding global new
static int alloc_count = 0;

void* operator new(size_t size) {
    ++alloc_count;
    std::cout << "  [alloc #" << alloc_count << "] " << size << " bytes\n";
    void* p = std::malloc(size);
    if (!p) throw std::bad_alloc();
    return p;
}

void operator delete(void* p) noexcept {
    std::free(p);
}
void operator delete(void* p, size_t) noexcept {
    std::free(p);
}

struct Data {
    int x, y, z;
    Data(int a, int b, int c) : x(a), y(b), z(c) {}
};

int main() {
    std::cout << "=== shared_ptr(new T) — TWO allocations ===\n";
    alloc_count = 0;
    {
        // Allocation 1: new Data -> heap block for Data object
        // Allocation 2: shared_ptr ctor -> heap block for control block (refcount)
        std::shared_ptr<Data> p(new Data(1, 2, 3));
    }
    std::cout << "Total allocations: " << alloc_count << "\n\n";

    std::cout << "=== make_shared<T>() — ONE allocation ===\n";
    alloc_count = 0;
    {
        // Single allocation: Data + control block in one memory block
        auto p = std::make_shared<Data>(4, 5, 6);
    }
    std::cout << "Total allocations: " << alloc_count << "\n\n";

    std::cout << "=== Why this matters ===\n";
    std::cout << "make_shared:\n";
    std::cout << "  + 1 allocation instead of 2 (50% fewer mallocs)\n";
    std::cout << "  + Better cache locality (object + refcount contiguous)\n";
    std::cout << "  + Less memory overhead (no separate block header)\n";
    std::cout << "  - Object memory not freed until all weak_ptrs gone\n";

    return 0;
}
// Expected output:
// === shared_ptr(new T) — TWO allocations ===
//   [alloc #1] 12 bytes     <- Data object
//   [alloc #2] ~24-48 bytes <- control block
// Total allocations: 2
//
// === make_shared<T>() — ONE allocation ===
//   [alloc #1] ~36-64 bytes <- Data + control block together
// Total allocations: 1
```

### Q3: List all the ways `make_unique` differs from `unique_ptr<T>(new T{...})` for array types

The most important difference for arrays is initialization: `make_unique<T[]>(n)` value-initializes all elements (zeros them out), while raw `new T[n]` default-initializes them, which for POD types means garbage. That is a common source of subtle bugs. The other differences are about expressiveness and exception safety.

```cpp
#include <iostream>
#include <memory>

int main() {
    std::cout << "=== make_unique vs raw new for arrays ===\n\n";

    // 1. make_unique<T[]> value-initializes (zeroes out)
    auto arr1 = std::make_unique<int[]>(5);  // All zeros!
    std::cout << "make_unique<int[]>(5): ";
    for (int i = 0; i < 5; ++i) std::cout << arr1[i] << " ";
    std::cout << " (value-initialized)\n";

    // raw new default-initializes (INDETERMINATE for POD!)
    std::unique_ptr<int[]> arr2(new int[5]);  // Garbage values
    std::cout << "unique_ptr(new int[5]): uninitialized (reading = UB)\n";

    // raw new with value-init
    std::unique_ptr<int[]> arr3(new int[5]());  // Zeroed, but verbose
    std::cout << "unique_ptr(new int[5]()): ";
    for (int i = 0; i < 5; ++i) std::cout << arr3[i] << " ";
    std::cout << " (explicitly value-initialized)\n\n";

    // 2. make_unique doesn't support initializer lists for arrays
    // auto bad = std::make_unique<int[]>({1,2,3});  // Error!
    std::unique_ptr<int[]> arr4(new int[]{1, 2, 3, 4, 5});  // Works
    std::cout << "new int[]{1,2,3,4,5}: ";
    for (int i = 0; i < 5; ++i) std::cout << arr4[i] << " ";
    std::cout << "\n";

    // C++20: make_unique_for_overwrite — default-initializes (no zeroing)
    // auto arr5 = std::make_unique_for_overwrite<int[]>(5);

    std::cout << "\n=== Summary of differences ===\n";
    std::cout << "1. make_unique<T[]>(n):      value-initializes (zeroed)\n";
    std::cout << "   unique_ptr(new T[n]):     default-initializes (garbage for POD)\n\n";
    std::cout << "2. make_unique<T[]>(n):      no initializer list support\n";
    std::cout << "   unique_ptr(new T[]{...}): supports brace init\n\n";
    std::cout << "3. make_unique<T[]>:         type written once\n";
    std::cout << "   unique_ptr(new T[]):      type written twice\n\n";
    std::cout << "4. make_unique:              exception safe in expressions\n";
    std::cout << "   raw new:                  can leak if another arg throws\n\n";
    std::cout << "5. make_unique_for_overwrite (C++20): default-init (skip zeroing)\n";
    std::cout << "   useful when you'll immediately overwrite the buffer\n";

    return 0;
}
```

---

## Notes

- `make_unique` was added in C++14 (one standard after `unique_ptr` itself).
- `make_shared` coalesces object + control block into one allocation - better performance and cache locality.
- Use raw `new` with smart pointers only when you need custom deleters or aggregate initialization.
- `make_shared` keeps object memory alive until the last `weak_ptr` dies (control block holds both).
- `make_unique_for_overwrite` / `make_shared_for_overwrite` (C++20) skip value-initialization for performance.
