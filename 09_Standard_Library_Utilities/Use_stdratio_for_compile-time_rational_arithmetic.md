# Use std::ratio for compile-time rational arithmetic

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/ratio>  

---

## Topic Overview

`std::ratio` (header `<ratio>`) represents a compile-time fraction as a ratio of two `std::intmax_t` values. It's the backbone of `std::chrono`'s type-safe duration system and can be used anywhere you need exact rational arithmetic at compile time without floating-point imprecision. The key insight is that these are purely type-level values - there are no runtime instances, only types with `::num` and `::den` members you read at compile time.

### Core Type

The template stores a fraction in its simplest reduced form. You put in any numerator and denominator, and the type gives back the reduced version:

```cpp
template <std::intmax_t Num, std::intmax_t Den = 1>
class ratio {
public:
    static constexpr std::intmax_t num = /* reduced Num */;
    static constexpr std::intmax_t den = /* reduced Den */;
    using type = ratio<num, den>;  // reduced form
};
```

The ratio is always stored in **reduced form**: `std::ratio<6, 4>::num == 3`, `std::ratio<6, 4>::den == 2`. This automatic reduction means you never have to manually simplify - the type system does it for you.

### Predefined Ratios (SI Prefixes)

The standard library ships with all the SI prefixes pre-defined. You'll encounter these most often when working with `std::chrono` durations.

| Name | Ratio | Decimal |
| --- | --- | --- |
| `std::atto` | `ratio<1, 1'000'000'000'000'000'000>` | 10⁻¹⁸ |
| `std::femto` | `ratio<1, 1'000'000'000'000'000>` | 10⁻¹⁵ |
| `std::pico` | `ratio<1, 1'000'000'000'000>` | 10⁻¹² |
| `std::nano` | `ratio<1, 1'000'000'000>` | 10⁻⁹ |
| `std::micro` | `ratio<1, 1'000'000>` | 10⁻⁶ |
| `std::milli` | `ratio<1, 1'000>` | 10⁻³ |
| `std::centi` | `ratio<1, 100>` | 10⁻² |
| `std::kilo` | `ratio<1'000, 1>` | 10³ |
| `std::mega` | `ratio<1'000'000, 1>` | 10⁶ |
| `std::giga` | `ratio<1'000'000'000, 1>` | 10⁹ |

### Compile-Time Arithmetic Operations

All arithmetic happens through type aliases - you get a new type back, not a runtime value. The result is always in reduced form.

| Operation | Type alias | Result |
| --- | --- | --- |
| `ratio_add<R1, R2>` | Addition | `R1 + R2` as reduced ratio |
| `ratio_subtract<R1, R2>` | Subtraction | `R1 - R2` |
| `ratio_multiply<R1, R2>` | Multiplication | `R1 × R2` |
| `ratio_divide<R1, R2>` | Division | `R1 / R2` |
| `ratio_equal<R1, R2>` | Equality | `::value` is `true`/`false` |
| `ratio_less<R1, R2>` | Less-than | `::value` is `true`/`false` |

### Core Examples

Here's `std::ratio` in action. Notice that `static_assert` is used to verify results at compile time - this is typical; you're working with types, not runtime values.

```cpp
#include <ratio>
#include <iostream>

int main() {
    // Define fractions at compile time
    using one_third = std::ratio<1, 3>;
    using two_thirds = std::ratio<2, 3>;

    // Addition: 1/3 + 2/3 = 3/3 = 1/1
    using sum = std::ratio_add<one_third, two_thirds>;
    std::cout << sum::num << "/" << sum::den << "\n"; // 1/1

    // Multiplication: 1/3 × 2/3 = 2/9
    using product = std::ratio_multiply<one_third, two_thirds>;
    std::cout << product::num << "/" << product::den << "\n"; // 2/9

    // Automatic reduction: 6/4 -> 3/2
    using reduced = std::ratio<6, 4>;
    std::cout << reduced::num << "/" << reduced::den << "\n"; // 3/2

    // Comparison
    static_assert(std::ratio_less<one_third, two_thirds>::value);
    static_assert(std::ratio_equal<std::ratio<2,4>, std::ratio<1,2>>::value);

    // SI prefix values
    std::cout << "nano: " << std::nano::num << "/" << std::nano::den << "\n";
    // nano: 1/1000000000
}
```

The entire computation - addition, reduction, comparison - happens at compile time. By the time your program runs, these are just constants.

---

## Self-Assessment

### Q1: Define a compilation-time ratio for 'bits per byte' and use it in a template

**Answer:**

```cpp
#include <ratio>
#include <iostream>
#include <cstdint>

// Bits-per-byte as a compile-time ratio
using bits_per_byte = std::ratio<8, 1>;

// Generic unit converter template using ratio
template <typename FromRatio, typename ToRatio>
constexpr double convert(double value) {
    using conversion = std::ratio_divide<FromRatio, ToRatio>;
    return value * static_cast<double>(conversion::num) / conversion::den;
}

// Custom ratios for data sizes
using bits   = std::ratio<1, 1>;       // base unit: bits
using bytes  = std::ratio<8, 1>;       // 8 bits
using kilobytes = std::ratio<8 * 1024, 1>;  // 8192 bits
using megabytes = std::ratio<8 * 1024 * 1024, 1>;

int main() {
    // Use bits_per_byte in a template
    constexpr std::intmax_t num_bytes = 1024;
    constexpr std::intmax_t num_bits = num_bytes * bits_per_byte::num / bits_per_byte::den;
    static_assert(num_bits == 8192);
    std::cout << num_bytes << " bytes = " << num_bits << " bits\n";
    // 1024 bytes = 8192 bits

    // Generic conversions using ratio
    std::cout << "1 MB = " << convert<megabytes, bytes>(1.0) << " bytes\n";
    // 1 MB = 1048576 bytes

    std::cout << "8192 bits = " << convert<bits, kilobytes>(8192.0) << " KB\n";
    // 8192 bits = 1 KB

    // Compile-time computation
    using half_byte = std::ratio_divide<bits_per_byte, std::ratio<2>>;
    static_assert(half_byte::num == 4 && half_byte::den == 1);
    std::cout << "Half a byte = " << half_byte::num << " bits (nibble)\n";
    // Half a byte = 4 bits (nibble)
}
```

`std::ratio<8, 1>` represents the compile-time constant 8/1 (8 bits per byte). This can be used in template computations, `static_assert`s, and type-level arithmetic. The `convert` template divides the source ratio by the target ratio to get a conversion factor, all computed at compile time.

### Q2: Show how std::chrono uses std::ratio for period definitions (std::milli, std::nano)

**Answer:**

The reason `std::chrono` can prevent you from accidentally adding milliseconds to seconds without a cast is exactly because the period is encoded as a `std::ratio` in the type. The compiler compares those ratios and refuses conversions that would lose data.

```cpp
#include <chrono>
#include <ratio>
#include <iostream>

int main() {
    // std::chrono::duration is defined as:
    // template <class Rep, class Period = std::ratio<1>>
    // class duration;
    //
    // Where Period is a std::ratio representing seconds per tick:

    // std::chrono::seconds      = duration<int64_t, ratio<1>>       (1 tick = 1 second)
    // std::chrono::milliseconds = duration<int64_t, ratio<1,1000>>  (1 tick = 1/1000 sec)
    // std::chrono::microseconds = duration<int64_t, ratio<1,1000000>>
    // std::chrono::nanoseconds  = duration<int64_t, ratio<1,1000000000>>
    // std::chrono::minutes      = duration<int64_t, ratio<60>>      (1 tick = 60 seconds)
    // std::chrono::hours        = duration<int64_t, ratio<3600>>

    // Verify: milliseconds uses std::milli = ratio<1, 1000>
    using ms_period = std::chrono::milliseconds::period;
    static_assert(std::ratio_equal<ms_period, std::milli>::value);
    std::cout << "milliseconds period: " << ms_period::num << "/" << ms_period::den << "\n";
    // 1/1000

    // Verify: nanoseconds uses std::nano = ratio<1, 1000000000>
    using ns_period = std::chrono::nanoseconds::period;
    static_assert(std::ratio_equal<ns_period, std::nano>::value);

    // Custom duration using ratio: "ticks" at 120 Hz (1/120 second per tick)
    using game_tick = std::chrono::duration<int, std::ratio<1, 120>>;
    game_tick one_tick(1);
    auto in_ms = std::chrono::duration_cast<std::chrono::milliseconds>(one_tick);
    std::cout << "1 game tick = " << in_ms.count() << " ms\n"; // 8 ms

    // Custom: "fortnights" = 14 days = 14 × 24 × 3600 seconds
    using fortnights = std::chrono::duration<double, std::ratio<14 * 24 * 3600>>;
    fortnights fn(1);
    auto in_hours = std::chrono::duration_cast<std::chrono::hours>(fn);
    std::cout << "1 fortnight = " << in_hours.count() << " hours\n"; // 336

    // ratio enables implicit safe conversions (no data loss):
    std::chrono::seconds sec(1);
    std::chrono::milliseconds ms = sec;  // OK: 1s -> 1000ms (widening)
    std::cout << "1 second = " << ms.count() << " ms\n"; // 1000

    // Narrowing requires duration_cast:
    std::chrono::milliseconds ms2(1500);
    auto sec2 = std::chrono::duration_cast<std::chrono::seconds>(ms2);
    std::cout << "1500 ms = " << sec2.count() << " s (truncated)\n"; // 1
}
```

`std::chrono::duration<Rep, Period>` uses `std::ratio` as its `Period` template parameter - `std::ratio<1, 1000>` for milliseconds means each tick represents 1/1000 of a second. This type-level encoding of units prevents accidentally adding minutes to nanoseconds without conversion. The compiler uses ratio arithmetic to compute conversion factors at compile time.

### Q3: Use std::ratio_add, std::ratio_multiply to compute ratios at compile time

**Answer:**

```cpp
#include <ratio>
#include <iostream>

int main() {
    // ratio_add: 1/3 + 1/6 = 2/6 + 1/6 = 3/6 = 1/2
    using sum = std::ratio_add<std::ratio<1, 3>, std::ratio<1, 6>>;
    static_assert(sum::num == 1 && sum::den == 2);
    std::cout << "1/3 + 1/6 = " << sum::num << "/" << sum::den << "\n";
    // 1/3 + 1/6 = 1/2

    // ratio_subtract: 3/4 - 1/4 = 2/4 = 1/2
    using diff = std::ratio_subtract<std::ratio<3, 4>, std::ratio<1, 4>>;
    std::cout << "3/4 - 1/4 = " << diff::num << "/" << diff::den << "\n";
    // 3/4 - 1/4 = 1/2

    // ratio_multiply: 2/3 × 3/5 = 6/15 = 2/5
    using prod = std::ratio_multiply<std::ratio<2, 3>, std::ratio<3, 5>>;
    static_assert(prod::num == 2 && prod::den == 5);
    std::cout << "2/3 × 3/5 = " << prod::num << "/" << prod::den << "\n";
    // 2/3 × 3/5 = 2/5

    // ratio_divide: 2/3 ÷ 4/5 = 2/3 × 5/4 = 10/12 = 5/6
    using quot = std::ratio_divide<std::ratio<2, 3>, std::ratio<4, 5>>;
    std::cout << "2/3 ÷ 4/5 = " << quot::num << "/" << quot::den << "\n";
    // 2/3 ÷ 4/5 = 5/6

    // Chaining operations: (1/2 + 1/3) × 6 = 5/6 × 6 = 5
    using step1 = std::ratio_add<std::ratio<1, 2>, std::ratio<1, 3>>;   // 5/6
    using step2 = std::ratio_multiply<step1, std::ratio<6>>;             // 30/6 = 5
    static_assert(step2::num == 5 && step2::den == 1);
    std::cout << "(1/2 + 1/3) × 6 = " << step2::num << "\n";
    // (1/2 + 1/3) × 6 = 5

    // Compile-time comparison
    static_assert(std::ratio_less<std::ratio<1, 3>, std::ratio<1, 2>>::value,
                  "1/3 < 1/2");
    static_assert(!std::ratio_less<std::ratio<2, 3>, std::ratio<1, 2>>::value,
                  "2/3 is NOT less than 1/2");

    // Practical: compute frame interval from FPS
    using fps_60 = std::ratio<1, 60>;       // 1/60 second per frame
    using two_frames = std::ratio_multiply<fps_60, std::ratio<2>>;  // 2/60 = 1/30
    std::cout << "2 frames at 60fps = " << two_frames::num
              << "/" << two_frames::den << " second\n";
    // 2 frames at 60fps = 1/30 second
}
```

All `ratio_*` operations produce a new `std::ratio` type with the result in reduced form. The entire computation happens at compile time - zero runtime cost. These type aliases can chain: `ratio_multiply<ratio_add<A, B>, C>` computes `(A + B) × C`. The results are available as `::num` and `::den` static constexpr members.

---

## Notes

- **Always reduced:** `std::ratio<6,9>` has `num=2, den=3`. The sign is always on the numerator: `std::ratio<1,-3>::num == -1`, `den == 3`.
- **Overflow protection:** If the numerator or denominator would overflow `std::intmax_t` during arithmetic, the program is ill-formed (compile error). This is by design - silent overflow would defeat the purpose.
- **Not a runtime type:** `std::ratio` has no runtime instances. It's purely a type-level construct - you use `::num` and `::den` to extract values.
- **`chrono` connection:** Every `std::chrono::duration` has a `::period` member type that's a `std::ratio`. This enables the type system to prevent mixing incompatible time units.
- **Zero denominator:** `std::ratio<1, 0>` is ill-formed - compile error.
- Compile with `-std=c++20 -Wall -Wextra`.
