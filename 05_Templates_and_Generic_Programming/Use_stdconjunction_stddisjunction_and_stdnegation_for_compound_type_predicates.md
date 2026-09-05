# Use `std::conjunction`, `std::disjunction`, and `std::negation` for Compound Type Predicates

**Category:** Templates and Generic Programming  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/types/conjunction>  

---

## Topic Overview

### What Are These Traits

C++17 gives you three logical combinators for type traits. Think of them as `&&`, `||`, and `!` that work on type predicates rather than runtime booleans:

```cpp
#include <type_traits>

// AND: all traits must be true
std::conjunction_v<std::is_integral<T>, std::is_signed<T>>

// OR: at least one trait must be true
std::disjunction_v<std::is_integral<T>, std::is_floating_point<T>>

// NOT: negate a single trait
std::negation_v<std::is_pointer<T>>
```

### Key Properties

| Trait | Behavior | Short-circuits? |
| --- | --- | :---: |
| `conjunction<Bs...>` | `true` if all `Bs::value` are `true` | Yes - stops at first `false` |
| `disjunction<Bs...>` | `true` if any `Bs::value` is `true` | Yes - stops at first `true` |
| `negation<B>` | `!B::value` | N/A |

### Why Short-Circuiting Matters

This is the part that trips people up: when you write `(std::is_integral_v<Ts> && ...)` as a fold expression, the compiler still instantiates every trait regardless of whether earlier ones are `false`. That means if any instantiation would be ill-formed, you get a hard error even if you never actually needed that branch.

`conjunction` and `disjunction` stop early, so later traits are never instantiated at all:

```cpp
// This instantiates ALL traits (no short-circuit):
(std::is_integral_v<Ts> && ...)

// This stops at the first false (short-circuits):
std::conjunction_v<std::is_integral<Ts>...>
```

The practical payoff is that you can guard an expensive or potentially ill-formed trait behind a cheap guard, and the compiler will never even look at the problematic one if the guard fires.

---

## Self-Assessment

### Q1: Write a template constrained to types that are both copyable and comparable using `conjunction_v`

Here we build a `safe_min` that requires two separate properties at once, which is exactly where `conjunction_v` earns its keep.

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// Helper trait: "is less-than comparable"
template <typename T, typename = void>
struct is_less_comparable : std::false_type {};

template <typename T>
struct is_less_comparable<T,
    std::void_t<decltype(std::declval<T>() < std::declval<T>())>>
    : std::true_type {};

// === Constrained with conjunction ===
template <typename T>
std::enable_if_t<
    std::conjunction_v<
        std::is_copy_constructible<T>,    // must be copyable
        is_less_comparable<T>              // must support operator<
    >,
    T>
safe_min(T a, T b) {
    return (a < b) ? a : b;
}

// === C++20 concepts version (cleaner) ===
template <typename T>
concept CopyableAndComparable = std::is_copy_constructible_v<T>
                             && requires(T a, T b) { { a < b } -> std::convertible_to<bool>; };

template <CopyableAndComparable T>
T safe_min_v2(T a, T b) {
    return (a < b) ? a : b;
}

// === Multi-trait conjunction ===
template <typename T>
constexpr bool is_numeric_v = std::conjunction_v<
    std::is_arithmetic<T>,              // int, float, etc.
    std::negation<std::is_same<T, bool>>,  // NOT bool
    std::negation<std::is_same<T, char>>   // NOT char
>;

template <typename T>
std::enable_if_t<is_numeric_v<T>, T>
safe_divide(T a, T b) {
    return a / b;
}

int main() {
    std::cout << "=== conjunction: copyable AND comparable ===\n";
    std::cout << "safe_min(3, 7) = " << safe_min(3, 7) << "\n";          // 3
    std::cout << "safe_min(3.14, 2.71) = " << safe_min(3.14, 2.71) << "\n";  // 2.71
    std::cout << "safe_min(\"abc\", \"xyz\") = " << safe_min(std::string("abc"), std::string("xyz")) << "\n";

    std::cout << "\n=== C++20 concepts version ===\n";
    std::cout << "safe_min_v2(10, 5) = " << safe_min_v2(10, 5) << "\n";

    std::cout << "\n=== Multi-trait conjunction ===\n";
    std::cout << "is_numeric<int>: " << is_numeric_v<int> << "\n";      // 1
    std::cout << "is_numeric<double>: " << is_numeric_v<double> << "\n"; // 1
    std::cout << "is_numeric<bool>: " << is_numeric_v<bool> << "\n";    // 0
    std::cout << "is_numeric<char>: " << is_numeric_v<char> << "\n";    // 0

    std::cout << "safe_divide(10, 3) = " << safe_divide(10, 3) << "\n";   // 3
    std::cout << "safe_divide(10.0, 3.0) = " << safe_divide(10.0, 3.0) << "\n";  // 3.333

    return 0;
}
```

Notice the `is_numeric_v` alias - giving the conjunction predicate a name like that makes it reusable and makes the `enable_if` line much easier to read.

### Q2: Show short-circuit evaluation: conjunction stops at the first false, disjunction at the first true

This example is a bit more conceptual, but the key thing to see is the `safe_check` alias near the bottom: because `std::is_integral<double>` is `false`, `conjunction` stops and `safe_sizeof<int>` is never instantiated.

```cpp
#include <iostream>
#include <type_traits>

// === Custom traits that print when instantiated ===
template <int N, bool Value>
struct TrackedTrait {
    static constexpr bool value = Value;
    using type = TrackedTrait;
    // In a real scenario, the compiler instantiates ::value
    // We'll demonstrate conceptually
};

// === Dangerous trait that causes a hard error if instantiated ===
template <typename T>
struct safe_sizeof {
    static_assert(sizeof(T) > 0, "Incomplete type!");
    static constexpr bool value = (sizeof(T) <= 8);
};

struct Incomplete;

// conjunction SHORT-CIRCUITS: if first trait is false, rest are NOT instantiated
// This is SAFE — std::is_integral<double> is false, so safe_sizeof<int> never fires
using safe_check = std::conjunction<
    std::is_integral<double>,    // false -> STOPS HERE
    safe_sizeof<int>             // never instantiated (would be fine anyway)
    // safe_sizeof<Incomplete>   // if this were reached, it would be a hard error
>;

// With fold expression &&, ALL traits are instantiated:
// (std::is_integral_v<double> && safe_sizeof<Incomplete>::value)
// -> safe_sizeof<Incomplete> IS instantiated -> HARD ERROR!

int main() {
    std::cout << "=== Conjunction short-circuit ===\n";

    // All true -> evaluates all
    constexpr bool r1 = std::conjunction_v<
        std::is_integral<int>,    // true
        std::is_signed<int>,      // true
        std::is_arithmetic<int>   // true
    >;
    std::cout << "all true: " << r1 << "\n";  // 1

    // First is false -> stops immediately
    constexpr bool r2 = std::conjunction_v<
        std::is_floating_point<int>,  // false -> STOPS
        std::is_signed<int>,          // NOT evaluated
        std::is_arithmetic<int>       // NOT evaluated
    >;
    std::cout << "first false: " << r2 << "\n";  // 0

    std::cout << "\n=== Disjunction short-circuit ===\n";

    // First is true -> stops immediately
    constexpr bool r3 = std::disjunction_v<
        std::is_integral<int>,         // true -> STOPS
        std::is_floating_point<int>,   // NOT evaluated
        std::is_pointer<int>           // NOT evaluated
    >;
    std::cout << "first true: " << r3 << "\n";  // 1

    // All false -> evaluates all
    constexpr bool r4 = std::disjunction_v<
        std::is_pointer<int>,          // false
        std::is_reference<int>,        // false
        std::is_void<int>              // false
    >;
    std::cout << "all false: " << r4 << "\n";  // 0

    std::cout << "\n=== Negation ===\n";
    constexpr bool r5 = std::negation_v<std::is_pointer<int>>;
    std::cout << "negation<is_pointer<int>>: " << r5 << "\n";  // 1

    std::cout << "\n=== Safe check (short-circuit saved us) ===\n";
    std::cout << "safe_check::value = " << safe_check::value << "\n";  // 0

    return 0;
}
```

### Q3: Replace a nested `enable_if` chain with a conjunction-based predicate

The before version is something you will see a lot in older C++ codebases. It works, but when you have to compose four conditions with `&&` and `!` inside a template argument, it becomes unreadable fast. The conjunction version lets you name the compound predicate and reuse it everywhere.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === BEFORE: Nested enable_if chain (hard to read) ===
template <typename T>
std::enable_if_t<
    std::is_arithmetic_v<T> &&
    !std::is_same_v<T, bool> &&
    !std::is_same_v<T, char> &&
    std::is_signed_v<T>,
    T>
abs_ugly(T val) {
    return val < 0 ? -val : val;
}

// === AFTER: conjunction-based predicate (cleaner) ===
template <typename T>
using is_signed_numeric = std::conjunction<
    std::is_arithmetic<T>,
    std::negation<std::is_same<T, bool>>,
    std::negation<std::is_same<T, char>>,
    std::is_signed<T>
>;

template <typename T>
std::enable_if_t<is_signed_numeric<T>::value, T>
abs_clean(T val) {
    return val < 0 ? -val : val;
}

// === BEST: C++20 concept (cleanest) ===
template <typename T>
concept SignedNumeric = std::is_arithmetic_v<T>
    && !std::is_same_v<T, bool>
    && !std::is_same_v<T, char>
    && std::is_signed_v<T>;

SignedNumeric auto abs_best(SignedNumeric auto val) {
    return val < 0 ? -val : val;
}

// === Complex example: Serializable type ===
// BEFORE:
template <typename T>
std::enable_if_t<
    std::is_trivially_copyable_v<T> &&
    std::is_standard_layout_v<T> &&
    !std::is_pointer_v<T>,
    void>
serialize_ugly(const T& val) {
    std::cout << "  Serializing " << sizeof(T) << " bytes\n";
}

// AFTER: Named predicate
template <typename T>
using is_pod_serializable = std::conjunction<
    std::is_trivially_copyable<T>,
    std::is_standard_layout<T>,
    std::negation<std::is_pointer<T>>
>;

template <typename T>
std::enable_if_t<is_pod_serializable<T>::value>
serialize_clean(const T& val) {
    std::cout << "  Serializing " << sizeof(T) << " bytes\n";
}

int main() {
    std::cout << "=== Ugly nested enable_if ===\n";
    std::cout << "abs_ugly(-42) = " << abs_ugly(-42) << "\n";
    std::cout << "abs_ugly(-3.14) = " << abs_ugly(-3.14) << "\n";

    std::cout << "\n=== Clean conjunction ===\n";
    std::cout << "abs_clean(-42) = " << abs_clean(-42) << "\n";
    std::cout << "abs_clean(-3.14) = " << abs_clean(-3.14) << "\n";

    std::cout << "\n=== Best: C++20 concept ===\n";
    std::cout << "abs_best(-42) = " << abs_best(-42) << "\n";
    std::cout << "abs_best(-3.14) = " << abs_best(-3.14) << "\n";

    std::cout << "\n=== Serialization ===\n";
    int x = 42;
    double d = 3.14;
    serialize_clean(x);
    serialize_clean(d);
    // serialize_clean("hello");  // ERROR: pointer, not pod_serializable

    std::cout << "\n=== Comparison ===\n";
    std::cout << "Nested &&:     hard to read, no short-circuit benefit\n";
    std::cout << "conjunction:   named, reusable, short-circuits\n";
    std::cout << "C++20 concept: cleanest syntax, best error messages\n";

    return 0;
}
```

The `is_pod_serializable` alias is now something you can test in a `static_assert`, document in comments, and reuse across multiple function templates without repeating the conditions.

---

## Notes

- `std::conjunction<Bs...>` = logical AND with **short-circuit** instantiation. Stops at first `false`.
- `std::disjunction<Bs...>` = logical OR with **short-circuit** instantiation. Stops at first `true`.
- `std::negation<B>` = logical NOT.
- Short-circuiting prevents instantiation of later traits - avoids hard errors on invalid types.
- Use named predicates (`using is_xyz = conjunction<...>`) for readability and reuse.
- In C++20, prefer **concepts** over conjunction/enable_if - cleaner syntax and better error messages.
