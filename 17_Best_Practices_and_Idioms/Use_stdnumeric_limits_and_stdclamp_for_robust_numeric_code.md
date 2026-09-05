# Use std::numeric_limits and std::clamp for robust numeric code

**Category:** Best Practices and Idioms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/clamp>  

---

## Topic Overview

`std::clamp(val, lo, hi)` constrains a value to `[lo, hi]`. `std::numeric_limits<T>` provides type-specific min/max/epsilon values. Together they make numeric code safe and portable - no more hardcoded `255` magic numbers or platform-dependent `INT_MAX` macros.

```cpp
#include <algorithm>
#include <limits>
int safe = std::clamp(input, 0, 255);  // pixel value: 0-255
auto max_int = std::numeric_limits<int>::max();  // 2147483647
```

---

## Self-Assessment

### Q1: Use `std::clamp` to constrain values

`std::clamp` is the clean replacement for the common `std::min(hi, std::max(lo, val))` pattern. It is easier to read and has `constexpr` support, so it works in compile-time contexts too. The most important practical use is clamping before a narrowing conversion - always clamp first, then cast.

```cpp
#include <algorithm>
#include <iostream>
#include <cstdint>

int main() {
    // Basic usage: clamp(value, low, high)
    int pixel = std::clamp(300, 0, 255);  // 255 (capped)
    std::cout << "pixel = " << pixel << '\n';

    int temp = std::clamp(-50, -40, 50);  // -40 (floored)
    std::cout << "temp = " << temp << '\n';

    double pct = std::clamp(1.5, 0.0, 1.0);  // 1.0 (capped)
    std::cout << "pct = " << pct << '\n';

    // Practical: safe narrow conversion
    int big = 70000;
    uint8_t byte = static_cast<uint8_t>(
        std::clamp(big, 0, 255)  // clamp BEFORE cast!
    );
    std::cout << "byte = " << static_cast<int>(byte) << '\n';

    // Array of values
    int values[] = {-10, 50, 200, 300, 0, 128};
    std::cout << "Clamped: ";
    for (int v : values)
        std::cout << std::clamp(v, 0, 255) << ' ';
    std::cout << '\n';
}
// Expected output:
// pixel = 255
// temp = -40
// pct = 1
// byte = 255
// Clamped: 0 50 200 255 0 128
```

Notice that `std::clamp` returns a reference to one of its arguments (the value, lo, or hi), so there is no copy overhead for cheap types. For expensive types, this matters.

### Q2: Show NaN problems with manual min/max

Floating-point code has a subtle trap: `std::min` and `std::max` are not commutative when NaN is involved. Which argument is NaN changes whether you get NaN or the other value back, depending on how `operator<` propagates NaN. This makes the naive `max(lo, min(val, hi))` pattern unreliable for float inputs.

```cpp
#include <algorithm>
#include <cmath>
#include <iostream>
#include <limits>

int main() {
    double val = std::numeric_limits<double>::quiet_NaN();
    double lo = 0.0, hi = 100.0;

    // Manual min/max: BROKEN with NaN!
    // std::min(NaN, x) is UNSPECIFIED (depends on argument order)
    double r1 = std::max(lo, std::min(val, hi));  // NaN propagation!
    double r2 = std::max(lo, std::min(hi, val));   // Different result!

    std::cout << "min/max attempt 1: " << r1 << '\n';
    std::cout << "min/max attempt 2: " << r2 << '\n';

    // std::clamp: also undefined for NaN, but at least consistent
    // The solution: check for NaN first!
    auto safe_clamp = [](double v, double lo, double hi) -> double {
        if (std::isnan(v)) return lo;  // default to low bound
        return std::clamp(v, lo, hi);
    };

    std::cout << "safe_clamp(NaN) = " << safe_clamp(val, lo, hi) << '\n';
    std::cout << "safe_clamp(50)  = " << safe_clamp(50.0, lo, hi) << '\n';

    // numeric_limits useful constants:
    std::cout << "epsilon = " << std::numeric_limits<double>::epsilon() << '\n';
    std::cout << "inf     = " << std::numeric_limits<double>::infinity() << '\n';
}
// Expected output:
// min/max attempt 1: 0 (or NaN, implementation-defined)
// min/max attempt 2: 100 (or NaN, implementation-defined)
// safe_clamp(NaN) = 0
// safe_clamp(50)  = 50
// epsilon = 2.22045e-16
// inf     = inf
```

The standard mandates that `std::clamp`'s behavior is undefined when the input is NaN, so wrapping it in an `isnan` check is the correct approach for any code that may receive floating-point values from user input or external data.

### Q3: Use `numeric_limits` for signed/unsigned overflow detection

Signed integer overflow is undefined behavior in C++. The only safe way to check for it is before the addition happens - once you add and overflow, the compiler has already been given permission to assume it never occurred. `std::numeric_limits` gives you the portable way to get the exact boundary values for any type.

```cpp
#include <iostream>
#include <limits>
#include <type_traits>

template<typename T>
bool safe_add(T a, T b, T& result) {
    if constexpr (std::numeric_limits<T>::is_signed) {
        // Signed overflow: UB in C++, must check BEFORE adding
        if (b > 0 && a > std::numeric_limits<T>::max() - b) return false;
        if (b < 0 && a < std::numeric_limits<T>::min() - b) return false;
    } else {
        // Unsigned overflow: well-defined (wraps), but may be unwanted
        if (a > std::numeric_limits<T>::max() - b) return false;
    }
    result = a + b;
    return true;
}

template<typename T>
void print_limits() {
    std::cout << "Type: " << (std::numeric_limits<T>::is_signed ? "signed" : "unsigned")
              << ", bits: " << std::numeric_limits<T>::digits
              << ", min: " << +std::numeric_limits<T>::min()
              << ", max: " << +std::numeric_limits<T>::max() << '\n';
}

int main() {
    print_limits<int>();
    print_limits<unsigned>();
    print_limits<int8_t>();
    print_limits<uint8_t>();

    int result;
    if (safe_add(2'000'000'000, 1'000'000'000, result))
        std::cout << "Sum: " << result << '\n';
    else
        std::cout << "Overflow detected!\n";

    unsigned uresult;
    if (safe_add(4'000'000'000u, 1'000'000'000u, uresult))
        std::cout << "Sum: " << uresult << '\n';
    else
        std::cout << "Unsigned overflow detected!\n";
}
// Expected output:
// Type: signed, bits: 31, min: -2147483648, max: 2147483647
// Type: unsigned, bits: 32, min: 0, max: 4294967295
// Type: signed, bits: 7, min: -128, max: 127
// Type: unsigned, bits: 8, min: 0, max: 255
// Overflow detected!
// Unsigned overflow detected!
```

The `if constexpr` branch on `is_signed` lets the same function template handle both signed and unsigned types correctly, since the overflow semantics differ between them.

---

## Notes

- `std::clamp` is in `<algorithm>` since C++17, and `constexpr`.
- `std::clamp` requires `lo <= hi` - violating this is UB.
- `std::clamp` with NaN is undefined; always check `std::isnan` first for floats.
- Prefer `std::numeric_limits<T>::max()` over macros like `INT_MAX`.
- `numeric_limits` has `constexpr` members: usable in `static_assert` and `if constexpr`.
