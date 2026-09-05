# Use std::bit_cast (C++20) for safe type punning

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/bit_cast>  

---

## Topic Overview

`std::bit_cast<To>(from)` reinterprets the object representation of `from` as type `To`, copying the underlying bytes. It is the **only** well-defined way to do type punning in C++ (besides `memcpy`), and unlike `memcpy` it is `constexpr`.

Type punning - treating the same bits as two different types - comes up constantly in low-level code: extracting IEEE 754 float fields, writing network serializers, working with hardware registers. C++ has strict aliasing rules that make most naive approaches undefined behavior. `std::bit_cast` is the sanctioned escape hatch.

### Requirements

- `sizeof(To) == sizeof(From)` - must be the same size.
- Both `To` and `From` must be **trivially copyable**.
- `To` must be **trivially default constructible** (automatically satisfied for POD types).

### Why Not the Alternatives

The reason this trips people up is that several approaches *look* correct and often *work* in practice, but are still undefined behavior under the C++ standard. Here is the full picture:

| Method | Status | Problem |
| --- | --- | --- |
| `reinterpret_cast<T*>(&x)` | **UB** | Violates strict aliasing |
| `union { float f; uint32_t u; }` | **UB in C++** | Reading inactive union member is undefined |
| `memcpy(&to, &from, sizeof(to))` | **Valid** | Not `constexpr`, verbose |
| `std::bit_cast<To>(from)` | **Valid** | Safety-checked, `constexpr`, concise |

### Core Syntax

Both approaches produce the same result. The difference is that `bit_cast` is a single expression, works at compile time, and gives you a compile error if the types are incompatible.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>
#include <cstring>

int main() {
    float f = 1.0f;

    // --- Old way: memcpy ---
    uint32_t via_memcpy;
    std::memcpy(&via_memcpy, &f, sizeof(f));

    // --- New way: bit_cast ---
    auto via_bitcast = std::bit_cast<uint32_t>(f);

    std::cout << std::hex;
    std::cout << "memcpy:   0x" << via_memcpy  << "\n";
    std::cout << "bit_cast: 0x" << via_bitcast << "\n";
    // Output (IEEE 754):
    // memcpy:   0x3f800000
    // bit_cast: 0x3f800000

    // Round-trip
    float back = std::bit_cast<float>(via_bitcast);
    std::cout << std::dec << "back: " << back << "\n";
    // Output: back: 1
}
```

---

## Self-Assessment

### Q1: Rewrite a float-to-uint32 type pun from memcpy to std::bit_cast and explain the safety improvement

**Answer:**

Notice that the `bit_cast` version is `constexpr` - you can use it in `static_assert` and other compile-time contexts, which `memcpy` simply cannot do.

```cpp
#include <bit>
#include <cstdint>
#include <cstring>
#include <iostream>
#include <type_traits>

// Old (C++11): memcpy-based type punning
uint32_t float_bits_old(float f) {
    static_assert(sizeof(float) == sizeof(uint32_t));
    uint32_t result;
    std::memcpy(&result, &f, sizeof(result));
    return result;
}

// New (C++20): bit_cast
constexpr uint32_t float_bits_new(float f) {
    return std::bit_cast<uint32_t>(f);
}

int main() {
    float f = -0.0f;

    uint32_t old_result = float_bits_old(f);
    uint32_t new_result = float_bits_new(f);

    std::cout << std::hex;
    std::cout << "memcpy:   0x" << old_result << "\n";
    std::cout << "bit_cast: 0x" << new_result << "\n";
    // Output:
    // memcpy:   0x80000000
    // bit_cast: 0x80000000

    // Compile-time usage - impossible with memcpy!
    constexpr uint32_t ct = float_bits_new(1.0f);
    static_assert(ct == 0x3f800000);
    std::cout << "constexpr: 0x" << ct << "\n";
    // Output: constexpr: 0x3f800000
}
```

**Safety improvements of `bit_cast` over `memcpy`:**

1. **`constexpr`** - can be used at compile time; `memcpy` cannot.
2. **Size check** - `bit_cast` is a compile error if sizes don't match; `memcpy` silently corrupts.
3. **Trivially copyable check** - `bit_cast` rejects non-trivially-copyable types at compile time.
4. **No pointer aliasing** - `memcpy` requires taking addresses; `bit_cast` works on values.
5. **Concise** - a single expression instead of a declaration + memcpy + variable.

---

### Q2: Explain why reinterpret_cast and union punning are UB for type punning but bit_cast is not

**Answer:**

**`reinterpret_cast` - strict aliasing violation:**

The strict aliasing rule says the compiler is allowed to assume that a `float*` and a `uint32_t*` never point to the same memory. When you break that assumption, the compiler's optimizer can silently produce wrong code.

```cpp
float f = 3.14f;
// UB: accessing a float object through a uint32_t pointer
uint32_t bits = *reinterpret_cast<uint32_t*>(&f);  // UNDEFINED BEHAVIOR
```

The **strict aliasing rule** ([basic.lval] §6.7.2) states that accessing an object through a pointer/reference of a different type is undefined behavior (with exceptions for `char`, `unsigned char`, `std::byte`). The compiler is free to assume `float*` and `uint32_t*` never alias, and may optimize away the read entirely.

**Union punning - reading inactive member:**

This one surprises people who know C, because C99 explicitly allows it. C++ does not.

```cpp
union Pun {
    float f;
    uint32_t u;
};
Pun p;
p.f = 3.14f;
uint32_t bits = p.u;  // UB in C++ (defined in C99, but NOT in C++)
```

In C++, reading from a union member that wasn't the last one written to is undefined behavior. (C allows it as an extension, but C++ does not - only `std::launder` or `memcpy` can make this work.)

**`std::bit_cast` - well-defined:**

`bit_cast` sidesteps the aliasing problem entirely because it never creates a pointer to the original object. It creates a brand-new object of the destination type with the copied bytes.

```cpp
float f = 3.14f;
uint32_t bits = std::bit_cast<uint32_t>(f);  // Well-defined!
```

`bit_cast` copies the **object representation** (raw bytes) of the source into a new object of the destination type. No aliasing occurs - a new `uint32_t` object is created with the copied bytes. The compiler generates the same code as `memcpy` but with compile-time safety guarantees.

---

### Q3: Show a bit_cast use case for extracting the exponent bits of an IEEE 754 float

**Answer:**

This is exactly the scenario `bit_cast` was designed for. You need to inspect the raw binary layout of a float, which means treating its bytes as an integer - and all three utility functions below are `constexpr`, so they can be verified at compile time too.

```cpp
#include <bit>
#include <cstdint>
#include <iostream>
#include <cmath>

// IEEE 754 single-precision float layout (32 bits):
//   [31]    sign (1 bit)
//   [30:23] exponent (8 bits, biased by 127)
//   [22:0]  mantissa (23 bits)

constexpr int extract_exponent(float f) {
    uint32_t bits = std::bit_cast<uint32_t>(f);
    uint32_t exponent_field = (bits >> 23) & 0xFF;  // extract bits [30:23]
    return static_cast<int>(exponent_field) - 127;   // remove bias
}

constexpr bool is_denormalized(float f) {
    uint32_t bits = std::bit_cast<uint32_t>(f);
    uint32_t exponent_field = (bits >> 23) & 0xFF;
    return exponent_field == 0 && (bits & 0x7FFFFF) != 0;
}

constexpr float set_sign_bit(float f) {
    uint32_t bits = std::bit_cast<uint32_t>(f);
    bits |= (1u << 31);  // set sign bit
    return std::bit_cast<float>(bits);
}

int main() {
    std::cout << "exponent of 1.0   = " << extract_exponent(1.0f)   << "\n";
    std::cout << "exponent of 8.0   = " << extract_exponent(8.0f)   << "\n";
    std::cout << "exponent of 0.5   = " << extract_exponent(0.5f)   << "\n";
    std::cout << "exponent of 0.125 = " << extract_exponent(0.125f) << "\n";
    // Output:
    // exponent of 1.0   = 0
    // exponent of 8.0   = 3
    // exponent of 0.5   = -1
    // exponent of 0.125 = -3

    // Compile-time usage
    static_assert(extract_exponent(1.0f) == 0);
    static_assert(extract_exponent(2.0f) == 1);
    static_assert(extract_exponent(0.25f) == -2);

    // Negate a float by flipping sign bit
    float neg = set_sign_bit(3.14f);
    std::cout << "set_sign_bit(3.14) = " << neg << "\n";
    // Output: set_sign_bit(3.14) = -3.14

    // Denormalized check
    float tiny = 1e-40f;
    std::cout << "is_denormalized(1e-40) = " << std::boolalpha
              << is_denormalized(tiny) << "\n";
    // Output: is_denormalized(1e-40) = true
}
```

**Why `bit_cast` is ideal here:**

- Extracting IEEE 754 fields requires interpreting float bytes as an integer - exactly what `bit_cast` is designed for.
- Being `constexpr`, all these utility functions work at compile time (`static_assert`).
- No UB, no `memcpy` verbosity, single-expression conversions.

---

## Notes

- `std::bit_cast` is in `<bit>` (not `<type_traits>` or `<cstring>`).
- If `From` contains padding bits, the value of those bits in `To` is unspecified.
- `bit_cast` with pointer types is rarely useful - the result is a pointer whose bits happen to match, but it doesn't point to a valid object.
- For double <-> uint64_t punning, the same pattern applies: `std::bit_cast<uint64_t>(some_double)`.
- On all major compilers (GCC, Clang, MSVC), `bit_cast` compiles to zero instructions in optimized builds - same as `memcpy`.
