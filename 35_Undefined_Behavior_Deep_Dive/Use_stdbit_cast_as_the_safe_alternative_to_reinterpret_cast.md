# Use std::bit_cast as the Safe Alternative to reinterpret_cast

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++20, C++23  
**Reference:** [cppreference - std::bit_cast](https://en.cppreference.com/w/cpp/numeric/bit_cast)  

---

## Topic Overview

`std::bit_cast<To>(from)` (C++20, `<bit>` header) performs a **value-preserving reinterpretation** of the bit pattern of `from` as type `To`. It is the standard-blessed replacement for the common but UB-ridden pattern of `reinterpret_cast`-ing between unrelated types or union type-punning.

Unlike `reinterpret_cast`, `std::bit_cast` operates on **values**, not pointers. It copies the bytes of the source object and reinterprets them as the destination type. Both types must be **trivially copyable** and have the **same size**. Crucially, `std::bit_cast` is `constexpr` - it works at compile time, which neither `reinterpret_cast` nor `std::memcpy` can achieve.

Here is how the three approaches compare:

| Feature | `reinterpret_cast` | `std::memcpy` | `std::bit_cast` |
| --- | --- | --- | --- |
| Strict aliasing safe | No | Yes | Yes |
| Compile-time (`constexpr`) | No | No | **Yes** |
| Type size check | No | Manual | **Automatic** (compile error) |
| Trivially copyable check | No | No | **Automatic** |
| Pointer or value | Pointer | Value (manual) | **Value** |
| Standard since | C++98 | C++98 | **C++20** |
| Zero runtime overhead | - | Yes (optimized away) | **Yes** |

Here is a plain-text summary of the four type-punning methods and their status:

```cpp
Type Punning Methods:

  reinterpret_cast<int*>(&f)  ->  UB (strict aliasing violation)

  memcpy(&i, &f, sizeof(f))  ->  OK but not constexpr

  std::bit_cast<int>(f)      ->  OK, constexpr, type-safe

  union { float f; int i; }  ->  UB in C++ (defined in C)
```

`std::bit_cast` should be the **default choice** for all type punning in C++20 and later. Fall back to `std::memcpy` only when targeting pre-C++20 or when working with non-trivially-copyable types (which cannot be bit-cast).

---

## Self-Assessment

### Q1: Implement IEEE 754 float inspection using std::bit_cast

One of the most compelling uses of `std::bit_cast` is inspecting the internal bit layout of floating-point numbers. Before C++20, this required a `memcpy` dance that the compiler could not evaluate at compile time. Now you can write `constexpr` functions that decompose a float into its sign, exponent, and mantissa fields - and verify them with `static_assert`:

```cpp
#include <bit>
#include <bitset>
#include <cstdint>
#include <cstdio>
#include <cmath>
#include <iostream>

// IEEE 754 single-precision layout:
// [sign:1][exponent:8][mantissa:23]

struct FloatParts {
    uint32_t mantissa : 23;
    uint32_t exponent : 8;
    uint32_t sign     : 1;
};

// Compile-time float inspection using bit_cast
constexpr uint32_t float_to_bits(float f) {
    return std::bit_cast<uint32_t>(f);
}

constexpr bool is_negative(float f) {
    return (float_to_bits(f) >> 31) & 1;
}

constexpr int biased_exponent(float f) {
    return (float_to_bits(f) >> 23) & 0xFF;
}

constexpr uint32_t raw_mantissa(float f) {
    return float_to_bits(f) & 0x7FFFFF;
}

constexpr bool is_nan_ct(float f) {
    return biased_exponent(f) == 0xFF && raw_mantissa(f) != 0;
}

constexpr bool is_inf_ct(float f) {
    return biased_exponent(f) == 0xFF && raw_mantissa(f) == 0;
}

void inspect_float(float f) {
    uint32_t bits = std::bit_cast<uint32_t>(f);
    auto parts = std::bit_cast<FloatParts>(f);

    std::printf("Value:    %g\n", f);
    std::printf("Hex:      0x%08X\n", bits);
    std::printf("Binary:   %s\n", std::bitset<32>(bits).to_string().c_str());
    std::printf("Sign:     %u\n", parts.sign);
    std::printf("Exponent: %u (biased), %d (unbiased)\n",
                parts.exponent, static_cast<int>(parts.exponent) - 127);
    std::printf("Mantissa: 0x%06X\n", parts.mantissa);
    std::printf("NaN:      %s\n", is_nan_ct(f) ? "yes" : "no");
    std::printf("Inf:      %s\n", is_inf_ct(f) ? "yes" : "no");
    std::printf("\n");
}

// Compile-time assertions using bit_cast
static_assert(float_to_bits(0.0f) == 0x00000000);
static_assert(float_to_bits(1.0f) == 0x3F800000);
static_assert(float_to_bits(-1.0f) == 0xBF800000);
static_assert(is_negative(-1.0f));
static_assert(!is_negative(1.0f));

int main() {
    inspect_float(0.0f);
    inspect_float(1.0f);
    inspect_float(-1.0f);
    inspect_float(3.14159f);
    inspect_float(std::numeric_limits<float>::infinity());
    inspect_float(std::numeric_limits<float>::quiet_NaN());
}
```

`std::bit_cast` enables compile-time IEEE 754 inspection that was impossible before C++20. The `static_assert` calls demonstrate compile-time type punning. The `FloatParts` bitfield struct can itself be bit-cast from a float for structured access.

---

### Q2: Compare std::bit_cast with reinterpret_cast and memcpy in generated code

All three methods produce the same assembly at `-O2`. The difference is entirely in what the standard allows and what the compiler can do with the code at compile time. Notice especially the fast inverse square root at the bottom - it is now a legitimate compile-time constant:

```cpp
#include <bit>
#include <cstdint>
#include <cstring>
#include <iostream>

// All three methods should produce identical assembly at -O2.
// The difference is in compile-time safety guarantees.

// Method 1: reinterpret_cast (UB - strict aliasing violation)
uint32_t float_bits_reinterpret(float f) {
    return *reinterpret_cast<uint32_t*>(&f);
    // Compiles to: movd eax, xmm0 (on x86-64)
    // BUT: undefined behavior per the standard
}

// Method 2: std::memcpy (well-defined, not constexpr)
uint32_t float_bits_memcpy(float f) {
    uint32_t result;
    std::memcpy(&result, &f, sizeof(result));
    return result;
    // Compiles to: movd eax, xmm0 (identical assembly)
    // Well-defined, but cannot be used in constexpr context
}

// Method 3: std::bit_cast (well-defined, constexpr)
constexpr uint32_t float_bits_bitcast(float f) {
    return std::bit_cast<uint32_t>(f);
    // Compiles to: movd eax, xmm0 (identical assembly)
    // Well-defined AND constexpr
}

// Reverse direction: bits -> float
constexpr float bits_to_float(uint32_t bits) {
    return std::bit_cast<float>(bits);
}

// Round-trip test
static_assert(float_bits_bitcast(bits_to_float(0x40490FDB)) == 0x40490FDB);

// Practical application: fast inverse square root (educational)
constexpr float fast_inv_sqrt_modern(float x) {
    // The famous Quake III algorithm, now legal in C++20
    auto i = std::bit_cast<uint32_t>(x);
    i = 0x5f3759df - (i >> 1);     // Magic number approximation
    float y = std::bit_cast<float>(i);
    y = y * (1.5f - (x * 0.5f * y * y));  // Newton-Raphson iteration
    return y;
}

// This is now a compile-time constant!
static_assert(fast_inv_sqrt_modern(4.0f) > 0.49f);
static_assert(fast_inv_sqrt_modern(4.0f) < 0.51f);

// Compile error examples (uncomment to see):
// struct NonTriviallyCopyable {
//     NonTriviallyCopyable(const NonTriviallyCopyable&) { /* custom */ }
//     int data;
// };
// auto x = std::bit_cast<int>(NonTriviallyCopyable{});  // Compile error!

// struct WrongSize {
//     int a, b;
// };
// auto y = std::bit_cast<int>(WrongSize{1, 2});  // Compile error: size mismatch

int main() {
    float f = 3.14159f;
    std::printf("reinterpret: 0x%08X\n", float_bits_reinterpret(f));
    std::printf("memcpy:      0x%08X\n", float_bits_memcpy(f));
    std::printf("bit_cast:    0x%08X\n", float_bits_bitcast(f));

    constexpr float inv_sqrt_2 = fast_inv_sqrt_modern(2.0f);
    std::printf("fast_inv_sqrt(2.0) = %f (expected ~0.707)\n", inv_sqrt_2);
}
```

All three methods produce identical assembly at `-O2`. The difference is safety: `reinterpret_cast` is UB, `memcpy` is well-defined but not `constexpr`, and `bit_cast` is well-defined AND `constexpr`. The fast inverse square root is now a valid compile-time constant in C++20.

---

### Q3: Use std::bit_cast for cross-platform serialization

`std::bit_cast` pairs naturally with `std::array<std::byte, N>` for serialization. Because both the struct and the byte array are trivially copyable and the same size, the conversion is a single `bit_cast` call - no pointer casting, no memcpy, and it all works at compile time:

```cpp
#include <array>
#include <bit>
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <type_traits>

// Serialization of trivially-copyable types to bytes using bit_cast.

// Serialize any trivially-copyable type to a byte array
template <typename T>
    requires std::is_trivially_copyable_v<T>
constexpr auto to_bytes(const T& value)
    -> std::array<std::byte, sizeof(T)>
{
    return std::bit_cast<std::array<std::byte, sizeof(T)>>(value);
}

// Deserialize a byte array back to a type
template <typename T>
    requires std::is_trivially_copyable_v<T>
constexpr T from_bytes(const std::array<std::byte, sizeof(T)>& bytes) {
    return std::bit_cast<T>(bytes);
}

// Network-endian conversion using bit_cast
constexpr uint32_t to_network_order(uint32_t host) {
    if constexpr (std::endian::native == std::endian::little) {
        auto bytes = std::bit_cast<std::array<uint8_t, 4>>(host);
        return std::bit_cast<uint32_t>(
            std::array<uint8_t, 4>{bytes[3], bytes[2], bytes[1], bytes[0]}
        );
    } else {
        return host;
    }
}

// Compile-time network-order conversion
static_assert(to_network_order(0x01020304) == 0x04030201
              || std::endian::native == std::endian::big);

struct Packet {
    uint32_t seq;
    float    value;
    uint16_t flags;
    uint16_t padding;
};
static_assert(sizeof(Packet) == 12);
static_assert(std::is_trivially_copyable_v<Packet>);

void serialization_demo() {
    Packet pkt{42, 3.14f, 0x00FF, 0};

    // Serialize to bytes
    auto bytes = to_bytes(pkt);

    std::printf("Serialized %zu bytes: ", bytes.size());
    for (auto b : bytes) {
        std::printf("%02X ", static_cast<unsigned>(b));
    }
    std::printf("\n");

    // Deserialize back
    Packet restored = from_bytes<Packet>(bytes);
    std::printf("Restored: seq=%u, value=%.2f, flags=0x%04X\n",
                restored.seq, restored.value, restored.flags);

    // Compile-time serialization round-trip
    constexpr Packet ct_pkt{100, 2.71f, 0x0001, 0};
    constexpr auto ct_bytes = to_bytes(ct_pkt);
    constexpr Packet ct_restored = from_bytes<Packet>(ct_bytes);
    static_assert(ct_restored.seq == 100);
    static_assert(ct_restored.flags == 0x0001);
}

int main() {
    serialization_demo();
}
```

`std::bit_cast` enables type-safe, `constexpr` serialization of trivially copyable types to/from byte arrays. Combined with `std::endian` detection, this provides a portable, zero-overhead serialization layer that works at compile time. The `std::array<std::byte, N>` to `T` round-trip is guaranteed to be lossless.

---

## Notes

- `std::bit_cast` requires both source and destination types to be **trivially copyable** and have the **same size**. Violations are compile-time errors.
- Unlike `reinterpret_cast`, `std::bit_cast` creates a **new value** - it does not create an alias. There are no aliasing concerns.
- For pre-C++20 code, `std::memcpy` is the correct substitute. Wrap it in a template for ergonomics.
- Padding bits may have indeterminate values; `bit_cast` preserves them as-is. This matters for comparison and hashing.
- `std::bit_cast` of a pointer type is implementation-defined (the pointer value may not be meaningful on the other side).
- GCC, Clang, and MSVC all support `constexpr std::bit_cast` since their C++20 modes.
