# Use std::byte for type-safe raw memory manipulation

**Category:** Standard Library Utilities  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/types/byte>  

---

## Topic Overview

`std::byte` (defined in `<cstddef>`) is an enumeration type that represents raw memory bytes without the implicit conversions and arithmetic operations that `unsigned char` and `char` allow. It forces deliberate conversion, making raw memory code safer.

The design philosophy is worth understanding. When you manipulate raw memory, you are working with *bit patterns*, not numbers. `unsigned char` blurs that line: it behaves as both a byte type and an arithmetic type, letting you accidentally do things like add two raw bytes together. `std::byte` draws a hard boundary - it is a byte, not a number, and the compiler will tell you if you confuse the two.

### Definition

```cpp
enum class byte : unsigned char {};
```

Because it's a scoped enum, you **cannot** accidentally use it in arithmetic or comparisons with integers.

### Allowed Operations

| Operation | Supported | Syntax |
| --- | --- | --- |
| Bitwise OR | Yes | `b1 \| b2` |
| Bitwise AND | Yes | `b1 & b2` |
| Bitwise XOR | Yes | `b1 ^ b2` |
| Bitwise NOT | Yes | `~b` |
| Shift left/right | Yes | `b << n`, `b >> n` (n is an integer) |
| Addition | **No** | Compile error |
| Subtraction | **No** | Compile error |
| Multiplication | **No** | Compile error |
| Comparison (`==`, `<`) | Yes | Via default comparison |
| Convert to integer | Explicit | `std::to_integer<int>(b)` |
| Convert from integer | Explicit | `std::byte{42}` |

### Core Syntax

The compilation failure for `b + std::byte{1}` is the whole point. With `unsigned char`, that compiles silently. With `std::byte`, the compiler catches it.

```cpp
#include <cstddef>
#include <iostream>

int main() {
    std::byte b{0xFF};

    // Bitwise operations
    std::byte masked = b & std::byte{0x0F};
    std::cout << std::to_integer<int>(masked) << "\n";  // Output: 15

    // Shift
    std::byte shifted = std::byte{1} << 4;
    std::cout << std::to_integer<int>(shifted) << "\n"; // Output: 16

    // This would NOT compile:
    // std::byte x = b + std::byte{1};  // Error: no operator+ for std::byte
    // int y = b;                        // Error: no implicit conversion
}
```

---

## Self-Assessment

### Q1: Rewrite a function using `unsigned char*` to use `std::byte*` and show the semantic improvement

**Answer:**

The `hexdump_new` version is functionally identical to `hexdump_old`, but the compiler now prevents accidental arithmetic on the bytes. Any time you need the numeric value, you have to say so explicitly with `std::to_integer`.

```cpp
#include <cstddef>
#include <cstring>
#include <iostream>
#include <vector>

// OLD: using unsigned char - prone to accidental arithmetic
void hexdump_old(const unsigned char* data, size_t len) {
    for (size_t i = 0; i < len; ++i) {
        // Bug risk: nothing prevents data[i] + 1 or data[i] * 2
        unsigned char val = data[i];  // implicit arithmetic allowed
        printf("%02X ", val);
    }
    printf("\n");
}

// NEW: using std::byte - type-safe
void hexdump_new(const std::byte* data, size_t len) {
    for (size_t i = 0; i < len; ++i) {
        // Must explicitly convert to integer to print
        int val = std::to_integer<int>(data[i]);
        printf("%02X ", val);

        // These would NOT compile, catching bugs at compile time:
        // data[i] + std::byte{1};   // No arithmetic on std::byte
        // int x = data[i];          // No implicit conversion
    }
    printf("\n");
}

// Memory copy utility using std::byte
void safe_memcpy(std::byte* dst, const std::byte* src, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        dst[i] = src[i];  // Assignment is allowed
    }
}

int main() {
    int value = 0x12345678;

    // Old way
    hexdump_old(reinterpret_cast<const unsigned char*>(&value), sizeof(value));
    // Output (little-endian): 78 56 34 12

    // New way - same output, but type-safe
    hexdump_new(reinterpret_cast<const std::byte*>(&value), sizeof(value));
    // Output (little-endian): 78 56 34 12

    // Build raw bytes
    std::vector<std::byte> buffer = {
        std::byte{0xDE}, std::byte{0xAD},
        std::byte{0xBE}, std::byte{0xEF}
    };
    hexdump_new(buffer.data(), buffer.size());
    // Output: DE AD BE EF
}
```

**Semantic improvement:** With `unsigned char*`, the compiler allows arithmetic on bytes - addition, multiplication, comparisons with integers are all silently permitted. With `std::byte*`, accidental arithmetic is a compile error. You must explicitly use `std::to_integer` when you actually need the numeric value.

---

### Q2: Explain why std::byte does not support arithmetic but does support bitwise operators

**Answer:**

`std::byte` is designed to represent **raw memory** - individual bytes of storage with no numeric meaning. The design principle is:

**Bitwise operations make sense for raw memory:**

- Masking specific bits: `byte & mask` (extracting flags, protocol fields)
- Setting bits: `byte | flag`
- Toggling bits: `byte ^ toggle`
- Shifting bit patterns: `byte << n`

These are all meaningful operations on bit patterns.

**Arithmetic operations do NOT make sense:**

- "Add 1 to a byte of raw memory" has no inherent meaning
- "Multiply two memory bytes" is semantically meaningless
- These would indicate you're treating the byte as a number, which is what `uint8_t` is for

Notice in the code below that when you genuinely need the numeric value, you convert explicitly. The cast makes the intent visible.

```cpp
#include <cstddef>
#include <iostream>

int main() {
    std::byte flags{0b1010'0101};

    // Bitwise: meaningful - extract bit 3
    std::byte bit3 = flags & std::byte{0b0000'1000};
    bool is_set = std::to_integer<int>(bit3) != 0;
    std::cout << "bit 3 set: " << std::boolalpha << is_set << "\n";
    // Output: bit 3 set: false

    // Bitwise: meaningful - set bit 3
    flags = flags | std::byte{0b0000'1000};
    std::cout << "after set: " << std::to_integer<int>(flags) << "\n";
    // Output: after set: 173  (0b1010'1101)

    // These correctly do NOT compile:
    // flags + std::byte{1};       // Error
    // flags * std::byte{2};       // Error
    // flags - std::byte{1};       // Error
    // if (flags > std::byte{0})   // Error (no ordering - use to_integer first)

    // When you need arithmetic, be explicit:
    int numeric = std::to_integer<int>(flags);
    numeric += 1;  // Now you're working with a number, not raw memory
    std::cout << "numeric + 1 = " << numeric << "\n";
    // Output: numeric + 1 = 174
}
```

---

### Q3: Show how std::byte interacts with std::span<std::byte> for binary serialization

**Answer:**

`std::span<std::byte>` is the modern way to pass a view of raw binary data. It replaces the old `void* data, size_t len` pattern with something typed, bounds-aware, and composable.

```cpp
#include <cstddef>
#include <span>
#include <vector>
#include <iostream>
#include <cstring>
#include <bit>

// Serialize any trivially copyable type to a byte span
template<typename T>
    requires std::is_trivially_copyable_v<T>
std::span<const std::byte> as_bytes_view(const T& obj) {
    return std::span<const std::byte>(
        reinterpret_cast<const std::byte*>(&obj), sizeof(T));
}

// Deserialize from a byte span
template<typename T>
    requires std::is_trivially_copyable_v<T>
T from_bytes(std::span<const std::byte> bytes) {
    T result;
    std::memcpy(&result, bytes.data(), sizeof(T));
    return result;
}

// A simple packet structure
struct Packet {
    uint16_t type;
    uint32_t payload_size;
    uint16_t checksum;
};

int main() {
    // Serialize a Packet to bytes
    Packet pkt{.type = 1, .payload_size = 1024, .checksum = 0xABCD};
    auto bytes = as_bytes_view(pkt);

    std::cout << "Packet as bytes (" << bytes.size() << " bytes): ";
    for (std::byte b : bytes) {
        printf("%02X ", std::to_integer<unsigned>(b));
    }
    std::cout << "\n";

    // Deserialize back
    Packet restored = from_bytes<Packet>(bytes);
    std::cout << "type=" << restored.type
              << " payload_size=" << restored.payload_size
              << " checksum=0x" << std::hex << restored.checksum << "\n";
    // Output: type=1 payload_size=1024 checksum=0xabcd

    // Using std::as_bytes and std::as_writable_bytes (C++20)
    std::vector<int> data{1, 2, 3, 4};
    std::span<int> data_span(data);

    // Get a read-only byte view of the vector
    std::span<const std::byte> byte_view = std::as_bytes(data_span);
    std::cout << std::dec << "ints: " << data.size() * sizeof(int)
              << " bytes total\n";
    // Output: ints: 16 bytes total

    // Get a writable byte view
    std::span<std::byte> writable = std::as_writable_bytes(data_span);
    writable[0] = std::byte{0xFF};  // modify first byte of data[0]
    std::cout << "data[0] after byte edit: " << data[0] << "\n";
    // Output: data[0] after byte edit: 255 (on little-endian)
}
```

**Why `std::span<std::byte>` is ideal for serialization:**

- Type-safe: you cannot accidentally do arithmetic on the bytes
- Zero-copy: `std::as_bytes` creates a view, no data is copied
- Composable: spans can be subspanned for header/payload separation
- Standard: replaces `void*` + `size_t` pairs with a single typed object

---

## Notes

- `std::byte` is in `<cstddef>`, **not** `<cstdint>`.
- `std::byte` has the same size and alignment as `unsigned char` (1 byte).
- Like `unsigned char` and `char`, `std::byte` is allowed to alias any object's memory (no strict aliasing violation).
- `std::to_integer<T>(b)` is the only way to get the numeric value - there is no implicit conversion.
- `std::as_bytes(span)` and `std::as_writable_bytes(span)` (C++20, in `<span>`) provide the cleanest way to get byte views.
