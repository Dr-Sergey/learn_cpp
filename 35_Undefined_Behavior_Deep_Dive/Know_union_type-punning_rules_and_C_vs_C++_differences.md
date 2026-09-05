# Know Union Type-Punning Rules and C vs C++ Differences

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17, C++20, C++23, C++11  
**Reference:** [cppreference - Union declaration](https://en.cppreference.com/w/cpp/language/union)  

---

## Topic Overview

Unions in C and C++ share the same syntax but have **fundamentally different type-punning rules**. In C (C99+), reading from a union member that was not the most recently written is explicitly defined behavior (with some constraints). In C++, this is **undefined behavior** - only the "active member" may be read. This difference is the single most common source of cross-language UB in codebases that mix C and C++ headers.

The reason this trips people up is that code using union type-punning very often works correctly in practice - MSVC is permissive about it, GCC and Clang allow it in C mode, and even in C++ mode the compiler may happen to generate the code you expected at lower optimization levels. The bug surfaces when you upgrade the compiler, enable higher optimizations, or move to a stricter build environment.

| Feature | C99/C11/C23 | C++17/C++20/C++23 |
| --- | --- | --- |
| Reading inactive member | **Defined** (reinterprets bytes) | **UB** |
| Active member tracking | Implicit (last written) | Enforced by standard |
| Common initial sequence | Readable through any member | Readable only via union (C++17) or only specific cases |
| `std::bit_cast` available | No (use `memcpy`) | Yes (C++20) |
| `constexpr` union access | N/A | Active member only |

The **common initial sequence** rule allows reading the common prefix of two struct members in a union - but in C++17/20, this is legal only when inspected through the union itself, not through a pointer to one of the structs. The rules were further clarified in C++20 defect reports.

The diagram below shows what happens at the memory level when you write one member and read another. The bytes are the same in both C and C++, but the language rules differ on whether accessing them is defined:

```cpp
Union memory layout:

    ┌──────────────────────────┐
    │  union U { int i;        │
    │            float f; };   │
    │                          │
    │  Memory: [4 bytes]       │
    │                          │
    │  u.i = 42;  -> active: i │
    │                          │
    │  C:   u.f -> reads same  │
    │            bytes as float │
    │            (defined)      │
    │                          │
    │  C++: u.f -> UB          │
    │            (i is active)  │
    └──────────────────────────┘
```

In modern C++, the correct alternatives are `std::bit_cast` (C++20), `std::memcpy`, or placement new with `std::start_lifetime_as` (C++23). Using unions for type punning in C++ should be considered a legacy practice.

---

## Self-Assessment

### Q1: Demonstrate the C vs C++ union type-punning difference

This example shows the same operation in three ways: the C-style union version that is UB in C++, and two C++-correct alternatives. All three produce the same output in practice - the difference is entirely in whether the standard permits them:

```cpp
#include <bit>
#include <cstdint>
#include <cstdio>
#include <cstring>

union FloatBits {
    float    f;
    uint32_t u;
};

// In C99+, this is DEFINED behavior.
// In C++, this is UNDEFINED behavior.
uint32_t float_to_bits_union(float value) {
    FloatBits fb;
    fb.f = value;  // f is now the active member
    return fb.u;   // UB in C++: reading inactive member u
}

// Safe C++ approach 1: std::memcpy
uint32_t float_to_bits_memcpy(float value) {
    uint32_t bits;
    std::memcpy(&bits, &value, sizeof(bits));
    return bits;  // Well-defined in both C and C++
}

// Safe C++ approach 2: std::bit_cast (C++20)
uint32_t float_to_bits_bitcast(float value) {
    return std::bit_cast<uint32_t>(value);  // Well-defined, constexpr
}

// Compile-time type punning (C++20): only bit_cast works
consteval uint32_t float_bits_at_compile_time(float value) {
    return std::bit_cast<uint32_t>(value);
    // Note: memcpy is NOT constexpr, union punning is UB.
    // bit_cast is the only option for compile-time type punning.
}

int main() {
    float pi = 3.14159f;

    std::printf("Union (C++ UB): 0x%08X\n", float_to_bits_union(pi));
    std::printf("memcpy:         0x%08X\n", float_to_bits_memcpy(pi));
    std::printf("bit_cast:       0x%08X\n", float_to_bits_bitcast(pi));

    constexpr uint32_t ct = float_bits_at_compile_time(3.14159f);
    std::printf("compile-time:   0x%08X\n", ct);
}
```

Notice that `float_bits_at_compile_time` can only use `std::bit_cast`. Neither the union trick nor `memcpy` work in a `consteval` context, so `bit_cast` is not just cleaner - it is the only option when you need compile-time type punning.

**Answer:** `float_to_bits_union` relies on union type-punning, which is defined in C99+ but UB in C++. The `std::memcpy` and `std::bit_cast` alternatives are well-defined in C++. For compile-time punning, `std::bit_cast` is the only option.

---

### Q2: Explain the common initial sequence rule and its limitations

The common initial sequence rule is a nuanced corner of the standard that is easy to misapply. The key thing to understand is that in C++, accessing the common initial sequence is only guaranteed when you go through the union itself - not through a pointer to one of the struct members:

```cpp
#include <cstdio>
#include <cstdint>
#include <type_traits>

// Two structs sharing a "common initial sequence":
// the initial members have layout-compatible types in order.
struct Header {
    uint32_t tag;
    uint16_t version;
};

struct MessageA {
    uint32_t tag;       // common with Header
    uint16_t version;   // common with Header
    float    payload;
};

struct MessageB {
    uint32_t tag;       // common with Header
    uint16_t version;   // common with Header
    int      code;
};

union Message {
    Header   header;
    MessageA a;
    MessageB b;
};

// C++ rule (pre-C++20 / CWG 2567):
// Reading the common initial sequence through the union is allowed.
// Reading it through a POINTER to one member is NOT guaranteed.

void inspect_through_union(Message& msg) {
    // OK: reading common initial sequence via union member access
    uint32_t tag = msg.header.tag;
    uint16_t ver = msg.header.version;
    std::printf("Tag: %u, Version: %u\n", tag, ver);
}

void inspect_through_pointer(MessageA* a) {
    // QUESTIONABLE: accessing through pointer, not through union
    // In C: defined (common initial sequence rule applies)
    // In C++: technically the standard only guarantees this through
    //         the union itself, not through arbitrary pointers.
    // CWG 2567 (resolved in C++23) clarifies the rules.
    std::printf("Tag via pointer: %u\n", a->tag);
}

// Practical pattern: discriminated union (tagged union)
struct Event {
    enum class Type : uint8_t { MouseMove, KeyPress, WindowResize };

    union Data {
        struct { int x, y; }       mouse_move;
        struct { int keycode; }    key_press;
        struct { int w, h; }       window_resize;
    };

    Type type;
    Data data;

    // Safe access: check tag before reading
    void print() const {
        switch (type) {
            case Type::MouseMove:
                std::printf("Mouse: %d, %d\n", data.mouse_move.x,
                            data.mouse_move.y);
                break;
            case Type::KeyPress:
                std::printf("Key: %d\n", data.key_press.keycode);
                break;
            case Type::WindowResize:
                std::printf("Resize: %dx%d\n", data.window_resize.w,
                            data.window_resize.h);
                break;
        }
    }
};

int main() {
    Message msg;
    msg.a = {1, 2, 3.14f};
    inspect_through_union(msg);

    Event evt;
    evt.type = Event::Type::KeyPress;
    evt.data.key_press = {65};  // 'A'
    evt.print();
}
```

The `Event` struct is the practical takeaway: if you need a discriminated union in C++, the safe pattern is to store a type tag alongside the union and always check the tag before reading a member. In modern C++, `std::variant` does all of this for you automatically.

**Answer:** The common initial sequence rule allows reading shared leading members through the union, but C++ restricts this more than C. Through a pointer to one member (not through the union), the guarantees are weaker in C++ until CWG 2567 (C++23). In practice, use `std::variant` for tagged unions in modern C++.

---

### Q3: Show the safe modern C++ alternatives to union type-punning

Modern C++ offers four distinct tools depending on what you need. Here they all are in one place:

```cpp
#include <bit>
#include <cstdint>
#include <cstring>
#include <iostream>
#include <variant>

// Modern C++ offers several safe alternatives to union type-punning.

// 1. std::bit_cast (C++20) - for reinterpreting bit patterns
struct Color {
    uint8_t r, g, b, a;
};

uint32_t color_to_packed(Color c) {
    static_assert(sizeof(Color) == sizeof(uint32_t));
    static_assert(std::is_trivially_copyable_v<Color>);
    return std::bit_cast<uint32_t>(c);
}

Color packed_to_color(uint32_t packed) {
    return std::bit_cast<Color>(packed);
}

// 2. std::variant (C++17) - for discriminated unions
using Value = std::variant<int, double, std::string>;

void process_value(const Value& v) {
    std::visit([](const auto& val) {
        using T = std::decay_t<decltype(val)>;
        if constexpr (std::is_same_v<T, int>)
            std::cout << "int: " << val << "\n";
        else if constexpr (std::is_same_v<T, double>)
            std::cout << "double: " << val << "\n";
        else if constexpr (std::is_same_v<T, std::string>)
            std::cout << "string: " << val << "\n";
    }, v);
}

// 3. std::start_lifetime_as (C++23) - for reinterpreting byte buffers
// template <class T>
// T* start_lifetime_as(void* p) noexcept;
//
// Implicitly creates an object of type T at address p,
// making it legal to access the bytes as T.
//
// Example usage (C++23):
// alignas(Header) std::byte buffer[sizeof(Header)];
// read_from_network(buffer, sizeof(Header));
// auto* hdr = std::start_lifetime_as<Header>(buffer);

// 4. std::memcpy - universal safe type punning (all standards)
double int_bits_to_double(uint64_t bits) {
    static_assert(sizeof(double) == sizeof(uint64_t));
    double result;
    std::memcpy(&result, &bits, sizeof(result));
    return result;
}

// Comparison table:
/*
| Method                     | Standard | constexpr | Type check |
| --- | --- | --- | --- |
| Union type-punning         | UB (C++) | No        | None       |
| std::memcpy                | C++98    | No        | Manual     |
| std::bit_cast              | C++20    | Yes       | static_assert |
| std::start_lifetime_as     | C++23    | No        | Implicit   |
| std::variant               | C++17    | Partial   | Runtime    |
*/

int main() {
    // bit_cast for reinterpretation
    Color c{255, 128, 64, 255};
    uint32_t packed = color_to_packed(c);
    std::cout << "Packed: 0x" << std::hex << packed << "\n";

    Color unpacked = packed_to_color(packed);
    std::cout << "Unpacked: ("
              << +unpacked.r << ", " << +unpacked.g << ", "
              << +unpacked.b << ", " << +unpacked.a << ")\n";

    // variant for tagged unions
    Value v1 = 42;
    Value v2 = 3.14;
    Value v3 = std::string("hello");
    process_value(v1);
    process_value(v2);
    process_value(v3);
}
```

The `bit_cast` approach is the cleanest for reinterpreting types because it is `constexpr`, does a static size check, and requires both types to be trivially copyable. Use `std::variant` whenever the goal is a type-safe union that can hold one of several alternatives at a time.

**Answer:** Modern C++ provides `std::bit_cast` for bit-pattern reinterpretation, `std::variant` for type-safe discriminated unions, `std::start_lifetime_as` (C++23) for safe byte-buffer reinterpretation, and `std::memcpy` as the universal fallback. Union type-punning should be replaced with these alternatives in new C++ code.

---

## Notes

- **C allows union type-punning; C++ does not.** This is the most practically important difference between C and C++ union semantics.
- The common initial sequence rule is more restrictive in C++ than in C, especially when accessed through pointers rather than through the union directly.
- `std::variant` is the idiomatic C++ replacement for tagged unions, with full type safety and compile-time exhaustive visitation.
- `std::bit_cast` is the only way to do type-punning at compile time in C++ (constexpr contexts).
- When wrapping C APIs that use union type-punning, isolate the union access in C translation units compiled as C, or use `std::memcpy` in the C++ wrapper.
- MSVC is more permissive than GCC/Clang regarding union type-punning in practice, but the code is still technically UB.
