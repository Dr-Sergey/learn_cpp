# Distinguish Implementation-Defined, Unspecified, and Undefined Behavior

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17, C++20, C++23  
**Reference:** [cppreference - Implementation-Defined Behavior](https://en.cppreference.com/w/cpp/language/implementation_defined_behavior)  

---

## Topic Overview

The C++ standard defines three categories of behavior that are not fully specified. Confusing them is a common source of portability bugs and security vulnerabilities. Understanding the precise guarantees (or lack thereof) for each category is essential for writing code that behaves correctly across compilers, platforms, and optimization levels.

| Category | Standard Requirement | Compiler Obligation | Portable? |
| --- | --- | --- | --- |
| **Implementation-defined** | Must choose a behavior and **document** it | Must be consistent and documented | Yes, if you query/check |
| **Unspecified** | Must choose from a set of valid behaviors | No obligation to document or be consistent | Partially - all outcomes are valid |
| **Undefined** | No requirements at all | None - may do literally anything | **No** |

**Implementation-defined behavior** means the compiler must pick a specific behavior from a set permitted by the standard and must document that choice. Examples: the size of `int`, the representation of negative integers (until C++20 mandated two's complement), the value of `sizeof(bool)`. You can write portable code by querying these values at compile time.

**Unspecified behavior** means the standard allows multiple valid outcomes, but the compiler need not document which it chooses, and different executions of the same code may yield different results. The program is still well-defined - no "nasal demons" - but you cannot rely on a specific outcome. Example: evaluation order of function arguments (pre-C++17).

**Undefined behavior** means the standard imposes no requirements. The compiler may assume it never happens and optimize accordingly. This is qualitatively different from the other two: UB is not "one of several valid outcomes" but rather "no valid outcome exists." The reason this trips people up is that UB often *looks* fine in testing - it may produce the result you expected, right up until you change the optimization level or compile on a different platform. The compiler is not required to warn you, crash your program, or do anything predictable. It may do literally anything, including silently generating code that does something entirely different from what you wrote.

The diagram below maps this out visually. Think of it as a spectrum from "fully specified" on the left to "anything goes" on the right:

```cpp
┌─────────────────────────────────────────────────────────────┐
│                  C++ Standard Behavior Spectrum              │
├──────────────┬──────────────────┬──────────────┬────────────┤
│  Well-Defined │ Impl-Defined     │ Unspecified  │ Undefined  │
│  (portable)  │ (documented)     │ (valid set)  │ (anything) │
│              │                  │              │            │
│  int x = 5;  │ sizeof(long)==8  │ f(a(),b())   │ *nullptr   │
│  x + 3 == 8  │ on this platform │ order of     │ signed     │
│              │ (documented)     │ a(), b()     │ overflow   │
│              │                  │ varies       │            │
└──────────────┴──────────────────┴──────────────┴────────────┘
```

---

## Self-Assessment

### Q1: Classify each behavior and explain the portability impact

Take your time with this example - it covers the full range of all three categories. Each annotated block is a distinct case worth understanding on its own.

```cpp
#include <climits>
#include <cstddef>
#include <cstdint>
#include <iostream>
#include <new>

// --- Example 1: sizeof(int) ---
// Implementation-defined: compiler must document this.
static_assert(sizeof(int) >= 2, "int must be at least 16 bits");

void classify_behaviors() {
    // --- Example 2: Signed integer representation ---
    // Before C++20: implementation-defined (could be 1's complement, sign-magnitude)
    // Since C++20: mandated two's complement - now well-defined
    int neg = -1;
    // Bit pattern: implementation-defined (pre-C++20), defined (C++20+)

    // --- Example 3: Right shift of negative value ---
    // Implementation-defined: arithmetic or logical shift
    int shifted = (-16) >> 2;
    // Could be -4 (arithmetic) or large positive (logical)
    // Compiler must document which.

    // --- Example 4: Evaluation order of function arguments ---
    // Unspecified behavior (in all C++ versions)
    auto f = [](int a, int b) { return a + b; };
    int i = 0;
    // int result = f(++i, ++i);  // Unspecified order + UB before C++17
    // C++17: each argument is sequenced, but order between them is unspecified

    // --- Example 5: Order of evaluation of operands ---
    // Pre-C++17: unspecified for most operators
    // C++17: left-to-right for <<, >>, assignment right-to-left, etc.
    std::cout << "A" << "B";  // C++17: guaranteed left-to-right

    // --- Example 6: Dereferencing nullptr ---
    // Undefined behavior - not implementation-defined, not unspecified.
    int* p = nullptr;
    // int val = *p;  // UB: the compiler owes you nothing

    // --- Example 7: Signed integer overflow ---
    int a = INT_MAX;
    // int b = a + 1;  // UB: not implementation-defined, not unspecified

    // --- Example 8: Conversion of out-of-range value to signed ---
    unsigned big = UINT_MAX;
    int s = static_cast<int>(big);
    // Implementation-defined (C++17): must produce a value or signal
    // C++20: well-defined - two's complement congruence

    std::cout << "shifted: " << shifted << "\n";
    std::cout << "s: " << s << "\n";
}

int main() {
    classify_behaviors();
}
```

The table below summarizes where each example lands and which part of the standard governs it:

**Answer Summary:**

| Example | Category | Standard Reference |
| --- | --- | --- |
| `sizeof(int)` | Implementation-defined | [basic.fundamental] |
| Right shift of negative | Implementation-defined | [expr.shift] |
| Function argument eval order | Unspecified | [expr.call] |
| Null dereference | **Undefined** | [expr.unary.op] |
| Signed overflow | **Undefined** | [expr.add] |
| Out-of-range -> signed | Impl-defined (C++17), defined (C++20) | [conv.integral] |

---

### Q2: Write a compile-time check that adapts to implementation-defined behavior

The right way to handle implementation-defined properties is to query them at compile time rather than hardcode an assumption. Here is a function that does exactly that across several platform properties:

```cpp
#include <bit>
#include <climits>
#include <cstdint>
#include <iostream>
#include <type_traits>

// Portable approach: query implementation-defined properties at compile time
// and adapt behavior accordingly.

template <typename T>
consteval auto bit_width_of() -> int {
    return sizeof(T) * CHAR_BIT;
}

// Since C++20 mandates two's complement, this is now well-defined:
consteval auto safe_negate(int value) -> int {
    if (value == INT_MIN) {
        // INT_MIN cannot be negated in two's complement without overflow.
        // In a consteval context, this would be a compile error if triggered.
        return INT_MAX;  // Saturate instead of UB
    }
    return -value;
}

// Portable endianness detection (C++20)
void detect_platform_properties() {
    std::cout << "int:        " << bit_width_of<int>() << " bits\n";
    std::cout << "long:       " << bit_width_of<long>() << " bits\n";
    std::cout << "long long:  " << bit_width_of<long long>() << " bits\n";
    std::cout << "pointer:    " << bit_width_of<void*>() << " bits\n";

    if constexpr (std::endian::native == std::endian::little) {
        std::cout << "Endianness: little-endian\n";
    } else if constexpr (std::endian::native == std::endian::big) {
        std::cout << "Endianness: big-endian\n";
    } else {
        std::cout << "Endianness: mixed\n";
    }

    // Right-shift behavior: detect at compile time
    constexpr int test = (-1) >> 1;
    if constexpr (test == -1) {
        std::cout << "Right shift: arithmetic (sign-extending)\n";
    } else {
        std::cout << "Right shift: logical (zero-filling)\n";
    }

    constexpr int neg_result = safe_negate(42);
    static_assert(neg_result == -42);
}

int main() {
    detect_platform_properties();
}
```

Notice how `if constexpr` branches on the endianness check, and `constexpr` evaluation is used for the shift test. This is the idiomatic way to write portable code that adapts to implementation-defined choices: let the compiler evaluate those choices at compile time and branch accordingly.

**Answer:** Querying implementation-defined properties via `sizeof`, `std::endian`, `constexpr` evaluation, and `<climits>` macros is the portable approach. Never assume specific values for implementation-defined behavior - always query.

---

### Q3: Demonstrate how unspecified behavior differs from UB in practice

This is the distinction that confuses most people. Unspecified behavior is not dangerous in the "your program is broken" sense - it just means you cannot predict which of several valid outcomes you will get. UB is categorically different.

```cpp
#include <functional>
#include <iostream>
#include <string>
#include <vector>

// Unspecified behavior: multiple valid outcomes, all well-defined.
// The program cannot crash or misbehave - it just may produce
// different results on different compilers.

std::string log_buffer;

std::string make_entry(const char* label, int value) {
    std::string entry = std::string(label) + "=" + std::to_string(value);
    log_buffer += entry + ";";
    return entry;
}

void unspecified_demo() {
    // Unspecified: order of argument evaluation
    auto concat = [](std::string a, std::string b, std::string c) {
        return a + " | " + b + " | " + c;
    };

    int counter = 0;
    // The order in which make_entry calls execute is unspecified.
    // All three calls will happen, but we don't know the order.
    std::string result = concat(
        make_entry("first", ++counter),
        make_entry("second", ++counter),
        make_entry("third", ++counter)
    );

    std::cout << "Result: " << result << "\n";
    std::cout << "Log:    " << log_buffer << "\n";
    // Different compilers may produce different log orderings.
    // GCC traditionally evaluates right-to-left,
    // MSVC traditionally evaluates left-to-right.
    // BOTH are correct. This is NOT UB.
}

// Contrast: this IS undefined behavior
void ub_demo() {
    int i = 0;
    // Pre-C++17:
    // i = i++ + ++i;  // UB: multiple unsequenced modifications

    // C++17 fixed assignment ordering, but the operands of +
    // are still indeterminately sequenced. With side-effects
    // on the same scalar, this remains UB.
}

// Practical rule:
// - Unspecified -> your program is valid, just non-deterministic
// - Undefined  -> your program is invalid, compiler can do anything

int main() {
    unspecified_demo();
    ub_demo();
}
```

The `concat` call above is a real-world scenario. The three `make_entry` calls all run and all have their results placed into the function - but the order they run in is the compiler's choice, and it may vary. Your program is still correct. The `i = i++ + ++i` case is a completely different story: there is no valid execution of that expression, and the compiler is free to generate whatever it wants.

**Answer:** Unspecified behavior produces one of several valid results - the program remains well-formed. UB means the program is broken. In the concat example, different argument evaluation orders produce different but all-valid outputs. In the `i = i++ + ++i` case, the program has no valid execution.

---

## Notes

- **Portability strategy:** Treat implementation-defined behavior as a compile-time query; treat unspecified behavior as non-determinism you must tolerate; treat UB as a hard error.
- C++20 eliminated several historical implementation-defined behaviors: two's complement is now mandatory, out-of-range unsigned->signed conversion is defined.
- The `-Wsequence-point` (GCC) and `-Wunsequenced` (Clang) warnings catch some unspecified/UB evaluation order issues.
- Static analyzers (Clang-Tidy `bugprone-*`, PVS-Studio) flag code that relies on unspecified evaluation order.
- When in doubt, introduce named temporaries to force sequencing and eliminate both unspecified behavior and UB.
