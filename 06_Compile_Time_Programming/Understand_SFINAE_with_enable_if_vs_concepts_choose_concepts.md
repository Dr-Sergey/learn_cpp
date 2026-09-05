# Understand SFINAE with `enable_if` vs Concepts - Choose Concepts

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constraints>  

---

## Topic Overview

### SFINAE: The Legacy Approach

**SFINAE** (Substitution Failure Is Not An Error) is a C++98 mechanism where template argument substitution silently removes overloads from the candidate set instead of causing a hard error. The name captures the idea: when the compiler tries to substitute a type into a template and something goes wrong - a missing member, an invalid expression - that substitution *failure* is not treated as an error; the overload is just dropped and the compiler keeps looking. `std::enable_if` (C++11) exploits this deliberately:

```cpp
// C++11 SFINAE: enable only for integral types
template <typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
T add(T a, T b) { return a + b; }
```

This works, but the syntax is telling the compiler "I want this overload to disappear when `T` is not integral" in a very roundabout way. The reason this trips people up is that the mechanism is implicit - the `enable_if_t` trick only works because a failed substitution quietly removes the overload rather than failing loudly.

### Concepts: The Modern Replacement

C++20 **concepts** provide the same functionality with drastically cleaner syntax and better error messages:

```cpp
// C++20 concepts: clean and readable
template <std::integral T>
T add(T a, T b) { return a + b; }
```

Same behavior - `add` is only available for integral types - but now the intent is visible in the code.

### Comparison

The table below shows where SFINAE causes real pain in practice. The "error messages" and "composability" rows are the ones that will affect you day-to-day.

| Aspect | `enable_if` / SFINAE | Concepts (C++20) |
| --- | --- | --- |
| Readability | Poor - embedded in template params or return types | Clear - constraint is a named predicate |
| Error messages | Cryptic (long substitution failure chains) | Direct ("constraint not satisfied: ...") |
| Composability | Nested `enable_if` is unreadable | `&&`, `\|\|` work naturally |
| Overload resolution | No subsumption - can be ambiguous | Subsumption - more constrained wins |
| Code location | Template params, return type, or dummy params | Before function, after function, or in abbreviated syntax |

**Rule:** In new C++20 code, always prefer concepts over `enable_if` / SFINAE.

---

## Self-Assessment

### Q1: Convert a three-overload `enable_if` SFINAE set to a `requires`-clause equivalent

This example shows the same `to_string` overload set written three ways: the old SFINAE approach, the `requires`-clause approach, and the abbreviated template syntax. Read all three and notice how the constraint location and readability change.

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <concepts>

// ============================================================
// BEFORE (C++11 SFINAE with enable_if) - hard to read
// ============================================================

// Overload 1: integral types
template <typename T>
std::enable_if_t<std::is_integral_v<T>, std::string>
to_string_sfinae(T value) {
    return "int:" + std::to_string(value);
}

// Overload 2: floating point types
template <typename T>
std::enable_if_t<std::is_floating_point_v<T>, std::string>
to_string_sfinae(T value) {
    return "float:" + std::to_string(value);
}

// Overload 3: string-like types
template <typename T>
std::enable_if_t<std::is_convertible_v<T, std::string>, std::string>
to_string_sfinae(T value) {
    return "str:" + std::string(value);
}

// ============================================================
// AFTER (C++20 Concepts) - clean and readable
// ============================================================

// Overload 1: integral types
template <std::integral T>
std::string to_string_concepts(T value) {
    return "int:" + std::to_string(value);
}

// Overload 2: floating point types
template <std::floating_point T>
std::string to_string_concepts(T value) {
    return "float:" + std::to_string(value);
}

// Overload 3: string-like types
template <typename T>
    requires std::convertible_to<T, std::string>
std::string to_string_concepts(T value) {
    return "str:" + std::string(value);
}

// ============================================================
// Even cleaner: abbreviated function template syntax
// ============================================================

std::string to_string_short(std::integral auto value) {
    return "int:" + std::to_string(value);
}

std::string to_string_short(std::floating_point auto value) {
    return "float:" + std::to_string(value);
}

std::string to_string_short(std::convertible_to<std::string> auto value) {
    return "str:" + std::string(value);
}

int main() {
    std::cout << "=== SFINAE (enable_if) ===\n";
    std::cout << to_string_sfinae(42) << "\n";
    std::cout << to_string_sfinae(3.14) << "\n";
    std::cout << to_string_sfinae("hello") << "\n";

    std::cout << "\n=== Concepts ===\n";
    std::cout << to_string_concepts(42) << "\n";
    std::cout << to_string_concepts(3.14) << "\n";
    std::cout << to_string_concepts("hello") << "\n";

    std::cout << "\n=== Abbreviated ===\n";
    std::cout << to_string_short(42) << "\n";
    std::cout << to_string_short(3.14) << "\n";
    std::cout << to_string_short("hello") << "\n";

    return 0;
}
```

**Expected output:**

```text
=== SFINAE (enable_if) ===
int:42
float:3.140000
str:hello

=== Concepts ===
int:42
float:3.140000
str:hello

=== Abbreviated ===
int:42
float:3.140000
str:hello
```

### Q2: Show how concept error messages are cleaner than SFINAE substitution failure messages

The code comments in this example are deliberately written to show the actual GCC error output you'd see from each approach. Pay attention to the difference - the SFINAE message makes you decode a substitution failure chain, while the concept message tells you the exact unsatisfied constraint and the type that failed it.

```cpp
#include <iostream>
#include <concepts>
#include <type_traits>
#include <string>
#include <vector>

// ============================================================
// SFINAE version: terrible error messages
// ============================================================
template <typename T>
std::enable_if_t<
    std::is_integral_v<T> && std::is_signed_v<T>,
    T
>
abs_sfinae(T value) {
    return value < 0 ? -value : value;
}

// ============================================================
// Concept version: clear error messages
// ============================================================
template <typename T>
concept SignedInteger = std::integral<T> && std::is_signed_v<T>;

template <SignedInteger T>
T abs_concepts(T value) {
    return value < 0 ? -value : value;
}

int main() {
    // These work:
    std::cout << abs_sfinae(-42) << "\n";    // 42
    std::cout << abs_concepts(-42) << "\n";  // 42

    // === Error message comparison ===
    // Uncommenting these will show the difference:

    // SFINAE error (typical GCC output):
    //   abs_sfinae(std::string("hello"));
    // Error: no matching function for call to 'abs_sfinae(std::string)'
    // note: candidate: template<class T> enable_if_t<is_integral_v<T> && is_signed_v<T>, T> abs_sfinae(T)
    // note: template argument deduction/substitution failed:
    // note: enable_if_t<false, ...> does not define type
    //   ^^^ Cryptic! Why did it fail? Which part of the condition?

    // Concept error (typical GCC output):
    //   abs_concepts(std::string("hello"));
    // Error: no matching function for call to 'abs_concepts(std::string)'
    // note: constraints not satisfied
    // note: the required expression 'integral<T>' is not satisfied
    //   with T = std::string
    //   ^^^ Clear! "integral<T> is not satisfied with T = string"

    std::cout << "\n=== Error Message Comparison ===\n";
    std::cout << "SFINAE error:\n";
    std::cout << "  'enable_if_t<is_integral_v<T> && is_signed_v<T>, T>'\n";
    std::cout << "  'substitution failed: enable_if_t<false> has no type'\n";
    std::cout << "  -> WHY did it fail? Which condition? Unclear!\n";
    std::cout << "\nConcept error:\n";
    std::cout << "  'constraint not satisfied: SignedInteger<std::string>'\n";
    std::cout << "  'integral<std::string> is not satisfied'\n";
    std::cout << "  -> Exact reason shown: string is not integral\n";

    return 0;
}
```

**Expected output:**

```text
42
42

=== Error Message Comparison ===
SFINAE error:
  'enable_if_t<is_integral_v<T> && is_signed_v<T>, T>'
  'substitution failed: enable_if_t<false> has no type'
  -> WHY did it fail? Which condition? Unclear!

Concept error:
  'constraint not satisfied: SignedInteger<std::string>'
  'integral<std::string> is not satisfied'
  -> Exact reason shown: string is not integral
```

### Q3: Benchmark compile time: SFINAE-heavy header vs concepts-based equivalent

Beyond readability and error messages, concepts can also improve compile times. The reason is that SFINAE forces the compiler to attempt substitution for every overload candidate on every call, whereas named concepts are evaluated as atomic constraints and can be cached. The code below shows both approaches producing identical output, followed by an explanation of the compile-time impact.

```cpp
#include <iostream>
#include <type_traits>
#include <concepts>
#include <chrono>
#include <string>

// ============================================================
// SFINAE-heavy: multiple nested enable_if checks
// ============================================================
template <typename T>
std::enable_if_t<std::is_integral_v<T> && !std::is_same_v<T, bool>, T>
process_sfinae(T v) { return v * 2; }

template <typename T>
std::enable_if_t<std::is_floating_point_v<T>, T>
process_sfinae(T v) { return v * 2.0; }

template <typename T>
std::enable_if_t<std::is_same_v<T, bool>, int>
process_sfinae(T v) { return v ? 1 : 0; }

template <typename T>
std::enable_if_t<std::is_convertible_v<T, std::string> && !std::is_arithmetic_v<T>, std::string>
process_sfinae(T v) { return std::string(v) + std::string(v); }

// ============================================================
// Concepts-based: cleaner, potentially faster to compile
// ============================================================
template <typename T>
concept IntegralNotBool = std::integral<T> && !std::same_as<T, bool>;

template <IntegralNotBool T>
T process_concepts(T v) { return v * 2; }

template <std::floating_point T>
T process_concepts(T v) { return v * 2.0; }

int process_concepts(bool v) { return v ? 1 : 0; }

template <typename T>
    requires std::convertible_to<T, std::string> && (!std::is_arithmetic_v<T>)
std::string process_concepts(T v) { return std::string(v) + std::string(v); }

int main() {
    // Both produce the same results
    std::cout << "=== SFINAE ===\n";
    std::cout << "int:    " << process_sfinae(21) << "\n";      // 42
    std::cout << "double: " << process_sfinae(1.5) << "\n";     // 3.0
    std::cout << "bool:   " << process_sfinae(true) << "\n";    // 1
    std::cout << "string: " << process_sfinae("ab") << "\n";    // abab

    std::cout << "\n=== Concepts ===\n";
    std::cout << "int:    " << process_concepts(21) << "\n";
    std::cout << "double: " << process_concepts(1.5) << "\n";
    std::cout << "bool:   " << process_concepts(true) << "\n";
    std::cout << "string: " << process_concepts("ab") << "\n";

    std::cout << "\n=== Compile-Time Impact ===\n";
    std::cout << "SFINAE:\n";
    std::cout << "  - Each call: compiler tries ALL overloads\n";
    std::cout << "  - enable_if: instantiates is_integral, is_floating_point, etc.\n";
    std::cout << "  - Substitution failure generates candidate, then discards\n";
    std::cout << "  - N overloads x M traits = O(N x M) template instantiations\n";
    std::cout << "\nConcepts:\n";
    std::cout << "  - Concepts are evaluated as atomic constraints\n";
    std::cout << "  - Subsumption prunes non-matching candidates early\n";
    std::cout << "  - Named concepts are cached by the compiler\n";
    std::cout << "  - Typical improvement: 10-40% faster compile times\n";
    std::cout << "\nBenchmark method:\n";
    std::cout << "  time g++ -std=c++20 -c sfinae_heavy.cpp\n";
    std::cout << "  time g++ -std=c++20 -c concepts_equiv.cpp\n";
    std::cout << "  Or use -ftime-report for detailed breakdown\n";

    return 0;
}
```

**Expected output:**

```text
=== SFINAE ===
int:    42
double: 3
bool:   1
string: abab

=== Concepts ===
int:    42
double: 3
bool:   1
string: abab

=== Compile-Time Impact ===
SFINAE:

  - Each call: compiler tries ALL overloads
  - enable_if: instantiates is_integral, is_floating_point, etc.
  - Substitution failure generates candidate, then discards
  - N overloads x M traits = O(N x M) template instantiations

Concepts:

  - Concepts are evaluated as atomic constraints
  - Subsumption prunes non-matching candidates early
  - Named concepts are cached by the compiler
  - Typical improvement: 10-40% faster compile times

Benchmark method:
  time g++ -std=c++20 -c sfinae_heavy.cpp
  time g++ -std=c++20 -c concepts_equiv.cpp
  Or use -ftime-report for detailed breakdown
```

---

## Notes

- In C++20 and later, **always prefer concepts** over `enable_if` / SFINAE.
- Concepts provide **subsumption**: the compiler automatically selects the most constrained overload.
- SFINAE is still needed for pre-C++20 code and edge cases (e.g., expression SFINAE in specific contexts).
- Concept error messages directly name the unsatisfied constraint - no more deciphering substitution failures.
- Named concepts are compiled once and cached - `enable_if` re-evaluates traits each time.
- Combining concepts: `Integral<T> && Signed<T>` is clean; compare to nested `enable_if_t<conjunction_v<...>>`.
