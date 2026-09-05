# Use std::bit_cast and std::start_lifetime_as for safe binary parsing

**Category:** Serialization and Data Formats  
**Standard:** C++20, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/bit_cast> · <https://en.cppreference.com/w/cpp/memory/start_lifetime_as>  

---

## Topic Overview

Parsing binary data traditionally relies on `reinterpret_cast` or `memcpy`, both of which can violate strict aliasing rules. C++20 provides `std::bit_cast` for safe type punning of trivially copyable types, and C++23 provides `std::start_lifetime_as` for reinterpreting byte buffers as typed objects.

The reason this matters is subtle but important. The compiler is allowed to assume that two pointers of unrelated types do not alias the same memory. When you use `reinterpret_cast` to treat a `uint8_t*` buffer as a `PacketHeader*`, you break that assumption - the compiler may optimize away reads that it thinks cannot have changed, producing values that are silently wrong. This class of bug is notoriously hard to find because the code looks correct and usually produces the right answer until an optimizer gets involved.

### The Problem with reinterpret_cast

Here is the problem and its two safe alternatives side by side. The `memcpy` solution is always correct - every compiler understands that `memcpy` creates a valid copy - and modern compilers optimize it to a single load instruction on aligned data:

```cpp
#include <cstdint>
#include <cstring>

struct PacketHeader {
    uint32_t magic;
    uint16_t version;
    uint16_t payload_length;
};
static_assert(std::is_trivially_copyable_v<PacketHeader>);

// BAD: strict aliasing violation - undefined behavior!
void parse_bad(const uint8_t* buffer) {
    auto* header = reinterpret_cast<const PacketHeader*>(buffer);
    // Reading through header is UB - buffer's actual type is uint8_t[]
    uint32_t magic = header->magic;  // UB!
}

// GOOD: memcpy approach (always safe)
PacketHeader parse_memcpy(const uint8_t* buffer) {
    PacketHeader hdr;
    std::memcpy(&hdr, buffer, sizeof(hdr));
    return hdr;  // Values are well-defined
}
```

The `reinterpret_cast` version compiles without warnings and often produces the right result in debug builds. The UB only tends to show up when the optimizer is enabled - which is exactly when you want your production code to work correctly.

### std::bit_cast (C++20)

`std::bit_cast` is the clean, modern alternative to the `memcpy` trick for scalar types. Both types must be the same size and trivially copyable. The big advantage over `memcpy` is that `bit_cast` is `constexpr` - you can use it in compile-time computations, which opens up things like inspecting IEEE 754 floating-point bit patterns at compile time:

```cpp
#include <bit>
#include <cstdint>
#include <array>
#include <iostream>

// bit_cast: safe reinterpretation of one trivially copyable type as another
// Both types must have the same size

void demonstrate_bit_cast() {
    // Float -> uint32 (inspect IEEE 754 bits)
    float pi = 3.14159f;
    auto bits = std::bit_cast<uint32_t>(pi);
    std::cout << "pi bits: 0x" << std::hex << bits << "\n";
    // 0x40490fd0

    // Round-trip: uint32 -> float
    auto restored = std::bit_cast<float>(bits);
    std::cout << "Restored: " << std::dec << restored << "\n";
    // 3.14159

    // Array of bytes -> struct
    std::array<uint8_t, 8> packet = {0x4D, 0x50, 0x50, 0x43,  // magic
                                      0x01, 0x00,               // version=1
                                      0x20, 0x00};              // length=32

    // bit_cast requires same size
    static_assert(sizeof(packet) == sizeof(PacketHeader));
    auto header = std::bit_cast<PacketHeader>(packet);
    std::cout << "Version: " << header.version << "\n";
}
```

Notice that `bit_cast` from `std::array<uint8_t, 8>` to `PacketHeader` works only because both are exactly 8 bytes. The compiler enforces this at compile time - if the sizes differ, you get a hard error rather than silent UB.

### std::start_lifetime_as (C++23)

`start_lifetime_as` solves a slightly different problem: instead of copying bytes into a new object, it tells the runtime "an object of this type now exists at this address in the existing buffer." Think of it as the thing `reinterpret_cast` should have been from the start - it legitimizes the reinterpretation rather than just pretending the problem away.

The practical difference from `bit_cast` is that `start_lifetime_as` does not copy anything. It works in-place on the original buffer, which is essential for zero-copy parsing scenarios:

```cpp
#include <memory>
#include <cstdint>
#include <iostream>
#include <vector>

// start_lifetime_as: implicitly creates an object of type T
// in an existing byte buffer, making it legal to access.
// This is what reinterpret_cast SHOULD have been.

void parse_with_start_lifetime_as(std::vector<uint8_t>& buffer) {
    // Before C++23: reinterpret_cast is UB
    // After C++23: start_lifetime_as makes it defined behavior

    auto* header = std::start_lifetime_as<PacketHeader>(buffer.data());
    // Now header points to a valid PacketHeader object
    // whose lifetime has been implicitly started in the buffer

    std::cout << "Magic: 0x" << std::hex << header->magic << "\n";
    std::cout << "Version: " << std::dec << header->version << "\n";
    std::cout << "Payload: " << header->payload_length << " bytes\n";
}

// For arrays within a buffer:
void parse_array(uint8_t* buffer, size_t count) {
    auto* arr = std::start_lifetime_as_array<uint32_t>(buffer, count);
    for (size_t i = 0; i < count; ++i) {
        std::cout << "arr[" << i << "] = " << arr[i] << "\n";
    }
}
```

---

## Self-Assessment

### Q1: Show why reinterpret_cast violates strict aliasing and how memcpy/bit_cast fix it

The strict aliasing rule says that the compiler is allowed to assume a `uint8_t[]` and a `Header*` can never point to the same memory - so it is free to reorder or cache reads across them. `memcpy` and `bit_cast` both copy the bits into a fresh object of the right type, which the compiler knows is a valid `Header`, so there is no aliasing issue at all:

```cpp
#include <cstdint>
#include <cstring>
#include <bit>
#include <iostream>
#include <array>

// The strict aliasing rule says: you can only access an object through
// a pointer/reference of the same type (or char/byte types).
// Accessing a uint8_t[] through a PacketHeader* violates this.

struct Header {
    uint32_t id;
    uint32_t size;
};

void example() {
    uint8_t buffer[8] = {1, 0, 0, 0, 64, 0, 0, 0};

    // BAD: strict aliasing violation
    // auto* h = reinterpret_cast<Header*>(buffer);
    // std::cout << h->id;  // UB - compiler may optimize this away

    // GOOD: memcpy (always safe, compiler optimizes to single load)
    Header h1;
    std::memcpy(&h1, buffer, sizeof(h1));
    std::cout << "memcpy: id=" << h1.id << " size=" << h1.size << "\n";

    // GOOD: bit_cast (C++20, constexpr-friendly)
    std::array<uint8_t, 8> arr;
    std::memcpy(arr.data(), buffer, 8);
    auto h2 = std::bit_cast<Header>(arr);
    std::cout << "bit_cast: id=" << h2.id << " size=" << h2.size << "\n";

    // Output: id=1 size=64
}

int main() { example(); }
```

### Q2: Parse a network packet using start_lifetime_as without UB

The `[[gnu::packed]]` attribute on `NetworkPacket` removes alignment padding, which is exactly what you want for network packet headers that arrive in a fixed byte layout. The `start_lifetime_as` call then makes accessing those bytes through the struct pointer well-defined:

<!-- compile: needs C++26 library support for `std::start_lifetime_as` (not in the CI standard library yet) -->
```cpp
#include <cstdint>
#include <memory>
#include <vector>
#include <iostream>
#include <algorithm>
#include <type_traits>

struct [[gnu::packed]] NetworkPacket {
    uint8_t  version;
    uint8_t  type;
    uint16_t length;
    uint32_t sequence;
    // followed by payload bytes
};
static_assert(std::is_trivially_copyable_v<NetworkPacket>);

void process_packet(std::vector<uint8_t>& raw) {
    if (raw.size() < sizeof(NetworkPacket)) {
        std::cerr << "Buffer too small\n";
        return;
    }

    // C++23: start lifetime of NetworkPacket in the buffer
    auto* pkt = std::start_lifetime_as<NetworkPacket>(raw.data());

    std::cout << "Version:  " << static_cast<int>(pkt->version) << "\n";
    std::cout << "Type:     " << static_cast<int>(pkt->type) << "\n";
    std::cout << "Length:   " << pkt->length << "\n";
    std::cout << "Sequence: " << pkt->sequence << "\n";

    // Access payload after the header
    const uint8_t* payload = raw.data() + sizeof(NetworkPacket);
    size_t payload_len = raw.size() - sizeof(NetworkPacket);
    std::cout << "Payload: " << payload_len << " bytes\n";
}

int main() {
    std::vector<uint8_t> raw = {
        0x01,                   // version
        0x03,                   // type
        0x10, 0x00,             // length = 16
        0x2A, 0x00, 0x00, 0x00, // sequence = 42
        'H', 'e', 'l', 'l', 'o' // payload
    };
    process_packet(raw);
}
```

### Q3: Use bit_cast in a constexpr context to inspect floating-point representation

Because `bit_cast` is `constexpr`, you can use it to write compile-time assertions about IEEE 754 bit patterns. This example extracts the sign bit, exponent, and mantissa at compile time - the `static_assert` lines verify your understanding of the float layout before the program even runs:

```cpp
#include <bit>
#include <cstdint>
#include <iostream>

// bit_cast is constexpr - can inspect bit patterns at compile time!
constexpr uint32_t float_bits(float f) {
    return std::bit_cast<uint32_t>(f);
}

constexpr bool is_negative(float f) {
    return (float_bits(f) >> 31) & 1;
}

constexpr uint8_t float_exponent(float f) {
    return static_cast<uint8_t>((float_bits(f) >> 23) & 0xFF);
}

// Compile-time assertions about IEEE 754
static_assert(float_bits(1.0f) == 0x3F800000);
static_assert(float_bits(-1.0f) == 0xBF800000);
static_assert(is_negative(-3.14f));
static_assert(!is_negative(3.14f));
static_assert(float_exponent(1.0f) == 127);  // Bias = 127

int main() {
    constexpr float val = 3.14159f;
    constexpr auto bits = float_bits(val);

    std::cout << "3.14159f bits: 0x" << std::hex << bits << "\n";
    std::cout << "Sign: " << is_negative(val) << "\n";
    std::cout << "Exponent (biased): " << std::dec
              << static_cast<int>(float_exponent(val)) << "\n";

    // NaN detection at compile time
    constexpr float nan = 0.0f / 0.0f;
    constexpr auto nan_bits = float_bits(nan);
    constexpr bool is_nan = (nan_bits & 0x7F800000) == 0x7F800000
                         && (nan_bits & 0x007FFFFF) != 0;
    static_assert(is_nan);

    std::cout << "NaN detected at compile time: " << is_nan << "\n";
}
```

The NaN detection at compile time is a nice demonstration of how useful constexpr bit manipulation can be - you can encode knowledge about floating-point edge cases directly into your type system.

---

## Notes

- `std::bit_cast` requires both types to be **trivially copyable** and the **same size**.
- `std::bit_cast` is **constexpr** - use it for compile-time bit manipulation.
- `std::start_lifetime_as` replaces the `reinterpret_cast` pattern for parsing buffers - it implicitly creates objects (P0593R6).
- `memcpy` is always safe and optimizes to zero instructions on modern compilers - prefer it over `reinterpret_cast` in pre-C++23 code.
- For network protocols, always validate buffer sizes before accessing headers.
