# Use [[assume(expr)]] (C++23) to communicate assumptions to the optimizer

**Category:** Best Practices and Idioms  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/assume>  

---

## Topic Overview

`[[assume(expr)]]` is a way to hand the compiler a signed guarantee: "I promise this expression is always true at this point in the program - now use that fact to generate better code." The compiler never evaluates the expression at runtime; it simply treats it as axiomatic. If the assumption turns out to be wrong at runtime, the behavior is **undefined** - the compiler may have generated code that assumes the impossible case can never happen, and all bets are off.

### How It Works

The idea is straightforward: you know something about your data that the compiler cannot infer from the code alone. Rather than leaving the compiler to guess conservatively, you tell it directly.

```cpp
void process(int* data, int n) {
    [[assume(n > 0)]];           // promise: n is always positive
    [[assume(n % 4 == 0)]];      // promise: n is always a multiple of 4
    for (int i = 0; i < n; ++i)
        data[i] *= 2;
}
// Compiler can:
// 1. Skip the n <= 0 check (loop always executes)
// 2. Unroll by 4 without remainder loop
// 3. Vectorize more aggressively
```

With those two assumptions in place, the compiler no longer needs a guard for the empty-input case and can emit a clean SIMD loop without a scalar cleanup tail. Two lines of code, measurably better output.

---

## Self-Assessment

### Q1: Add `[[assume(n > 0)]]` before a loop and inspect the effect

The most direct way to understand what `[[assume]]` buys you is to look at the assembly. Without the annotation, the compiler must defensively handle `n == 0`; with it, that check disappears. The alignment assumption on top of that opens the door to auto-vectorization with no remainder path.

```cpp
#include <cstddef>
#include <iostream>

// Without assume: compiler must handle n == 0 case
int sum_without_assume(const int* data, int n) {
    int total = 0;
    for (int i = 0; i < n; ++i)  // generates: test n; jle skip_loop
        total += data[i];
    return total;
}

// With assume: compiler KNOWS n > 0
int sum_with_assume(const int* data, int n) {
    [[assume(n > 0)]];           // UB if n <= 0!
    int total = 0;
    for (int i = 0; i < n; ++i)  // no zero-check needed
        total += data[i];
    return total;
}

// With alignment assume: enables vectorization
void multiply(float* data, int n) {
    [[assume(n > 0)]];
    [[assume(n % 8 == 0)]];                         // n is multiple of 8
    [[assume(reinterpret_cast<uintptr_t>(data) % 32 == 0)]];  // 32-byte aligned
    for (int i = 0; i < n; ++i)
        data[i] *= 2.0f;
    // Compiler generates fully unrolled AVX code, no scalar cleanup
}

int main() {
    int data[] = {1, 2, 3, 4, 5};
    std::cout << "Sum: " << sum_with_assume(data, 5) << '\n';
}
// Expected output:
// Sum: 15
//
// Assembly comparison (godbolt.org with -O2 -std=c++23):
// without assume: includes "test edi, edi; jle .return_zero"
// with assume:    no zero-check, jumps straight to loop
```

Notice that the runtime output is identical - the difference only shows up in the generated assembly. Paste both versions into Compiler Explorer with `-O2 -std=c++23` and compare the instruction counts.

### Q2: Explain the UB semantics of `[[assume]]` and security implications

This is the part that trips people up. The expression inside `[[assume]]` is a contract, not a check. The compiler eliminates any code paths that would only be reached if the assumption were false - so if the assumption is violated, those paths are gone and there is nothing to fall back on. The result can range from a wrong answer to a security hole.

```cpp
#include <iostream>

int divide_by_nonzero(int a, int b) {
    [[assume(b != 0)]];     // promise: b is never zero
    return a / b;
    // Compiler removes the division-by-zero check
    // If b IS zero: UB -> crash, wrong result, or security vulnerability
}

// SECURITY RISK:
int check_bounds(int* arr, int index, int size) {
    [[assume(index >= 0 && index < size)]];  // dangerous if caller violates this!
    return arr[index];
    // Compiler removes bounds check
    // If index is out of bounds: buffer overflow vulnerability!
}

int main() {
    int arr[] = {10, 20, 30};
    std::cout << divide_by_nonzero(10, 2) << '\n';  // OK
    std::cout << check_bounds(arr, 1, 3) << '\n';   // OK

    // divide_by_nonzero(10, 0);    // UB!
    // check_bounds(arr, 100, 3);   // UB! Buffer overflow!
}
// Expected output:
// 5
// 20
```

The reason the security angle matters is that an attacker who can control `index` or `b` can now bypass what looks like a check but isn't one at all. The compiler has already eliminated the guard.

**Security rules for `[[assume]]`:**

1. NEVER assume user input is valid.
2. Only use in internal functions after validation has already happened at the boundary.
3. Prefer `assert()` during development; switch to `[[assume]]` only in release where the correctness is well established.

### Q3: Compare `[[assume]]`, `__builtin_assume`, and `assert`

All three let you express "this should be true," but they behave very differently. Here they are side by side so you can see exactly what each one does (and does not) do.

```cpp
#include <cassert>
#include <iostream>

void process(int n) {
    // 1. assert: checked in debug, removed in release (-DNDEBUG)
    assert(n > 0);               // debug: aborts if false; release: gone

    // 2. __builtin_assume (GCC/Clang extension): optimizer hint, never checked
    // __builtin_assume(n > 0);   // UB if false, no runtime check ever

    // 3. [[assume]] (C++23 standard): optimizer hint, never checked
    [[assume(n > 0)]];           // UB if false, no runtime check ever

    std::cout << "Processing " << n << " items\n";
}

int main() {
    process(5);  // OK for all three
}
// Expected output:
// Processing 5 items
```

The table below summarizes the key differences. Pay attention to the "side effects in expr" row - `[[assume]]` never evaluates its argument, so `[[assume(++x > 0)]]` would silently not increment `x`.

| Feature | `assert(expr)` | `__builtin_assume(expr)` | `[[assume(expr)]]` |
| --- | --- | --- | --- |
| Standard | C89 | GCC/Clang extension | C++23 |
| Debug build | Checks, aborts | No check | No check |
| Release build | Removed (`NDEBUG`) | Optimizer hint | Optimizer hint |
| If false (debug) | `abort()` | UB | UB |
| If false (release) | Nothing | UB | UB |
| Side effects in expr | Evaluated (debug) | NOT evaluated | NOT evaluated |
| Portability | Universal | GCC/Clang only | C++23 compilers |

---

## Notes

- `[[assume]]` expression is **never evaluated** - don't put side effects in it.
- MSVC equivalent: `__assume(expr)` (available since VS 2005).
- Best practice: use `assert()` + `[[assume]]` together: `assert(n > 0); [[assume(n > 0)]];` - you get runtime checking in debug and the optimizer hint in release.
- Profile first: `[[assume]]` only helps in hot paths where the compiler can't already deduce the fact.
