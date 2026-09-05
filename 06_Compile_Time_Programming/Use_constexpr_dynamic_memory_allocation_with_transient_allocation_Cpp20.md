# Use constexpr dynamic memory allocation with transient allocation (C++20)

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constexpr>  

---

## Topic Overview

C++20 allows `new` and `delete` inside `constexpr` functions, but all allocations must be freed before the constant expression finishes evaluating. This is called "transient allocation."

The idea is simpler than it sounds: you can do dynamic allocation during a `constexpr` computation, but the allocation has to live and die entirely within that computation. Nothing can escape. Think of it as the compiler lending you a scratchpad - you can use heap-like storage while you're working, but you have to hand it back before you're done.

### Basic Example

Here's the most direct demonstration - `new[]` and `delete[]` inside a `constexpr` function, with the deallocation happening before the return:

```cpp
#include <algorithm>
#include <memory>

constexpr int compute_sum() {
    int* arr = new int[5]{1, 2, 3, 4, 5};
    int sum = 0;
    for (int i = 0; i < 5; ++i) sum += arr[i];
    delete[] arr;  // Must free before returning
    return sum;
}

constexpr int result = compute_sum();  // OK: 15
// All allocations are freed - "transient"
```

The compiler tracks every allocation made during constant evaluation. If you return without freeing, it's a hard error.

### constexpr std::vector and std::string

In practice, you rarely use raw `new`/`delete` in `constexpr` code. The real payoff is that `std::vector` and `std::string` - which manage their own memory internally - now work in `constexpr` contexts. Their destructors handle the cleanup, so the transient rule is satisfied automatically when the objects go out of scope:

```cpp
#include <vector>
#include <string>
#include <algorithm>
#include <numeric>

constexpr int sum_of_sorted() {
    std::vector<int> v = {5, 3, 1, 4, 2};
    std::sort(v.begin(), v.end());
    return std::accumulate(v.begin(), v.end(), 0);
    // v's internal buffer is freed when v is destroyed
    // -> transient allocation -> OK
}

constexpr int answer = sum_of_sorted();  // 15

constexpr size_t count_words() {
    std::string s = "hello world foo bar";
    size_t count = 1;
    for (char c : s)
        if (c == ' ') ++count;
    return count;
    // s freed at end -> transient
}

constexpr size_t words = count_words();  // 4
```

### Non-Transient Allocation (Compile Error)

The transient rule means you can't return a `std::vector` from a `constexpr` context and store it in a `constexpr` variable. The vector's internal buffer would have to persist into the binary, and the compiler has no mechanism for that:

```cpp
constexpr auto make_vector() {
    return std::vector{1, 2, 3};  // Vector escapes constexpr context
}

// constexpr auto v = make_vector();
// ERROR: allocation is NOT transient - it "leaks" into runtime
// The vector's heap memory can't become a compile-time constant

// This works IF the result is used only at compile time:
constexpr auto sz = make_vector().size();  // OK: vector created and destroyed
```

That last line is fine because the temporary `std::vector` is fully constructed and destroyed during the evaluation of `make_vector().size()`. The integer result `sz` persists - but it's a scalar, not a pointer to heap memory.

---

## Self-Assessment

### Q1: What makes an allocation "transient"

A transient allocation is one that is both allocated AND deallocated during the same constant expression evaluation. If the allocation "escapes" - the pointer is returned, stored in a `constexpr` variable, or otherwise outlives the evaluation - it's non-transient and causes a compile error.

The reason this matters is architectural: the binary has `.data`, `.rodata`, and `.bss` sections, but no heap section. A `constexpr` value has to fit into one of those sections. A pointer to "compile-time heap memory" would have to be translated into a real runtime address, which the linker has no way to do.

### Q2: Can you have constexpr std::map

Yes in C++20 (if your standard library implements it as `constexpr`). The requirement is that all allocations are transient. You can populate a map, query it, and get a scalar result - all at compile time. You cannot return the map itself as a `constexpr` value, because a map contains heap-allocated nodes.

### Q3: What is the practical use case

Compile-time computation of lookup tables, string parsing, configuration validation, hash computation, sorting, and any algorithm that needs dynamic memory internally but produces a fixed-size result. The pattern is: use `std::vector` or `std::string` internally for flexibility, then copy the result into a `std::array` or return a scalar before the function exits.

---

## Notes

- `constexpr std::vector`/`string` work in GCC 12+, Clang 15+, MSVC 19.29+.
- All standard library algorithms are `constexpr` since C++20.
- Non-transient allocations are proposed for C++26 (P2670 - static storage from constexpr).
- This enables "compile-time programming with runtime data structures."
