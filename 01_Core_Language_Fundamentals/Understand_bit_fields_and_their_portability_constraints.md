# Understand bit fields and their portability constraints

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/bit_field>  

---

## Topic Overview

A **bit field** is a class data member with an explicit width in bits. Bit fields let you pack
multiple values into a single storage unit, which is useful for hardware registers, protocol
headers, and memory-constrained systems. The catch is that the C++ standard deliberately
leaves most of the layout details up to the implementation - so portability requires care.

### Basic Syntax

The width comes after the member name, separated by a colon. The total bits add up within
the underlying storage type:

```cpp
struct Flags {
    unsigned int readable  : 1;   // 1 bit
    unsigned int writable  : 1;   // 1 bit
    unsigned int executable: 1;   // 1 bit
    unsigned int reserved  : 5;   // 5 bits - padding to byte boundary
};
// sizeof(Flags) is typically 4 bytes (one unsigned int storage unit)
```

### What Is Implementation-Defined

Most of the layout details you might want to rely on are explicitly left unspecified by the
standard. Here's a summary of what you cannot count on:

| Aspect | Guaranteed? | Notes |
| --- | --- | --- |
| Total number of bits used | No | Padding between fields is impl-defined |
| Bit ordering within storage unit (MSB first or LSB first) | No | Big-endian vs little-endian machines differ |
| Whether a bit field can straddle storage unit boundaries | No | Some compilers pad, others bridge |
| Signedness of plain `int` bit field | No | Use `unsigned int` or `signed int` explicitly |
| Alignment of the storage unit | No | May differ across compilers |

### Hardware Register Example

This is the classic embedded use case - modeling a hardware control register. The
`static_assert` confirms the size on your platform, but you still cannot rely on bit order
being the same across compilers or targets:

```cpp
#include <cstdint>

// SPI control register - 8 bits
struct SPI_CR {
    uint8_t enable    : 1;   // bit 0
    uint8_t mode      : 2;   // bits 1-2 (4 SPI modes)
    uint8_t clock_div : 3;   // bits 3-5 (divide by 1..8)
    uint8_t interrupt : 1;   // bit 6
    uint8_t reserved  : 1;   // bit 7
};
static_assert(sizeof(SPI_CR) == 1);
// WARNING: bit ordering depends on compiler + target endianness!
```

### Unnamed Bit Fields (Padding)

An unnamed bit field reserves bits without giving them a name. A zero-width unnamed field
forces alignment to the next storage unit boundary:

```cpp
struct StatusReg {
    unsigned ready    : 1;
    unsigned          : 3;    // 3 bits of padding (unnamed)
    unsigned error    : 1;
    unsigned          : 0;    // force alignment to next storage unit boundary
    unsigned overflow : 1;
};
```

### Cannot Take Address

Because a bit field may not start on a byte boundary, you can't form a pointer or reference
to one. The compiler will reject the attempt:

```cpp
struct Bits { unsigned x : 3; };
Bits b;
// auto* p = &b.x;   // ERROR: cannot take address of bit field
// Can't use references either: auto& r = b.x; // ERROR
```

### Using `std::bit_cast` for Serialization (C++20)

When you need to treat a bit-field struct as a raw integer (for writing to a register or
serializing over a bus), `std::bit_cast` lets you do it without undefined behavior - as long
as the types have the same size and the source is trivially copyable:

```cpp
#include <bit>
#include <cstdint>
#include <iostream>

struct PackedFlags {
    uint8_t a : 1;
    uint8_t b : 3;
    uint8_t c : 4;
};
static_assert(sizeof(PackedFlags) == 1);

int main() {
    PackedFlags f{.a = 1, .b = 5, .c = 10};

    // Convert to integer for serialization
    uint8_t raw = std::bit_cast<uint8_t>(f);
    std::cout << "Raw byte: 0x" << std::hex << +raw << "\n";

    // Deserialize back
    PackedFlags f2 = std::bit_cast<PackedFlags>(raw);
    std::cout << "a=" << f2.a << " b=" << f2.b << " c=" << f2.c << "\n";
}
// WARNING: the raw byte value depends on the platform's bit layout!
```

The round-trip works fine on the same compiler and platform; the raw value is not portable
across different targets.

---

## Self-Assessment

### Q1: Write a struct with bit fields for a hardware register and explain layout guarantees

Here's a UART Line Control Register modeled as a bit-field struct. The `static_assert`
catches size surprises at compile time, but notice all the caveats about what is still not
guaranteed:

```cpp
#include <cstdint>
#include <iostream>

// Modeling a UART Line Control Register (LCR) - 8 bits
struct UART_LCR {
    uint8_t data_bits   : 2;   // 00=5, 01=6, 10=7, 11=8
    uint8_t stop_bits   : 1;   // 0=1 stop bit, 1=2 stop bits
    uint8_t parity_en   : 1;   // parity enable
    uint8_t parity_type : 2;   // 00=odd, 01=even, 10=mark, 11=space
    uint8_t break_ctrl  : 1;   // break control
    uint8_t dlab        : 1;   // divisor latch access bit
};

static_assert(sizeof(UART_LCR) == 1, "Must fit in one byte");

int main() {
    UART_LCR lcr{};
    lcr.data_bits = 3;     // 8-bit data
    lcr.stop_bits = 0;     // 1 stop bit
    lcr.parity_en = 0;     // no parity

    std::cout << "data_bits: " << +lcr.data_bits << "\n";
    std::cout << "stop_bits: " << +lcr.stop_bits << "\n";
}
```

**Layout guarantees (and lack thereof):**

- The **total size** respects the underlying type (`uint8_t` -> 1 byte), but padding may increase it.
- **Bit order within the byte is implementation-defined**: on x86 (little-endian, GCC/Clang), `data_bits` occupies bits 0-1 of the byte. On a big-endian platform, it might be bits 6-7.
- The standard guarantees bit fields of the same type declared consecutively are packed into the same storage unit, but not the internal ordering.
- For portable register access, use explicit shifts and masks instead.

### Q2: Show why the order of bit fields within a storage unit is implementation-defined

This example makes the platform-dependence concrete. Set two nibbles, read back the raw
byte, and you'll see that the mapping depends on the platform's bit ordering convention:

```cpp
#include <cstdint>
#include <cstring>
#include <iostream>
#include <bitset>

struct TwoBits {
    uint8_t low  : 4;   // "first" field
    uint8_t high : 4;   // "second" field
};
static_assert(sizeof(TwoBits) == 1);

int main() {
    TwoBits t{};
    t.low  = 0xA;   // 1010 in binary
    t.high = 0x5;   // 0101 in binary

    uint8_t raw;
    std::memcpy(&raw, &t, 1);

    std::cout << "Raw byte: 0x" << std::hex << +raw << "\n";
    std::cout << "Binary:   " << std::bitset<8>(raw) << "\n";

    // On little-endian GCC/Clang/MSVC x86:
    //   raw = 0x5A  (high nibble in upper bits, low nibble in lower bits)
    // On a big-endian platform:
    //   raw could be 0xA5 (low nibble stored first = at high bits)
    //
    // The standard says: "Allocation of bit-fields within a class object
    // is implementation-defined." [class.bit]

    // Portable alternative - manual bit packing:
    uint8_t portable = (uint8_t(0x5) << 4) | uint8_t(0xA);
    std::cout << "Portable: 0x" << +portable << "\n";   // always 0x5A
}
```

**Why it's implementation-defined:**

- The C++ standard ([class.bit]) intentionally leaves bit allocation order unspecified to allow compilers to match the target architecture's natural bit ordering.
- Little-endian machines (x86) typically fill from LSB to MSB.
- Big-endian machines (PowerPC, SPARC) may fill from MSB to LSB.
- Even on the same architecture, different compilers may pack differently.

### Q3: Use `std::bit_cast` to safely read a bit field struct as an integer for serialization

`std::bit_cast` is the right tool for this job - it's UB-free and the compiler can see
through it for optimization. Just remember the result is only meaningful on the same platform:

```cpp
#include <bit>
#include <cstdint>
#include <iostream>
#include <bitset>

struct Packet {
    uint16_t version  : 4;
    uint16_t type     : 4;
    uint16_t length   : 8;
};
static_assert(sizeof(Packet) == 2);
static_assert(std::is_trivially_copyable_v<Packet>);   // required for bit_cast

int main() {
    Packet pkt{.version = 3, .type = 7, .length = 200};

    // Serialize: struct -> integer
    uint16_t raw = std::bit_cast<uint16_t>(pkt);
    std::cout << "Serialized: 0x" << std::hex << raw << "\n";
    std::cout << "Bits: " << std::bitset<16>(raw) << "\n";

    // Transmit raw over network / write to file...

    // Deserialize: integer -> struct
    Packet pkt2 = std::bit_cast<Packet>(raw);
    std::cout << std::dec;
    std::cout << "version=" << pkt2.version
              << " type=" << pkt2.type
              << " length=" << pkt2.length << "\n";

    // CAUTION: the raw value is only meaningful on the SAME compiler + platform.
    // For cross-platform serialization, use explicit shifts and masks:
    uint16_t portable = (uint16_t(pkt.version) & 0xF)
                      | ((uint16_t(pkt.type) & 0xF) << 4)
                      | (uint16_t(pkt.length) << 8);
    std::cout << "Portable: 0x" << std::hex << portable << "\n";
}
```

**How it works:**

- `std::bit_cast<uint16_t>(pkt)` reinterprets the bit-field struct's bytes as a `uint16_t` with no UB (requires trivially copyable + same size).
- For same-platform use (e.g., writing to a memory-mapped register), `bit_cast` is safe and efficient.
- For cross-platform serialization, **always use manual bit packing** with explicit shifts - bit field layout varies across platforms.

---

## Notes

- **Never use bit fields for cross-platform binary protocols** - use shifts and masks instead.
- Use `uint8_t`, `uint16_t`, etc. as the underlying type for predictable size.
- Bit fields cannot be `static` or have `auto` storage duration for individual bits.
- C++20 allows designated initializers for bit-field structs: `Packet{.version = 3}`.
- In embedded programming, bit fields are common but always verify the compiler's layout with a `static_assert` and a known test value.
