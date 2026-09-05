# Use std::midpoint and std::lerp for safe numeric interpolation

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/midpoint>  

---

## Topic Overview

C++20 introduces `std::midpoint` and `std::lerp` in `<numeric>` and `<cmath>` respectively. They exist because the "obvious" implementations of both operations have subtle but real bugs - integer overflow for midpoint, and precision loss or incorrect endpoint behavior for lerp. These functions give you the mathematically correct result with no extra effort.

### The Problem with Naive Approaches

The bugs here are easy to miss because the naive code looks completely reasonable. The problem only surfaces at large values:

```cpp
// DANGER: signed integer overflow -> undefined behavior
int mid_bad(int a, int b) {
    return (a + b) / 2;  // UB when a + b > INT_MAX
}
// Example: a = 2'000'000'000, b = 2'000'000'000
// a + b = 4'000'000'000 -> overflows int32 -> UB

// DANGER: floating-point precision loss
double mid_bad(double a, double b) {
    return (a + b) / 2.0;  // Loss of precision for very large values
}
```

### std::midpoint - Signature and Behavior

```cpp
#include <numeric>

// For integers: rounds toward a when the true midpoint is between two integers
constexpr int    midpoint(int a, int b);
constexpr long   midpoint(long a, long b);
// Also: unsigned types, pointers

// For floating-point:
constexpr float  midpoint(float a, float b);
constexpr double midpoint(double a, double b);
```

| Property | Detail |
| --- | --- |
| Header | `<numeric>` |
| Integer overflow | Impossible - uses subtraction-based algorithm |
| Rounding (integers) | Rounds toward `a` (first argument) |
| Floating-point | At most one ULP error; no overflow for finite inputs |
| Constexpr | Yes |
| Pointer version | `midpoint(p, q)` returns pointer to midpoint element |

### std::lerp - Signature and Behavior

```cpp
#include <cmath>

// Linear interpolation: lerp(a, b, t) = a + t*(b - a)
constexpr float  lerp(float a, float b, float t);
constexpr double lerp(double a, double b, double t);
```

| Property | Detail |
| --- | --- |
| Header | `<cmath>` |
| Formula | `a + t * (b - a)` (but more precise) |
| `t = 0` | Returns exactly `a` |
| `t = 1` | Returns exactly `b` |
| `t = 0.5` | Returns `midpoint(a, b)` equivalent |
| Monotonic | Guaranteed: `lerp(a,b,t2) >= lerp(a,b,t1)` when `t2 >= t1` |
| Deterministic | `lerp(a,b,1) == b` exactly (not just approximately) |
| Constexpr | Yes |
| Extrapolation | Works for `t < 0` and `t > 1` |

### Core Examples

Here's a quick tour of both functions together. Notice the pointer version of `midpoint` and the exact endpoint guarantee of `lerp`:

```cpp
#include <numeric>
#include <cmath>
#include <iostream>
#include <climits>

int main() {
    // === midpoint: integers ===
    int a = INT_MAX - 1;  // 2147483646
    int b = INT_MAX;      // 2147483647
    std::cout << std::midpoint(a, b) << "\n"; // 2147483646 (rounds toward a)
    std::cout << std::midpoint(b, a) << "\n"; // 2147483647 (rounds toward b now)

    // Extreme case that would overflow with (a+b)/2
    int x = 2'000'000'000, y = 2'000'000'000;
    std::cout << std::midpoint(x, y) << "\n"; // 2000000000 — correct, no UB

    // Negative range
    std::cout << std::midpoint(-10, 10) << "\n"; // 0

    // === midpoint: floating-point ===
    std::cout << std::midpoint(1.0, 3.0) << "\n";     // 2
    std::cout << std::midpoint(1e308, 1e308) << "\n";  // 1e308 (no overflow)

    // === midpoint: pointers ===
    int arr[] = {10, 20, 30, 40, 50};
    int* mid_ptr = std::midpoint(arr, arr + 4);
    std::cout << *mid_ptr << "\n"; // 30 (arr[2])

    // === lerp ===
    double start = 0.0, end = 100.0;
    std::cout << std::lerp(start, end, 0.0)  << "\n"; // 0
    std::cout << std::lerp(start, end, 0.25) << "\n"; // 25
    std::cout << std::lerp(start, end, 0.5)  << "\n"; // 50
    std::cout << std::lerp(start, end, 1.0)  << "\n"; // 100 (exactly)
    std::cout << std::lerp(start, end, 1.5)  << "\n"; // 150 (extrapolation)
}
```

---

## Self-Assessment

### Q1: Use std::midpoint to compute the middle of two integers without overflow (vs (a+b)/2)

**Answer:**

The binary search use case is probably the most common place people write this bug without realizing it. `std::midpoint(lo, hi)` is cleaner than `lo + (hi - lo) / 2` and has the same safety guarantees:

```cpp
#include <numeric>
#include <iostream>
#include <climits>
#include <cassert>

int main() {
    int a = 2'000'000'000;
    int b = 1'800'000'000;
    // a + b = 3'800'000'000 -> overflows int32 (max 2'147'483'647) -> UB!

    // Naive approach — UNDEFINED BEHAVIOR:
    // int bad = (a + b) / 2;  // DON'T DO THIS

    // Safe approach:
    int safe = std::midpoint(a, b);
    std::cout << "midpoint(" << a << ", " << b << ") = " << safe << "\n";
    // Output: midpoint(2000000000, 1800000000) = 1900000000

    // Verify: no overflow occurred, result is mathematically correct
    // True midpoint: (2e9 + 1.8e9) / 2 = 1.9e9

    // Rounding behavior: rounds toward first argument
    std::cout << std::midpoint(1, 4) << "\n"; // 2 (rounds toward 1)
    std::cout << std::midpoint(4, 1) << "\n"; // 3 (rounds toward 4)

    // Classic binary search fix:
    int lo = 0, hi = INT_MAX;
    // int mid = (lo + hi) / 2;       // BUG: overflow when lo + hi > INT_MAX
    // int mid = lo + (hi - lo) / 2;  // Works but verbose
    int mid = std::midpoint(lo, hi);   // Clean, correct, constexpr
    std::cout << "binary search mid = " << mid << "\n";
    // Output: binary search mid = 1073741823
}
```

**Explanation:** `std::midpoint` internally uses a subtraction-based algorithm (roughly `a + (b - a) / 2` with careful unsigned arithmetic) that never causes overflow. The naive `(a + b) / 2` computes `a + b` first, which for large values exceeds `INT_MAX` - producing undefined behavior on signed integers. The midpoint function is also `constexpr`, making it usable in compile-time computations.

### Q2: Apply std::lerp(a, b, t) for smooth animation interpolation with t in [0,1]

**Answer:**

The guarantee that matters for animation is that `lerp(a, b, 1.0)` returns exactly `b` - not `b - epsilon`. Without that guarantee, your object never quite arrives at its destination, which can cause subtle rendering bugs:

```cpp
#include <cmath>
#include <iostream>
#include <iomanip>

struct Vec2 {
    double x, y;
};

Vec2 lerp2d(Vec2 a, Vec2 b, double t) {
    return {std::lerp(a.x, b.x, t), std::lerp(a.y, b.y, t)};
}

int main() {
    // Animate a sprite from position A to position B over 10 frames
    Vec2 start = {100.0, 200.0};
    Vec2 end   = {500.0, 50.0};

    int frames = 10;
    std::cout << std::fixed << std::setprecision(1);

    for (int i = 0; i <= frames; ++i) {
        double t = static_cast<double>(i) / frames; // 0.0 to 1.0
        Vec2 pos = lerp2d(start, end, t);
        std::cout << "Frame " << std::setw(2) << i
                  << ": t=" << t
                  << "  pos=(" << pos.x << ", " << pos.y << ")\n";
    }
    // Output:
    // Frame  0: t=0.0  pos=(100.0, 200.0)
    // Frame  1: t=0.1  pos=(140.0, 185.0)
    // Frame  2: t=0.2  pos=(180.0, 170.0)
    // ...
    // Frame  9: t=0.9  pos=(460.0, 65.0)
    // Frame 10: t=1.0  pos=(500.0, 50.0)   <- exactly end position

    // Color interpolation (0-255 range)
    double red_start = 255.0, red_end = 0.0;
    for (double t = 0.0; t <= 1.0; t += 0.25) {
        double red = std::lerp(red_start, red_end, t);
        std::cout << "t=" << t << " red=" << red << "\n";
    }
    // t=0.0 red=255.0
    // t=0.25 red=191.2 (actually 191.25)
    // t=0.5 red=127.5
    // t=0.75 red=63.8 (actually 63.75)
    // t=1.0 red=0.0
}
```

**Explanation:** `std::lerp` guarantees monotonicity and exact endpoint values (`lerp(a,b,0)==a`, `lerp(a,b,1)==b`). This is critical for animation - the object always starts exactly at `start` and arrives exactly at `end`. A naive `a + t*(b-a)` might not return exactly `b` at `t=1` due to floating-point rounding, causing visual glitches.

### Q3: Explain the UB that (a+b)/2 triggers for large signed integers and how midpoint avoids it

The expression `(a + b) / 2` performs **signed integer addition** first. When `a + b` exceeds `INT_MAX` (or falls below `INT_MIN`), signed integer overflow occurs - which is **undefined behavior** in C and C++. The reason this matters in practice is that optimizing compilers are allowed to assume signed overflow never happens, so they can generate code that produces any result at all.

| Scenario | `a` | `b` | `a + b` | UB? |
| --- | --- | --- | --- | --- |
| Normal | 10 | 20 | 30 | No |
| Near limit | 2'000'000'000 | 1'000'000'000 | 3'000'000'000 | **YES** - exceeds INT_MAX (2'147'483'647) |
| Negative overflow | -2'000'000'000 | -1'000'000'000 | -3'000'000'000 | **YES** - below INT_MIN |
| Mixed safe | -1'000'000'000 | 1'000'000'000 | 0 | No |

**How `std::midpoint` avoids it:**

The implementation never adds `a + b`. Instead it works with the *difference*, which is safe because the difference between two same-sign integers always fits:

```cpp
// Conceptual implementation for unsigned integers:
constexpr unsigned midpoint(unsigned a, unsigned b) {
    // Uses SUBTRACTION (never adds a + b):
    if (a <= b)
        return a + (b - a) / 2;  // b - a is always non-negative, no overflow
    else
        return b + (a - b) / 2;
}

// For signed integers, the implementation converts to unsigned internally:
// 1. Compute the unsigned distance |b - a| (well-defined for unsigned)
// 2. Divide by 2
// 3. Add to the smaller value
// This NEVER adds a + b, so overflow is impossible
```

Here's the concrete proof with boundary values:

```cpp
#include <numeric>
#include <iostream>
#include <climits>

int main() {
    int a = INT_MAX;     // 2147483647
    int b = INT_MAX - 2; // 2147483645

    // (a + b) would be 4294967292 -> overflows int -> UB
    // But midpoint computes: a + (b - a) / 2 = 2147483647 + (-2) / 2
    //                      = 2147483647 + (-1) = 2147483646
    int result = std::midpoint(a, b);
    std::cout << result << "\n"; // 2147483646

    // For pointers: midpoint(p, q) divides the distance, never adds addresses
    int arr[100]{};
    int* p = arr + 90;
    int* q = arr + 98;
    int* m = std::midpoint(p, q); // arr + 94
    std::cout << (m - arr) << "\n"; // 94
}
```

**Key takeaway:** `std::midpoint` is a drop-in replacement for `(a + b) / 2` that is always correct. Use it in binary search (`std::midpoint(lo, hi)` instead of `lo + (hi - lo) / 2`), running averages, and any context where two values must be averaged safely.

---

## Notes

- **Header locations:** `std::midpoint` is in `<numeric>`; `std::lerp` is in `<cmath>`. Both are C++20.
- **`constexpr`:** Both functions are `constexpr` - usable in compile-time expressions, `static_assert`, template parameters, etc.
- **Pointer midpoint:** `std::midpoint(p, q)` requires `p` and `q` to point into the same array (or one past the end). It returns the pointer to the element at the midpoint index.
- **`lerp` extrapolation:** `std::lerp(a, b, t)` works for `t` outside [0,1]. When `t = -1`, it returns `a - (b - a)` = `2a - b`. Useful for extending motion beyond endpoints.
- **Binary search idiom:** Replace `int mid = lo + (hi - lo) / 2;` with `int mid = std::midpoint(lo, hi);` - cleaner, more intent-revealing, and equally correct.
- Compile with `-std=c++20 -Wall -Wextra`.
