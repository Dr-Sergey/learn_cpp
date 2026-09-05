# Implement Fixed-Point Arithmetic Without Floating Point

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** https://en.wikipedia.org/wiki/Fixed-point_arithmetic  

---

## Topic Overview

Many embedded targets (Cortex-M0, M0+, some RISC-V RV32I) lack a hardware FPU, making software floating-point emulation 10-100x slower than integer operations. Fixed-point arithmetic maps real numbers onto integers by reserving a compile-time-fixed number of fractional bits. A Q16.16 format, for example, uses 16 integer bits and 16 fractional bits in a 32-bit word, giving a resolution of ~0.0000153 and a range of ±32767.

The key design decisions are: (1) choosing the Q format (QM.N where M = integer bits, N = fractional bits), (2) handling intermediate widths to prevent overflow during multiply/accumulate, and (3) providing rounding and saturation policies. A well-designed C++ template encodes all of this at compile time with zero runtime overhead.

| Q Format  | Storage | Range               | Resolution      | Use Case                    |
| --- | --- | --- | --- | --- |
| Q1.15     | 16-bit  | [-1, +0.99997]      | 3.05e-5         | Audio DSP coefficients      |
| Q16.16    | 32-bit  | [-32768, +32767.99] | 1.53e-5         | Sensor fusion, PID control  |
| Q8.24     | 32-bit  | [-128, +127.999]    | 5.96e-8         | High-precision control      |
| Q32.32    | 64-bit  | [-2^31, +2^31-1]    | 2.33e-10        | Navigation, financial calc  |

Here is what a Q16.16 value looks like inside a 32-bit word. The raw integer value 245760 represents 3.75 because 245760 / 65536 = 3.75. That division is the conceptual mapping - in practice you never actually divide, you just shift.

```text
  32-bit word (Q16.16):
  +----------------------+----------------------+
  |   Integer part (16)  | Fractional part (16)  |
  |  sign + 15 bits      |  16 bits              |
  +----------------------+----------------------+
  Value = raw_integer / 2^16
  3.75  = 245760 / 65536 = 0x0003_C000
```

Overflow during multiplication is the primary hazard. Multiplying two Q16.16 numbers produces a Q32.32 intermediate in 64 bits, which must be shifted right by N before truncating back to 32 bits. The reason this trips people up is that if you forget to widen before the multiply and do it in 32 bits, you silently lose the upper bits and get catastrophically wrong results with no warning from the compiler.

---

## Self-Assessment

### Q1: Design a type-safe fixed-point template that prevents accidental mixing of incompatible Q formats at compile time

The template below uses the type system to make mixing Q formats a compile error. A `Q16_16` and a `Q8_24` are different types, so you can't accidentally add them - you have to explicitly convert first. The private `RawTag` constructor prevents anyone from accidentally constructing a fixed-point value from a raw integer without going through the named factory functions like `from_int` or `from_double`.

```cpp
#include <cstdint>
#include <type_traits>
#include <limits>

// Saturation policy: clamp on overflow instead of wrapping
enum class OverflowPolicy { Wrap, Saturate };

template <typename StorageType, int FracBits,
          OverflowPolicy Policy = OverflowPolicy::Saturate>
class FixedPoint {
    static_assert(std::is_signed_v<StorageType>,
                  "Storage must be signed for proper arithmetic");
    static_assert(FracBits > 0 && FracBits < sizeof(StorageType) * 8 - 1,
                  "FracBits must leave at least 1 integer + 1 sign bit");

    StorageType raw_;

    // Private tagged constructor for raw values
    struct RawTag {};
    constexpr FixedPoint(StorageType raw, RawTag) noexcept : raw_(raw) {}

    // Double-width type for intermediate results
    using Wide = std::conditional_t<sizeof(StorageType) <= 2,
                                    std::int32_t, std::int64_t>;

    static constexpr StorageType saturate(Wide value) noexcept {
        constexpr Wide lo = std::numeric_limits<StorageType>::min();
        constexpr Wide hi = std::numeric_limits<StorageType>::max();
        if constexpr (Policy == OverflowPolicy::Saturate) {
            return static_cast<StorageType>(
                value < lo ? lo : (value > hi ? hi : value));
        } else {
            return static_cast<StorageType>(value);
        }
    }

public:
    static constexpr int fractional_bits = FracBits;
    static constexpr StorageType scale = StorageType{1} << FracBits;

    constexpr FixedPoint() noexcept : raw_(0) {}

    // Conversion from integer
    static constexpr FixedPoint from_int(StorageType i) noexcept {
        return FixedPoint(static_cast<StorageType>(
            saturate(static_cast<Wide>(i) << FracBits)), RawTag{});
    }

    // Conversion from raw representation
    static constexpr FixedPoint from_raw(StorageType r) noexcept {
        return FixedPoint(r, RawTag{});
    }

    // Conversion from double (compile-time only for constants)
    static constexpr FixedPoint from_double(double d) noexcept {
        return FixedPoint(static_cast<StorageType>(d * scale + (d >= 0 ? 0.5 : -0.5)),
                          RawTag{});
    }

    constexpr StorageType raw()      const noexcept { return raw_; }
    constexpr double      to_double() const noexcept {
        return static_cast<double>(raw_) / scale;
    }

    // Same-format arithmetic: no format confusion possible
    constexpr FixedPoint operator+(FixedPoint rhs) const noexcept {
        return FixedPoint(saturate(static_cast<Wide>(raw_) + rhs.raw_), RawTag{});
    }
    constexpr FixedPoint operator-(FixedPoint rhs) const noexcept {
        return FixedPoint(saturate(static_cast<Wide>(raw_) - rhs.raw_), RawTag{});
    }

    // Multiply: widen, multiply, shift back
    constexpr FixedPoint operator*(FixedPoint rhs) const noexcept {
        Wide product = static_cast<Wide>(raw_) * static_cast<Wide>(rhs.raw_);
        // Round-to-nearest by adding half the divisor before shift
        product += Wide{1} << (FracBits - 1);
        return FixedPoint(saturate(product >> FracBits), RawTag{});
    }

    // Divide: shift left first, then divide
    constexpr FixedPoint operator/(FixedPoint rhs) const noexcept {
        Wide numerator = static_cast<Wide>(raw_) << FracBits;
        return FixedPoint(saturate(numerator / rhs.raw_), RawTag{});
    }

    constexpr bool operator>(FixedPoint rhs)  const noexcept { return raw_ > rhs.raw_; }
    constexpr bool operator<(FixedPoint rhs)  const noexcept { return raw_ < rhs.raw_; }
    constexpr bool operator==(FixedPoint rhs) const noexcept { return raw_ == rhs.raw_; }
};

// Type aliases for common formats
using Q16_16 = FixedPoint<std::int32_t, 16>;
using Q8_24  = FixedPoint<std::int32_t, 24>;
using Q1_15  = FixedPoint<std::int16_t, 15>;

// Compile-time test
static_assert(Q16_16::from_double(3.75).raw() == 245760);
static_assert((Q16_16::from_double(2.5) * Q16_16::from_double(1.5)).raw() ==
              Q16_16::from_double(3.75).raw());
```

The `static_assert` checks at the bottom are your sanity check that the arithmetic is correct. If the multiplication implementation ever produces the wrong shift or rounding, the build breaks. These run entirely at compile time - zero cost.

### Q2: Implement a fixed-point PID controller suitable for a bare-metal control loop with no FPU

A PID controller is the canonical use case for fixed-point on MCUs without an FPU. The implementation below is entirely in Q16.16 integer arithmetic. Every operation - multiply, add, clamp - resolves to a handful of integer instructions. On a Cortex-M0 running at 48 MHz, this loop can easily run at 10 kHz where the floating-point equivalent (in software emulation) might only manage 1 kHz.

```cpp
#include <cstdint>

using FP = FixedPoint<std::int32_t, 16>;  // Q16.16

struct PidState {
    FP integral;
    FP prev_error;
};

struct PidGains {
    FP kp;
    FP ki;
    FP kd;
    FP integral_limit;  // anti-windup clamp
    FP output_min;
    FP output_max;
};

// Deterministic, no allocation, no FP, bounded execution
FP pid_update(const PidGains& gains, PidState& state,
              FP setpoint, FP measurement) noexcept {
    FP error = setpoint - measurement;

    // Integral with anti-windup saturation
    state.integral = state.integral + error;
    if (state.integral > gains.integral_limit)
        state.integral = gains.integral_limit;
    else if (state.integral < FP::from_int(0) - gains.integral_limit)
        state.integral = FP::from_int(0) - gains.integral_limit;

    // Derivative (backward difference)
    FP derivative = error - state.prev_error;
    state.prev_error = error;

    // PID output: u = Kp*e + Ki*integral(e) + Kd*de/dt
    FP output = gains.kp * error
              + gains.ki * state.integral
              + gains.kd * derivative;

    // Output clamping
    if (output > gains.output_max)      return gains.output_max;
    else if (output < gains.output_min) return gains.output_min;
    return output;
}

// Example initialization at 1kHz sample rate
// Kp=2.5, Ki=0.01, Kd=0.8 - all embedded in Q16.16
PidGains make_motor_gains() noexcept {
    return PidGains{
        .kp = FP::from_double(2.5),
        .ki = FP::from_double(0.01),
        .kd = FP::from_double(0.8),
        .integral_limit = FP::from_int(500),
        .output_min = FP::from_int(-1000),
        .output_max = FP::from_int(1000),
    };
}
```

The gain constants (`from_double(2.5)`, etc.) are evaluated entirely at compile time - `from_double` is `constexpr` and the `double` values are compile-time constants. At runtime, the gains are just fixed integer values embedded in flash. There's no floating-point involved at runtime whatsoever.

### Q3: How do you convert between fixed-point formats of different precision safely at compile time

Format conversions are where fixed-point code gets error-prone. If you have a high-precision Q8.24 value from a sensor and need to feed it into a Q16.16 PID controller, you need to shift right by 8. But shifting right loses precision, and if the value is too large for the target range it must saturate rather than wrap. The `fixed_convert` function below resolves the shift direction and amount entirely at compile time using `if constexpr`.

```cpp
#include <cstdint>
#include <type_traits>

// Convert between two FixedPoint types with different FracBits
// Shift direction and saturation are resolved at compile time
template <typename To, typename From>
constexpr To fixed_convert(From value) noexcept {
    constexpr int shift = To::fractional_bits - From::fractional_bits;

    using Wide = std::int64_t;
    Wide raw = static_cast<Wide>(value.raw());

    if constexpr (shift > 0) {
        // Increasing precision: shift left (no loss)
        raw <<= shift;
    } else if constexpr (shift < 0) {
        // Decreasing precision: round-to-nearest, then shift right
        raw += Wide{1} << (-shift - 1);  // round
        raw >>= -shift;
    }

    // Saturate to target range
    using TargetStorage = decltype(To::from_int(0).raw());
    constexpr Wide lo = std::numeric_limits<TargetStorage>::min();
    constexpr Wide hi = std::numeric_limits<TargetStorage>::max();
    raw = raw < lo ? lo : (raw > hi ? hi : raw);

    return To::from_raw(static_cast<TargetStorage>(raw));
}

// Application: sensor fusion pipeline
// ADC gives Q8.24 high-precision, PID needs Q16.16
void sensor_pipeline() noexcept {
    Q8_24 high_precision = Q8_24::from_double(3.14159265);

    // Compile-time resolved shift: 24 -> 16, shifts right by 8
    Q16_16 control_value = fixed_convert<Q16_16>(high_precision);

    // Verify precision is maintained within Q16.16 resolution
    // 3.14159265 -> Q8.24 raw: 52707178
    // -> Q16.16 raw: 205887 -> 3.14158630... (error < 2^-16)

    // Reverse: widen for accumulation
    Q8_24 accumulated = fixed_convert<Q8_24>(control_value);
    (void)accumulated;
}
```

The conversion table below shows what happens for common format pairs. Widening (increasing precision) is lossless - you just shift left and the value is exact. Narrowing (decreasing precision) loses fractional bits, but the round-to-nearest step in `fixed_convert` halves the average error compared to truncation.

| From    | To      | Shift | Direction      | Precision Impact        |
|---------|---------|-------|----------------|-------------------------|
| Q8.24   | Q16.16  |  -8   | Right shift 8  | Loses 8 bits of frac    |
| Q16.16  | Q8.24   |  +8   | Left shift 8   | No loss, range narrows  |
| Q1.15   | Q16.16  |  +1   | Left shift 1   | No loss, range widens   |
| Q16.16  | Q1.15   |  -1   | Right shift 1  | Slight frac loss, clamp |

---

## Notes

- Always use double-width intermediates for multiplication: Q16.16 x Q16.16 needs a 64-bit intermediate.
- Round-to-nearest (add `1 << (N-1)` before shifting) halves the average error vs truncation.
- Saturation arithmetic prevents silent wrap-around; on Cortex-M, `__SSAT` and `__USAT` intrinsics do this in one cycle.
- `constexpr` fixed-point lets you compute lookup tables (sin, cos, sqrt) entirely at compile time.
- For division-heavy code, consider reciprocal multiplication: precompute `1/x` in Q format and multiply.
- Profile the generated assembly: a well-optimized Q16.16 multiply on Cortex-M4 should be 2-3 instructions (`SMULL` + shift).
- Avoid mixing Q formats implicitly - the type system should enforce explicit conversion at every boundary.
