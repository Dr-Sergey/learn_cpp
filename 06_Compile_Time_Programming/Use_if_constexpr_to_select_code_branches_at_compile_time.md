# Use `if constexpr` to Select Code Branches at Compile Time

**Category:** Compile Time Programming  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/if>  

---

## Topic Overview

### What Is `if constexpr`

`if constexpr` evaluates a compile-time boolean condition and **discards the non-taken branch entirely**. The discarded branch is not instantiated, not type-checked for template-dependent expressions, and generates no code. It's as if that branch were erased by a preprocessor - except it's properly scoped and syntactically structured.

Here's a quick example to make it concrete:

```cpp
template <typename T>
auto process(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;          // Only compiled for integral T
    } else if constexpr (std::is_floating_point_v<T>) {
        return value * 2.0;        // Only compiled for floating-point T
    } else {
        return std::string(value); // Only compiled for other T
    }
}
```

Each branch is only compiled when it's actually selected. The compiler doesn't even try to parse `std::string(value)` when `T` is `int`.

### Key Property: Branch Discarding

The discarded branch can contain code that **would not compile** for the given `T`. This is the main thing that makes `if constexpr` different from a regular `if` - with a regular `if`, both branches must compile for every instantiation of the template, even if the condition makes one of them unreachable:

```cpp
template <typename T>
void print_size(const T& v) {
    if constexpr (requires { v.size(); }) {
        std::cout << "size = " << v.size();  // Only compiled if T has .size()
    } else {
        std::cout << "no size";              // Only compiled otherwise
    }
}
```

Without `if constexpr`, this would be a hard error - `v.size()` doesn't exist for `int`. With `if constexpr`, the compiler discards whichever branch doesn't apply and never tries to compile it.

### `if constexpr` vs Regular `if`

| Feature | `if constexpr` | `if` |
| --- | --- | --- |
| Condition | Must be constexpr | Any expression |
| Non-taken branch | Discarded - not compiled | Compiled - must be valid |
| Use in templates | Enables branch-specific invalid code | Both branches must compile |
| Code generation | Zero-cost: only taken branch exists | Both branches in binary (optimizer may remove) |

---

## Self-Assessment

### Q1: Write a `to_string` function that uses `if constexpr` to handle `int` vs `float` vs `std::string` differently

This function handles several types, including the special case of `bool`. Notice how nesting `if constexpr` inside another `if constexpr` is perfectly legal - the inner check is only reached when the outer condition is true.

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <vector>

// === Universal to_string using if constexpr ===
template <typename T>
std::string to_string(const T& value) {
    if constexpr (std::is_same_v<T, std::string>) {
        return value;                              // Already a string
    } else if constexpr (std::is_same_v<T, const char*> || std::is_same_v<T, char>) {
        return std::string(1, '\0') + value;       // char* or char
        // Actually, let's be more straightforward:
    } else if constexpr (std::is_integral_v<T>) {
        if constexpr (std::is_same_v<T, bool>) {
            return value ? "true" : "false";       // Bool special case
        } else {
            return "int:" + std::to_string(value);
        }
    } else if constexpr (std::is_floating_point_v<T>) {
        return "float:" + std::to_string(value);
    } else if constexpr (std::is_pointer_v<T>) {
        return "ptr:" + std::to_string(reinterpret_cast<std::uintptr_t>(value));
    } else {
        // This branch is only instantiated for types matching none of the above
        static_assert(!std::is_same_v<T, T>, "Unsupported type for to_string");
    }
}

// === Specialization for containers ===
template <typename T>
std::string container_to_string(const std::vector<T>& v) {
    std::string result = "[";
    for (std::size_t i = 0; i < v.size(); ++i) {
        if (i > 0) result += ", ";
        if constexpr (std::is_arithmetic_v<T>) {
            result += std::to_string(v[i]);
        } else if constexpr (std::is_same_v<T, std::string>) {
            result += "\"" + v[i] + "\"";
        } else {
            result += "?";
        }
    }
    return result + "]";
}

int main() {
    std::cout << to_string(42) << "\n";           // int:42
    std::cout << to_string(3.14) << "\n";         // float:3.140000
    std::cout << to_string(true) << "\n";         // true
    std::cout << to_string(std::string("hi")) << "\n";  // hi

    std::vector<int> vi = {1, 2, 3};
    std::cout << container_to_string(vi) << "\n"; // [1, 2, 3]

    std::vector<std::string> vs = {"a", "b"};
    std::cout << container_to_string(vs) << "\n"; // ["a", "b"]

    return 0;
}
```

**Expected output:**

```text
int:42
float:3.140000
true
hi
[1, 2, 3]
["a", "b"]
```

### Q2: Explain why `if constexpr` discards the non-taken branch even if it would fail to compile

This is the concept that's hardest to internalize coming from regular C++. With a regular `if` in a template, the condition is evaluated at runtime, but the template code is compiled for all `T` at instantiation time - so both branches have to be valid code for every `T`. With `if constexpr`, the condition is evaluated at compile time and the losing branch is genuinely never compiled.

The comment in the example explains the subtlety: non-dependent names (things that don't involve `T`) are still syntax-checked even in discarded branches. This is why `static_assert(false)` always fires even inside a discarded branch - `false` doesn't depend on `T`. The "dependent false" idiom (`!std::is_same_v<T, T>`) works around this by making the condition formally dependent on the template parameter.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === The problem WITHOUT if constexpr ===
// This would FAIL for int because int has no .size() method:
/*
template <typename T>
void print_info(const T& v) {
    if (std::is_class_v<T>) {
        std::cout << "size = " << v.size();  // ERROR for T=int: no .size()
    } else {
        std::cout << "value = " << v;
    }
}
*/
// WHY: Regular 'if' requires BOTH branches to be valid for all T.
// The condition is a runtime check, but template instantiation
// happens at compile time - both branches are compiled.

// === The solution WITH if constexpr ===
template <typename T>
void print_info(const T& v) {
    if constexpr (requires { v.size(); }) {
        // This branch is DISCARDED (not compiled) when T = int
        std::cout << "Has .size(): " << v.size() << "\n";
    } else {
        // This branch is DISCARDED when T = std::string
        std::cout << "Scalar value: " << v << "\n";
    }
}

// === How it works internally ===
// When the compiler instantiates print_info<int>:
//   1. Evaluates: requires { v.size(); } -> false (int has no .size())
//   2. DISCARDS the if-branch entirely
//   3. Only compiles the else-branch: std::cout << v;
//
// When the compiler instantiates print_info<std::string>:
//   1. Evaluates: requires { v.size(); } -> true
//   2. DISCARDS the else-branch
//   3. Only compiles the if-branch: std::cout << v.size();

// === The "dependent false" idiom for static_assert ===
template <typename T>
void must_handle(T) {
    if constexpr (std::is_integral_v<T>) {
        // handle integral
    } else if constexpr (std::is_floating_point_v<T>) {
        // handle floating
    } else {
        // This static_assert is only triggered if this branch is taken
        // If we wrote: static_assert(false, "...") - it would ALWAYS fire
        // because false is not dependent on T
        static_assert(!std::is_same_v<T, T>, "Unhandled type");
    }
}

int main() {
    print_info(42);                           // Scalar value: 42
    print_info(std::string("hello"));         // Has .size(): 5
    print_info(3.14);                         // Scalar value: 3.14

    std::cout << "\n=== Why Discarding Works ===\n";
    std::cout << "Template instantiation compiles ALL code in the template body.\n";
    std::cout << "Regular 'if': both branches compiled -> v.size() fails for int.\n";
    std::cout << "'if constexpr': only the matching branch is compiled.\n";
    std::cout << "The discarded branch is syntax-checked but NOT instantiated.\n";
    std::cout << "Non-dependent names are still checked even in discarded branches.\n";

    return 0;
}
```

**Expected output:**

```text
Scalar value: 42
Has .size(): 5
Scalar value: 3.14

=== Why Discarding Works ===
Template instantiation compiles ALL code in the template body.
Regular 'if': both branches compiled -> v.size() fails for int.
'if constexpr': only the matching branch is compiled.
The discarded branch is syntax-checked but NOT instantiated.
Non-dependent names are still checked even in discarded branches.
```

### Q3: Show a recursive template function rewritten as a non-recursive function using `if constexpr`

One of the biggest practical uses of `if constexpr` is replacing the classic "recursive template + base-case overload" pattern. The old pattern requires two separate function templates with `enable_if` conditions to dispatch between them. The `if constexpr` version puts everything in a single function and controls recursion with a compile-time check on `sizeof...(Rest)` or the current index.

The comparison here is instructive: the "before" version uses `enable_if` return types which are notoriously hard to read. The "after" version reads like normal code.

```cpp
#include <iostream>
#include <tuple>
#include <string>

// ============================================================
// BEFORE: Recursive template (C++11/14 style)
// ============================================================

// Base case
template <std::size_t I = 0, typename... Ts>
typename std::enable_if<I == sizeof...(Ts), void>::type
print_tuple_recursive(const std::tuple<Ts...>&) {
    std::cout << "\n";
}

// Recursive case
template <std::size_t I = 0, typename... Ts>
typename std::enable_if<I < sizeof...(Ts), void>::type
print_tuple_recursive(const std::tuple<Ts...>& t) {
    if (I > 0) std::cout << ", ";
    std::cout << std::get<I>(t);
    print_tuple_recursive<I + 1>(t);
}

// ============================================================
// AFTER: Non-recursive using if constexpr (C++17)
// ============================================================

template <std::size_t I = 0, typename... Ts>
void print_tuple_constexpr(const std::tuple<Ts...>& t) {
    if constexpr (I < sizeof...(Ts)) {
        if constexpr (I > 0) std::cout << ", ";
        std::cout << std::get<I>(t);
        print_tuple_constexpr<I + 1>(t);  // Not truly recursive - terminates via if constexpr
    } else {
        std::cout << "\n";
    }
}

// ============================================================
// Another example: compile-time sum of variadic args
// ============================================================

// BEFORE: recursive with base case overload
template <typename T>
constexpr T sum_recursive(T value) { return value; }

template <typename T, typename... Rest>
constexpr T sum_recursive(T first, Rest... rest) {
    return first + sum_recursive(rest...);
}

// AFTER: fold expression (C++17) - even simpler
template <typename... Args>
constexpr auto sum_fold(Args... args) {
    return (args + ...);
}

// AFTER: if constexpr approach
template <typename T, typename... Rest>
constexpr auto sum_if_constexpr(T first, Rest... rest) {
    if constexpr (sizeof...(rest) == 0) {
        return first;
    } else {
        return first + sum_if_constexpr(rest...);
    }
}

// ============================================================
// Type name printer: recursive -> if constexpr
// ============================================================
template <typename T, typename... Rest>
void print_types() {
    // Print current type
    if constexpr (std::is_integral_v<T>) {
        std::cout << "integral";
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "floating";
    } else {
        std::cout << "other";
    }

    // Continue with remaining types
    if constexpr (sizeof...(Rest) > 0) {
        std::cout << ", ";
        print_types<Rest...>();
    } else {
        std::cout << "\n";
    }
}

int main() {
    auto t = std::make_tuple(1, 3.14, std::string("hello"), 'X');

    std::cout << "Recursive:     (";
    print_tuple_recursive(t);

    std::cout << "if constexpr:  (";
    print_tuple_constexpr(t);

    // Sum
    constexpr auto s1 = sum_recursive(1, 2, 3, 4, 5);
    constexpr auto s2 = sum_fold(1, 2, 3, 4, 5);
    constexpr auto s3 = sum_if_constexpr(1, 2, 3, 4, 5);
    static_assert(s1 == 15 && s2 == 15 && s3 == 15);

    std::cout << "Sum: " << s1 << " " << s2 << " " << s3 << "\n";

    // Type printer
    std::cout << "Types: ";
    print_types<int, double, std::string, char>();

    return 0;
}
```

**Expected output:**

```text
Recursive:     (1, 3.14, hello, X
if constexpr:  (1, 3.14, hello, X
Sum: 15 15 15
Types: integral, floating, other, integral
```

---

## Notes

- `if constexpr` requires a compile-time boolean condition.
- The discarded branch is syntax-checked but not instantiated for template-dependent expressions.
- Non-dependent expressions in discarded branches ARE still checked (e.g., `static_assert(false)` always fires).
- Use the "dependent false" idiom: `static_assert(!std::is_same_v<T,T>, "msg")` for catch-all error branches.
- `if constexpr` replaces many SFINAE patterns and recursive template specializations.
- Prefer fold expressions over `if constexpr` recursion for simple variadic operations.
