# Understand integer literal suffixes and their effect on type deduction

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/integer_literal>  

---

## Topic Overview

Every integer literal in C++ has a type determined by its value and **suffix**. The suffix controls the type, which in turn affects `auto` deduction, template argument deduction, and overload resolution.

### Standard Suffixes

| Suffix | Type | Example |
| --- | --- | --- |
| (none) | `int` (or wider if value doesn't fit) | `42` |
| `U` or `u` | `unsigned int` | `42U` |
| `L` or `l` | `long` | `42L` |
| `LL` or `ll` | `long long` | `42LL` |
| `UL` or `ul` | `unsigned long` | `42UL` |
| `ULL` or `ull` | `unsigned long long` | `42ULL` |

### C++23 Additions

| Suffix | Type | Example |
| --- | --- | --- |
| `Z` or `z` | `std::size_t` (signed: `std::ptrdiff_t` with `z`, unsigned: `std::size_t` with `uz`) | `42uz` |

### Type Deduction with auto

`auto` deduces the exact type of the literal, so the suffix is what controls what you get:

```cpp
auto a = 42;       // int
auto b = 42L;      // long
auto c = 42LL;     // long long
auto d = 42U;      // unsigned int
auto e = 42ULL;    // unsigned long long
auto f = 0xFF;     // int (hex literal, no suffix)
auto g = 0xFFU;    // unsigned int
```

### Why Suffixes Matter

Three concrete reasons you'll run into this in real code:

```cpp
// 1. Overflow prevention
auto big = 1000000 * 1000000;      // int * int -> may overflow!
auto safe = 1000000LL * 1000000LL; // long long * long long -> no overflow

// 2. Template argument deduction
template<typename T> void f(T x);
f(42);      // T = int
f(42LL);    // T = long long
f(42U);     // T = unsigned int

// 3. Signed/unsigned comparison warnings
std::vector<int> v(10);
for (auto i = 0U; i < v.size(); ++i) {}   // no warning: unsigned < size_t
// for (auto i = 0; i < v.size(); ++i) {} // warning: signed/unsigned comparison
```

---

## Self-Assessment

### Q1: Show that `42LL` has type `long long` while `42` has type `int`, and the impact on `auto` deduction

The `std::is_same_v` checks make the compiler state the types out loud, and the overflow example shows why this matters in practice:

```cpp
#include <iostream>
#include <type_traits>

int main() {
    auto a = 42;
    auto b = 42LL;
    auto c = 42U;
    auto d = 42ULL;

    std::cout << std::boolalpha;
    std::cout << "42    is int:           " << std::is_same_v<decltype(a), int> << "\n";           // true
    std::cout << "42LL  is long long:     " << std::is_same_v<decltype(b), long long> << "\n";     // true
    std::cout << "42U   is unsigned int:  " << std::is_same_v<decltype(c), unsigned int> << "\n";  // true
    std::cout << "42ULL is unsigned long long: " << std::is_same_v<decltype(d), unsigned long long> << "\n"; // true

    // Practical impact: overflow behavior
    auto x = 2000000000 * 2;      // int overflow! UB on most platforms
    auto y = 2000000000LL * 2;    // long long -- no overflow -> 4000000000

    std::cout << "int multiply:       " << x << "\n";         // undefined behavior
    std::cout << "long long multiply: " << y << "\n";         // 4000000000

    // sizeof differs:
    std::cout << "sizeof(42):   " << sizeof(42) << "\n";     // typically 4
    std::cout << "sizeof(42LL): " << sizeof(42LL) << "\n";   // typically 8
}
```

**How it works:**

- `auto` deduces the literal's type directly - suffix determines the type.
- Without a suffix, the literal is `int` (32-bit on most platforms), which can overflow.
- `LL` suffix forces `long long` (at least 64-bit), preventing overflow for large values.

### Q2: Use `ULL` to prevent signed/unsigned comparison warnings in template arguments

`v.size()` returns `std::size_t` (unsigned), so comparing it with a plain `0` (signed `int`) triggers a compiler warning. Using an unsigned literal stops the mismatch:

```cpp
#include <iostream>
#include <vector>
#include <array>
#include <cstddef>

template<typename T, std::size_t N>
void print_array(const std::array<T, N>& arr) {
    // Using ULL avoids signed/unsigned comparison warning:
    for (auto i = 0ULL; i < arr.size(); ++i) {
        std::cout << arr[i] << " ";
    }
    std::cout << "\n";
}

template<typename T>
bool contains(const std::vector<T>& v, std::size_t index) {
    // Without U/ULL: warning about signed/unsigned comparison
    // return index < v.size();  // OK but index might be wrongly typed

    // Clean: use consistent unsigned types
    return index < v.size();   // both are unsigned -> no warning
}

int main() {
    std::array<int, 5> arr{1, 2, 3, 4, 5};
    print_array(arr);

    std::vector<int> v{10, 20, 30};

    // Compare with 0ULL instead of 0 to match size_t:
    if (v.size() > 0ULL) {
        std::cout << "Vector is not empty\n";
    }

    // Template argument with explicit unsigned type:
    constexpr auto N = 256ULL;   // unsigned long long
    std::array<char, N> buffer{};
    std::cout << "Buffer size: " << buffer.size() << "\n";

    // C++23: use uz suffix for size_t directly
    // for (auto i = 0uz; i < v.size(); ++i) {}
}
```

**How it works:**

- `v.size()` returns `std::size_t` (unsigned). Comparing with `int` (signed) triggers `-Wsign-compare`.
- Using `0ULL` or `0U` makes the comparison value unsigned, suppressing the warning.
- C++23's `uz` suffix creates `std::size_t` directly - the cleanest solution.

### Q3: Write a user-defined literal that creates a `std::chrono::seconds` value from an integer

User-defined literals for integers must take `unsigned long long` - that's why the integer suffixes always produce the widest unsigned type internally. Here you can see the pattern in action:

```cpp
#include <chrono>
#include <iostream>

// User-defined literal: integer -> chrono::seconds
constexpr std::chrono::seconds operator""_sec(unsigned long long n) {
    return std::chrono::seconds(n);
}

// Another: integer -> chrono::milliseconds
constexpr std::chrono::milliseconds operator""_ms(unsigned long long n) {
    return std::chrono::milliseconds(n);
}

int main() {
    auto timeout = 30_sec;
    auto delay = 500_ms;

    std::cout << "Timeout: " << timeout.count() << " seconds\n";
    std::cout << "Delay:   " << delay.count() << " milliseconds\n";

    // Arithmetic works:
    auto total = timeout + std::chrono::duration_cast<std::chrono::seconds>(delay);
    std::cout << "Total: " << total.count() << " seconds\n";

    // Note: the standard library already provides these in <chrono>:
    using namespace std::chrono_literals;
    auto std_timeout = 30s;    // std::chrono::seconds
    auto std_delay = 500ms;    // std::chrono::milliseconds
    auto std_micro = 100us;    // std::chrono::microseconds

    std::cout << "Std: " << std_timeout.count() << "s, "
              << std_delay.count() << "ms\n";
}
```

**How it works:**

- UDL for integers must take `unsigned long long` as the parameter type.
- The returned `std::chrono::seconds` wraps the integer value.
- The standard `<chrono>` library already provides `s`, `ms`, `us`, `ns`, `min`, `h` literals via `std::chrono_literals`.

---

## Notes

- Avoid lowercase `l` suffix (`42l`) - it looks like `41` (digit one). Use uppercase `42L`.
- Hex literals without suffix follow different type rules: the compiler tries `int`, then `unsigned int`, then `long`, etc.
- `0` is `int`, not `unsigned` - this matters for overload resolution.
- C++23 `uz` suffix is the definitive fix for the `size_t` comparison problem.
- Binary literals (`0b1010`) follow the same suffix rules. `0b1010ULL` is `unsigned long long`.
