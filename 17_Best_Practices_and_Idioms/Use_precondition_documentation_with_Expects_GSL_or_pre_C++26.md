# Use precondition documentation with Expects (GSL) or [[pre]] (C++26)

**Category:** Best Practices and Idioms  
**Standard:** C++26  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#i-in-and-out-parameters>  

---

## Topic Overview

**Preconditions** are the rules a caller must obey before invoking a function. Documenting them explicitly - rather than silently assuming them or defensively checking them everywhere - shifts responsibility to the right place: the caller, not the implementation. A function that documents "n must be positive" is cleaner, faster, and easier to reason about than one that checks `n > 0` internally and returns an error code.

### Precondition Mechanisms

The table below shows what's available and what each option does for you. The key distinction is between mechanisms that actually check at runtime (catching bugs early) and those that only serve as optimizer hints (improving performance).

| Mechanism | Available | Runtime check | Optimizer hint |
| --- | --- | --- | --- |
| Comments | Always | No | No |
| `assert()` | Always | Debug only | No |
| `Expects()` (GSL) | With GSL | Debug only | No |
| `[[pre: expr]]` | C++26 | Configurable | Yes |
| `[[assume(expr)]]` | C++23 | Never | Yes |

---

## Self-Assessment

### Q1: Add `Expects(n > 0)` and show it fires in debug builds

`Expects` is a macro from the GSL (Guidelines Support Library) that behaves like a self-documenting `assert` - it fires in debug builds and disappears in release. The version below is a simplified implementation you can use without pulling in the full GSL. Notice the `Ensures` alias for postconditions: same mechanism, different semantics.

```cpp
#include <cassert>
#include <iostream>
#include <vector>

// GSL-style Expects macro (simplified)
#ifdef NDEBUG
    #define Expects(cond) ((void)0)
#else
    #define Expects(cond) \
        do { \
            if (!(cond)) { \
                std::cerr << "Precondition failed: " #cond \
                           << " at " << __FILE__ << ":" << __LINE__ << '\n'; \
                std::abort(); \
            } \
        } while(false)
#endif

#define Ensures(cond) Expects(cond)  // postcondition

double average(const std::vector<int>& data) {
    Expects(!data.empty());  // precondition: non-empty input
    double sum = 0;
    for (int x : data) sum += x;
    double result = sum / data.size();
    Ensures(result >= 0 || result < 0);  // postcondition: result is valid
    return result;
}

int factorial(int n) {
    Expects(n >= 0);   // precondition: non-negative
    Expects(n <= 20);  // precondition: fits in int
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    Ensures(result > 0);  // postcondition: positive result
    return result;
}

int main() {
    std::cout << "avg({1,2,3}) = " << average({1, 2, 3}) << '\n';
    std::cout << "5! = " << factorial(5) << '\n';

    // In debug build, this would fire:
    // factorial(-1);  // "Precondition failed: n >= 0"
    // average({});    // "Precondition failed: !data.empty()"
}
// Expected output:
// avg({1,2,3}) = 2
// 5! = 120
```

By placing `Expects` at the very top of the function, you're telling every reader (and every future maintainer) exactly what must be true before the function can do its job. That's the documentation aspect. In debug builds you also get an immediate, informative abort if the caller violates the contract, rather than a mysterious crash deep inside the function body.

### Q2: Compare `Expects` with `static_assert` for compile-time vs runtime preconditions

The reason both tools exist is that they operate in different worlds. `static_assert` runs during compilation, so it can only check things the compiler knows at compile time - types and constant expressions. `Expects`/`assert` runs at runtime, so it can check actual values that only exist when the program is executing.

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <cassert>

// Compile-time precondition: static_assert
template<typename T>
T safe_divide(T a, T b) {
    static_assert(std::is_arithmetic_v<T>,
                  "safe_divide requires arithmetic type");
    // This is checked at COMPILE TIME
    // Can't use Expects for type constraints

    // Runtime precondition: Expects/assert
    assert(b != 0 && "division by zero");
    // b's value is only known at RUNTIME
    // Can't use static_assert for runtime values

    return a / b;
}

constexpr int compile_time_fact(int n) {
    // In constexpr context, we can use both:
    // static_assert works when called at compile time with known args
    // assert/Expects works when called at runtime
    return (n <= 1) ? 1 : n * compile_time_fact(n - 1);
}

int main() {
    std::cout << safe_divide(10, 3) << '\n';       // OK: int arithmetic
    std::cout << safe_divide(10.0, 3.0) << '\n';   // OK: double arithmetic

    // safe_divide(std::string("a"), std::string("b"));  // static_assert FAILS

    constexpr int f5 = compile_time_fact(5);  // computed at compile time
    std::cout << "5! = " << f5 << '\n';
}
// Expected output:
// 3
// 3.33333
// 5! = 120
```

Use them together: `static_assert` for type-level contracts, `Expects`/`assert` for value-level contracts. They complement each other perfectly.

| Check type | `static_assert` | `Expects`/`assert` |
| --- | --- | --- |
| When | Compile time | Runtime (debug) |
| What it checks | Types, constexpr values | Runtime values |
| Failure mode | Compilation error | `abort()` in debug |
| Release build | Always active | Removed (`NDEBUG`) |

### Q3: Explain contract-based programming and the C++26 contracts proposal

**Contract-based programming** is a design philosophy: instead of sprinkling defensive checks throughout your implementation, you define clear obligations at function boundaries. The caller promises to satisfy the preconditions; the function promises to deliver the postconditions. If the caller's code is correct, the checks never fire. This is fundamentally different from defensive programming, where you check inputs because you distrust the caller.

C++26 is bringing standardized syntax for this pattern directly into the language:

```cpp
// C++26 contracts syntax (proposed):
int factorial(int n)
    pre(n >= 0)
    pre(n <= 20)
    post(r: r > 0)
{
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    return result;
}

// The contract attributes are:
// pre(expr)      - precondition (caller's responsibility)
// post(r: expr)  - postcondition (r is the return value)
// assert(expr)   - assertion (mid-function check)
```

The reason this is better than the macro approach is that the compiler understands the contracts natively. It can use them as optimizer hints (like `[[assume]]`), pass them to static analysis tools, and handle violations through a customizable handler rather than a hard-coded `abort()`.

**Three contract levels:**

| Level | Enforcement | Use case |
| --- | --- | --- |
| `default` | Check in debug | Normal preconditions |
| `audit` | Check only with extra flag | Expensive checks |
| `axiom` | Never checked | Documentation only |

**Benefits over `assert`/`Expects`:**

1. Standardized syntax (not a macro).
2. Compiler can use contracts as optimizer hints.
3. Contract violation handler is customizable without modifying the function.
4. Static analysis tools can reason about contracts across call boundaries.

---

## Notes

- GSL `Expects`/`Ensures` is available today via Microsoft's GSL library (`#include <gsl/gsl>`).
- Place `Expects` at the very beginning of the function and `Ensures` at the end - this mirrors the structure of a formal contract.
- Don't use `Expects` for user input validation - that's a different concern and deserves proper error handling with return values, not contract assertions.
- C++26 contracts are still being finalized. Use `assert()`/`Expects()` in the meantime, and the migration to native contract syntax should be straightforward when it arrives.
