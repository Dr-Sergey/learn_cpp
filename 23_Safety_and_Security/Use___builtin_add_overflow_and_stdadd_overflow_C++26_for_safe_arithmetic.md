# Use __builtin_add_overflow and std::add_overflow (C++26) for safe arithmetic

**Category:** Safety and Security  
**Standard:** C++26  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Integer-Overflow-Builtins.html>  

---

## Topic Overview

Signed integer overflow is **undefined behavior** in C++. The compiler is allowed to assume it never happens and optimize accordingly - removing overflow checks, making infinite loops, or producing completely unexpected code. `__builtin_add_overflow` (GCC/Clang) and the upcoming `std::add_overflow` (C++26) perform the addition and **report whether overflow occurred** without triggering UB.

The reason this trips people up is that overflow checks that look reasonable in source code can be silently removed by the optimizer. You write a check, you think you're protected, and the compiled binary does not contain the check at all. That is not a compiler bug - it is the standard explicitly allowing this optimization because signed overflow is UB.

### The Overflow Problem

The distinction between signed and unsigned overflow is important because the language treats them completely differently:

```cpp
Signed overflow is UB:
  int x = INT_MAX;
  x + 1  ->  UNDEFINED BEHAVIOR
  Compiler may: return INT_MIN (wrapping), optimize away checks, anything

Unsigned "overflow" is well-defined (modular arithmetic):
  unsigned x = UINT_MAX;
  x + 1  ->  0  (wrapping, always defined)
  But the wrap is rarely what you want!

__builtin_add_overflow:
  int result;
  if (__builtin_add_overflow(a, b, &result)) {
      // overflow occurred - result contains the wrapped value
      // but we detected it without UB!
  }
```

### Overflow Detection Functions

The GCC/Clang builtins exist today; the `std::` versions are coming in C++26. For portable code that needs to compile with both GCC/Clang and MSVC in the meantime, you can fall back to the pre-check approach shown in Q3:

| Function | Available | Detects |
| --- | --- | --- |
| `__builtin_add_overflow(a, b, &r)` | GCC/Clang | Addition overflow |
| `__builtin_sub_overflow(a, b, &r)` | GCC/Clang | Subtraction overflow |
| `__builtin_mul_overflow(a, b, &r)` | GCC/Clang | Multiplication overflow |
| `std::add_overflow(a, b, &r)` | C++26 | Addition overflow (portable) |
| `std::sub_overflow(a, b, &r)` | C++26 | Subtraction overflow |
| `std::mul_overflow(a, b, &r)` | C++26 | Multiplication overflow |
| `-ftrapv` | GCC/Clang | Traps on signed overflow (debug) |
| `-fsanitize=signed-integer-overflow` | UBSan | Reports at runtime |

### Core Example

Here is the minimal form. Notice the compile output comment - `INT_MAX + 1` is detected cleanly without triggering any UB, and the compiler generates a single `add` + `jo` instruction pair for it:

```cpp
#include <iostream>
#include <climits>
#include <cstdint>

int main() {
    int a = INT_MAX;
    int b = 1;
    int result;

    if (__builtin_add_overflow(a, b, &result)) {
        std::cout << "Overflow detected! " << a << " + " << b
                  << " doesn't fit in int\n";
    } else {
        std::cout << "Result: " << result << "\n";
    }
    // Output: Overflow detected! 2147483647 + 1 doesn't fit in int
}
```

---

## Self-Assessment

### Q1: Detect signed integer overflow without UB using __builtin_add_overflow(a, b, &result)

**Answer:**

The most important use case here is the allocation size check at the bottom - multiplying a count by an element size to get a `malloc` argument is one of the most common integer overflow vulnerabilities in real code:

```cpp
#include <iostream>
#include <climits>
#include <cstdint>

// __builtin_add_overflow
//
// Signature:
//   bool __builtin_add_overflow(type1 a, type2 b, type3 *result);
//
// Returns: true if overflow occurred, false if result is exact
// Stores: the (possibly wrapped) result in *result
// Types: works with any integer type, can mix signed/unsigned
// NO UB: even on signed overflow, the check itself is well-defined

bool safe_add(int a, int b, int& result) {
    if (__builtin_add_overflow(a, b, &result)) {
        return false;  // overflow!
    }
    return true;  // result is valid
}

bool safe_sub(int a, int b, int& result) {
    if (__builtin_sub_overflow(a, b, &result)) {
        return false;
    }
    return true;
}

bool safe_mul(int a, int b, int& result) {
    if (__builtin_mul_overflow(a, b, &result)) {
        return false;
    }
    return true;
}

// Cross-type overflow detection
// Can detect overflow when converting between types:

bool safe_size_to_int(size_t sz, int& result) {
    // Detects if sz fits in int
    if (__builtin_add_overflow(sz, size_t{0}, &result)) {
        return false;  // doesn't fit
    }
    return true;
}

int main() {
    int result;

    // Normal addition: no overflow
    if (safe_add(100, 200, result)) {
        std::cout << "100 + 200 = " << result << "\n";
    }
    // Output: 100 + 200 = 300

    // Overflow: INT_MAX + 1
    if (!safe_add(INT_MAX, 1, result)) {
        std::cout << "INT_MAX + 1: OVERFLOW!\n";
    }
    // Output: INT_MAX + 1: OVERFLOW!

    // Underflow: INT_MIN - 1
    if (!safe_sub(INT_MIN, 1, result)) {
        std::cout << "INT_MIN - 1: OVERFLOW!\n";
    }
    // Output: INT_MIN - 1: OVERFLOW!

    // Multiplication overflow
    if (!safe_mul(INT_MAX, 2, result)) {
        std::cout << "INT_MAX * 2: OVERFLOW!\n";
    }
    // Output: INT_MAX * 2: OVERFLOW!

    // Common security pattern: allocation size check
    int count = 1000000;
    int elem_size = 4000;
    int total_size;
    if (!safe_mul(count, elem_size, total_size)) {
        std::cerr << "Allocation size overflow - reject!\n";
    }
    // Output: Allocation size overflow - reject!

    // Compiles to efficient code:
    //   addl %esi, %edi
    //   jo   .overflow    ; single instruction checks overflow flag!
    // No branch prediction penalty on non-overflow path.
}
```

**Explanation:** `__builtin_add_overflow(a, b, &result)` performs the addition and returns `true` if the mathematical result cannot be represented in the type of `*result`. It does NOT trigger undefined behavior - the overflow is detected cleanly. On x86, it compiles to a single `add` instruction followed by a `jo` (jump on overflow) check of the CPU's overflow flag, making it extremely efficient.

### Q2: Implement a saturating_add function that returns INT_MAX on overflow instead of wrapping

**Answer:**

Saturating arithmetic is exactly what you want for things like audio mixing - if two loud samples add up past the maximum, you clamp to the maximum rather than wrapping around to a completely wrong value. The `if constexpr` on signedness means this single template handles both signed and unsigned correctly:

```cpp
#include <iostream>
#include <climits>
#include <cstdint>
#include <limits>
#include <type_traits>

// Saturating arithmetic
// Instead of overflow -> UB or wrap, clamp to min/max.
// Used in: audio processing, image processing, financial calculations

template <typename T>
    requires std::is_integral_v<T>
T saturating_add(T a, T b) {
    T result;
    if (__builtin_add_overflow(a, b, &result)) {
        // Overflow occurred - determine direction
        if constexpr (std::is_signed_v<T>) {
            // If a and b are both positive -> overflow to max
            // If a and b are both negative -> underflow to min
            return (a > 0) ? std::numeric_limits<T>::max()
                           : std::numeric_limits<T>::min();
        } else {
            // Unsigned: overflow always wraps high -> saturate to max
            return std::numeric_limits<T>::max();
        }
    }
    return result;
}

template <typename T>
    requires std::is_integral_v<T>
T saturating_sub(T a, T b) {
    T result;
    if (__builtin_sub_overflow(a, b, &result)) {
        if constexpr (std::is_signed_v<T>) {
            return (b > 0) ? std::numeric_limits<T>::min()
                           : std::numeric_limits<T>::max();
        } else {
            return std::numeric_limits<T>::min(); // 0 for unsigned
        }
    }
    return result;
}

template <typename T>
    requires std::is_integral_v<T>
T saturating_mul(T a, T b) {
    T result;
    if (__builtin_mul_overflow(a, b, &result)) {
        if constexpr (std::is_signed_v<T>) {
            bool negative = (a > 0) != (b > 0);
            return negative ? std::numeric_limits<T>::min()
                            : std::numeric_limits<T>::max();
        } else {
            return std::numeric_limits<T>::max();
        }
    }
    return result;
}

int main() {
    // Signed overflow -> clamp to INT_MAX
    std::cout << "saturating_add(INT_MAX, 1):       "
              << saturating_add(INT_MAX, 1) << "\n";
    // Output: 2147483647 (INT_MAX)

    // Signed underflow -> clamp to INT_MIN
    std::cout << "saturating_add(INT_MIN, -1):      "
              << saturating_add(INT_MIN, -1) << "\n";
    // Output: -2147483648 (INT_MIN)

    // No overflow -> normal result
    std::cout << "saturating_add(100, 200):          "
              << saturating_add(100, 200) << "\n";
    // Output: 300

    // Unsigned overflow -> clamp to UINT_MAX
    std::cout << "saturating_add(UINT_MAX, 1u):     "
              << saturating_add(UINT_MAX, 1u) << "\n";
    // Output: 4294967295 (UINT_MAX)

    // Multiplication overflow
    std::cout << "saturating_mul(INT_MAX, 2):        "
              << saturating_mul(INT_MAX, 2) << "\n";
    // Output: 2147483647 (INT_MAX)

    // Practical use: audio sample mixing (int16_t)
    int16_t sample1 = 30000;
    int16_t sample2 = 20000;
    int16_t mixed = saturating_add(sample1, sample2);
    std::cout << "Audio mix 30000 + 20000:           "
              << mixed << "\n";
    // Output: 32767 (INT16_MAX - no clipping distortion from wrap)

    // Without saturation:
    // int16_t bad_mix = sample1 + sample2;  // UB! (signed overflow)
    // Wrapped result on most platforms: -15536 (sounds terrible)

    // Output:
    // saturating_add(INT_MAX, 1):       2147483647
    // saturating_add(INT_MIN, -1):      -2147483648
    // saturating_add(100, 200):          300
    // saturating_add(UINT_MAX, 1u):     4294967295
    // saturating_mul(INT_MAX, 2):        2147483647
    // Audio mix 30000 + 20000:           32767
}
```

**Explanation:** `saturating_add` uses `__builtin_add_overflow` to detect overflow, then clamps to `INT_MAX` or `INT_MIN` (for signed) or `T_MAX` (for unsigned) depending on the overflow direction. This is the correct behavior for audio processing (prevents digital clipping), image processing (pixel values stay in range), and any domain where wrapping/UB is worse than clamping.

### Q3: Show the UB from assuming signed overflow wraps and how the compiler exploits it

**Answer:**

This is one of the most important examples in the safety section. The reason this trips people up is that the "obvious" overflow check `x + 100 < x` is mathematically correct and works fine at `-O0`, but is provably removed by the optimizer at `-O2`. The compiler is not wrong to remove it - the standard says it can assume overflow never happens, so the comparison is always false:

```cpp
#include <iostream>
#include <climits>
#include <cstdint>

// The compiler ASSUMES signed overflow never happens
//
// The C++ standard says signed integer overflow is UB.
// Compilers use this to OPTIMIZE: if x + 1 > x is always true
// (because overflow "can't happen"), the compiler removes the check.

// Example 1: Infinite loop

// void count_forever() {
//     for (int i = 0; i >= 0; ++i) {
//         // Programmer assumes: when i reaches INT_MAX and wraps to INT_MIN,
//         //                     i < 0, loop exits.
//         // Compiler assumes:   i starts at 0, only increments, so i >= 0 ALWAYS.
//         //                     Loop condition is always true -> infinite loop!
//     }
// }
// With -O2: compiler removes the check -> truly infinite loop.
// With -O0: may appear to work (wraps on most architectures).

// Example 2: Removed bounds check

bool check_overflow_buggy(int x) {
    // Programmer's intent: detect if x+100 overflows
    return x + 100 < x;
    // Compiler reasoning: for signed int, x + 100 < x is NEVER true
    //   (because if it were, overflow occurred, which is UB, which "can't happen")
    // Compiler optimizes to: return false;  (always!)
}

// Example 3: Compiler eliminates null check
//
// void process(int* p) {
//     int x = *p;       // dereferences p -> UB if p is null
//     if (p == nullptr)  // compiler: "p was already dereferenced,
//         return;        //  so p can't be null (otherwise UB already happened)"
//     use(x);           // dead code from compiler's perspective
// }
// Result: null check is REMOVED, crash on null pointer.

// Safe overflow check

bool check_overflow_safe(int x) {
    int result;
    return __builtin_add_overflow(x, 100, &result);
    // This is well-defined! Returns true if overflow occurs.
}

// Pre-check without builtins

bool will_add_overflow(int a, int b) {
    // Check BEFORE the operation to avoid UB
    if (b > 0 && a > INT_MAX - b) return true;   // positive overflow
    if (b < 0 && a < INT_MIN - b) return true;   // negative overflow
    return false;
}

int main() {
    // Demonstrate the compiler optimization
    std::cout << "check_overflow_buggy(INT_MAX): "
              << check_overflow_buggy(INT_MAX) << "\n";
    // With -O2: always prints 0 (false) - check was optimized away!
    // The compiler treated x+100 < x as always false.

    std::cout << "check_overflow_safe(INT_MAX):  "
              << check_overflow_safe(INT_MAX) << "\n";
    // Always prints 1 (true) - correctly detects overflow.

    // Pre-check method
    std::cout << "will_add_overflow(INT_MAX, 1):  "
              << will_add_overflow(INT_MAX, 1) << "\n";
    // Output: 1 (true)

    std::cout << "will_add_overflow(100, 200):    "
              << will_add_overflow(100, 200) << "\n";
    // Output: 0 (false)

    // Why -fwrapv is not the answer
    //
    // g++ -fwrapv: makes signed overflow defined (two's complement wrap)
    // Problem: disables ALL overflow-based optimizations
    //   - Compiler can't assume loops terminate
    //   - Compiler can't simplify x+1 > x to true
    //   - Performance cost: 1-5% on benchmarks
    //
    // Better approach: __builtin_add_overflow (or -ftrapv for debugging)
    //   - Only pay the cost where you check
    //   - Compiler can still optimize everywhere else

    // Output:
    // check_overflow_buggy(INT_MAX): 0
    // check_overflow_safe(INT_MAX):  1
    // will_add_overflow(INT_MAX, 1):  1
    // will_add_overflow(100, 200):    0
}
```

**Explanation:** The compiler assumes signed overflow never occurs (because it's UB). This means overflow checks written as `x + 100 < x` are optimized to `false` at `-O2` - the compiler proves that for non-overflowing `int` arithmetic, the result of `x + 100` is always >= `x`. The check is silently removed, leaving the code exactly as vulnerable as without the check. `__builtin_add_overflow` is the correct tool: it's explicitly defined to check for overflow without triggering UB, and compiles to a single CPU flag check.

---

## Notes

- **`std::add_overflow`** (C++26, P0543) will standardize `__builtin_add_overflow` as a portable API.
- **`__builtin_*_overflow`** works with ANY integer types and cross-type conversions (e.g., add two `int`s and store in `int64_t` - no overflow).
- **`-ftrapv`** (GCC/Clang) traps (aborts) on signed overflow at runtime - useful for debugging but not production.
- **`-fwrapv`** makes signed overflow defined as two's complement wrapping - disables useful optimizations, not recommended for general use.
- **`-fsanitize=signed-integer-overflow`** (UBSan) reports overflow at runtime with a diagnostic - use in CI testing.
- **Allocation size checks** (`count * size`) are the most critical use case - integer overflow in `malloc(n * sizeof(T))` leads to heap overflow (CWE-190).
