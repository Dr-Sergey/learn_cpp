# Understand How to Use `if consteval` (C++23) to Branch on Constant Evaluation Context

**Category:** Compile Time Programming  
**Standard:** C++23, C++20 (`if consteval`)  
**Reference:** <https://en.cppreference.com/w/cpp/language/if>  

---

## Topic Overview

### What Is `if consteval`

`if consteval` (C++23) is a compile-time branching construct that checks whether code is being evaluated **during constant evaluation** (compile time) or at **runtime**. It replaces the awkward `if (std::is_constant_evaluated())` pattern from C++20.

```cpp
constexpr int compute(int x) {
    if consteval {
        // This branch runs ONLY during constant evaluation
        return x * x;  // safe compile-time path
    } else {
        // This branch runs ONLY at runtime
        return fast_intrinsic(x);  // can use non-constexpr functions
    }
}
```

The key thing `if consteval` unlocks that `std::is_constant_evaluated()` could not is the ability to call non-`constexpr` functions in the `else` branch. At runtime, you're free to use hardware intrinsics, `memcpy`, platform-specific APIs - anything. The compile-time branch gets a strict `constexpr`-safe path. The two branches are truly independent.

### `if consteval` vs `std::is_constant_evaluated()`

The reason `if consteval` was added even though `std::is_constant_evaluated()` already existed comes down to two concrete problems with the older approach. The table captures both:

| Feature | `if consteval` (C++23) | `std::is_constant_evaluated()` (C++20) |
| --- | --- | --- |
| Syntax | `if consteval { ... } else { ... }` | `if (std::is_constant_evaluated()) { ... }` |
| Type | Language keyword - special statement | Library function returning `bool` |
| Compile-time branch | `if consteval` block is `consteval` context | Still `constexpr`, not `consteval` |
| Call non-constexpr in else | Yes - the else branch is explicitly runtime | No - the whole function is still constexpr; calling non-constexpr functions is ill-formed |
| False positive risk | None | `if constexpr (std::is_constant_evaluated())` is **always true** - a common bug |
| Negation | `if !consteval { ... }` | `if (!std::is_constant_evaluated()) { ... }` |

### The Critical Difference: `consteval` Context

This is the part that trips people up, so let's be explicit about it.

Inside `if consteval { ... }`, you are in a **manifestly constant-evaluated** context. This means:

- You can call `consteval` functions.
- The compiler **guarantees** this code only runs at compile time.

Inside the `else` branch:

- You can call **non-constexpr** functions (intrinsics, syscalls, etc.).
- The compiler **guarantees** this code only runs at runtime.

With `std::is_constant_evaluated()`, neither guarantee holds - the function body must remain valid `constexpr` code in *both* branches, because the compiler doesn't treat each branch differently for the purposes of what's allowed inside it.

---

## Self-Assessment

### Q1: Write `if consteval { /* compile-time path */ } else { /* runtime path */ }` in a constexpr function

This example shows two real use cases: picking between a constexpr-safe Newton's-method sqrt and the hardware-accelerated `std::sqrt`, and adding debug logging only on the runtime path where `std::cout` is available.

```cpp
#include <iostream>
#include <cmath>  // std::sqrt (not constexpr in most implementations)

// === Compile-time sqrt using Newton's method ===
constexpr double constexpr_sqrt(double x) {
    if (x < 0) return -1.0;
    double guess = x / 2.0;
    for (int i = 0; i < 100; ++i) {
        guess = (guess + x / guess) / 2.0;
    }
    return guess;
}

// === Dual-path function using if consteval ===
constexpr double smart_sqrt(double x) {
    if consteval {
        // Compile-time path: use our constexpr Newton's method
        return constexpr_sqrt(x);
    } else {
        // Runtime path: use the optimized std::sqrt (hardware instruction)
        return std::sqrt(x);
    }
}

// === Another example: debug logging only at runtime ===
constexpr int factorial(int n) {
    if consteval {
        // Pure calculation at compile time - no side effects
        int result = 1;
        for (int i = 2; i <= n; ++i) result *= i;
        return result;
    } else {
        // At runtime, we can log
        std::cout << "  Computing factorial(" << n << ") at runtime\n";
        int result = 1;
        for (int i = 2; i <= n; ++i) result *= i;
        return result;
    }
}

int main() {
    // Compile-time usage
    constexpr double compile_time_result = smart_sqrt(144.0);
    static_assert(compile_time_result > 11.99 && compile_time_result < 12.01);
    std::cout << "Compile-time sqrt(144) = " << compile_time_result << "\n";

    // Runtime usage - will call std::sqrt
    double runtime_val = 144.0;
    double runtime_result = smart_sqrt(runtime_val);
    std::cout << "Runtime sqrt(144) = " << runtime_result << "\n";

    // Compile-time factorial
    constexpr int ct_fact = factorial(5);
    static_assert(ct_fact == 120);
    std::cout << "Compile-time factorial(5) = " << ct_fact << "\n";

    // Runtime factorial - will print log message
    int n = 5;
    int rt_fact = factorial(n);
    std::cout << "Runtime factorial(5) = " << rt_fact << "\n";

    return 0;
}
```

**Expected output:**

```text
Compile-time sqrt(144) = 12
Runtime sqrt(144) = 12
Compile-time factorial(5) = 120
  Computing factorial(5) at runtime
Runtime factorial(5) = 120
```

### Q2: Explain the difference between `if consteval` and `if (std::is_constant_evaluated())`

This example demonstrates the classic `if constexpr (std::is_constant_evaluated())` bug where the condition is always evaluated as `true`, making the else branch dead code. Watch the `buggy` function - both the compile-time and runtime calls return `20` (x*2), even though the intent was for runtime to return `30` (x*3).

```cpp
#include <iostream>
#include <type_traits>

// === PROBLEM with std::is_constant_evaluated() ===

// Bug 1: if constexpr + is_constant_evaluated = ALWAYS TRUE
constexpr int buggy(int x) {
    // WARNING: This is ALWAYS true! 'if constexpr' forces constant evaluation
    // of the condition, and is_constant_evaluated() returns true during
    // constant evaluation of the condition itself.
    if constexpr (std::is_constant_evaluated()) {
        return x * 2;  // Always taken!
    } else {
        return x * 3;  // Dead code!
    }
}

// Bug 2: Cannot call non-constexpr in else branch
/*
constexpr int buggy2(int x) {
    if (std::is_constant_evaluated()) {
        return x;
    } else {
        // ERROR: printf is not constexpr, and the whole function
        // must be valid constexpr - both branches are checked
        printf("runtime path\n");  // ill-formed
        return x;
    }
}
*/

// === CORRECT with if consteval ===
constexpr int correct(int x) {
    if consteval {
        return x * 2;  // Only at compile time
    } else {
        return x * 3;  // Only at runtime
    }
}

// if consteval allows non-constexpr in else branch
constexpr int correct2(int x) {
    if consteval {
        return x * x;
    } else {
        // This is fine! The else branch is runtime-only
        std::cout << "Runtime execution\n";
        return x * x;
    }
}

// === Negated form: if !consteval ===
constexpr int runtime_only_path(int x) {
    if !consteval {
        // Only at runtime
        std::cout << "Running at runtime with x = " << x << "\n";
    }
    return x + 1;
}

int main() {
    // Demonstrate the bug
    constexpr int ct = buggy(10);
    int rt_val = 10;
    int rt = buggy(rt_val);
    std::cout << "buggy(10) compile-time: " << ct << "\n";  // 20
    std::cout << "buggy(10) runtime:      " << rt << "\n";  // 20 (BUG! Should be 30)

    // Demonstrate correct behavior
    constexpr int ct2 = correct(10);
    int rt2 = correct(rt_val);
    std::cout << "correct(10) compile-time: " << ct2 << "\n";  // 20
    std::cout << "correct(10) runtime:      " << rt2 << "\n";  // 30 (correct!)

    // Non-constexpr in else branch
    constexpr int ct3 = correct2(5);
    std::cout << "correct2(5) compile-time: " << ct3 << "\n";
    int five = 5;
    int rt3 = correct2(five);  // prints "Runtime execution"
    std::cout << "correct2(5) runtime: " << rt3 << "\n";

    std::cout << "\n=== Summary ===\n";
    std::cout << "std::is_constant_evaluated():\n";
    std::cout << "  - Library function, returns bool\n";
    std::cout << "  - if constexpr with it = ALWAYS TRUE (bug!)\n";
    std::cout << "  - Cannot use non-constexpr in else branch\n";
    std::cout << "if consteval:\n";
    std::cout << "  - Language keyword\n";
    std::cout << "  - No false-positive risk\n";
    std::cout << "  - else branch allows non-constexpr code\n";
    std::cout << "  - Supports negation: if !consteval\n";

    return 0;
}
```

**Expected output:**

```text
buggy(10) compile-time: 20
buggy(10) runtime:      20
correct(10) compile-time: 20
correct(10) runtime:      30
correct2(5) compile-time: 25
Runtime execution
correct2(5) runtime: 25

=== Summary ===
std::is_constant_evaluated():

  - Library function, returns bool
  - if constexpr with it = ALWAYS TRUE (bug!)
  - Cannot use non-constexpr in else branch

if consteval:

  - Language keyword
  - No false-positive risk
  - else branch allows non-constexpr code
  - Supports negation: if !consteval
```

### Q3: Use `if consteval` to call a non-constexpr intrinsic only at runtime

This is the pattern you'll reach for most often in performance-sensitive code: a constexpr-safe fallback for the compile-time path, and a hardware-accelerated implementation at runtime. The `smart_copy` function shows this for buffer copies; `smart_popcount` shows it for bit manipulation.

```cpp
#include <iostream>
#include <cstring>   // std::memcpy (not constexpr)
#include <algorithm>  // std::copy
#include <array>

// === Scenario: optimized buffer copy ===
// At runtime: use memcpy (compiler intrinsic, highly optimized)
// At compile time: use element-by-element copy (constexpr-safe)

constexpr void smart_copy(int* dst, const int* src, std::size_t n) {
    if consteval {
        // Compile-time path: element-by-element (constexpr-safe)
        for (std::size_t i = 0; i < n; ++i) {
            dst[i] = src[i];
        }
    } else {
        // Runtime path: use hardware-optimized memcpy
        std::memcpy(dst, src, n * sizeof(int));
    }
}

// === Another example: constexpr popcount ===
// At runtime: use __builtin_popcount (single CPU instruction)
// At compile time: manual bit counting

constexpr int smart_popcount(unsigned int x) {
    if consteval {
        // Compile-time: count bits manually
        int count = 0;
        while (x) {
            count += (x & 1);
            x >>= 1;
        }
        return count;
    } else {
        // Runtime: use compiler intrinsic (compiles to POPCNT instruction)
        return __builtin_popcount(x);
    }
}

// === Example: hash function that uses runtime intrinsic ===
constexpr std::size_t smart_hash(const char* str, std::size_t len) {
    if consteval {
        // Compile-time: simple FNV-1a hash
        std::size_t hash = 14695981039346656037ULL;
        for (std::size_t i = 0; i < len; ++i) {
            hash ^= static_cast<std::size_t>(str[i]);
            hash *= 1099511628211ULL;
        }
        return hash;
    } else {
        // Runtime: could use CRC32 intrinsic or platform-specific hash
        // For demo, still use FNV-1a but log that we're at runtime
        std::cout << "  [runtime hash for \"";
        for (std::size_t i = 0; i < len; ++i) std::cout << str[i];
        std::cout << "\"]\n";
        std::size_t hash = 14695981039346656037ULL;
        for (std::size_t i = 0; i < len; ++i) {
            hash ^= static_cast<std::size_t>(str[i]);
            hash *= 1099511628211ULL;
        }
        return hash;
    }
}

int main() {
    // === Compile-time copy ===
    constexpr auto ct_result = [] {
        std::array<int, 4> src = {10, 20, 30, 40};
        std::array<int, 4> dst = {};
        smart_copy(dst.data(), src.data(), 4);
        return dst;
    }();
    static_assert(ct_result[0] == 10 && ct_result[3] == 40);
    std::cout << "Compile-time copy: " << ct_result[0] << " "
              << ct_result[1] << " " << ct_result[2] << " "
              << ct_result[3] << "\n";

    // Runtime copy (uses memcpy)
    int src[] = {1, 2, 3, 4};
    int dst[4] = {};
    smart_copy(dst, src, 4);
    std::cout << "Runtime copy: " << dst[0] << " " << dst[1]
              << " " << dst[2] << " " << dst[3] << "\n";

    // === Compile-time popcount ===
    constexpr int ct_pop = smart_popcount(0b10110011);  // 5 bits set
    static_assert(ct_pop == 5);
    std::cout << "Compile-time popcount(0b10110011) = " << ct_pop << "\n";

    // Runtime popcount (uses __builtin_popcount)
    unsigned int val = 0b10110011;
    std::cout << "Runtime popcount(0b10110011) = " << smart_popcount(val) << "\n";

    // === Compile-time hash ===
    constexpr auto ct_hash = smart_hash("hello", 5);
    std::cout << "Compile-time hash(\"hello\") = " << ct_hash << "\n";

    // Runtime hash
    const char* s = "hello";
    auto rt_hash = smart_hash(s, 5);
    std::cout << "Runtime hash(\"hello\") = " << rt_hash << "\n";

    return 0;
}
```

**Expected output:**

```text
Compile-time copy: 10 20 30 40
Runtime copy: 1 2 3 4
Compile-time popcount(0b10110011) = 5
Runtime popcount(0b10110011) = 5
Compile-time hash("hello") = 11831194018420276491
  [runtime hash for "hello"]
Runtime hash("hello") = 11831194018420276491
```

---

## Notes

- `if consteval` requires C++23. Use `std::is_constant_evaluated()` for C++20 code - but avoid `if constexpr (std::is_constant_evaluated())`.
- The `else` branch of `if consteval` can contain non-constexpr code - this is the primary advantage over `std::is_constant_evaluated()`.
- Use `if !consteval { ... }` as a shorthand for "runtime-only" logic.
- Common use cases: hardware intrinsics (popcount, CRC, SIMD), syscalls, `memcpy`, logging, platform-specific optimizations.
- `if consteval` cannot appear outside a `constexpr` or `consteval` function - it only makes sense in contexts where constant evaluation is possible.
