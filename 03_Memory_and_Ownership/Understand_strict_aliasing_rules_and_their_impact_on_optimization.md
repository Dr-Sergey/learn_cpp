# Understand Strict Aliasing Rules and Their Impact on Optimization

**Category:** Memory and Ownership  
**Standard:** C++98  
**Reference:** <https://en.cppreference.com/w/cpp/language/reinterpret_cast>  

---

## Topic Overview

### What Is Strict Aliasing

The **strict aliasing rule** says: an object of type `T` can only be accessed through a pointer/reference of:

1. The same type `T` (with any cv-qualification)
2. A signed/unsigned variant of `T`
3. `char`, `unsigned char`, or `std::byte` (the "byte aliasing" exemption)

Violating this rule is **undefined behavior**. The reason this matters so much in practice is that the compiler is allowed - and actively encouraged - to assume pointers to unrelated types can never point at the same memory. That assumption is what unlocks a whole class of powerful optimizations.

### Why Compilers Care

When the compiler sees `int*` and `float*` in the same function, it knows they can't possibly alias. That means it can freely cache, reorder, and eliminate redundant loads. Here is a simplified illustration:

```cpp
// The compiler sees:
void update(int* a, float* b) {
    *a = 1;
    *b = 2.0f;
    // Compiler KNOWS *a is still 1 — int* and float* can't alias
    // -> can reorder, cache in registers, eliminate reloads
}
```

If aliasing were allowed between arbitrary types, the compiler would have to re-read `*a` after every write through `*b` just in case they overlap. Strict aliasing is what lets it skip those extra loads.

| Access through | Aliasing allowed? | Standard basis |
| --- | --- | --- |
| `T*` -> `T` | Yes | Same type |
| `int*` -> `unsigned int` | Yes | Signed/unsigned variant |
| `char*` -> anything | Yes | Byte aliasing exemption |
| `int*` -> `float` | **NO - UB** | Unrelated types |
| `Base*` -> `Derived` | Yes | Polymorphic access |

---

## Self-Assessment

### Q1: Show that accessing a `float` through an `int*` violates strict aliasing and can be misoptimized

The classic example of this mistake is type-punning by casting a pointer - something that was common before `memcpy` and `bit_cast` were understood as the correct tools. The fast inverse square root hack is the most famous victim. Notice how the "safe" version using `memcpy` expresses the same intent without the UB.

```cpp
#include <iostream>
#include <cstring>

// UNDEFINED BEHAVIOR: type punning via pointer cast
float bad_int_bits_of_float(float f) {
    // This violates strict aliasing!
    int* ip = reinterpret_cast<int*>(&f);
    *ip += 1;  // UB: accessing float storage through int*
    return f;
    // The compiler may assume f is unchanged (int* can't alias float*)
    // and optimize away the modification entirely!
}

// What ACTUALLY can happen with -O2:
// 1. Compiler caches f in a register before the int* write
// 2. Returns the ORIGINAL value of f (modification "lost")
// 3. Or generates completely broken code
// 4. Different behavior between -O0 and -O2

// Classic "fast inverse square root" — also UB!
float fast_inv_sqrt_BAD(float x) {
    // float -> int type punning via pointer cast = UB
    int i = *reinterpret_cast<int*>(&x);  // STRICT ALIASING VIOLATION
    i = 0x5f3759df - (i >> 1);
    return *reinterpret_cast<float*>(&i);  // UB again
}

// Correct version using memcpy (no UB)
float fast_inv_sqrt_GOOD(float x) {
    int i;
    std::memcpy(&i, &x, sizeof(i));       // OK: memcpy is always safe
    i = 0x5f3759df - (i >> 1);
    float result;
    std::memcpy(&result, &i, sizeof(result));
    return result;
}

int main() {
    std::cout << "=== Strict aliasing violation demo ===\n\n";

    float val = 1.0f;
    std::cout << "Original:   " << val << "\n";

    // The UB version may or may not "work" depending on optimization
    // float bad = bad_int_bits_of_float(val);
    // std::cout << "UB result:  " << bad << " (unpredictable!)\n";

    // Safe version always works
    float good = fast_inv_sqrt_GOOD(2.0f);
    std::cout << "Safe inv_sqrt(2): " << good << "\n";

    // Demonstration of the problem:
    std::cout << "\n=== Why it matters ===\n";
    std::cout << "With -O2, the compiler assumes int* and float* don't alias.\n";
    std::cout << "It can reorder/eliminate writes, cache in registers,\n";
    std::cout << "and produce different results than -O0.\n";
    std::cout << "Use -fno-strict-aliasing to disable (GCC/Clang),\n";
    std::cout << "but the CORRECT fix is to use memcpy or bit_cast.\n";

    return 0;
}
```

### Q2: Use `std::memcpy` or `std::bit_cast` as the correct alternatives for type punning

Both approaches get you the bits of one type reinterpreted as another type without touching the aliasing rules. `memcpy` has been the safe idiom since C++98 and all major compilers optimize it to a no-op register move at `-O1`. `std::bit_cast` (C++20) is the modern replacement - it is `constexpr`, type-safe, and reads like it means. The `show_float_bits` helper at the bottom is a nice practical application of `bit_cast`.

```cpp
#include <iostream>
#include <cstring>
#include <bit>        // C++20: std::bit_cast
#include <cstdint>
#include <iomanip>

// Method 1: std::memcpy — works in all C++ standards
uint32_t float_to_bits_memcpy(float f) {
    uint32_t bits;
    static_assert(sizeof(float) == sizeof(uint32_t));
    std::memcpy(&bits, &f, sizeof(bits));  // No UB — byte copy
    return bits;
}

float bits_to_float_memcpy(uint32_t bits) {
    float f;
    std::memcpy(&f, &bits, sizeof(f));
    return f;
}

// Method 2: std::bit_cast (C++20) — constexpr, zero overhead
uint32_t float_to_bits_bitcast(float f) {
    return std::bit_cast<uint32_t>(f);  // No UB, constexpr-capable
}

float bits_to_float_bitcast(uint32_t bits) {
    return std::bit_cast<float>(bits);
}

// Demonstration: examining IEEE 754 representation
void show_float_bits(float f) {
    uint32_t bits = std::bit_cast<uint32_t>(f);
    std::cout << std::setw(12) << f << " -> 0x"
              << std::hex << std::setfill('0') << std::setw(8) << bits
              << std::dec << std::setfill(' ')
              << "  sign=" << (bits >> 31)
              << " exp=" << ((bits >> 23) & 0xFF)
              << " mant=0x" << std::hex << (bits & 0x7FFFFF)
              << std::dec << "\n";
}

int main() {
    std::cout << "=== Safe type punning ===\n\n";

    // memcpy approach
    float original = 3.14f;
    uint32_t bits = float_to_bits_memcpy(original);
    float roundtrip = bits_to_float_memcpy(bits);
    std::cout << "memcpy: " << original << " -> 0x"
              << std::hex << bits << std::dec
              << " -> " << roundtrip << "\n";

    // bit_cast approach (C++20)
    uint32_t bits2 = float_to_bits_bitcast(original);
    float roundtrip2 = bits_to_float_bitcast(bits2);
    std::cout << "bit_cast: " << original << " -> 0x"
              << std::hex << bits2 << std::dec
              << " -> " << roundtrip2 << "\n";

    // constexpr bit_cast!
    constexpr float pi = 3.14159f;
    constexpr uint32_t pi_bits = std::bit_cast<uint32_t>(pi);
    std::cout << "\nconstexpr bit_cast: pi bits = 0x"
              << std::hex << pi_bits << std::dec << "\n";

    // IEEE 754 analysis
    std::cout << "\n=== IEEE 754 bit layout ===\n";
    show_float_bits(1.0f);
    show_float_bits(-1.0f);
    show_float_bits(0.0f);
    show_float_bits(3.14f);

    std::cout << "\n=== Comparison ===\n";
    std::cout << "reinterpret_cast: UB (strict aliasing violation)\n";
    std::cout << "union type pun:   UB in C++ (allowed in C)\n";
    std::cout << "memcpy:           Safe, all standards, optimized to no-op\n";
    std::cout << "bit_cast (C++20): Safe, constexpr, zero overhead\n";

    return 0;
}
```

### Q3: Explain the `char*` exemption from strict aliasing and why it exists

The exemption exists because byte-level access to any object is fundamental - `memcpy`, `memset`, serialization, and network I/O all need to read an object as raw bytes without caring about its type. However, the exemption is strictly one-way: `char*` can alias anything, but that does not mean any pointer can alias a `char` buffer as if it were a different type. Reading back through an `int*` into a `char` buffer requires that a proper `int` object actually lives there.

```cpp
#include <iostream>
#include <cstring>
#include <cstddef>
#include <cstdint>

// The standard allows accessing ANY object through:
//   char*, unsigned char*, std::byte* (C++17)
//
// WHY? Because:
// 1. memcpy, memmove, memset need to work on any type
// 2. Serialization/deserialization reads raw bytes
// 3. Network I/O operates on byte buffers
// 4. The "object representation" IS a sequence of bytes

struct Packet {
    uint32_t id;
    float value;
    char tag;
};

int main() {
    std::cout << "=== char* aliasing exemption ===\n\n";

    // Legal: examine any object through char*
    Packet pkt{42, 3.14f, 'A'};

    // Reading bytes of any object — ALWAYS LEGAL
    const unsigned char* bytes = reinterpret_cast<const unsigned char*>(&pkt);
    std::cout << "Packet bytes: ";
    for (size_t i = 0; i < sizeof(pkt); ++i) {
        std::cout << std::hex << static_cast<int>(bytes[i]) << " ";
    }
    std::cout << std::dec << "\n";

    // Legal: copy bytes via char*
    Packet copy;
    std::memcpy(&copy, &pkt, sizeof(Packet));
    std::cout << "Copied: id=" << copy.id << " value=" << copy.value
              << " tag=" << copy.tag << "\n";

    // Legal: std::byte* (C++17)
    std::byte* bp = reinterpret_cast<std::byte*>(&pkt);
    std::cout << "First byte via std::byte: "
              << static_cast<int>(std::to_integer<int>(bp[0])) << "\n";

    // IMPORTANT: The exemption is ONE-WAY!
    // char* can alias anything -> OK
    // anything can alias char -> NOT necessarily OK!
    // int* CANNOT alias char[] unless the char[] actually contains an int object

    std::cout << "\n=== What the exemption DOES NOT allow ===\n";
    std::cout << "1. char[] -> cast to int* -> dereference: UB (no int object there)\n";
    std::cout << "2. Two unrelated types still can't alias each other\n";
    std::cout << "3. Only char/byte types get the exemption, not short/etc.\n";

    // Correct way to "create" an int in a char buffer:
    alignas(int) char buffer[sizeof(int)];
    int val = 42;
    std::memcpy(buffer, &val, sizeof(int));  // OK: byte copy
    int result;
    std::memcpy(&result, buffer, sizeof(int));  // OK: byte copy back
    std::cout << "\nSafe buffer roundtrip: " << result << "\n";

    return 0;
}
// Expected output (bytes vary by platform endianness):
// === char* aliasing exemption ===
//
// Packet bytes: 2a 0 0 0 c3 f5 48 40 41 ...
// Copied: id=42 value=3.14 tag=A
// First byte via std::byte: 42
//
// === What the exemption DOES NOT allow ===
// 1. char[] -> cast to int* -> dereference: UB (no int object there)
// 2. Two unrelated types still can't alias each other
// 3. Only char/byte types get the exemption, not short/etc.
//
// Safe buffer roundtrip: 42
```

---

## Notes

- Strict aliasing enables major optimizations: register caching, instruction reordering, dead store elimination.
- Use `-fno-strict-aliasing` (GCC/Clang) to disable, but prefer fixing the code instead.
- `std::bit_cast` (C++20) is the best modern solution - zero overhead, `constexpr`, no UB.
- `std::memcpy` is optimized away by all major compilers - no actual copy at `-O1` and above.
- Common violation: union type punning is UB in C++ (allowed in C99+). Use `memcpy` or `bit_cast` instead.
