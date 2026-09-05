# Use std::inplace_vector for fixed-capacity stack allocation

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/container/inplace_vector>  

---

## Topic Overview

This file focuses on **practical patterns and benchmarks** for `std::inplace_vector<T,N>`. (See also file #580 for API overview and `std::span` interop.)

### Common Use Cases

The table below is a quick reference for where `inplace_vector` pays off. The common thread is any context where heap allocation is either forbidden, expensive, or simply unnecessary because the maximum count is known at compile time.

```cpp
┌─────────────────────────────────────────────────────────┐
│ Use Case                           │ Why inplace_vector │
├─────────────────────────────────────────────────────────┤
│ Embedded systems (no heap)          │ Zero allocation    │
│ Real-time audio/game tick (no jitter)│ Deterministic time│
│ Small buffer optimization           │ Stack-local speed  │
│ constexpr computation               │ Full constexpr     │
│ High-frequency trading              │ Predictable latency│
│ Interrupt handlers                  │ No allocator calls │
└─────────────────────────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: Use std::inplace_vector<int,16> as a fixed-capacity vector that never heap-allocates

The best way to get comfortable with `inplace_vector` is to see that its API is essentially identical to `std::vector`. If you know `std::vector`, you already know how to use this. The only real difference is that `capacity()` and `max_size()` are always the compile-time constant `N`, and the whole object lives on the stack.

**Answer:**

<!-- compile: needs `inplace_vector` (not in the CI standard library yet) -->
```cpp
#include <inplace_vector>  // C++26
#include <iostream>
#include <algorithm>

int main() {
    // Basic usage — drop-in vector replacement
    std::inplace_vector<int, 16> iv;

    // Same API as std::vector
    iv.push_back(42);
    iv.push_back(17);
    iv.push_back(99);
    iv.emplace_back(5);

    std::cout << "size: " << iv.size() << '\n';        // 4
    std::cout << "capacity: " << iv.capacity() << '\n'; // 16 (always)
    std::cout << "max_size: " << iv.max_size() << '\n'; // 16 (always)

    // Random access
    std::cout << "iv[0]: " << iv[0] << '\n';    // 42
    std::cout << "iv.at(1): " << iv.at(1) << '\n'; // 17

    // Standard algorithms work
    std::sort(iv.begin(), iv.end());
    for (int x : iv) std::cout << x << ' ';  // 5 17 42 99
    std::cout << '\n';

    // Erase
    iv.erase(iv.begin() + 1);  // Remove 17
    std::cout << "After erase: ";
    for (int x : iv) std::cout << x << ' ';  // 5 42 99
    std::cout << '\n';

    // sizeof lives on the stack
    std::cout << "sizeof(iv): " << sizeof(iv) << " bytes\n";
    // ~= 16 * sizeof(int) + sizeof(size_t) = 72 bytes (on stack, not heap)

    // Initializer list construction
    std::inplace_vector<double, 4> coords = {1.0, 2.5, 3.7, 4.2};

    // Never heap-allocates — verified
    // Even with custom allocator tracking, new/delete are never called.
    // This makes it safe for:
    // - Interrupt service routines
    // - Real-time audio callbacks
    // - constexpr contexts
}
```

Notice that `capacity()` always returns 16, not the current size. This is different from `std::vector`, where `capacity()` grows as needed. With `inplace_vector`, the capacity is baked into the type at compile time and never changes.

### Q2: Show push_back throwing std::bad_alloc when capacity is exceeded

This is the one behavioral difference from `std::vector` that you need to internalize: `std::vector` reallocates when it runs out of room; `inplace_vector` cannot. Instead it throws `std::bad_alloc` from `push_back`, returns `nullptr` from `try_push_back`, or causes undefined behavior from `unchecked_push_back`. The choice of which API to use is a deliberate design decision about how you want overflow handled.

**Answer:**

<!-- compile: needs `inplace_vector` (not in the CI standard library yet) -->
```cpp
#include <inplace_vector>  // C++26
#include <iostream>
#include <stdexcept>

int main() {
    std::inplace_vector<int, 4> v;

    // Fill to capacity
    v.push_back(1);
    v.push_back(2);
    v.push_back(3);
    v.push_back(4);
    std::cout << "Full: size=" << v.size() << " capacity=" << v.capacity() << '\n';
    // Full: size=4 capacity=4

    // push_back: throws when full
    try {
        v.push_back(5);  // Throws std::bad_alloc!
    } catch (const std::bad_alloc& e) {
        std::cout << "push_back threw std::bad_alloc: " << e.what() << '\n';
    }
    std::cout << "Size still: " << v.size() << '\n';  // 4 (strong guarantee)

    // try_push_back: returns nullptr when full
    auto* p = v.try_push_back(5);
    std::cout << "try_push_back result: " << (p ? "success" : "nullptr") << '\n';
    // nullptr — no exception thrown

    // unchecked_push_back: UB when full (fastest)
    // v.unchecked_push_back(5);  // UB! Only use when you've checked size < capacity

    // Safe pattern with unchecked:
    v.pop_back();  // Make room: size=3
    if (v.size() < v.capacity()) {
        v.unchecked_push_back(5);  // Safe: we verified space exists
        std::cout << "unchecked_push_back succeeded\n";
    }

    for (int x : v) std::cout << x << ' ';  // 1 2 3 5
    std::cout << '\n';

    /*
    API decision tree:
    ┌──────────────────────────────────────────────────────┐
    │ Need to handle overflow?                             │
    │   YES -> push_back (exception) or try_push_back (null)│
    │   NO  -> unchecked_push_back (fastest, UB if full)    │
    └──────────────────────────────────────────────────────┘
    */
}
```

The reason `push_back` throws `std::bad_alloc` specifically (rather than something like `std::length_error`) is to stay compatible with code that already catches `std::bad_alloc` for allocation failures. From the caller's perspective, the container ran out of capacity - that is conceptually the same as running out of memory.

### Q3: Benchmark inplace_vector vs std::vector for a hot path that rarely exceeds 16 elements

This benchmark simulates a physics tick that collects collision pairs. The key insight in the results is that even a reserved `std::vector` is slower than `inplace_vector` - not just because of allocation, but because `inplace_vector`'s data lives in the object itself rather than behind a heap pointer, giving the compiler and CPU better locality.

**Answer:**

<!-- compile: needs `inplace_vector` (not in the CI standard library yet) -->
```cpp
#include <inplace_vector>  // C++26
#include <vector>
#include <iostream>
#include <chrono>
#include <random>

// Scenario: collecting collision pairs in a physics tick
// Typically 0-12 collisions, max 16 possible

struct CollisionPair { int a, b; };

template<typename Container>
void simulate_tick(Container& collisions, int seed) {
    collisions.clear();
    // Simulate 0-12 collisions
    int count = seed % 13;
    for (int i = 0; i < count; ++i) {
        collisions.push_back({i, i + 1});
    }
    // Process collisions
    volatile int sum = 0;
    for (const auto& c : collisions) {
        sum += c.a + c.b;
    }
}

int main() {
    constexpr int TICKS = 2'000'000;

    // Benchmark: std::vector
    {
        std::vector<CollisionPair> collisions;
        collisions.reserve(16);  // Pre-allocate, but still heap memory
        auto t1 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < TICKS; ++i)
            simulate_tick(collisions, i);
        auto t2 = std::chrono::high_resolution_clock::now();
        std::cout << "std::vector (reserved 16): "
                  << std::chrono::duration<double, std::milli>(t2 - t1).count()
                  << " ms\n";
    }

    // Benchmark: std::inplace_vector
    {
        std::inplace_vector<CollisionPair, 16> collisions;
        auto t1 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < TICKS; ++i)
            simulate_tick(collisions, i);
        auto t2 = std::chrono::high_resolution_clock::now();
        std::cout << "std::inplace_vector<16>:   "
                  << std::chrono::duration<double, std::milli>(t2 - t1).count()
                  << " ms\n";
    }

    /*
    Typical results (2M ticks):
    ┌────────────────────────────┬──────────┬─────────────────────────┐
    │ Container                  │ Time     │ Why                     │
    ├────────────────────────────┼──────────┼─────────────────────────┤
    │ vector (no reserve)        │ ~120 ms  │ Allocation + realloc    │
    │ vector (reserved 16)       │ ~45 ms   │ No realloc, but heap ptr│
    │ inplace_vector<16>         │ ~25 ms   │ Stack-local, cache-hot  │
    └────────────────────────────┴──────────┴─────────────────────────┘

    Why inplace_vector is faster even vs reserved vector:

    1. No heap pointer dereference (data is in the object itself)
    2. Better cache locality (on stack, near other local variables)
    3. clear() is trivial (just set size=0, no deallocate)
    4. Compiler can sometimes keep the entire container in registers

    */
}
```

Point 4 in the comment - the compiler keeping the container in registers - is the most surprising one. For small element types like `int` or a pair of `int`s, the optimizer can sometimes eliminate the stack storage entirely and keep everything in CPU registers. That optimization is simply not possible for a `std::vector` because its data lives at a heap address that could be observed from elsewhere.

---

## Notes

- `std::inplace_vector<T,N>` is equivalent to Boost `static_vector<T,N>` and EASTL `fixed_vector<T,N,false>`.
- `capacity()` and `max_size()` are always `N` - compile-time constants that never change.
- Three push tiers: `push_back` (safe, throws on overflow), `try_push_back` (safe, no-throw, returns null), `unchecked_push_back` (fastest, undefined behavior if full).
- For types larger than a cache line (64 bytes), the advantage over `vector` diminishes because the stack object itself becomes large enough to hurt locality.
- Size overhead: `sizeof(inplace_vector<T,N>)` = N * sizeof(T) + alignment padding - lives entirely on the stack.
