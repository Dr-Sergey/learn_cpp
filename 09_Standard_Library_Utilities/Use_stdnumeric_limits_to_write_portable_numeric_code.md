# Use std::numeric_limits to write portable numeric code

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/types/numeric_limits>  

---

## Topic Overview

`std::numeric_limits<T>` (header `<limits>`) is a class template that gives you compile-time information about any arithmetic type - its range, precision, special values, and representation details. It replaces C macros like `INT_MAX` and `FLT_EPSILON` with a type-safe, generic, `constexpr` interface that actually works inside templates and compile-time contexts.

The big practical reason to prefer it over macros is genericity: if you're writing a template function that needs "the largest possible value of type T", `std::numeric_limits<T>::max()` works for any arithmetic type, while `INT_MAX` only works for `int`.

### Key Members

| Member | Type | Example (`int`) | Example (`double`) |
| --- | --- | --- | --- |
| `min()` | Minimum finite value | `-2147483648` | `2.22507e-308` (smallest positive normal) |
| `max()` | Maximum finite value | `2147483647` | `1.79769e+308` |
| `lowest()` | Most negative finite value | `-2147483648` | `-1.79769e+308` |
| `epsilon()` | Smallest value e where `1 + e != 1` | 0 | `2.22045e-16` |
| `digits` | Number of radix digits | 31 (sign bit excluded) | 53 (mantissa bits) |
| `digits10` | Decimal digits of precision | 9 | 15 |
| `is_signed` | Whether type is signed | `true` | `true` |
| `is_integer` | Whether type is integral | `true` | `false` |
| `is_iec559` | Whether IEEE 754 compliant | `false` | `true` (usually) |
| `has_infinity` | Whether `infinity()` exists | `false` | `true` |
| `infinity()` | Positive infinity | N/A | `+inf` |
| `quiet_NaN()` | Quiet NaN | N/A | `NaN` |
| `has_denorm` | Denormalized number support | N/A | `denorm_present` |

### `min()` vs `lowest()` - The Critical Distinction

This is the trap that catches almost everyone at least once. For integers the two are the same, but for floating-point they are very different:

```text
For integers:     min() == lowest()    (both are the most negative value)
For floats:       min() = smallest POSITIVE normal
                  lowest() = most NEGATIVE value

                  <--- lowest() ----- 0 -- min() -- max() --->
                  -1.8e308           0   2.2e-308   1.8e308
                                         ^ smallest positive normal
```

If you initialize a running minimum with `std::numeric_limits<double>::min()`, thinking it means "most negative", you'll get a very hard to find bug because it's actually a tiny positive number close to zero.

### Core Usage Patterns

Here are the four patterns you'll reach for most often: initializing min/max accumulators, floating-point comparison with epsilon, overflow detection, and querying type properties:

```cpp
#include <limits>
#include <iostream>
#include <iomanip>
#include <cmath>
#include <vector>
#include <algorithm>

int main() {
    // === Initialize running min/max without magic constants ===
    int running_min = std::numeric_limits<int>::max();     // Start high
    int running_max = std::numeric_limits<int>::lowest();  // Start low

    std::vector<int> data = {42, -7, 100, 3, -50};
    for (int v : data) {
        running_min = std::min(running_min, v);
        running_max = std::max(running_max, v);
    }
    std::cout << "min=" << running_min << " max=" << running_max << "\n";
    // min=-50 max=100

    // === Floating-point comparison with epsilon ===
    double a = 0.1 + 0.2;
    double b = 0.3;
    // a == b is FALSE due to floating-point representation

    // Naive epsilon check (only works near 1.0):
    bool naive_eq = std::abs(a - b) < std::numeric_limits<double>::epsilon();
    // This is too strict for most practical use

    // Better: relative epsilon with a minimum absolute threshold
    auto nearly_equal = [](double x, double y, double rel_eps = 1e-9) {
        double diff = std::abs(x - y);
        double largest = std::max(std::abs(x), std::abs(y));
        return diff <= largest * rel_eps;
    };
    std::cout << std::boolalpha << nearly_equal(a, b) << "\n"; // true

    // === Check for overflow before it occurs ===
    int x = 2'000'000'000;
    int y = 1'000'000'000;
    if (x > std::numeric_limits<int>::max() - y) {
        std::cout << "Addition would overflow!\n";
    } else {
        int sum = x + y;
        std::cout << "Sum = " << sum << "\n";
    }
    // Output: Addition would overflow!

    // === Type properties ===
    std::cout << "double digits10 = " << std::numeric_limits<double>::digits10 << "\n"; // 15
    std::cout << "float epsilon = " << std::numeric_limits<float>::epsilon() << "\n";   // 1.19209e-07
    std::cout << "int is_signed = " << std::numeric_limits<int>::is_signed << "\n";     // 1
}
```

---

## Self-Assessment

### Q1: Use `std::numeric_limits<int>::max()` to initialize a running minimum without magic constants

**Answer:**

The pattern is: initialize your accumulator to the most extreme possible value in the opposite direction, so the first real element always beats it. For a running minimum, start at `max()`. For a running maximum, start at `lowest()` (not `min()` - remember the floating-point trap):

```cpp
#include <limits>
#include <iostream>
#include <vector>
#include <algorithm>

template <typename T>
T find_minimum(const std::vector<T>& values) {
    // Initialize to the largest possible value of type T
    // so ANY element will be smaller
    T result = std::numeric_limits<T>::max();

    for (const T& v : values) {
        result = std::min(result, v);
    }
    return result;
}

template <typename T>
T find_maximum(const std::vector<T>& values) {
    // Initialize to the most negative value of type T
    // Use lowest() — works correctly for both int and double
    T result = std::numeric_limits<T>::lowest();

    for (const T& v : values) {
        result = std::max(result, v);
    }
    return result;
}

int main() {
    std::vector<int> ints = {42, -7, 100, 3, -50};
    std::cout << "int min = " << find_minimum(ints) << "\n";     // -50
    std::cout << "int max = " << find_maximum(ints) << "\n";     // 100

    std::vector<double> doubles = {3.14, -2.71, 1.618, 0.0};
    std::cout << "double min = " << find_minimum(doubles) << "\n"; // -2.71
    std::cout << "double max = " << find_maximum(doubles) << "\n"; // 3.14

    // Why not use 999999 or INT_MAX macro?
    // - Magic constants fail if data contains larger values
    // - INT_MAX macro is not generic — doesn't work in templates
    // - numeric_limits<T>::max() works for ANY arithmetic type T
    // - It's constexpr, so it can be used in compile-time contexts
}
```

**Explanation:** By initializing `result` to `numeric_limits<T>::max()`, any value in the input collection will be less than or equal to it, ensuring the first comparison always updates the minimum. This pattern is generic (works for `int`, `long long`, `float`, `double`, etc.) and avoids platform-dependent magic constants.

### Q2: Show how to detect overflow before it occurs using numeric_limits

**Answer:**

The key technique is to rearrange the check so you never perform the potentially overflowing operation in order to test it. For addition, instead of checking `a + b > MAX` (which itself overflows), check `a > MAX - b` - that subtraction is always safe:

```cpp
#include <limits>
#include <iostream>
#include <cstdint>

// Safe addition: returns true if a + b would overflow
bool would_overflow_add(int a, int b) {
    if (b > 0 && a > std::numeric_limits<int>::max() - b)
        return true;  // positive overflow
    if (b < 0 && a < std::numeric_limits<int>::min() - b)
        return true;  // negative overflow
    return false;
}

// Safe multiplication: returns true if a * b would overflow
bool would_overflow_mul(int a, int b) {
    if (a == 0 || b == 0) return false;
    if (a > 0 && b > 0 && a > std::numeric_limits<int>::max() / b) return true;
    if (a < 0 && b < 0 && a < std::numeric_limits<int>::max() / b) return true;
    if (a > 0 && b < 0 && b < std::numeric_limits<int>::min() / a) return true;
    if (a < 0 && b > 0 && a < std::numeric_limits<int>::min() / b) return true;
    return false;
}

int main() {
    int a = 2'000'000'000;
    int b = 1'000'000'000;

    if (would_overflow_add(a, b)) {
        std::cout << a << " + " << b << " would OVERFLOW\n";
    }
    // Output: 2000000000 + 1000000000 would OVERFLOW

    int c = 100'000;
    int d = 30'000;
    if (would_overflow_mul(c, d)) {
        std::cout << c << " * " << d << " would OVERFLOW\n";
    }
    // Output: 100000 * 30000 would OVERFLOW
    // Because 100000 * 30000 = 3'000'000'000 > INT_MAX (2'147'483'647)

    // Safe operation
    int e = 1'000, f = 2'000;
    if (!would_overflow_add(e, f)) {
        std::cout << e << " + " << f << " = " << (e + f) << "\n";
    }
    // Output: 1000 + 2000 = 3000

    // For unsigned, overflow wraps (well-defined), but you can still detect:
    uint32_t ua = 4'000'000'000u, ub = 1'000'000'000u;
    if (ua > std::numeric_limits<uint32_t>::max() - ub) {
        std::cout << "unsigned overflow detected\n";
    }
    // Output: unsigned overflow detected
}
```

**Explanation:** The key technique is to rearrange the overflow condition using subtraction/division instead of performing the potentially overflowing operation. For addition: instead of checking `a + b > MAX` (which requires computing `a + b`), check `a > MAX - b` (which is always safe). This pattern extends to subtraction and multiplication.

### Q3: Compare `numeric_limits<float>::epsilon()` with a practical floating-point comparison function

`epsilon()` is the smallest `float` value e such that `1.0f + e != 1.0f`. For `float`, it's approximately `1.19209e-07`. The reason this trips people up is that using raw `epsilon()` for equality comparison is almost always wrong - it only makes sense for values near 1.0.

| Approach | Code | Problem |
| --- | --- | --- |
| Raw epsilon | `abs(a-b) < epsilon()` | Too strict for values > 1.0; too loose for values near 0 |
| Scaled epsilon | `abs(a-b) < epsilon() * max(abs(a),abs(b))` | Fails near zero |
| Practical relative | `abs(a-b) <= max_val * rel_tol` | Good general-purpose approach |
| ULP-based | Compare ULP distance | Most rigorous but complex |

```cpp
#include <limits>
#include <cmath>
#include <iostream>

// BAD: Raw epsilon comparison
bool bad_equal(float a, float b) {
    return std::abs(a - b) < std::numeric_limits<float>::epsilon();
}

// GOOD: Relative + absolute tolerance comparison
bool nearly_equal(float a, float b,
                  float rel_tol = 1e-5f,
                  float abs_tol = 1e-8f)
{
    float diff = std::abs(a - b);
    // Absolute check handles values near zero
    if (diff <= abs_tol) return true;
    // Relative check handles larger values
    float largest = std::max(std::abs(a), std::abs(b));
    return diff <= largest * rel_tol;
}

int main() {
    // Case 1: Values near 1.0 — both work
    float a = 1.0f, b = 1.0f + 1e-7f;
    std::cout << "bad_equal(1.0, 1.0+1e-7):  " << bad_equal(a, b) << "\n";    // 1
    std::cout << "nearly_equal(1.0, 1.0+1e-7): " << nearly_equal(a, b) << "\n"; // 1

    // Case 2: Large values — raw epsilon FAILS
    float c = 1'000'000.0f, d = 1'000'000.125f;
    // Difference = 0.125 >> epsilon (1.19e-07) -> raw epsilon says "not equal"
    // But the relative difference is tiny: 0.125 / 1e6 = 1.25e-07
    std::cout << "bad_equal(1e6, 1e6+0.125):  " << bad_equal(c, d) << "\n";    // 0 (WRONG)
    std::cout << "nearly_equal(1e6, 1e6+0.125): " << nearly_equal(c, d) << "\n"; // 1 (correct)

    // Case 3: Values near zero — raw epsilon may be too loose
    float e = 1e-10f, f = 2e-10f;
    // Relative difference is 100%, but absolute difference is tiny
    std::cout << "nearly_equal(1e-10, 2e-10): " << nearly_equal(e, f) << "\n"; // 1 (abs_tol handles it)

    // epsilon() IS useful for understanding precision limits:
    std::cout << "float epsilon:  " << std::numeric_limits<float>::epsilon() << "\n";
    std::cout << "double epsilon: " << std::numeric_limits<double>::epsilon() << "\n";
    std::cout << "float digits10: " << std::numeric_limits<float>::digits10 << "\n"; // 6
    // digits10=6 means float reliably represents ~6 significant decimal digits
}
```

**Key takeaway:** `epsilon()` tells you the precision limit of the type, but it should NOT be used directly as a comparison tolerance. Instead, use a combined relative + absolute tolerance function that adapts to the magnitude of the values being compared.

---

## Notes

- **`min()` vs `lowest()` trap:** For floating-point, `min()` returns the smallest *positive normal* value, NOT the most negative. Use `lowest()` for the most negative representable value. For integers, they're the same.
- **`constexpr`:** All `numeric_limits` members are `constexpr` - usable in template arguments, `static_assert`, `constexpr` functions, array sizes, etc.
- **Custom types:** You can specialize `numeric_limits` for user-defined numeric types (e.g., a `BigInt` class). Just specialize the full template.
- **`is_iec559`:** If `numeric_limits<double>::is_iec559` is `true`, the type conforms to IEEE 754, guaranteeing special values (±infinity, NaN, denormals) behave as specified.
- **`quiet_NaN()` and `signaling_NaN()`:** Only meaningful when `has_quiet_NaN` / `has_signaling_NaN` is `true`. NaN != NaN by definition.
- **`digits` vs `digits10`:** `digits` is the number of bits in the mantissa (binary precision); `digits10` is the number of decimal digits that survive round-trip conversion.
- Compile with `-std=c++20 -Wall -Wextra`.
