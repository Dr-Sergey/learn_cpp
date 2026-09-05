# Use Fold Expressions (C++17) to Operate on Parameter Packs

**Category:** Templates and Generic Programming  
**Standard:** C++17  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#fold-expressions>  

---

## Topic Overview

### What Are Fold Expressions

Before C++17, operating on a parameter pack required recursive templates - one overload for the base case, one for the recursive step, and a fair amount of boilerplate. Fold expressions let you **reduce a parameter pack** with an operator in a single expression:

```cpp
template <typename... Args>
auto sum(Args... args) {
    return (args + ...);  // unary right fold
}
sum(1, 2, 3, 4);  // -> 1 + (2 + (3 + 4)) = 10
```

That's the whole pack reduction, in one line, with no recursion.

### The Four Forms

The four forms vary along two axes: unary vs binary (whether you supply an initial value) and left vs right (which end the folding starts from). The difference between left and right matters for non-commutative operators:

| Form | Syntax | Expansion | Name |
| --- | --- | --- | --- |
| Unary right fold | `(pack op ...)` | `a1 op (a2 op (... op aN))` | Right-associative |
| Unary left fold | `(... op pack)` | `((a1 op a2) op ...) op aN` | Left-associative |
| Binary right fold | `(pack op ... op init)` | `a1 op (a2 op (... op (aN op init)))` | Right with initial value |
| Binary left fold | `(init op ... op pack)` | `(((init op a1) op a2) op ...) op aN` | Left with initial value |

### Supported Operators

All 32 binary operators work with fold expressions:
`+  -  *  /  %  ^  &  |  =  <  >  <<  >>  +=  -=  *=  /=  %=  ^=  &=  |=  <<=  >>=  ==  !=  <=  >=  &&  ||  ,  .*  ->*`

### Empty Pack Behavior

For unary folds, an empty pack is only valid for `&&` (-> `true`), `||` (-> `false`), and `,` (-> `void()`). Other operators with empty packs are ill-formed - use a binary fold with an init value to handle the empty case:

```cpp
(args + ...);        // ERROR if pack is empty
(args + ... + 0);    // OK: binary fold, returns 0 for empty pack
```

---

## Self-Assessment

### Q1: Write a variadic sum using a unary right fold: `(args + ... + 0)`

The binary form with `+ 0` is the version you actually want to use in production - it handles the empty pack and documents the identity element clearly. The other variants here show how the same idea extends to products, strings, and booleans:

```cpp
#include <iostream>
#include <string>

// === Unary right fold ===
template <typename... Args>
auto sum_unary(Args... args) {
    return (args + ...);  // Unary right fold: a1 + (a2 + (a3 + a4))
    // Empty pack -> compile error!
}

// === Binary right fold with init value ===
template <typename... Args>
auto sum(Args... args) {
    return (args + ... + 0);  // Binary right fold: a1 + (a2 + (... + (aN + 0)))
    // Empty pack -> returns 0
}

// === Product fold ===
template <typename... Args>
auto product(Args... args) {
    return (args * ... * 1);  // Binary right fold with identity element
}

// === String concatenation fold ===
template <typename... Args>
std::string concat(Args&&... args) {
    return (std::string{} + ... + std::string(args));
    // Binary left fold: ((("" + a1) + a2) + a3)
}

// === Boolean folds ===
template <typename... Args>
bool all_true(Args... args) {
    return (args && ...);  // Unary: empty pack -> true
}

template <typename... Args>
bool any_true(Args... args) {
    return (args || ...);  // Unary: empty pack -> false
}

int main() {
    std::cout << "=== Sum (binary right fold) ===\n";
    std::cout << "sum(1,2,3,4) = " << sum(1, 2, 3, 4) << "\n";      // 10
    std::cout << "sum(1.5, 2.5) = " << sum(1.5, 2.5) << "\n";       // 4.0
    std::cout << "sum() = " << sum() << "\n";                         // 0 (empty)

    std::cout << "\n=== Product ===\n";
    std::cout << "product(2,3,4) = " << product(2, 3, 4) << "\n";    // 24
    std::cout << "product() = " << product() << "\n";                  // 1

    std::cout << "\n=== String concatenation ===\n";
    std::cout << concat("hello", " ", "world") << "\n";   // "hello world"

    std::cout << "\n=== Boolean folds ===\n";
    std::cout << "all_true(1,1,1) = " << all_true(true, true, true) << "\n";   // 1
    std::cout << "all_true(1,0,1) = " << all_true(true, false, true) << "\n";  // 0
    std::cout << "any_true(0,0,1) = " << any_true(false, false, true) << "\n"; // 1
    std::cout << "all_true() = " << all_true() << "\n";   // 1 (vacuous truth)
    std::cout << "any_true() = " << any_true() << "\n";   // 0

    return 0;
}
```

### Q2: Explain the difference between left fold, right fold, and their unary vs binary forms

For commutative, associative operations like addition, left and right folds give the same result. The difference only matters when the operator is not associative - subtraction is the clearest example:

| Form | Syntax | Expansion Example (a,b,c) | Associativity |
| --- | --- | --- | --- |
| **Unary left fold** | `(... op pack)` | `(a op b) op c` | Left |
| **Unary right fold** | `(pack op ...)` | `a op (b op c)` | Right |
| **Binary left fold** | `(init op ... op pack)` | `((init op a) op b) op c` | Left |
| **Binary right fold** | `(pack op ... op init)` | `a op (b op (c op init))` | Right |

**When does it matter?** Watch the subtraction case - the two folds give different answers because subtraction isn't associative:

```cpp
#include <iostream>

// For commutative + associative ops (like +), left vs right gives same result
template<typename... Args>
auto sum_left(Args... args) { return (... + args); }   // left fold

template<typename... Args>
auto sum_right(Args... args) { return (args + ...); }   // right fold

// For non-associative ops (like -), order matters!
template<typename... Args>
auto sub_left(Args... args) { return (... - args); }
// sub_left(1,2,3) -> (1 - 2) - 3 = -4

template<typename... Args>
auto sub_right(Args... args) { return (args - ...); }
// sub_right(1,2,3) -> 1 - (2 - 3) = 2

// For << (stream output), LEFT fold is natural:
template<typename... Args>
void print_left(Args&&... args) {
    (std::cout << ... << args);  // binary left fold with std::cout as init
    // Expands to: ((std::cout << a1) << a2) << a3
    std::cout << "\n";
}

int main() {
    std::cout << "=== Sum (associative — same result) ===\n";
    std::cout << "left:  " << sum_left(1, 2, 3) << "\n";   // 6
    std::cout << "right: " << sum_right(1, 2, 3) << "\n";  // 6

    std::cout << "\n=== Subtraction (non-associative — different!) ===\n";
    std::cout << "left (1-2)-3:  " << sub_left(1, 2, 3) << "\n";   // -4
    std::cout << "right 1-(2-3): " << sub_right(1, 2, 3) << "\n";  // 2

    std::cout << "\n=== Stream output (left fold) ===\n";
    print_left("x=", 42, " y=", 3.14);  // x=42 y=3.14

    return 0;
}
```

### Q3: Use a fold expression to print all arguments with spaces between them

Printing with separators is slightly tricky because you don't want a leading or trailing space. The four methods below show different ways to handle this, each with a different trade-off between simplicity and flexibility:

```cpp
#include <iostream>
#include <string>

// === Method 1: Comma fold with lambda trick ===
template <typename... Args>
void print_spaced(Args&&... args) {
    bool first = true;
    // Uses comma fold: (expr, ...)
    ((std::cout << (first ? "" : " ") << args, first = false), ...);
    std::cout << "\n";
}

// === Method 2: Comma fold with helper ===
template <typename T>
void emit(const T& val, bool& first) {
    if (!first) std::cout << " ";
    std::cout << val;
    first = false;
}

template <typename... Args>
void print_v2(Args&&... args) {
    bool first = true;
    (emit(args, first), ...);  // Comma fold over function calls
    std::cout << "\n";
}

// === Method 3: Print first, then fold the rest with space prefix ===
template <typename First, typename... Rest>
void print_v3(First&& first, Rest&&... rest) {
    std::cout << first;
    ((std::cout << " " << rest), ...);  // Comma fold: each gets space prefix
    std::cout << "\n";
}

// === Method 4: Fold to build a string ===
template <typename... Args>
std::string to_string_spaced(Args&&... args) {
    std::string result;
    bool first = true;
    auto append = [&](const auto& val) {
        if (!first) result += " ";
        result += std::to_string(val);
        first = false;
    };
    (append(args), ...);
    return result;
}

int main() {
    std::cout << "=== Method 1: Comma fold with lambda ===\n";
    print_spaced(1, 2.5, "hello", 'X', 42);
    // Output: 1 2.5 hello X 42

    std::cout << "\n=== Method 2: Helper function ===\n";
    print_v2("a", "b", "c", "d");
    // Output: a b c d

    std::cout << "\n=== Method 3: First + rest ===\n";
    print_v3(10, 20, 30, 40);
    // Output: 10 20 30 40

    std::cout << "\n=== Method 4: Build string ===\n";
    std::cout << to_string_spaced(1, 2, 3) << "\n";
    // Output: 1 2 3

    return 0;
}
```

---

## Notes

- Fold expressions (C++17) eliminate recursive template expansion for parameter pack operations.
- Four forms: unary/binary x left/right. Use **binary** to handle empty packs.
- **Left fold**: `(... op pack)` - natural for stream `<<` and left-associative ops.
- **Right fold**: `(pack op ...)` - natural for right-associative construction.
- **Comma fold** `(expr, ...)` is the most versatile - lets you call functions or execute statements per argument.
- For printing with separators, use the "first + rest" pattern or a `bool first` flag with comma fold.
