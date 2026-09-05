# Detect integer overflow at compile time and runtime safely

**Category:** Safety and Security  
**Standard:** Not version-specific  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Integer-Overflow-Builtins.html>  

---

## Topic Overview

This topic focuses on compile-time overflow detection and UBSan - complementing #557 (safe_add wrapper + __builtin_add_overflow).

Here is the thing that surprises most C++ programmers the first time they really think about it: signed integer overflow is not just "wrong behavior" - it is **undefined behavior** in the C++ standard. That means the compiler is not required to produce any particular result, and in practice, optimizing compilers actively exploit the assumption that signed overflow never happens in order to eliminate checks and simplify code. The result is that code you think is protecting you may be silently removed.

```cpp
Signed integer overflow in C++:
  int x = INT_MAX;
  x + 1;   // UNDEFINED BEHAVIOR!

  Why UB? The C++ standard says signed overflow has no defined result.
  Compilers EXPLOIT this: they assume it NEVER happens.

  Compiler optimization example:
    bool will_overflow(int x) { return x + 1 < x; }
    // Compiler: "signed overflow can't happen, so x+1 >= x always"
    // Optimized to: return false;  (WRONG for INT_MAX!)
```

The table below shows your detection toolkit at a glance. Think of UBSan as your safety net during testing, the overflow builtins as production-ready lightweight checks for specific operations, and the constexpr path as the way to catch problems at compile time for constant expressions.

| Approach | When | Overhead | Catches |
| --- | --- | --- | --- |
| `__builtin_add_overflow` | Runtime | ~1 instruction (checks OF flag) | Specific ops |
| `constexpr` evaluation | Compile time | Zero | Constant expressions |
| `-fsanitize=signed-integer-overflow` | Testing | ~20% | All signed ops |
| `-fsanitize=unsigned-integer-overflow` | Testing | ~20% | All unsigned wraps |
| `-ftrapv` | Runtime | Moderate | Signed overflow (trap) |

---

## Self-Assessment

### Q1: __builtin_add_overflow and std::add_overflow

The overflow builtins are remarkably cheap - they map to a single extra instruction (a branch on the CPU's overflow flag) that the processor would have set anyway. Here you can see each operation in action: addition, multiplication, and subtraction all checked without touching undefined behavior.

```cpp
#include <iostream>
#include <climits>
#include <cstdint>

int main() {
    // __builtin_add_overflow: returns true if overflow, stores result
    // Available in GCC 5+ and Clang 3.8+

    int result;

    // Safe addition (no overflow)
    if (__builtin_add_overflow(100, 200, &result)) {
        std::cout << "100 + 200: OVERFLOW!\n";
    } else {
        std::cout << "100 + 200 = " << result << '\n';
    }

    // Overflow detected!
    if (__builtin_add_overflow(INT_MAX, 1, &result)) {
        std::cout << "INT_MAX + 1: OVERFLOW detected (no UB!)\n";
        // result contains the wrapped value, but we don't use it
    }

    // Works for all integer types
    int64_t big_result;
    if (__builtin_mul_overflow(INT_MAX, (int64_t)2, &big_result)) {
        std::cout << "INT_MAX * 2 (int64): OVERFLOW\n";
    } else {
        std::cout << "INT_MAX * 2 = " << big_result << '\n';
    }

    // Subtraction
    if (__builtin_sub_overflow(INT_MIN, 1, &result)) {
        std::cout << "INT_MIN - 1: OVERFLOW detected\n";
    }

    // C++26 will have std::add_overflow, std::sub_overflow, std::mul_overflow
    // Until then: use the builtins or write wrappers (see #557)

    std::cout << "\nKey: these builtins check the CPU overflow flag\n";
    std::cout << "Cost: ~1 extra instruction (jno/jo) = nearly free\n";
}
// Output:
//   100 + 200 = 300
//   INT_MAX + 1: OVERFLOW detected (no UB!)
//   INT_MAX * 2 = 4294967294
//   INT_MIN - 1: OVERFLOW detected
```

Notice that the builtins store the wrapped (mathematically incorrect) result in the output parameter even when overflow occurs. This is intentional - you can inspect it if you want, but the whole point is that you now have a flag telling you to not trust it.

### Q2: Why signed overflow is UB and compiler exploits it

This is the deep end. The reason this trips people up is that the compiler is doing exactly what the standard says it can do - and what the standard allows is surprisingly aggressive.

**Why UB?** The C++ standard ([basic.fundamental]) says signed integers use a mathematical model where overflow has no result. This is intentional:

1. **Portability**: Different CPUs handle overflow differently (wrap, saturate, trap). UB allows all.
2. **Optimization**: If overflow "can't happen", the compiler can assume `x + 1 > x` always.

**How compilers exploit this:**

```cpp
#include <iostream>
#include <climits>

// Example 1: Loop optimization
void count_up(int start) {
    // Compiler assumes: i + 1 > i (always true if no overflow)
    // -> Loop MUST terminate (even if start = INT_MAX)
    // -> With UB: may become infinite loop!
    for (int i = start; i >= start; ++i) {
        // Compiler may optimize: "i starts >= start and always increases"
        // -> condition always true -> infinite loop removed or kept
    }
}

// Example 2: Dead code elimination
bool check_overflow(int x) {
    return (x + 1) < x;  // Compiler: "overflow can't happen"
    // Optimized to: return false;
    // Even when x == INT_MAX !
}

// Example 3: Range propagation
void process(int x) {
    if (x > 0) {
        int doubled = x * 2;  // Compiler assumes no overflow
        // -> Compiler deduces: doubled > 0 (always!)
        // -> Can optimize away checks on 'doubled'
        if (doubled < 0)  // "This can never be true" -> eliminated!
            std::cout << "Negative!\n";
    }
}

int main() {
    std::cout << "check_overflow(INT_MAX) = "
              << std::boolalpha << check_overflow(INT_MAX) << '\n';
    // May print: false (optimizer removed the check!)

    std::cout << "\nCompiler assumptions from no-overflow:\n";
    std::cout << "  x + 1 > x  (always true)\n";
    std::cout << "  x * 2 > 0  if x > 0 (always true)\n";
    std::cout << "  loop counter increases monotonically\n";
}
```

The `check_overflow` function is the most striking example. Its entire body is the overflow check - but because the compiler knows signed overflow is UB, it concludes that `x + 1 < x` can never be true, and compiles the function to unconditionally return `false`. This is not a compiler bug. This is the compiler faithfully implementing the C++ standard. The lesson: you cannot use arithmetic on signed integers to detect whether that arithmetic would overflow. You have to check before the operation, using the builtins or pre-condition math.

### Q3: UBSan for overflow detection

UBSan (Undefined Behavior Sanitizer) instruments every arithmetic operation at compile time to insert a runtime check. It does not prevent overflow - it detects it and reports it so you can fix the bug. The overhead is real (~20%), so this is a testing tool, not a production one.

```cpp
// Compile with: g++ -std=c++20 -fsanitize=signed-integer-overflow,undefined overflow.cpp
#include <iostream>
#include <climits>

int main() {
    // UBSan catches these at runtime:

    // 1. Signed addition overflow
    int a = INT_MAX;
    int b = a + 1;  // UBSan: "signed integer overflow: 2147483647 + 1"
    std::cout << "INT_MAX + 1 = " << b << '\n';

    // 2. Signed multiplication overflow
    int c = 1000000;
    int d = c * c;  // UBSan: "signed integer overflow: 1000000 * 1000000"
    std::cout << "1M * 1M = " << d << '\n';

    // 3. Signed negation overflow
    int e = INT_MIN;
    int f = -e;  // UBSan: "negation of -2147483648 cannot be represented"
    std::cout << "-INT_MIN = " << f << '\n';

    std::cout << "\nUBSan flags:\n";
    std::cout << "  -fsanitize=signed-integer-overflow   (signed ops)\n";
    std::cout << "  -fsanitize=unsigned-integer-overflow  (unsigned wraps)\n";
    std::cout << "  -fsanitize=integer                    (both + shifts)\n";
    std::cout << "  -fno-sanitize-recover=all             (abort on first)\n";
    std::cout << "\nCI/CD: always run tests with sanitizers!\n";
}
```

Note the negation case at the end: `-INT_MIN` overflows because `INT_MIN` is `-2147483648` and `INT_MAX` is only `2147483647` - there is no positive value of the same magnitude. This is one of the less obvious overflow cases that UBSan will catch for you.

---

## Notes

- Complementary to #557 (safe_add wrapper + builtin details).
- Unsigned overflow is NOT UB (wraps modulo 2^N), but `-fsanitize=unsigned-integer-overflow` is still useful.
- `-ftrapv`: traps on signed overflow (GCC/Clang). Simpler than UBSan but less info.
- C++26: `std::add_overflow`, `std::sub_overflow`, `std::mul_overflow` are standardized.
- In constexpr context, overflow is a compile error (the compiler evaluates and rejects it).
