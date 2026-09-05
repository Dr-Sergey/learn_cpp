# Understand Substitution Failure vs Hard Error in SFINAE

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/sfinae>  

---

## Topic Overview

### The Critical Distinction

Not all invalid code in templates triggers SFINAE. This is probably the most common source of confusion when people first start writing SFINAE-heavy code: they expect a compile error to be silently swallowed, but instead the build breaks. The compiler distinguishes sharply between two situations:

| Situation | Result | Where |
| --- | --- | --- |
| **Substitution failure** | Overload silently removed (SFINAE) | **Immediate context** only |
| **Hard error** | Compilation aborts | **Non-immediate context** (inside function body, nested templates, etc.) |

### What Is "Immediate Context"

The **immediate context** includes:

- Function return type
- Function parameter types
- Template parameter default arguments
- `requires` expressions (C++20)
- `decltype` in the above contexts

Errors **outside** immediate context (inside the function body, inside instantiated nested templates, inside `static_assert`) are **hard errors**.

The mental model is this: the immediate context is everything the compiler checks before committing to instantiate the function body. Think of it as the "surface" of the function - the parts visible in the declaration. Once the compiler steps inside and starts generating code, it is committed, and any error there is fatal:

```cpp
┌─── Immediate context (SFINAE-friendly) ───┐
│ template <typename T>                      │
│ auto f(T x) -> decltype(x.size())         │<- failure here = SFINAE
│ {                                          │
│ ┌─ Non-immediate context (hard error) ──┐ │
│ │   return x.size();                     │ │<- failure here = HARD ERROR
│ │   static_assert(sizeof(T) > 8);       │ │<- always hard error
│ └────────────────────────────────────────┘ │
│ }                                          │
└────────────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: Show that a syntax error in a non-immediate context is a hard error, not SFINAE

The reason this trips people up is that the error looks like it should be SFINAE-able. After all, `inner_check<T>::type` is used in what looks like a return type position - which is immediate context. But the error does not happen there: it happens inside `inner_check<T>` itself, during instantiation of that helper. That is non-immediate context, and the `static_assert` fires as a hard error:

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// Helper that generates a hard error inside the function body
template <typename T>
struct inner_check {
    // This static_assert is in a NON-immediate context
    // If it fails, it's a HARD ERROR - not SFINAE
    static_assert(std::is_integral_v<T>, "T must be integral!");
    using type = T;
};

// This function uses inner_check<T>::type
// The error from static_assert is NOT in the immediate context
// -> it's a HARD ERROR, not SFINAE
template <typename T>
typename inner_check<T>::type process(T val) {
    return val * 2;
}

// Fallback overload
std::string process(const std::string& s) {
    return s + s;
}

// If we call process(std::string("hi")):
//   1. Compiler tries process<string>(string) -> inner_check<string>::type
//   2. Inside inner_check<string>, static_assert fails -> HARD ERROR
//   3. The program does NOT fall through to the string overload
//   4. Instead, compilation ABORTS

// === FIXED VERSION: Move the check to immediate context ===

template <typename T>
auto process_fixed(T val) -> std::enable_if_t<std::is_integral_v<T>, T> {
    // enable_if is in IMMEDIATE context -> SFINAE-friendly
    return val * 2;
}

std::string process_fixed(const std::string& s) {
    return s + s;
}

int main() {
    std::cout << "=== Hard error vs SFINAE ===\n\n";

    // This works (int is integral):
    std::cout << "process(21):           " << process(21) << "\n";  // 42

    // This would be a HARD ERROR (uncomment to see):
    // std::cout << process(std::string("hi")) << "\n";
    // Error: static_assert failed: T must be integral!
    // (NOT SFINAE - compilation aborts)

    std::cout << "\n=== Fixed version (SFINAE-friendly) ===\n";
    std::cout << "process_fixed(21):     " << process_fixed(21) << "\n";           // 42
    std::cout << "process_fixed(\"hi\"): " << process_fixed(std::string("hi")) << "\n"; // hihi
    // Works! enable_if failure is SFINAE -> falls through to string overload

    return 0;
}
```

The fix is always the same: move the condition into the immediate context - either the return type, a parameter type, or a template parameter default - instead of hiding it inside a helper template's body.

### Q2: Demonstrate that accessing a non-existent member in a `decltype`/`sizeof` is SFINAE-friendly

Using `decltype` in the trailing return type is the most common way to make a check SFINAE-friendly. The compiler evaluates the `decltype` expression as part of the function signature, which is immediate context, so a failure there becomes a substitution failure rather than a hard error:

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// === decltype in IMMEDIATE context -> SFINAE-friendly ===

// Trailing return type with decltype - in immediate context
template <typename T>
auto get_size(const T& c) -> decltype(c.size()) {
    // If T has no .size(), decltype(c.size()) is invalid
    // This is in IMMEDIATE context -> SFINAE (overload removed)
    return c.size();
}

// Fallback
template <typename T>
int get_size(const T&) {
    return -1;  // No size available
}

// === sizeof in immediate context -> also SFINAE-friendly ===

// Check if T has a .data() member via decltype in enable_if
template <typename T>
auto has_data_check(const T& c)
    -> decltype(c.data(), bool{})  // Comma operator: check c.data(), return bool
{
    std::cout << "  has .data(): pointer = " << static_cast<const void*>(c.data()) << "\n";
    return true;
}

template <typename T>
bool has_data_check(const T&) {
    std::cout << "  no .data()\n";
    return false;
}

// === Multiple expressions in decltype (all must be valid) ===
template <typename T>
auto full_container_check(const T& c)
    -> decltype(c.size(), c.begin(), c.end(), c.empty(), std::string{})
{
    return "full container interface";
}

template <typename T>
std::string full_container_check(const T&) {
    return "not a full container";
}

int main() {
    std::cout << "=== decltype SFINAE (immediate context) ===\n\n";

    std::vector<int> v{1, 2, 3};
    int x = 42;
    std::string s = "hello";

    std::cout << "get_size(vector):  " << get_size(v) << "\n";   // 3
    std::cout << "get_size(int):     " << get_size(x) << "\n";   // -1 (no .size())
    std::cout << "get_size(string):  " << get_size(s) << "\n";   // 5

    std::cout << "\nhas_data_check:\n";
    std::cout << "vector: "; has_data_check(v);      // has .data()
    std::cout << "string: "; has_data_check(s);      // has .data()
    std::cout << "int:    "; has_data_check(x);      // no .data()

    std::cout << "\nfull_container_check:\n";
    std::cout << "vector: " << full_container_check(v) << "\n";  // full container
    std::cout << "int:    " << full_container_check(x) << "\n";  // not a full container

    return 0;
}
```

The comma-operator trick in `decltype(c.data(), bool{})` is worth noting: the comma operator evaluates both operands, so the `decltype` checks that `c.data()` is valid and then yields the type of `bool{}` as the actual return type.

### Q3: Explain the immediate context rule and why it matters for SFINAE robustness

This example demonstrates the dangerous pattern more explicitly: a function whose return type uses `enable_if`, which looks SFINAE-safe, but which internally triggers a helper template that fires a `static_assert`. The `enable_if` is in the immediate context, but the `Validator<T>` instantiation is not - so the hard error still fires:

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === The Immediate Context Rule ===
//
// SFINAE only applies to errors in the "immediate context" of:
//   1. Template parameter substitution
//   2. Return type deduction
//   3. Function parameter types
//   4. requires expressions (C++20)
//
// Errors OUTSIDE immediate context are HARD ERRORS:
//   1. Inside function body
//   2. Inside instantiated helper templates
//   3. static_assert inside templates
//   4. Default member initializers

// EXAMPLE: Hard error from nested template instantiation
template <typename T>
struct Validator {
    // This is NOT in the immediate context of any calling function
    static_assert(!std::is_void_v<T>, "Cannot validate void!");
    static constexpr bool valid = true;
};

// DANGEROUS: uses Validator<T> in a context that seems SFINAE-friendly
template <typename T>
auto validate(T val) -> std::enable_if_t<Validator<T>::valid, bool> {
    return true;
}
// If T=void, Validator<void> triggers static_assert -> HARD ERROR
// Even though enable_if is in immediate context, Validator<void>
// is instantiated in NON-immediate context

// === How to make it SFINAE-safe ===

// Option 1: Move ALL checks to immediate context
template <typename T>
auto validate_safe(T val)
    -> std::enable_if_t<!std::is_void_v<T> && std::is_arithmetic_v<T>, bool> {
    return true;
}

// Option 2: Use C++20 requires (everything inside is immediate context)
template <typename T>
    requires (!std::is_void_v<T>) && std::is_arithmetic_v<T>
bool validate_concept(T val) {
    return true;
}

// Fallback
bool validate_safe(...) { return false; }
bool validate_concept(...) { return false; }

int main() {
    std::cout << "=== Immediate Context Rule ===\n\n";

    std::cout << "Rule: SFINAE only catches errors in IMMEDIATE context.\n";
    std::cout << "Errors in nested templates, function bodies, or static_asserts\n";
    std::cout << "are HARD ERRORS that abort compilation.\n\n";

    std::cout << "Immediate context includes:\n";
    std::cout << "  - Return types:  auto f(T) -> decltype(expr)\n";
    std::cout << "  - Parameters:    void f(enable_if_t<cond, T>)\n";
    std::cout << "  - Template args: template<class T, enable_if_t<cond, int> = 0>\n";
    std::cout << "  - requires:      requires { expr; }  (C++20)\n\n";

    std::cout << "NON-immediate context (hard errors):\n";
    std::cout << "  - Function body: { static_assert(...); }\n";
    std::cout << "  - Nested templates: inner<T> where inner has static_assert\n";
    std::cout << "  - Default member initializers\n\n";

    std::cout << "Safe validate(42): " << validate_safe(42) << "\n";     // true
    std::cout << "Safe validate(\"hi\"): " << validate_safe("hi") << "\n";  // false (not arithmetic)

    std::cout << "\nBest practice:\n";
    std::cout << "  1. Keep ALL SFINAE checks in the immediate context\n";
    std::cout << "  2. Never rely on static_assert inside helper templates for SFINAE\n";
    std::cout << "  3. Prefer C++20 requires expressions (always immediate context)\n";

    return 0;
}
```

The C++20 `requires` clause is the cleanest solution precisely because everything inside a `requires` expression is always treated as immediate context - there is no risk of accidentally triggering a hard error through a nested instantiation.

---

## Notes

- **Substitution failure** (SFINAE) only works in the **immediate context** - return type, parameter types, template args, requires.
- Errors inside function bodies, nested template instantiations, or `static_assert` are **hard errors** that abort compilation.
- Classic trap: using a helper trait that has `static_assert` inside - the assert fires as a hard error, not SFINAE.
- C++20 `requires` expressions make everything inside them part of the immediate context - much safer.
- Rule of thumb: keep all conditions checked via `enable_if`/`decltype`/`requires` at the function signature level.
