# Use `std::unreachable()` (C++23) to Mark Impossible Code Paths

**Category:** Standard Library New Cpp23 26  
**Standard:** C++23  
**Reference:** [cppreference — std::unreachable](https://en.cppreference.com/w/cpp/utility/unreachable)  

---

## Topic Overview

`std::unreachable()`, defined in `<utility>`, informs the compiler that a particular point in the code is never reached during valid execution. If control flow does reach it, the behavior is undefined. This gives the optimizer license to assume the path is dead, enabling elimination of redundant checks, better branch prediction hints, and tighter code generation.

Before C++23, developers relied on compiler-specific intrinsics: `__builtin_unreachable()` (GCC/Clang) and `__assume(false)` (MSVC). `std::unreachable()` standardizes this pattern, making portable code cleaner and eliminating `#ifdef` blocks.

| Mechanism | Portable | Standard | UB if reached | Optimizer hint |
| --------- | -------- | -------- | ------------- | -------------- |
| `__builtin_unreachable()` | GCC/Clang only | No | Yes | Yes |
| `__assume(false)` | MSVC only | No | Yes | Yes |
| `std::unreachable()` | Yes | C++23 | Yes | Yes |
| `assert(false)` | Yes | Yes | No (abort) | No |
| `std::abort()` | Yes | Yes | No (abort) | No |

**Critical safety rule:** `std::unreachable()` is not a debugging tool - it is a *performance* tool for provably dead code. If there is any possibility the path is reachable, use `assert(false)` or `std::abort()` instead. In debug builds, consider guarding with an assertion before the unreachable hint.

Here's the classic example - an exhaustive switch where the compiler can eliminate the default branch entirely once it knows the other cases cover everything:

```cpp
switch (color) {
    case Red:   ...
    case Green: ...
    case Blue:  ...
    default:    std::unreachable();   // enum is exhaustive
}           ^
            |  Compiler may now omit the default branch entirely,
               eliminate bounds checks, and assume only 3 cases exist.
```

---

## Self-Assessment

### Q1: How does `std::unreachable()` eliminate generated code in an exhaustive switch

The way the optimizer uses this hint is concrete: without `unreachable()`, the compiler must generate a fallback path in case `d` holds an out-of-range value. With it, the compiler knows only four cases are possible and can produce a simple jump table with no defensive branch. Notice that `unreachable()` goes *after* the switch, not inside `default:` - this preserves `-Wswitch` warnings if you add a new enumerator without a case:

```cpp
#include <utility>
#include <cstdint>
#include <string_view>

enum class Direction : uint8_t { North, South, East, West };

// Without unreachable: compiler generates a fallback path for default
// With unreachable: compiler knows all cases are covered -> tighter codegen
constexpr std::string_view to_string(Direction d) {
    switch (d) {
        case Direction::North: return "North";
        case Direction::South: return "South";
        case Direction::East:  return "East";
        case Direction::West:  return "West";
    }
    std::unreachable();  // provably dead if Direction has only 4 enumerators
}

// Compiler explorer comparison (x86-64, -O2):
// Without unreachable:  generates a "mov eax, 0" fallback + conditional jumps
// With unreachable:     generates a simple jump table, no fallback

static_assert(to_string(Direction::North) == "North");
static_assert(to_string(Direction::West)  == "West");

int main() {
    // All four calls resolve at compile time with constexpr
    return to_string(Direction::East).size();  // returns 4
}
```

The placement outside the switch body rather than in `default:` is the key ergonomic detail. Putting it in `default: std::unreachable()` would suppress the compiler warning when a new enumerator is added. Placing it after the switch keeps that warning alive, so you get the optimization hint *and* the safety net.

---

### Q2: How do you safely combine `std::unreachable()` with debug assertions

The right pattern for production code is a wrapper that gives you a hard assertion in debug builds and the optimizer hint in release builds. `std::source_location` (C++20) makes the debug message maximally useful - you get the exact file, line, and function where the impossible path was actually reached:

```cpp
#include <utility>
#include <cassert>
#include <iostream>
#include <source_location>

// Production: unreachable hint. Debug: hard assertion with location info.
[[noreturn]] inline void unreachable_safe(
    std::source_location loc = std::source_location::current())
{
#ifndef NDEBUG
    std::cerr << "FATAL: reached unreachable code at "
              << loc.file_name() << ':' << loc.line()
              << " in " << loc.function_name() << '\n';
    std::abort();
#else
    std::unreachable();
#endif
}

enum class Op : uint8_t { Add, Sub, Mul, Div };

double compute(double a, double b, Op op) {
    switch (op) {
        case Op::Add: return a + b;
        case Op::Sub: return a - b;
        case Op::Mul: return a * b;
        case Op::Div: return a / b;
    }
    unreachable_safe();  // debug: aborts with location; release: UB hint
}

// Modular usage in constexpr context
consteval double compute_checked(double a, double b, Op op) {
    switch (op) {
        case Op::Add: return a + b;
        case Op::Sub: return a - b;
        case Op::Mul: return a * b;
        case Op::Div: return a / b;
    }
    // In consteval: reaching std::unreachable() is a compile error
    // because UB is not allowed in constant evaluation
    std::unreachable();
}

static_assert(compute_checked(3.0, 4.0, Op::Add) == 7.0);

int main() {
    std::cout << compute(10, 3, Op::Mul) << '\n';  // 30
}
```

The `consteval` case is a particularly nice safety property: in constant evaluation, undefined behavior is not allowed, so the compiler will issue a compile-time error if `std::unreachable()` is ever actually reached. You get free correctness checking at compile time when the function is used in a constant context.

---

### Q3: What are valid vs. dangerous uses of `std::unreachable()`? Show an optimization benchmark pattern

The reason this trips people up is that `std::unreachable()` looks like it might be a useful error-handling tool - "if we get here something went wrong, so mark it unreachable." That thinking is backwards and dangerous. A user passing invalid input *can* reach that path, and the result is silent UB, not a helpful crash. The rule is: only use it when the *logic of the program* (not user behavior) makes the path impossible:

```cpp
#include <utility>
#include <cstdint>
#include <vector>
#include <numeric>
#include <iostream>

// VALID USE: value known to be in range
// After validation at system boundary, hint the optimizer
uint32_t fast_lookup(const uint32_t* table, uint32_t index) {
    // Caller guarantees index < 256 (validated at API boundary)
    if (index >= 256) std::unreachable();
    return table[index];  // compiler omits bounds check
}

// VALID USE: division where denominator is provably nonzero
int fast_div(int num, int denom) {
    if (denom == 0) std::unreachable();
    return num / denom;  // compiler omits zero-check on some architectures
}

// DANGEROUS: DO NOT use unreachable as error handling
// int parse_int(std::string_view s) {
//     auto result = try_parse(s);
//     if (!result) std::unreachable();  // BUG: user input can be invalid!
//     return *result;
// }

// VALID USE: eliminate impossible modular arithmetic outcomes
int classify_mod3(int value) {
    switch (value % 3) {
        case 0: return 0;
        case 1: return 1;
        case 2: return 2;
        // Mathematically, x%3 can only be 0,1,2 for non-negative x
        // For negative x, implementation-defined but still in {-2,-1,0,1,2}
    }
    std::unreachable();
}

// Portable polyfill for pre-C++23
#if __cplusplus < 202302L
namespace std {
    [[noreturn]] inline void unreachable() {
#if defined(__GNUC__) || defined(__clang__)
        __builtin_unreachable();
#elif defined(_MSC_VER)
        __assume(false);
#endif
    }
}
#endif

int main() {
    uint32_t table[256];
    std::iota(std::begin(table), std::end(table), 0);

    std::cout << fast_lookup(table, 42) << '\n';   // 42
    std::cout << fast_div(100, 7) << '\n';          // 14
    std::cout << classify_mod3(17) << '\n';         // 2
}
```

The commented-out dangerous example is the most important part of this question. The distinction between "validated at API boundary" (safe) and "comes from user input" (not safe) is what separates a valid use of `std::unreachable()` from a latent UB bug waiting to surface.

---

## Notes

- `std::unreachable()` is in `<utility>`, declared `[[noreturn]]`.
- Reaching it is **undefined behavior** - treat it as a performance tool for provably dead paths, not as error handling.
- Place it after a switch (not inside `default:`) to preserve `-Wswitch` / `-Wswitch-enum` warnings when new enumerators are added.
- In `constexpr` evaluation, reaching `std::unreachable()` is a hard compile error - UB is diagnosed at compile time, which is a useful free safety check.
- Combine with `assert(false)` in debug builds via a wrapper like `unreachable_safe()` so you get actionable diagnostics during development.
- The optimizer benefits most in tight loops and lookup tables where eliminating a single branch matters.
- Feature-test macro: `__cpp_lib_unreachable >= 202202L`.
- Compilers: GCC 12+, Clang 15+, MSVC 19.32+ (VS 2022 17.2).
