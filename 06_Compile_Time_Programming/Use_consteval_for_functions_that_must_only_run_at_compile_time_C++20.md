# Use `consteval` for Functions That Must Only Run at Compile Time (C++20)

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/consteval>  

---

## Topic Overview

### What Is `consteval`

`consteval` declares an **immediate function** - a function that **must** produce a compile-time constant. If the function is called with a non-constant argument, the compiler emits a **hard error**. There is no silent fallback to runtime evaluation.

Here's the simplest possible example:

```cpp
consteval int square(int n) { return n * n; }

constexpr int a = square(5);   // OK: compile-time - result is 25
// int x = 5;
// int b = square(x);          // ERROR: x is not a constant expression
```

That second call is not a warning or a degraded mode - it's a straight compile failure. That's the whole point: `consteval` is a guarantee, not a hint.

### `consteval` vs `constexpr`

The distinction between `consteval` and `constexpr` is important enough to be worth spelling out clearly. `constexpr` functions are *allowed* to run at compile time, but they can also silently fall back to runtime if the arguments aren't constant. `consteval` functions are *required* to run at compile time - no fallback, no exceptions.

| Feature | `consteval` | `constexpr` |
| --- | --- | --- |
| Compile-time evaluation | Guaranteed - always | Possible - only when required |
| Runtime evaluation | Forbidden - hard error | Allowed - falls back to runtime |
| Use case | Lookup tables, compile-time-only computation | Dual-use (compile-time or runtime) |
| Function pointers | Cannot take address at runtime | Can take address |
| Calling non-constexpr | Forbidden | Forbidden in constant context, allowed at runtime |

Here's a direct comparison so you can see the behavioral difference side by side:

```cpp
constexpr int f(int n) { return n * 2; }
consteval int g(int n) { return n * 2; }

int x = 5;
int a = f(x);   // OK: evaluated at runtime
// int b = g(x);   // ERROR: x is not a constant expression
constexpr int c = f(5);  // OK: evaluated at compile-time
constexpr int d = g(5);  // OK: always evaluated at compile-time
```

The takeaway: use `constexpr` when you want a function that's useful both ways; use `consteval` when runtime evaluation would be a bug.

---

## Self-Assessment

### Q1: Write a `consteval` function that generates a lookup table as a `std::array` at compile time

Because `consteval` forces evaluation at compile time, it's a natural fit for generating lookup tables that you want baked directly into the binary. Here's a set of three table generators, all verified with `static_assert` before `main` even runs:

```cpp
#include <iostream>
#include <array>
#include <cstddef>

// === consteval: generates a squares lookup table at compile time ===
template <std::size_t N>
consteval auto make_squares_table() {
    std::array<int, N> table{};
    for (std::size_t i = 0; i < N; ++i) {
        table[i] = static_cast<int>(i * i);
    }
    return table;
}

// === consteval: generates a Fibonacci lookup table ===
template <std::size_t N>
consteval auto make_fibonacci_table() {
    std::array<unsigned long long, N> table{};
    if constexpr (N > 0) table[0] = 0;
    if constexpr (N > 1) table[1] = 1;
    for (std::size_t i = 2; i < N; ++i) {
        table[i] = table[i - 1] + table[i - 2];
    }
    return table;
}

// === consteval: ASCII character classification table ===
consteval auto make_char_class_table() {
    // 0 = other, 1 = digit, 2 = uppercase, 3 = lowercase
    std::array<int, 128> table{};
    for (int c = '0'; c <= '9'; ++c) table[c] = 1;
    for (int c = 'A'; c <= 'Z'; ++c) table[c] = 2;
    for (int c = 'a'; c <= 'z'; ++c) table[c] = 3;
    return table;
}

// All tables are computed at compile time - zero runtime cost
constexpr auto squares   = make_squares_table<16>();
constexpr auto fibonacci = make_fibonacci_table<20>();
constexpr auto char_class = make_char_class_table();

// Verify at compile time
static_assert(squares[5] == 25);
static_assert(squares[10] == 100);
static_assert(fibonacci[10] == 55);
static_assert(fibonacci[19] == 4181);
static_assert(char_class['A'] == 2);
static_assert(char_class['z'] == 3);
static_assert(char_class['5'] == 1);
static_assert(char_class[' '] == 0);

int main() {
    std::cout << "=== Squares Table ===\n";
    for (std::size_t i = 0; i < squares.size(); ++i) {
        std::cout << i << "^2 = " << squares[i] << "\n";
    }

    std::cout << "\n=== Fibonacci Table ===\n";
    for (std::size_t i = 0; i < fibonacci.size(); ++i) {
        std::cout << "fib(" << i << ") = " << fibonacci[i] << "\n";
    }

    std::cout << "\n=== Character Classification ===\n";
    auto classify = [](char c) -> const char* {
        switch (char_class[c]) {
            case 1: return "digit";
            case 2: return "uppercase";
            case 3: return "lowercase";
            default: return "other";
        }
    };
    for (char c : {'A', 'z', '5', ' ', '!'}) {
        std::cout << "'" << c << "' -> " << classify(c) << "\n";
    }

    return 0;
}
```

**Expected output:**

```text
=== Squares Table ===
0^2 = 0
1^2 = 1
2^2 = 4
3^2 = 9
4^2 = 16
5^2 = 25
6^2 = 36
7^2 = 49
8^2 = 64
9^2 = 81
10^2 = 100
11^2 = 121
12^2 = 144
13^2 = 169
14^2 = 196
15^2 = 225

=== Fibonacci Table ===
fib(0) = 0
fib(1) = 1
fib(2) = 1
fib(3) = 2
fib(4) = 3
fib(5) = 5
fib(6) = 8
fib(7) = 13
fib(8) = 21
fib(9) = 34
fib(10) = 55
fib(11) = 89
fib(12) = 144
fib(13) = 233
fib(14) = 377
fib(15) = 610
fib(16) = 987
fib(17) = 1597
fib(18) = 2584
fib(19) = 4181

=== Character Classification ===
'A' -> uppercase
'z' -> lowercase
'5' -> digit
' ' -> other
'!' -> other
```

### Q2: Show the compile error that occurs when a `consteval` function is called with a runtime argument

There are three distinct ways to misuse a `consteval` function, each with its own error message. The example below shows all three in commented-out form, along with the valid uses for contrast:

```cpp
#include <iostream>

// === consteval function ===
consteval int compile_time_only(int n) {
    return n * n + 1;
}

int main() {
    // === Valid: constant expression argument ===
    constexpr int a = compile_time_only(10);  // OK: 101
    static_assert(a == 101);
    std::cout << "compile_time_only(10) = " << a << "\n";

    // A literal is a constant expression
    int b = compile_time_only(5);  // OK: 5 is a literal constant
    std::cout << "compile_time_only(5) = " << b << "\n";

    // === INVALID: runtime argument ===
    // Uncomment to see the error:
    /*
    int x = 5;
    int c = compile_time_only(x);   // ERROR!
    */
    // GCC error:
    //   error: the value of 'x' is not usable in a constant expression
    //   note: 'x' was not declared 'constexpr'
    //
    // Clang error:
    //   error: call to consteval function 'compile_time_only' is not
    //          a constant expression
    //   note: read of non-const variable 'x' is not allowed in a
    //          constant expression

    // === INVALID: cannot take address ===
    // Uncomment to see the error:
    /*
    auto fn_ptr = &compile_time_only;  // ERROR!
    */
    // error: taking address of an immediate function

    // === VALID: constexpr variable as argument ===
    constexpr int y = 7;
    int d = compile_time_only(y);   // OK: y is constexpr
    std::cout << "compile_time_only(7) = " << d << "\n";

    // === VALID: consteval calling consteval ===
    // consteval is a function specifier, so it goes on the lambda's call
    // operator - `consteval auto x = ...` would declare a consteval variable,
    // which is ill-formed.
    auto doubled_square = [](int n) consteval {
        return compile_time_only(n) * 2;
    };
    constexpr int e = doubled_square(3);  // OK: 3 -> 10 -> 20
    std::cout << "doubled_square(3) = " << e << "\n";

    std::cout << "\n=== Summary of Errors ===\n";
    std::cout << "1. Runtime variable as arg  -> 'not usable in constant expression'\n";
    std::cout << "2. Taking address           -> 'taking address of immediate function'\n";
    std::cout << "3. Non-constexpr variable   -> 'read of non-const variable not allowed'\n";

    return 0;
}
```

**Expected output:**

```text
compile_time_only(10) = 101
compile_time_only(5) = 26
compile_time_only(7) = 50
doubled_square(3) = 20

=== Summary of Errors ===

1. Runtime variable as arg  -> 'not usable in constant expression'
2. Taking address           -> 'taking address of immediate function'
3. Non-constexpr variable   -> 'read of non-const variable not allowed'
```

### Q3: Explain the difference between `consteval` and `constexpr` in terms of guaranteed evaluation

This example puts both specifiers through the same set of scenarios so you can see exactly where their behavior diverges. Pay attention to Case 2 - that's where `consteval` refuses to compile while `constexpr` quietly does runtime work instead:

```cpp
#include <iostream>
#include <array>

// === constexpr: MAY run at compile time ===
constexpr int compute_constexpr(int n) {
    int sum = 0;
    for (int i = 1; i <= n; ++i) sum += i;
    return sum;
}

// === consteval: MUST run at compile time ===
consteval int compute_consteval(int n) {
    int sum = 0;
    for (int i = 1; i <= n; ++i) sum += i;
    return sum;
}

int main() {
    // === Case 1: Used in constexpr context ===
    // Both produce compile-time results
    constexpr int a = compute_constexpr(10);  // Compile-time: guaranteed by constexpr var
    constexpr int b = compute_consteval(10);  // Compile-time: guaranteed by consteval
    static_assert(a == 55);
    static_assert(b == 55);

    // === Case 2: Used in runtime context ===
    int n = 10;
    int c = compute_constexpr(n);   // RUNTIME: falls back because n is not constant
    // int d = compute_consteval(n);   // ERROR: n is not a constant expression

    std::cout << "constexpr result: " << a << "\n";
    std::cout << "consteval result: " << b << "\n";
    std::cout << "runtime constexpr: " << c << "\n";

    // === Case 3: Template argument (must be compile-time) ===
    std::array<int, compute_constexpr(5)> arr1;  // OK: 15 at compile time
    std::array<int, compute_consteval(5)> arr2;   // OK: 15 at compile time
    std::cout << "arr1 size: " << arr1.size() << "\n";
    std::cout << "arr2 size: " << arr2.size() << "\n";

    // === Case 4: constexpr can be called from non-constexpr code freely ===
    auto process = [](int val) {
        return compute_constexpr(val);  // OK: runtime call
        // return compute_consteval(val);  // ERROR: val is not constant
    };
    std::cout << "process(5): " << process(5) << "\n";

    // === Case 5: Function pointers ===
    auto ptr = &compute_constexpr;   // OK
    // auto ptr2 = &compute_consteval;   // ERROR: cannot take address
    std::cout << "via pointer: " << ptr(10) << "\n";

    std::cout << "\n=== Evaluation Guarantee Matrix ===\n";
    std::cout << "Context                    constexpr   consteval\n";
    std::cout << "──────────────────────────────────────────────────\n";
    std::cout << "constexpr int x = f(5);   Compile     Compile\n";
    std::cout << "int x = f(5);             Compile*    Compile\n";
    std::cout << "int n=5; int x = f(n);    Runtime     ERROR!\n";
    std::cout << "array<int, f(5)> arr;     Compile     Compile\n";
    std::cout << "auto p = &f;              OK          ERROR!\n";
    std::cout << "\n* May be compile-time (as-if rule) but not guaranteed\n";

    return 0;
}
```

**Expected output:**

```text
constexpr result: 55
consteval result: 55
runtime constexpr: 55
arr1 size: 15
arr2 size: 15
process(5): 15
via pointer: 55

=== Evaluation Guarantee Matrix ===
Context                    constexpr   consteval
──────────────────────────────────────────────────
constexpr int x = f(5);   Compile     Compile
int x = f(5);             Compile*    Compile
int n=5; int x = f(n);    Runtime     ERROR!
array<int, f(5)> arr;     Compile     Compile
auto p = &f;              OK          ERROR!

* May be compile-time (as-if rule) but not guaranteed
```

---

## Notes

- `consteval` is C++20. Use `constexpr` for functions that should also work at runtime.
- `consteval` functions cannot have their address taken - they exist only during compilation.
- Use `consteval` for: lookup table generators, compile-time validators, DSL parsers, hash generation.
- Use `constexpr` when: the function is useful both at compile time and runtime.
- `consteval` can call `constexpr` functions, but not vice versa (unless in `if consteval` context).
- In C++23, `if consteval` provides a bridge between `consteval` and runtime paths.
