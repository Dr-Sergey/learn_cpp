# Know constexpr exception support status and workarounds

**Category:** Compile Time Programming  
**Standard:** C++20, C++26  
**Reference:** <https://wg21.link/P3068>  

---

## Topic Overview

Exceptions are not allowed in constant expressions (C++20/23). This creates friction when constexpr functions need error handling - you can't just `throw` and `catch` at compile time the way you would at runtime.

The reason this limitation exists is fundamental: exception unwinding relies on runtime stack frame information (RTTI, stack unwinding tables) that simply doesn't exist during the compile-time interpreter. The C++ standard therefore disallows `try`/`catch` in constant expressions.

### Current Limitations

Here's the key thing to understand: `throw` in a constexpr function is not a compile error in itself. What matters is *when* that throw is hit. If it's reached during constant evaluation, the evaluation is deemed non-constant and the compiler produces an error (with the thrown value appearing in the diagnostic). At runtime, the throw behaves normally.

```cpp
constexpr int divide(int a, int b) {
    if (b == 0) {
        throw std::runtime_error("divide by zero");
        // OK at compile time: throw makes the call non-constexpr
        // The throw is evaluated -> compilation error with clear message
        // At runtime: throws normally
    }
    return a / b;
}

constexpr int a = divide(10, 2);   // OK: 5
// constexpr int b = divide(10, 0); // Compile error: "divide by zero"
int c = divide(10, 0);             // Runtime: throws exception
```

This "throw as a compile-time assertion" trick is actually quite useful in practice - the thrown string literal shows up in the compiler error output.

### Workarounds for Constexpr Error Handling

When you need to handle errors gracefully in both contexts, you have three main patterns to choose from:

```cpp
#include <optional>
#include <stdexcept>
#include <type_traits>

// Pattern 1: Return optional
constexpr std::optional<int> safe_divide(int a, int b) {
    if (b == 0) return std::nullopt;
    return a / b;
}

// Pattern 2: Compile-time/runtime split
constexpr int checked_divide(int a, int b) {
    if (b == 0) {
        if (std::is_constant_evaluated()) {
            // Can't throw at compile time - use a trick:
            // Calling a non-constexpr function makes this non-constant
            throw "divide by zero";  // Forces compile error
        } else {
            throw std::runtime_error("divide by zero");
        }
    }
    return a / b;
}

// Pattern 3: assert-like with immediate function
consteval int must_divide(int a, int b) {
    if (b == 0) throw "compile-time division by zero";
    return a / b;
}
```

Pattern 1 (returning `optional`) is the safest because it works in both contexts without any branching tricks. Pattern 2 lets you give different error types to the compile-time path and the runtime path. Pattern 3 (`consteval`) is appropriate when the function is *only* ever called at compile time - there's no runtime fallback at all.

---

## Self-Assessment

### Q1: Why can't you catch exceptions at compile time

The compiler evaluates constant expressions as an interpreter. Exception unwinding requires runtime stack frame information that doesn't exist during compilation. The C++ standard simply disallows `try`/`catch` in constant expressions.

### Q2: What happens if a constexpr function throws during constant evaluation

The throw expression makes the function no longer a valid constant expression. If the result must be constexpr (`constexpr int x = f();`), this is a compile error. The thrown value appears in the error message, making it a useful diagnostic technique.

### Q3: What is the C++26 direction for constexpr exceptions

P3068 proposes allowing try/catch in constant expressions. The behavior would be the same as runtime: throw, unwind, catch. This would enable `constexpr std::vector` and `constexpr std::string` to use exceptions for their full API at compile time.

---

## Notes

- Use `throw` as a compile-time assertion: `constexpr int x = might_throw();` - the thrown message appears in the error.
- `std::is_constant_evaluated()` (C++20) or `if consteval` (C++23) to branch.
- C++26 may allow full exception handling at compile time.
- `consteval` functions that always throw are compile errors - by design.
