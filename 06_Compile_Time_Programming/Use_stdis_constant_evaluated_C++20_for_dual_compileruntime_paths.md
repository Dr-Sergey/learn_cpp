# Use `std::is_constant_evaluated` (C++20) for Dual Compile/Runtime Paths

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/types/is_constant_evaluated>  

---

## Topic Overview

### What Is `std::is_constant_evaluated()`

`std::is_constant_evaluated()` is a `constexpr` function that returns `true` when the compiler is evaluating the call as part of a constant expression, and `false` when the function is running normally at runtime. This lets you write a single `constexpr` function that uses one algorithm during compile-time evaluation and a different algorithm when called at runtime.

The motivating use case is mathematical functions. At compile time you often have to use a portable algorithm (like Newton's method for square roots), because hardware intrinsics and `<cmath>` functions may not be available in a constant-evaluation context. At runtime you want to call the optimized, hardware-backed version instead. Without `is_constant_evaluated()` you would need two separate functions and manual selection, which is fragile. With it, the selection is automatic:

```cpp
#include <type_traits>

constexpr double my_sqrt(double x) {
    if (std::is_constant_evaluated()) {
        // Compile-time: Newton's method (no <cmath>)
        double guess = x / 2.0;
        for (int i = 0; i < 20; ++i)
            guess = (guess + x / guess) / 2.0;
        return guess;
    } else {
        // Runtime: use hardware-optimized std::sqrt
        return std::sqrt(x);
    }
}
```

### When Does It Return `true`

The function returns `true` in any context that the standard calls "manifestly constant-evaluated." If you are not sure whether a given context qualifies, the table below covers the common cases.

| Context | Returns |
| --- | --- |
| `constexpr` variable initializer | `true` |
| `consteval` function body | `true` |
| `static_assert` argument | `true` |
| Template argument | `true` |
| `if constexpr` condition | `true` |
| Regular runtime call | `false` |
| `const` variable init (non-constexpr) | Implementation-defined |

### Common Pitfall: `if constexpr` vs `if`

This is the mistake that trips almost everyone up the first time they use `std::is_constant_evaluated()`. If you write `if constexpr` instead of plain `if`, the condition itself is evaluated at compile time - and since you are asking a compile-time function whether you are in a compile-time context, the answer is always `true`. The `else` branch becomes dead code forever.

The reason this is so easy to get wrong is that `if constexpr` looks like the "more compile-time-aware" version, so reaching for it feels natural. But that is exactly the wrong instinct here. The whole point of `is_constant_evaluated()` is that its answer changes depending on context - and `if constexpr` collapses that context to "always compile-time."

```cpp
constexpr int f() {
    if constexpr (std::is_constant_evaluated()) {
        return 1; // BUG: ALWAYS taken (condition itself is constant-evaluated)
    } else {
        return 2; // Dead code - never reached
    }
}

constexpr int g() {
    if (std::is_constant_evaluated()) {
        return 1; // Correct: plain if - taken only during constant evaluation
    } else {
        return 2; // Taken at runtime
    }
}
```

**Rule:** Always use plain `if`, never `if constexpr`, with `is_constant_evaluated()`.

### C++23: `if consteval` (Preferred)

C++23 introduced `if consteval` as the safe, explicit replacement for this pattern. It avoids the pitfall entirely because the syntax itself distinguishes the two paths, and you cannot accidentally mix them up:

```cpp
constexpr int h() {
    if consteval {
        return 1; // Compile-time path
    } else {
        return 2; // Runtime path
    }
}
```

If you are targeting C++23 or later, prefer `if consteval`. It is clearer and impossible to misuse in the `if constexpr` way.

---

## Self-Assessment

### Q1: Write a square root function that uses a lookup table at compile time and `std::sqrt` at runtime

This example shows the full pattern in action. At compile time, we build a lookup table using Newton's method (pure constexpr math). For runtime calls, we hand off to `std::sqrt` so the hardware FPU instruction is used instead of the slower iterative method. Notice that the compile-time and runtime paths can use completely different strategies - the caller does not need to care which one runs.

```cpp
#include <iostream>
#include <type_traits>
#include <cmath>
#include <array>

// === Compile-time sqrt via Newton's method ===
constexpr double constexpr_sqrt_newton(double x) {
    if (x < 0) return -1.0; // error
    if (x == 0) return 0.0;
    double guess = x / 2.0;
    for (int i = 0; i < 30; ++i)
        guess = (guess + x / guess) / 2.0;
    return guess;
}

// === Build a compile-time lookup table for sqrt(0..N-1) ===
template<std::size_t N>
constexpr auto make_sqrt_table() {
    std::array<double, N> table{};
    for (std::size_t i = 0; i < N; ++i)
        table[i] = constexpr_sqrt_newton(static_cast<double>(i));
    return table;
}

constexpr auto sqrt_table = make_sqrt_table<256>();

// === Dual-path sqrt using is_constant_evaluated ===
constexpr double my_sqrt(double x) {
    if (std::is_constant_evaluated()) {
        // Compile-time: Newton's method
        return constexpr_sqrt_newton(x);
    } else {
        // Runtime: hardware-optimized std::sqrt
        return std::sqrt(x);
    }
}

// Compile-time verification using the table
static_assert(sqrt_table[0] == 0.0);
static_assert(sqrt_table[1] == 1.0);
static_assert(sqrt_table[4] > 1.999 && sqrt_table[4] < 2.001);
static_assert(sqrt_table[9] > 2.999 && sqrt_table[9] < 3.001);

// Compile-time verification using my_sqrt
static_assert(my_sqrt(0.0) == 0.0);
static_assert(my_sqrt(1.0) == 1.0);
static_assert(my_sqrt(4.0) > 1.999 && my_sqrt(4.0) < 2.001);

int main() {
    std::cout << "=== Compile-Time Lookup Table ===\n";
    for (int i : {0, 1, 4, 9, 16, 25, 100, 255}) {
        std::cout << "sqrt_table[" << i << "] = " << sqrt_table[i] << "\n";
    }

    std::cout << "\n=== my_sqrt (runtime path uses std::sqrt) ===\n";
    double values[] = {2.0, 3.0, 10.0, 100.0, 12345.6789};
    for (double v : values) {
        std::cout << "my_sqrt(" << v << ") = " << my_sqrt(v) << "\n";
    }

    // Prove compile-time path works
    constexpr double ct_result = my_sqrt(144.0);
    std::cout << "\nconstexpr my_sqrt(144.0) = " << ct_result << "\n";

    // Prove runtime path works
    double rt_input = 144.0;
    double rt_result = my_sqrt(rt_input);
    std::cout << "runtime  my_sqrt(144.0) = " << rt_result << "\n";

    return 0;
}
```

Notice that `constexpr double ct_result = my_sqrt(144.0)` forces compile-time evaluation, while `double rt_result = my_sqrt(rt_input)` cannot be constant-evaluated because `rt_input` is a non-`constexpr` variable - so the runtime path runs for that call. Same function name, same argument value, different paths - that is exactly what this feature is for.

**Expected output:**

```text
=== Compile-Time Lookup Table ===
sqrt_table[0] = 0
sqrt_table[1] = 1
sqrt_table[4] = 2
sqrt_table[9] = 3
sqrt_table[16] = 4
sqrt_table[25] = 5
sqrt_table[100] = 10
sqrt_table[255] = 15.9687

=== my_sqrt (runtime path uses std::sqrt) ===
my_sqrt(2) = 1.41421
my_sqrt(3) = 1.73205
my_sqrt(10) = 3.16228
my_sqrt(100) = 10
my_sqrt(12345.7) = 111.111

constexpr my_sqrt(144.0) = 12
runtime  my_sqrt(144.0) = 12
```

### Q2: Explain when `std::is_constant_evaluated()` returns `true` vs `false`

`std::is_constant_evaluated()` returns `true` during **manifestly constant-evaluated** expressions. Here is a concrete demonstration with every important context shown side by side. The `const int b` case is the most surprising one - read the note below the code.

| Context | Result | Example |
| --- | --- | --- |
| `constexpr` variable init | `true` | `constexpr auto x = f();` |
| `consteval` function body | `true` | `consteval int f() { ... }` |
| `static_assert` argument | `true` | `static_assert(f() == 42);` |
| Template argument | `true` | `std::array<int, f()>` |
| `if constexpr` condition | `true` | `if constexpr (f()) ...` |
| `const` variable init (integral) | Maybe `true` | `const int x = f();` - compiler may try |
| Regular function call | `false` | `int x = f();` |
| Inside a loop at runtime | `false` | `for(...) f();` |

```cpp
#include <iostream>
#include <type_traits>

constexpr int detect() {
    if (std::is_constant_evaluated())
        return 1;  // compile-time
    else
        return 2;  // runtime
}

// === Demonstration ===
constexpr int a = detect();          // 1 - constexpr init, constant evaluation
const     int b = detect();          // 1 - const int init, compiler tries constant eval
int           c = detect();          // 2 - no constexpr/const, runtime
static    int d = detect();          // 2 - static with no const, runtime

static_assert(detect() == 1);        // 1 - static_assert is constant context

template<int N> struct S {};
using T = S<detect()>;               // 1 - template argument is constant context

int main() {
    std::cout << "constexpr int a = detect(): " << a << "\n";  // 1
    std::cout << "const     int b = detect(): " << b << "\n";  // 1
    std::cout << "int           c = detect(): " << c << "\n";  // 2
    std::cout << "static    int d = detect(): " << d << "\n";  // 2

    // Direct call - runtime context
    std::cout << "detect() in runtime: " << detect() << "\n";  // 2

    // Forced compile-time via constexpr
    constexpr int e = detect();
    std::cout << "constexpr int e = detect(): " << e << "\n";  // 1

    return 0;
}
```

The `const int b = detect()` case is implementation-defined - the standard permits but does not require the compiler to evaluate it at compile time. In practice, most compilers will do so for integral types, but you cannot rely on it. If you need a guarantee, use `constexpr`.

**Expected output:**

```text
constexpr int a = detect(): 1
const     int b = detect(): 1
int           c = detect(): 2
static    int d = detect(): 2
detect() in runtime: 2
constexpr int e = detect(): 1
```

### Q3: Show `constexpr if` used with `is_constant_evaluated()` to call non-constexpr code at runtime

The runtime branch of a dual-path function is allowed to call non-`constexpr` functions, use pointer arithmetic, and generally do things that are not permitted at compile time. The `is_constant_evaluated()` check is the gate: the compile-time path must be fully constexpr-legal, but the runtime path has no such restriction. In a real codebase, the runtime branch might use SIMD intrinsics, hardware CRC instructions, or platform-specific fast paths that you cannot express in portable `constexpr` code.

```cpp
#include <iostream>
#include <type_traits>
#include <cstring>  // std::memcpy - not constexpr

// === Constexpr function that uses non-constexpr code at runtime ===
constexpr int fast_hash(const char* str, int len) {
    if (std::is_constant_evaluated()) {
        // Compile-time: simple constexpr hash (djb2)
        unsigned hash = 5381;
        for (int i = 0; i < len; ++i)
            hash = hash * 33 + static_cast<unsigned>(str[i]);
        return static_cast<int>(hash & 0x7FFFFFFF);
    } else {
        // Runtime: can use non-constexpr operations
        // (e.g., memcpy to aligned buffer, SIMD, etc.)
        unsigned hash = 5381;
        for (int i = 0; i < len; ++i)
            hash = hash * 33 + static_cast<unsigned>(str[i]);
        // In real code, this branch could use platform intrinsics:
        // __builtin_ia32_crc32qi, etc.
        return static_cast<int>(hash & 0x7FFFFFFF);
    }
}

// === Another example: constexpr string compare ===
constexpr bool str_equal(const char* a, const char* b, int len) {
    if (std::is_constant_evaluated()) {
        // Compile-time: character-by-character
        for (int i = 0; i < len; ++i)
            if (a[i] != b[i]) return false;
        return true;
    } else {
        // Runtime: use optimized std::memcmp
        return std::memcmp(a, b, len) == 0;
    }
}

// Compile-time usage
constexpr int ct_hash = fast_hash("hello", 5);
static_assert(ct_hash > 0);

constexpr bool ct_eq = str_equal("abc", "abc", 3);
static_assert(ct_eq);

constexpr bool ct_neq = str_equal("abc", "abd", 3);
static_assert(!ct_neq);

int main() {
    std::cout << "=== Dual compile/runtime paths ===\n";

    // Compile-time result
    std::cout << "Compile-time hash(\"hello\") = " << ct_hash << "\n";

    // Runtime result - same algorithm, could use intrinsics
    const char* s = "hello";
    int rt_hash = fast_hash(s, 5);
    std::cout << "Runtime     hash(\"hello\") = " << rt_hash << "\n";

    // Both produce the same result (same algorithm in this demo)
    std::cout << "Match: " << (ct_hash == rt_hash ? "yes" : "no") << "\n";

    std::cout << "\n=== str_equal ===\n";
    std::cout << "Compile-time: \"abc\" == \"abc\": " << ct_eq << "\n";
    std::cout << "Compile-time: \"abc\" == \"abd\": " << ct_neq << "\n";

    // Runtime comparison
    const char* a = "test";
    const char* b = "test";
    std::cout << "Runtime: \"test\" == \"test\": " << str_equal(a, b, 4) << "\n";

    std::cout << "\n=== Common Pitfall ===\n";
    std::cout << "WRONG: if constexpr (is_constant_evaluated()) - always true!\n";
    std::cout << "RIGHT: if (is_constant_evaluated()) - plain if\n";
    std::cout << "BEST:  if consteval { } else { }  (C++23)\n";

    return 0;
}
```

The dual-path pattern gives you the best of both worlds: a `constexpr`-legal portable algorithm for compile-time use, and a fast platform-specific implementation for runtime use - all in one function, with zero friction for the caller.

**Expected output:**

```text
=== Dual compile/runtime paths ===
Compile-time hash("hello") = 261238937
Runtime     hash("hello") = 261238937
Match: yes

=== str_equal ===
Compile-time: "abc" == "abc": 1
Compile-time: "abc" == "abd": 0
Runtime: "test" == "test": 1

=== Common Pitfall ===
WRONG: if constexpr (is_constant_evaluated()) - always true!
RIGHT: if (is_constant_evaluated()) - plain if
BEST:  if consteval { } else { }  (C++23)
```

---

## Notes

- Use plain `if`, **never** `if constexpr`, with `std::is_constant_evaluated()`.
- In C++23, prefer `if consteval` - it's safer and clearer.
- The compile-time path cannot call non-`constexpr` functions, but the runtime path can.
- Typical use: constexpr-friendly algorithm at compile time, optimized intrinsic at runtime.
- `std::is_constant_evaluated()` is itself `constexpr` and `noexcept`.
- The `const` variable case is subtle: `const int x = f();` may or may not be constant-evaluated depending on context.
