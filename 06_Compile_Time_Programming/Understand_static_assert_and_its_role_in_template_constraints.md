# Understand `static_assert` and Its Role in Template Constraints

**Category:** Compile Time Programming  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/static_assert>  

---

## Topic Overview

### What Is `static_assert`

`static_assert` is a compile-time assertion that causes a **hard compile error** with a custom message if its condition evaluates to `false`. Unlike runtime `assert()`, it runs entirely during compilation - there's no overhead at all in the final binary, and no way for the condition to be wrong at a customer's site.

```cpp
static_assert(sizeof(int) == 4, "int must be 4 bytes on this platform");
static_assert(std::is_trivially_copyable_v<T>, "T must be trivially copyable");
```

The message string is what makes `static_assert` particularly valuable: you can explain exactly what the programmer needs to fix, not just tell them something failed.

### `static_assert` in Templates

In templates, `static_assert` serves as a **hard constraint** - if the condition fails when the template is instantiated with a particular type, compilation stops with a clear diagnostic. This is different from concepts or SFINAE, which silently remove overloads from consideration. With `static_assert`, there's no fallback - you're saying "this code cannot possibly work with T, period."

```cpp
template <typename T>
class Buffer {
    static_assert(std::is_trivially_copyable_v<T>,
                  "Buffer<T> requires T to be trivially copyable for memcpy safety");
    // ...
};
```

### `static_assert` vs SFINAE vs Concepts

The reason you have three tools here is that they do genuinely different things. The table captures when each one is appropriate:

| Feature | `static_assert` | SFINAE / `enable_if` | Concepts (C++20) |
| --- | --- | --- | --- |
| Effect | **Hard error** - stops compilation | **Soft failure** - removes overload | **Soft failure** - removes overload |
| Message quality | Custom, human-readable | Cryptic substitution failure | Clear constraint diagnostic |
| Overload resolution | No - it's a hard stop | Yes - enables/disables overloads | Yes - enables/disables overloads |
| Best for | Catching programming errors early | Conditional overload selection | Conditional overload selection |

**When to use `static_assert`:** When there is NO valid alternative - the code must fail if the condition is not met. For overload selection, use concepts instead.

---

## Self-Assessment

### Q1: Write a class template that `static_assert`s that `T` is trivially copyable

A `FixedBuffer` that's meant to use `memcpy` internally is a good candidate for a `static_assert` - if someone instantiates it with `std::string`, you want a clear error pointing to the root cause, not a mysterious `memcpy` memory corruption at runtime.

```cpp
#include <iostream>
#include <type_traits>
#include <cstring>
#include <string>

// === Buffer that requires trivially copyable types ===
// This ensures memcpy/memmove is safe to use on T
template <typename T, std::size_t N>
class FixedBuffer {
    static_assert(std::is_trivially_copyable_v<T>,
        "FixedBuffer requires T to be trivially copyable for safe binary operations");
    static_assert(N > 0, "Buffer size must be greater than 0");

    T data_[N]{};
    std::size_t size_ = 0;

public:
    void push(const T& val) {
        static_assert(sizeof(T) <= 64,
            "T is too large for stack-allocated FixedBuffer");
        if (size_ < N) {
            data_[size_++] = val;
        }
    }

    T& operator[](std::size_t i) { return data_[i]; }
    const T& operator[](std::size_t i) const { return data_[i]; }
    std::size_t size() const { return size_; }

    // Safe because T is trivially copyable
    void copy_to(T* dst) const {
        std::memcpy(dst, data_, size_ * sizeof(T));
    }
};

// === More static_assert use cases ===
template <typename T>
class Serializer {
    static_assert(!std::is_pointer_v<T>,
        "Serializer<T*> is forbidden - serialize the pointed-to object instead");
    static_assert(std::is_standard_layout_v<T>,
        "Serializer requires standard layout types for binary serialization");
};

struct Point { int x, y; };                       // OK: trivially copyable, standard layout
// struct Complex { std::string name; int val; };  // Would fail: string is not trivially copyable

int main() {
    // Works: int is trivially copyable
    FixedBuffer<int, 10> ibuf;
    ibuf.push(1);
    ibuf.push(2);
    ibuf.push(3);
    std::cout << "Buffer: ";
    for (std::size_t i = 0; i < ibuf.size(); ++i)
        std::cout << ibuf[i] << " ";
    std::cout << "\n";

    // Works: Point is trivially copyable
    FixedBuffer<Point, 5> pbuf;
    pbuf.push({10, 20});
    pbuf.push({30, 40});
    std::cout << "Points: (" << pbuf[0].x << "," << pbuf[0].y << ") "
              << "(" << pbuf[1].x << "," << pbuf[1].y << ")\n";

    // Copy to raw array
    int raw[3];
    ibuf.copy_to(raw);
    std::cout << "Copied: " << raw[0] << " " << raw[1] << " " << raw[2] << "\n";

    // Serializer checks
    Serializer<Point> s;  // OK
    // Serializer<int*> s2;               // ERROR: pointer forbidden
    // Serializer<std::string> s3;        // ERROR: not standard layout

    // These would fail:
    // FixedBuffer<std::string, 10> sbuf;  // ERROR: string is not trivially copyable
    // FixedBuffer<int, 0> zbuf;           // ERROR: size must be > 0

    (void)s;
    return 0;
}
```

**Expected output:**

```text
Buffer: 1 2 3
Points: (10,20) (30,40)
Copied: 1 2 3
```

### Q2: Show how `static_assert` with a descriptive message provides better diagnostics than SFINAE

The real value of `static_assert` over SFINAE is that you can tell the user what to do next. A SFINAE failure says "no matching function" - a `static_assert` failure can say "you passed a string, but this function only works with arithmetic types - use split() instead." That's actionable.

```cpp
#include <iostream>
#include <type_traits>
#include <concepts>

// ============================================================
// SFINAE approach: cryptic error when no overload matches
// ============================================================
template <typename T>
std::enable_if_t<std::is_arithmetic_v<T>, T>
safe_divide_sfinae(T a, T b) {
    return a / b;
}

// When called with std::string:
// Error: "no matching function for call to 'safe_divide_sfinae'"
// note: candidate template ignored: requirement
//       'is_arithmetic_v<basic_string<char>>' was not satisfied
//   ^^^ What should I do? Why is arithmetic required?

// ============================================================
// static_assert approach: clear, actionable error message
// ============================================================
template <typename T>
T safe_divide_assert(T a, T b) {
    static_assert(std::is_arithmetic_v<T>,
        "safe_divide requires an arithmetic type (int, float, double). "
        "For string division, use a split function instead.");

    static_assert(!std::is_same_v<T, bool>,
        "safe_divide does not support bool: use int instead.");

    if constexpr (std::is_integral_v<T>) {
        static_assert(std::is_signed_v<T>,
            "Use signed integers to avoid unexpected unsigned wraparound in division.");
    }

    return a / b;
}

// ============================================================
// Concepts approach (C++20): best of both worlds
// ============================================================
template <typename T>
concept Dividable = std::is_arithmetic_v<T> && !std::is_same_v<T, bool>;

template <Dividable T>
T safe_divide_concepts(T a, T b) {
    return a / b;
}

int main() {
    std::cout << "SFINAE:   " << safe_divide_sfinae(10, 3) << "\n";
    std::cout << "Assert:   " << safe_divide_assert(10, 3) << "\n";
    std::cout << "Concepts: " << safe_divide_concepts(10, 3) << "\n";

    std::cout << "\n=== Error Message Comparison ===\n";

    std::cout << "\nSFINAE with string:\n";
    std::cout << "  'no matching function for safe_divide_sfinae(string, string)'\n";
    std::cout << "  'requirement is_arithmetic_v<string> not satisfied'\n";
    std::cout << "  -> No guidance on what to do instead\n";

    std::cout << "\nstatic_assert with string:\n";
    std::cout << "  'safe_divide requires an arithmetic type (int, float, double).'\n";
    std::cout << "  'For string division, use a split function instead.'\n";
    std::cout << "  -> Tells you WHAT went wrong AND what to do\n";

    std::cout << "\nConcepts with string:\n";
    std::cout << "  'constraint not satisfied: Dividable<string>'\n";
    std::cout << "  'is_arithmetic_v<string> is false'\n";
    std::cout << "  -> Clear cause, but no fix suggestion\n";

    std::cout << "\n=== When to Use Each ===\n";
    std::cout << "static_assert: Hard errors with actionable messages\n";
    std::cout << "SFINAE/concepts: When overload fallback is desired\n";
    std::cout << "Best practice: concepts for overloads + static_assert for final checks\n";

    return 0;
}
```

**Expected output:**

```text
SFINAE:   3
Assert:   3
Concepts: 3

=== Error Message Comparison ===

SFINAE with string:
  'no matching function for safe_divide_sfinae(string, string)'
  'requirement is_arithmetic_v<string> not satisfied'
  -> No guidance on what to do instead

static_assert with string:
  'safe_divide requires an arithmetic type (int, float, double).'
  'For string division, use a split function instead.'
  -> Tells you WHAT went wrong AND what to do

Concepts with string:
  'constraint not satisfied: Dividable<string>'
  'is_arithmetic_v<string> is false'
  -> Clear cause, but no fix suggestion

=== When to Use Each ===
static_assert: Hard errors with actionable messages
SFINAE/concepts: When overload fallback is desired
Best practice: concepts for overloads + static_assert for final checks
```

### Q3: Demonstrate a `static_assert` that checks the size of a serializable struct against a protocol requirement

This is a textbook use of `static_assert` in systems programming: you have an external contract (the protocol spec says MessageHeader is 16 bytes) and you want the compiler to enforce it. A mismatched struct size will corrupt every packet you send - you want to know at compile time, not after a mysterious network bug in production.

```cpp
#include <iostream>
#include <type_traits>
#include <cstdint>
#include <cstring>

// === Network protocol: fixed-size message format ===
// Protocol spec says each message header is exactly 16 bytes
// and each data packet is exactly 64 bytes

#pragma pack(push, 1)  // Ensure no padding

struct MessageHeader {
    uint32_t magic;       // 4 bytes
    uint16_t version;     // 2 bytes
    uint16_t type;        // 2 bytes
    uint32_t length;      // 4 bytes
    uint32_t checksum;    // 4 bytes
};                        // Total: 16 bytes

struct DataPacket {
    MessageHeader header;      // 16 bytes
    uint8_t  payload[44];      // 44 bytes
    uint32_t sequence_number;  // 4 bytes
};                             // Total: 64 bytes

#pragma pack(pop)

// === Protocol constraints - compile-time enforcement ===
static_assert(sizeof(MessageHeader) == 16,
    "MessageHeader must be exactly 16 bytes per protocol spec v2.1");

static_assert(sizeof(DataPacket) == 64,
    "DataPacket must be exactly 64 bytes - network MTU alignment requirement");

static_assert(std::is_trivially_copyable_v<MessageHeader>,
    "MessageHeader must be trivially copyable for safe memcpy to network buffer");

static_assert(std::is_trivially_copyable_v<DataPacket>,
    "DataPacket must be trivially copyable for safe memcpy to network buffer");

static_assert(std::is_standard_layout_v<MessageHeader>,
    "MessageHeader must be standard layout for C interop and wire format compatibility");

static_assert(offsetof(MessageHeader, magic) == 0,
    "magic field must be at offset 0 for protocol handshake");

static_assert(offsetof(MessageHeader, length) == 8,
    "length field must be at offset 8 per protocol spec");

// === Template helper for protocol struct validation ===
template <typename T, std::size_t ExpectedSize>
struct ProtocolStruct {
    static_assert(sizeof(T) == ExpectedSize,
        "Struct size does not match protocol specification");
    static_assert(std::is_trivially_copyable_v<T>,
        "Protocol structs must be trivially copyable");
    static_assert(std::is_standard_layout_v<T>,
        "Protocol structs must be standard layout");
    static_assert(alignof(T) <= 4,
        "Protocol structs must have alignment <= 4 for wire compatibility");
};

// Validate at compile time
template struct ProtocolStruct<MessageHeader, 16>;
template struct ProtocolStruct<DataPacket, 64>;

// === Serialization (safe because of static_asserts) ===
void serialize(const MessageHeader& hdr, uint8_t* buffer) {
    std::memcpy(buffer, &hdr, sizeof(hdr));
}

MessageHeader deserialize(const uint8_t* buffer) {
    MessageHeader hdr;
    std::memcpy(&hdr, buffer, sizeof(hdr));
    return hdr;
}

int main() {
    std::cout << "=== Protocol Struct Sizes ===\n";
    std::cout << "MessageHeader: " << sizeof(MessageHeader) << " bytes\n";
    std::cout << "DataPacket:    " << sizeof(DataPacket) << " bytes\n";

    // Create and serialize a header
    MessageHeader hdr{0xDEADBEEF, 2, 1, 100, 0};
    uint8_t buffer[16];
    serialize(hdr, buffer);

    // Deserialize and verify
    auto restored = deserialize(buffer);
    std::cout << "\nSerialized and deserialized:\n";
    std::cout << "  magic:    0x" << std::hex << restored.magic << std::dec << "\n";
    std::cout << "  version:  " << restored.version << "\n";
    std::cout << "  type:     " << restored.type << "\n";
    std::cout << "  length:   " << restored.length << "\n";

    std::cout << "\n=== Field Offsets ===\n";
    std::cout << "  magic:    offset " << offsetof(MessageHeader, magic) << "\n";
    std::cout << "  version:  offset " << offsetof(MessageHeader, version) << "\n";
    std::cout << "  type:     offset " << offsetof(MessageHeader, type) << "\n";
    std::cout << "  length:   offset " << offsetof(MessageHeader, length) << "\n";
    std::cout << "  checksum: offset " << offsetof(MessageHeader, checksum) << "\n";

    return 0;
}
```

**Expected output:**

```text
=== Protocol Struct Sizes ===
MessageHeader: 16 bytes
DataPacket:    64 bytes

Serialized and deserialized:
  magic:    0xdeadbeef
  version:  2
  type:     1
  length:   100

=== Field Offsets ===
  magic:    offset 0
  version:  offset 4
  type:     offset 6
  length:   offset 8
  checksum: offset 12
```

---

## Notes

- `static_assert(condition)` without a message is valid since C++17.
- `static_assert` causes a **hard compilation error** - use it when there is no valid fallback.
- For overload selection, use concepts or SFINAE instead (they allow graceful fallback).
- Combine `static_assert` with `sizeof`, `alignof`, `offsetof`, and type traits for protocol/ABI checking.
- In templates, `static_assert` can depend on template parameters - it's evaluated when the template is instantiated.
- Common pattern: `static_assert` in class body for invariant enforcement + concepts for API constraints.
