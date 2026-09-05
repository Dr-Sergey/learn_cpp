# Understand the as-if rule and what the compiler is allowed to optimize

**Category:** Best Practices and Idioms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/as_if>  

---

## Topic Overview

The **as-if rule** states the compiler may perform any transformation that does not change the *observable behavior* of a conforming program. Observable behavior includes:

1. Reads/writes to `volatile` objects
2. I/O operations
3. Accesses to atomic objects

Everything else - memory layout, instruction order, intermediate values - can be transformed freely. The compiler doesn't have to execute your code the way you wrote it; it just has to produce a result that looks like it did.

### What the Compiler Can Do Under As-If

If the table feels like a lot, it boils down to this: anything that only affects values you compute internally is fair game, but anything that interacts with the outside world (files, devices, other threads via atomics) is protected.

| Optimization | What happens | Allowed? |
| --- | --- | --- |
| Dead code elimination | Remove unreachable code | Yes |
| Constant folding | `2+3` -> `5` at compile time | Yes |
| Loop vectorization | SIMD instead of scalar | Yes |
| Function inlining | Replace call with body | Yes |
| Copy elision (RVO/NRVO) | Skip copy/move construction | Yes (special) |
| Reorder non-volatile stores | Change write order | Yes |
| Reorder volatile stores | Change volatile write order | No |
| Remove I/O | Skip `printf` | No |

---

## Self-Assessment

### Q1: Show a loop that the compiler vectorizes under the as-if rule

The final result - the values in `out` - is all that matters. How the CPU gets there is entirely up to the optimizer.

```cpp
#include <iostream>
#include <vector>
#include <numeric>

// The compiler can transform this scalar loop into SIMD instructions
// because the observable behavior (final sum) is identical.
void add_arrays(const float* a, const float* b, float* out, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        out[i] = a[i] + b[i];
    }
    // With -O2/-O3, the compiler generates:
    //   vmovups ymm0, [a + i*4]     ; load 8 floats from a
    //   vaddps  ymm0, ymm0, [b + i*4]  ; add 8 floats from b
    //   vmovups [out + i*4], ymm0    ; store 8 results
    // instead of one-at-a-time scalar additions.
}

// Another example: loop elimination
int sum_1_to_n(int n) {
    int sum = 0;
    for (int i = 1; i <= n; ++i)
        sum += i;
    return sum;
}
// Compiler with -O2 replaces the entire loop with:
//   return n * (n + 1) / 2;
// The as-if rule allows this because the result is identical.

int main() {
    constexpr size_t N = 8;
    float a[N] = {1,2,3,4,5,6,7,8};
    float b[N] = {8,7,6,5,4,3,2,1};
    float out[N];

    add_arrays(a, b, out, N);
    for (size_t i = 0; i < N; ++i)
        std::cout << out[i] << ' ';
    std::cout << '\n';

    std::cout << "sum(100) = " << sum_1_to_n(100) << '\n';
}
// Expected output:
// 9 9 9 9 9 9 9 9
// sum(100) = 5050
```

The `sum_1_to_n` example is worth dwelling on: the compiler recognizes the summation pattern and replaces the entire loop body with the closed-form formula. Your loop never runs. That is the as-if rule at its most aggressive.

### Q2: Explain why `std::memcpy` through a non-aliased pointer is an optimization opportunity

Strict aliasing is the rule that two pointers of different types are assumed not to point to the same memory. The compiler uses this assumption to generate better code. Violating it through casts is undefined behavior - but `std::memcpy` is the sanctioned way to reinterpret bytes without breaking that rule.

```cpp
#include <cstring>
#include <iostream>

// Strict aliasing: compiler assumes pointers of different types
// don't point to the same memory. std::memcpy is the SAFE way
// to do type punning without violating aliasing rules.

// BAD: aliasing violation (UB)
float int_bits_to_float_bad(int i) {
    return *(float*)&i;  // UB! violates strict aliasing
}

// GOOD: memcpy is always safe and compilers optimize it away
float int_bits_to_float_good(int i) {
    float f;
    std::memcpy(&f, &i, sizeof(f));  // no aliasing violation
    return f;
    // Compiler generates the same code as the UB version:
    // just a register move (movd xmm0, edi) -- zero overhead
}

// The compiler knows that after memcpy, the destination pointer
// doesn't alias the source. This enables further optimizations:
void process(float* __restrict out, const int* __restrict in, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        float f;
        std::memcpy(&f, &in[i], sizeof(f));
        out[i] = f * 2.0f;
    }
    // __restrict + memcpy tells the compiler:
    // 1. out and in don't overlap (restrict)
    // 2. The type pun is legal (memcpy)
    // Result: fully vectorized loop
}

int main() {
    int bits = 0x40490FDB;  // IEEE 754 representation of ~pi
    float pi = int_bits_to_float_good(bits);
    std::cout << "pi approx " << pi << '\n';
}
// Expected output:
// pi approx 3.14159
```

The assembly output for `int_bits_to_float_good` and `int_bits_to_float_bad` is typically identical - a single register move. The `memcpy` is optimized away entirely. You get correctness for free.

### Q3: Demonstrate that the compiler can reorder memory accesses (unless volatile/atomic)

This is a genuinely important concept for concurrent code. The as-if rule allows the compiler to reorder plain memory writes freely - and even when it doesn't, the CPU itself may reorder them at runtime. Only `volatile` and `std::atomic` impose ordering constraints.

```cpp
#include <atomic>
#include <iostream>

int main() {
    // Scenario: compiler reordering non-volatile stores
    int a = 0, b = 0;

    // These two writes have no dependency:
    a = 1;
    b = 2;
    // Compiler may reorder to: b = 2; a = 1;
    // As-if rule allows this because a single-threaded program
    // can't tell the difference.

    std::cout << a << ' ' << b << '\n';  // always prints "1 2"

    // volatile prevents reordering (per-access, not inter-thread sync)
    volatile int va = 0, vb = 0;
    va = 10;   // MUST happen before vb = 20 (volatile semantics)
    vb = 20;
    std::cout << va << ' ' << vb << '\n';

    // atomic prevents reordering AND provides inter-thread visibility
    std::atomic<int> aa{0}, ab{0};
    aa.store(100, std::memory_order_release);  // guaranteed order
    ab.store(200, std::memory_order_release);
    std::cout << aa.load() << ' ' << ab.load() << '\n';
}
// Expected output:
// 1 2
// 10 20
// 100 200

// Key insight: in a multi-threaded context, another thread could
// see b=2 before a=1 (for plain ints). Only atomics provide
// cross-thread ordering guarantees.
```

The reason `volatile` doesn't help with multi-threading is subtle: it prevents the *compiler* from reordering, but the CPU itself can still reorder writes in its store buffer before they become visible to other cores. Only `std::atomic` coordinates with the CPU's memory model.

**Reordering summary:**

| Access type | Compiler reorder? | CPU reorder? |
| --- | --- | --- |
| Plain `int` | Yes | Yes |
| `volatile int` | No | Yes (on some CPUs) |
| `std::atomic<int>` (seq_cst) | No | No |
| `std::atomic<int>` (relaxed) | No (per-access) | Yes |

---

## Notes

- Copy elision (RVO/NRVO) is technically an exception to the as-if rule - it changes observable behavior (fewer constructor/destructor calls) but is explicitly permitted.
- The as-if rule is why `volatile` doesn't help with multi-threading - it prevents compiler reordering but not CPU reordering.
- Use `std::bit_cast` (C++20) instead of `memcpy` for type punning - same performance, cleaner syntax.
- Compiler Explorer (godbolt.org) is the best tool to see what the as-if rule lets the compiler do.
