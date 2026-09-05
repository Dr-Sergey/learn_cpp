# Know how integer promotion and usual arithmetic conversions work

**Category:** Core Language Fundamentals  
**Standard:** C++98, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/implicit_conversion>  

---

## Topic Overview

### What Are Integer Promotions

Here's something that surprises almost everyone the first time they hit it: C++ never actually does arithmetic on types smaller than `int`. Before any arithmetic or bitwise operation, the compiler quietly **promotes** "small" integer types up to `int`. This is called **integer promotion**, and the rules come straight from C:

| Source type | Promoted to |
| --- | --- |
| `bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short` | `int` (if `int` can represent all values), else `unsigned int` |
| Unscoped enum with underlying type smaller than `int` | `int` or `unsigned int` |
| Bit-field with width < `int` width | `int` or `unsigned int` |

So even `char + char` gives you an `int`. There's no such thing as "char arithmetic":

```cpp
char a = 100, b = 100;
auto result = a + b;          // result is int (200), NOT char
static_assert(std::is_same_v<decltype(a + b), int>);
```

If `a + b` stayed a `char`, `200` wouldn't even fit. Promotion is what saves you here - the math happens in `int`.

### Usual Arithmetic Conversions

Promotion handles a single small type. But what about a binary operator whose two operands have *different* types - say `int` and `double`, or `int` and `unsigned`? Then the compiler brings both to a single **common type** using the *usual arithmetic conversions*, applied in this order:

1. If either operand is `long double` -> the other becomes `long double`.
2. Else if either is `double` -> the other becomes `double`.
3. Else if either is `float` -> the other becomes `float`.
4. Else **integer promotions** are applied to both, then:
   - If both have the same signedness -> convert to the wider type.
   - If the unsigned type is >= rank of the signed type -> convert signed to unsigned. **(This is the dangerous case!)**
   - If the signed type can represent all values of the unsigned type -> convert unsigned to signed.
   - Otherwise -> both become the unsigned version of the signed type.

That fourth branch, second bullet, is the one that quietly ruins people's afternoons. Let's look at it.

### The Signed/Unsigned Comparison Trap

Watch what happens when you compare a negative `int` against an `unsigned`:

```cpp
int    s = -1;
unsigned u = 2;
// Step 1: Both are already >= int rank, no integer promotion needed
// Step 2: Different signedness, unsigned int rank == signed int rank
// Step 3: Convert signed to unsigned -> -1 becomes UINT_MAX (4294967295)
// Step 4: 4294967295 > 2 -> true!
if (s > u) {
    // This EXECUTES! -1 > 2u is true!
}
```

The `-1` gets reinterpreted as a gigantic unsigned value, and suddenly "negative one is greater than two" is true. This is one of the most common sources of real bugs in C and C++ - it hides in loop bounds, size comparisons, and index checks.

### Compiler Warnings

The good news: your compiler can flag these for you if you let it. GCC and Clang with `-Wsign-compare` (which `-Wall` already includes) emit:

```text
warning: comparison of integer expressions of different signedness
```

MSVC with `/W4` warns about the same mismatches. Turn these on and leave them on.

---

## Self-Assessment

### Q1: Show a signed/unsigned comparison bug where `-1 > 2u` evaluates to `true`

The point of this example is to *see* the bogus value the conversion produces, not just trust that it happens:

```cpp
#include <iostream>

int main() {
    int signed_val = -1;
    unsigned int unsigned_val = 2;

    // The compiler converts signed_val to unsigned before comparison.
    // -1 as unsigned int = 4294967295 (UINT_MAX on 32-bit int)
    // 4294967295 > 2 -> true
    if (signed_val > unsigned_val) {
        std::cout << "-1 > 2u is TRUE (surprising!)\n";
    } else {
        std::cout << "-1 > 2u is false\n";
    }

    // Demonstrating the actual converted value:
    std::cout << "signed_val as unsigned: "
              << static_cast<unsigned int>(signed_val) << "\n";
    // Output: 4294967295

    // Same problem with different types:
    short s = -1;
    unsigned int u = 0;
    std::cout << "(short)-1 > 0u: " << (s > u) << "\n";  // true!
    // short -1 -> int -1 (promotion) -> unsigned int UINT_MAX (conversion)

    return 0;
}
```

**Output:**

```text
-1 > 2u is TRUE (surprising!)
signed_val as unsigned: 4294967295
(short)-1 > 0u: 1
```

**How this works:**

- When comparing `int` to `unsigned int`, the `int` is the one that converts.
- `-1` in two's complement becomes `UINT_MAX` (all bits set = 4294967295).
- `4294967295 > 2` is honestly true - the comparison isn't "wrong," it's just operating on a value you didn't intend.
- This bites *every* relational and equality operator: `<`, `>`, `<=`, `>=`, `==`, `!=`.
- Notice the `short` case needs two steps: it's promoted to `int` first, then converted to `unsigned int`.

### Q2: Explain what happens when char arithmetic overflows and how to avoid UB

First, an uncomfortable fact: **whether `char` is signed or unsigned is implementation-defined.** On most desktop platforms (x86) it's signed, spanning `-128` to `127`.

Now, the thing people fear is overflow during the arithmetic - but promotion actually protects you there:

```cpp
signed char a = 120;
signed char b = 20;
// a + b performs INTEGER PROMOTION first:
// int(120) + int(20) = int(140) -> result is int, no overflow!
signed char c = a + b;  // 140 doesn't fit in signed char -> implementation-defined narrowing
```

So where's the catch? Not in the addition - that happens safely in `int`. The catch is **storing the result back** into a `char`:

- The narrowing from `int` to `signed char` is **implementation-defined** (a defined-but-platform-specific value, not UB).
- The real trap is logic errors, like a loop counter that can never reach its bound:

```cpp
// Logic bug: the comparison is fine, but a signed char can't hold 200
for (signed char i = 0; i < 200; ++i) {  // i can never reach 200!
    // Infinite loop: i goes 0..127, then wraps (impl-defined) to -128, etc.
}
```

For **`unsigned char`**, arithmetic wraps cleanly modulo 256, which is fully well-defined.

The practical guidance is simple:

1. **Use `int` or larger for arithmetic** - don't do math directly on `char`.
2. **Use `unsigned char` or `std::uint8_t`** when you genuinely mean byte manipulation.
3. **Turn on `-Wconversion` and `-Wsign-conversion`** so the compiler nags you.
4. **Use `static_cast` explicitly** when a narrowing conversion is what you actually want.

```cpp
#include <cstdint>
#include <iostream>

int main() {
    // GOOD: Use int for arithmetic
    int sum = 0;
    for (int i = 0; i < 1000; ++i) sum += i;

    // GOOD: Use uint8_t for byte data
    std::uint8_t byte = 200;
    std::uint8_t result = static_cast<std::uint8_t>(byte + 100);  // Wraps to 44, well-defined

    // GOOD: Explicit cast when narrowing
    int big = 300;
    auto small = static_cast<signed char>(big);  // Intentional truncation
    std::cout << static_cast<int>(small) << "\n"; // Implementation-defined value
    return 0;
}
```

### Q3: Demonstrate how to use `std::cmp_less` (C++20) to safely compare signed and unsigned integers

C++20 finally gives you a clean way out of the signed/unsigned trap. The `std::cmp_*` functions compare values *mathematically*, ignoring the representation games:

```cpp
#include <iostream>
#include <utility>  // std::cmp_less, std::cmp_greater, etc.

int main() {
    int    signed_val   = -1;
    unsigned unsigned_val = 2;

    // DANGEROUS: raw comparison (signed converted to unsigned)
    std::cout << "Raw:      -1 > 2u  = " << (signed_val > unsigned_val) << "\n";  // 1 (true!)

    // SAFE: std::cmp_less and friends (C++20)
    std::cout << "cmp_less: -1 < 2u  = " << std::cmp_less(signed_val, unsigned_val) << "\n";     // 1 (true)
    std::cout << "cmp_greater: -1 > 2u = " << std::cmp_greater(signed_val, unsigned_val) << "\n"; // 0 (false)
    std::cout << "cmp_equal: -1 == 2u = " << std::cmp_equal(signed_val, unsigned_val) << "\n";    // 0 (false)

    // Practical use: safe bounds checking
    int index = -5;
    std::size_t container_size = 10;

    if (std::cmp_less(index, container_size) && std::cmp_greater_equal(index, 0)) {
        std::cout << "Index is in bounds\n";
    } else {
        std::cout << "Index is out of bounds\n";
    }

    // All C++20 safe comparison functions:
    // std::cmp_equal(a, b)         - true if mathematically a == b
    // std::cmp_not_equal(a, b)     - true if mathematically a != b
    // std::cmp_less(a, b)          - true if mathematically a < b
    // std::cmp_greater(a, b)       - true if mathematically a > b
    // std::cmp_less_equal(a, b)    - true if mathematically a <= b
    // std::cmp_greater_equal(a, b) - true if mathematically a >= b
    // std::in_range<T>(value)      - true if value fits in T

    // std::in_range example:
    std::cout << "Can -1 fit in unsigned? " << std::in_range<unsigned>(-1) << "\n";  // 0 (false)
    std::cout << "Can 42 fit in unsigned? " << std::in_range<unsigned>(42) << "\n";  // 1 (true)
    std::cout << "Can 300 fit in uint8_t? " << std::in_range<uint8_t>(300) << "\n";  // 0 (false)

    return 0;
}
```

**Output:**

```text
Raw:      -1 > 2u  = 1
cmp_less: -1 < 2u  = 1
cmp_greater: -1 > 2u = 0
cmp_equal: -1 == 2u = 0
Index is out of bounds
Can -1 fit in unsigned? 0
Can 42 fit in unsigned? 1
Can 300 fit in uint8_t? 0
```

**How this works:**

- `std::cmp_less` and its siblings give you the answer you'd compute on paper, regardless of signedness.
- Under the hood they check the sign first: a negative signed value next to a non-negative unsigned one is always smaller, no conversion needed.
- `std::in_range<T>(value)` asks "does this value actually fit in `T`?" - perfect to call *before* a narrowing conversion.
- Best part: these are zero-cost. They compile down to the same instructions a careful programmer would write by hand.

---

## Additional Examples

### Integer Promotion Surprises

A couple of cases that catch people off guard - especially the bitwise `~`, which promotes before it inverts:

```cpp
#include <iostream>
#include <type_traits>

int main() {
    unsigned short a = 65535;
    unsigned short b = 1;

    // a + b promotes both to int (assuming int is 32-bit)
    auto sum = a + b;
    static_assert(std::is_same_v<decltype(sum), int>);  // NOT unsigned short!
    std::cout << "65535us + 1us = " << sum << " (type: int)\n";  // 65536

    // But on a platform where int is 16-bit (rare):
    // both would promote to unsigned int, and 65535 + 1 = 0 (wraps)

    // Bitwise NOT on unsigned char:
    unsigned char mask = 0x0F;
    auto notmask = ~mask;  // promotes to int first, then NOT
    // notmask = ~(int)0x0F = 0xFFFFFFF0 (int, negative!)
    static_assert(std::is_same_v<decltype(notmask), int>);
    std::cout << "~0x0F as int: " << notmask << "\n";  // -16

    // To get unsigned char result:
    unsigned char correct = static_cast<unsigned char>(~mask);  // 0xF0
    std::cout << "~0x0F as uchar: " << static_cast<int>(correct) << "\n";  // 240
}
```

### Rank-Based Conversion Table

A quick playground for seeing which type wins in mixed expressions:

```cpp
#include <iostream>
#include <typeinfo>
#include <cstdint>

// Helper to print the type of an expression
#define SHOW_TYPE(expr) \
    std::cout << #expr << " has type: " << typeid(expr).name() << " = " << (expr) << "\n"

int main() {
    SHOW_TYPE(1 + 1.0);       // double (int -> double)
    SHOW_TYPE(1.0f + 1.0);    // double (float -> double)
    SHOW_TYPE(1u + (-2));      // unsigned int (int -> unsigned)
    SHOW_TYPE('A' + 1);        // int (char promoted to int)
    SHOW_TYPE(true + true);    // int (bool promoted to int: 1 + 1 = 2)
}
```

---

## Notes

- Always enable `-Wsign-compare` and `-Wconversion` to catch promotion issues before they ship.
- Use the `std::cmp_*` functions (C++20) for any signed/unsigned comparison in your code.
- Prefer `int` or `std::size_t` for loop counters - avoid arithmetic on `char` or `short`.
- `auto` preserves the *promoted* type, which can surprise you: `unsigned char a = 0xFF; auto b = ~a;` gives `int`, not `unsigned char`.
- Use `std::in_range<T>()` before narrowing conversions to validate at runtime.
