# Use `constexpr` Lambdas (C++17) in Compile-Time Algorithms

**Category:** Compile Time Programming  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

### `constexpr` Lambdas in C++17

C++17 made lambdas **implicitly constexpr** whenever their body satisfies `constexpr` function requirements. You don't need to write `constexpr` on the lambda - if the body is eligible, the compiler treats it as `constexpr` automatically. This unlocked using lambdas as building blocks in compile-time algorithms.

```cpp
// C++17: this lambda is automatically constexpr
constexpr auto add = [](int a, int b) { return a + b; };
static_assert(add(3, 4) == 7);
```

That `static_assert` passes at compile time - no extra annotation needed.

### Using Lambdas in Compile-Time Algorithms

Where this gets useful is when you want to pass customizable behavior into a compile-time algorithm. Lambdas serve as **customization points** - comparators, transformers, predicates - exactly as they do at runtime, but now the whole chain runs at compile time.

Keep in mind that `std::sort` wasn't made `constexpr` until C++20, so in C++17 you'd implement your own sorting algorithm. Here's a classic bubble sort written as a lambda:

```cpp
constexpr auto sort_array = [](auto arr) {
    // Bubble sort at compile time (std::sort is constexpr only since C++20)
    for (std::size_t i = 0; i < arr.size(); ++i)
        for (std::size_t j = i + 1; j < arr.size(); ++j)
            if (arr[j] < arr[i]) std::swap(arr[i], arr[j]);
    return arr;
};
```

### Capture Rules in `constexpr` Context

Here's the rule that trips people up most often: reference captures don't work in `constexpr` lambdas, but value captures do.

The reason is fundamental. A reference is internally a pointer, and a pointer holds an address. Addresses are runtime values - they're assigned by the linker and loaded by the OS. A constant expression cannot depend on a runtime address, so reference captures are off-limits. Value captures embed the value directly into the closure object, which is why they work fine.

| Capture | Compile-time? | Why? |
| --- | :---: | --- |
| `[x]` (by value) | Yes | Value is embedded in closure |
| `[x = expr]` (init capture) | Yes, if `expr` is constexpr | Computed at compile time |
| `[&x]` (by reference) | No | References involve addresses - not constexpr |

---

## Self-Assessment

### Q1: Write a `constexpr` lambda that sorts a `std::array` at compile time and verify with `static_assert`

This example implements three sorting strategies as `constexpr` lambdas in C++17 style, since `std::sort` isn't available at compile time until C++20. Notice that `static_assert` runs before `main` - the sorting happens during compilation, not execution.

```cpp
#include <iostream>
#include <array>
#include <algorithm>
#include <cstddef>

// === C++17: constexpr bubble sort lambda ===
// (std::sort is not constexpr until C++20, so we implement our own)
constexpr auto bubble_sort = [](auto arr) {
    for (std::size_t i = 0; i < arr.size(); ++i) {
        for (std::size_t j = 0; j + 1 < arr.size() - i; ++j) {
            if (arr[j + 1] < arr[j]) {
                auto tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
    return arr;
};

// === Sort at compile time ===
constexpr std::array<int, 7> unsorted = {64, 25, 12, 22, 11, 90, 1};
constexpr auto sorted = bubble_sort(unsorted);

// Verify at compile time
static_assert(sorted[0] == 1);
static_assert(sorted[1] == 11);
static_assert(sorted[2] == 12);
static_assert(sorted[6] == 90);

// === constexpr lambda with custom comparator ===
constexpr auto sort_descending = [](auto arr) {
    for (std::size_t i = 0; i < arr.size(); ++i) {
        for (std::size_t j = 0; j + 1 < arr.size() - i; ++j) {
            if (arr[j + 1] > arr[j]) {
                auto tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
    return arr;
};

constexpr auto desc = sort_descending(unsorted);
static_assert(desc[0] == 90);
static_assert(desc[6] == 1);

// === constexpr selection sort ===
constexpr auto selection_sort = [](auto arr) {
    for (std::size_t i = 0; i < arr.size(); ++i) {
        std::size_t min_idx = i;
        for (std::size_t j = i + 1; j < arr.size(); ++j) {
            if (arr[j] < arr[min_idx]) min_idx = j;
        }
        if (min_idx != i) {
            auto tmp = arr[i];
            arr[i] = arr[min_idx];
            arr[min_idx] = tmp;
        }
    }
    return arr;
};

constexpr auto sel_sorted = selection_sort(unsorted);
static_assert(sel_sorted[0] == 1 && sel_sorted[6] == 90);

int main() {
    std::cout << "Unsorted:   ";
    for (int x : unsorted) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "Sorted:     ";
    for (int x : sorted) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "Descending: ";
    for (int x : desc) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "\nAll results verified at compile time with static_assert!\n";
    return 0;
}
```

**Expected output:**

```text
Unsorted:   64 25 12 22 11 90 1
Sorted:     1 11 12 22 25 64 90
Descending: 90 64 25 22 12 11 1

All results verified at compile time with static_assert!
```

### Q2: Show that `constexpr` lambdas can capture `constexpr` variables by value

Value captures and init captures both work in `constexpr` contexts. The captured value is embedded in the closure type, which means the compiler knows it at compile time. You can even do arithmetic in the capture initializer, as long as the initializing expression is itself a constant expression.

```cpp
#include <iostream>
#include <array>

// === Capture constexpr by value ===
constexpr int BASE = 16;
constexpr int MASK = 0xFF;

// Value capture: BASE is embedded in the closure
constexpr auto to_hex_digit = [base = BASE](int value) -> int {
    return value % base;
};

constexpr auto apply_mask = [m = MASK](int value) -> int {
    return value & m;
};

static_assert(to_hex_digit(255) == 15);   // 255 % 16 = 15
static_assert(to_hex_digit(32) == 0);     // 32 % 16 = 0
static_assert(apply_mask(0x1234) == 0x34);

// === Capture constexpr array by value ===
constexpr std::array<int, 4> weights = {1, 2, 4, 8};

constexpr auto weighted_sum = [w = weights](const std::array<int, 4>& values) {
    int sum = 0;
    for (std::size_t i = 0; i < 4; ++i) {
        sum += w[i] * values[i];
    }
    return sum;
};

constexpr std::array<int, 4> data = {1, 1, 1, 1};
static_assert(weighted_sum(data) == 15);  // 1+2+4+8

// === Multiple captures ===
constexpr int LO = 10;
constexpr int HI = 100;

constexpr auto clamp = [lo = LO, hi = HI](int x) -> int {
    if (x < lo) return lo;
    if (x > hi) return hi;
    return x;
};

static_assert(clamp(5) == 10);
static_assert(clamp(50) == 50);
static_assert(clamp(200) == 100);

// === Init capture with constexpr computation ===
constexpr auto half_base = [h = BASE / 2](int x) { return x * h; };
static_assert(half_base(3) == 24);  // 3 * 8

int main() {
    std::cout << "to_hex_digit(255) = " << to_hex_digit(255) << "\n";
    std::cout << "apply_mask(0x1234) = 0x" << std::hex << apply_mask(0x1234) << std::dec << "\n";
    std::cout << "weighted_sum({1,1,1,1}) = " << weighted_sum(data) << "\n";
    std::cout << "clamp(5) = " << clamp(5) << "\n";
    std::cout << "clamp(50) = " << clamp(50) << "\n";
    std::cout << "clamp(200) = " << clamp(200) << "\n";

    return 0;
}
```

**Expected output:**

```text
to_hex_digit(255) = 15
apply_mask(0x1234) = 0x34
weighted_sum({1,1,1,1}) = 15
clamp(5) = 10
clamp(50) = 50
clamp(200) = 100
```

### Q3: Explain why `constexpr` lambdas cannot capture variables by reference in a constant expression

This is one of the most common `constexpr` lambda pitfalls, so it's worth understanding the reasoning rather than just memorizing the rule.

A reference is semantically a pointer. In a constant expression, all values must be deterministic at compile time. A reference to a variable implies knowing its address - which is a runtime property assigned by the linker. The compiler places variables at addresses determined during linking, not during compilation. So any capture that relies on an address is off-limits in a constant expression.

The fix is straightforward: use a value capture or init capture instead. The value is embedded directly into the closure object, and no address is involved.

```cpp
#include <iostream>

// === WHY reference captures fail in constexpr ===
//
// A reference is semantically a pointer. In a constant expression, all
// values must be deterministic at compile time. A reference to a variable
// implies knowing its ADDRESS, which is a runtime property.
//
// The compiler places variables at addresses determined by the linker -
// these addresses are not available during constant evaluation.

// === Example: this would fail ===
// constexpr int N = 10;
// constexpr auto bad = [&N]() { return N * 2; };
// constexpr int result = bad();  // ERROR!
//
// GCC: "the value of 'N' is not usable in a constant expression"
// Reason: &N is an address, and addresses are not constant expressions

// === Value capture works because no address is needed ===
constexpr int N = 10;
constexpr auto good = [n = N]() { return n * 2; };
constexpr int result = good();  // OK: n is a value, no address involved
static_assert(result == 20);

// === Even constexpr globals fail with reference capture ===
// constexpr int G = 42;
// constexpr auto capture_ref = [&G]() { return G; };
// constexpr int x = capture_ref();  // ERROR: cannot reference G's address

// === The deeper reason: pointers in constant expressions ===
// C++ allows pointers in constexpr only in very limited cases:
// - nullptr
// - Past-the-end of constexpr arrays (comparisons only)
// - Pointers that are never dereferenced (never actually used)
//
// A reference capture generates a pointer member in the closure class:
//   struct Lambda {
//       const int& n;   // internally: const int* n_ptr;
//       int operator()() const { return *n_ptr; }
//   };
// The pointer value (address) is unknown at compile time -> not constexpr

// === Workaround: use value or init capture ===
constexpr int FACTOR = 5;

// BAD:  constexpr auto f = [&FACTOR](int x) { return x * FACTOR; };
// GOOD: constexpr auto f = [f = FACTOR](int x) { return x * f; };
constexpr auto f = [f = FACTOR](int x) { return x * f; };
static_assert(f(6) == 30);

int main() {
    std::cout << "=== Reference Capture in constexpr ===\n\n";

    std::cout << "Value capture:     [n = N]     -> OK (value embedded)\n";
    std::cout << "Init capture:      [f = EXPR]  -> OK if EXPR is constexpr\n";
    std::cout << "Reference capture: [&N]        -> ERROR (address is runtime)\n";

    std::cout << "\n=== Why? ===\n";
    std::cout << "Reference = pointer internally\n";
    std::cout << "Pointer = runtime address\n";
    std::cout << "Constant expression cannot depend on runtime address\n";
    std::cout << "Therefore: reference capture -> not a constant expression\n";

    std::cout << "\ngood() = " << result << "\n";
    std::cout << "f(6) = " << f(6) << "\n";

    return 0;
}
```

**Expected output:**

```text
=== Reference Capture in constexpr ===

Value capture:     [n = N]     -> OK (value embedded)
Init capture:      [f = EXPR]  -> OK if EXPR is constexpr
Reference capture: [&N]        -> ERROR (address is runtime)

=== Why? ===
Reference = pointer internally
Pointer = runtime address
Constant expression cannot depend on runtime address
Therefore: reference capture -> not a constant expression

good() = 20
f(6) = 30
```

---

## Notes

- C++17 made lambdas **implicitly constexpr** - no explicit keyword needed.
- Use constexpr lambdas for compile-time sorts, filters, transforms in `constexpr` algorithms.
- Value captures work in `constexpr`; reference captures never do (address is runtime).
- In C++17, implement own sort/search algorithms; in C++20, `std::sort`/`std::lower_bound` are `constexpr`.
- `constexpr` lambdas are the foundation for policy-based compile-time programming.
- Init captures (`[x = expr]`) are powerful - they compute derived values at capture time.
