# Use enum class bitmasks safely with a library helper

**Category:** Best Practices and Idioms  
**Standard:** C++11  
**Reference:** <https://github.com/Neargye/magic_enum>  

---

## Topic Overview

`enum class` is strictly typed, which is exactly what you want - but that strictness means the compiler refuses to let you do bitwise operations on enum values, since it doesn't know whether your enum is meant to be used as a bitmask. The fix is to provide those operators explicitly, either through a template helper or a small macro that opts a specific enum in.

### The Problem

```cpp
enum class Flags : uint32_t { Read = 1, Write = 2, Exec = 4 };
// Flags f = Flags::Read | Flags::Write;  // ERROR: no operator|
```

Without an `operator|`, combining flags requires ugly casts everywhere, which defeats the purpose of using an enum in the first place.

### The Solution: Enable bitwise ops for specific enums

```cpp
// Option 1: Macro
// Option 2: Template + trait
// Option 3: Operator overloads per enum
```

All three options produce the same result at runtime. The template-plus-trait approach (shown in Q1) is the most reusable because you write the operators once and just opt each enum in with a single line.

---

## Self-Assessment

### Q1: Implement `operator|`, `operator&`, `operator~` for an enum class

The pattern below uses a trait struct `EnableBitmask` that defaults to `false_type`. You specialize it to `true_type` for any enum that should support bitwise ops, and a set of constrained template operators becomes available for that enum and only that enum.

```cpp
#include <cstdint>
#include <iostream>
#include <type_traits>

// Generic template approach: enable bitwise ops for any enum class
template<typename E>
struct EnableBitmask : std::false_type {};

// Enable for specific enum:
#define ENABLE_BITMASK(E) \
    template<> struct EnableBitmask<E> : std::true_type {}

template<typename E>
requires EnableBitmask<E>::value
constexpr E operator|(E lhs, E rhs) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(static_cast<U>(lhs) | static_cast<U>(rhs));
}

template<typename E>
requires EnableBitmask<E>::value
constexpr E operator&(E lhs, E rhs) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(static_cast<U>(lhs) & static_cast<U>(rhs));
}

template<typename E>
requires EnableBitmask<E>::value
constexpr E operator~(E e) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(~static_cast<U>(e));
}

template<typename E>
requires EnableBitmask<E>::value
constexpr E& operator|=(E& lhs, E rhs) { return lhs = lhs | rhs; }

template<typename E>
requires EnableBitmask<E>::value
constexpr E& operator&=(E& lhs, E rhs) { return lhs = lhs & rhs; }

// Define the enum:
enum class Permissions : uint8_t {
    None    = 0,
    Read    = 1 << 0,
    Write   = 1 << 1,
    Execute = 1 << 2,
    All     = Read | Write | Execute  // works in the enum definition
};
ENABLE_BITMASK(Permissions);

bool has_flag(Permissions perms, Permissions flag) {
    return (perms & flag) != Permissions::None;
}

int main() {
    auto perms = Permissions::Read | Permissions::Write;

    std::cout << std::boolalpha;
    std::cout << "Has Read:    " << has_flag(perms, Permissions::Read) << '\n';
    std::cout << "Has Write:   " << has_flag(perms, Permissions::Write) << '\n';
    std::cout << "Has Execute: " << has_flag(perms, Permissions::Execute) << '\n';

    perms |= Permissions::Execute;
    std::cout << "After adding Execute: " << has_flag(perms, Permissions::Execute) << '\n';
}
// Expected output:
// Has Read:    true
// Has Write:   true
// Has Execute: false
// After adding Execute: true
```

Because the operators require `EnableBitmask<E>::value` to be true, they simply don't exist for any other enum. You get strong type safety - a `Permissions` value can never accidentally be ORed with a `Color` value - while still writing the bitmask operations naturally.

### Q2: Show why raw enum bitmasks allow mixing unrelated types while enum class prevents it

Here's why the old-style `enum` is risky for bitmask use: both your flags enum and an unrelated enum share `int` as their underlying type, so the compiler sees them as the same kind of thing. Mixing them is silently valid, even when the result is nonsensical.

```cpp
#include <cstdint>
#include <iostream>

// BAD: unscoped enums - can mix unrelated types!
enum OldPermission { OldRead = 1, OldWrite = 2 };
enum OldColor { OldRed = 1, OldGreen = 2 };

void demonstrate_raw_enum_danger() {
    int mixed = OldRead | OldRed;  // COMPILES! Nonsensical.
    if (OldWrite == OldGreen) {    // COMPILES! Both are 2.
        std::cout << "BAD: Write == Green? Makes no sense!\n";
    }
}

// GOOD: enum class - type-safe
enum class Permission : uint8_t { Read = 1, Write = 2 };
enum class Color : uint8_t { Red = 1, Green = 2 };

void demonstrate_enum_class_safety() {
    // auto mixed = Permission::Read | Color::Red;  // ERROR: different types
    // if (Permission::Write == Color::Green) {}     // ERROR: can't compare
    std::cout << "GOOD: compiler prevents mixing Permission and Color\n";
}

int main() {
    demonstrate_raw_enum_danger();
    demonstrate_enum_class_safety();
}
// Expected output:
// BAD: Write == Green? Makes no sense!
// GOOD: compiler prevents mixing Permission and Color
```

The `enum class` version catches this at compile time. That's the fundamental trade-off: you give up implicit-to-int conversions and gain protection against this whole category of silent mixup bug.

### Q3: Custom enum reflection solution

When you need to iterate over enum values or convert them to strings at runtime, a simple parallel array keyed on the underlying integer works well for small enums. The `COUNT` sentinel trick is a lightweight way to get the array size without any macros.

```cpp
#include <array>
#include <iostream>
#include <string_view>

enum class Color { Red, Green, Blue, Yellow, COUNT };

// Simple reflection: parallel array
constexpr std::array<std::string_view, static_cast<int>(Color::COUNT)> color_names = {
    "Red", "Green", "Blue", "Yellow"
};

constexpr std::string_view to_string(Color c) {
    auto idx = static_cast<int>(c);
    if (idx >= 0 && idx < static_cast<int>(Color::COUNT))
        return color_names[idx];
    return "Unknown";
}

// Iterate all values:
template<typename Func>
void for_each_color(Func&& f) {
    for (int i = 0; i < static_cast<int>(Color::COUNT); ++i)
        f(static_cast<Color>(i));
}

int main() {
    std::cout << "Color: " << to_string(Color::Blue) << '\n';

    std::cout << "All colors: ";
    for_each_color([](Color c) {
        std::cout << to_string(c) << ' ';
    });
    std::cout << '\n';
}
// Expected output:
// Color: Blue
// All colors: Red Green Blue Yellow
//
// For production use, consider magic_enum library:
// auto name = magic_enum::enum_name(Color::Blue);  // "Blue"
// auto val  = magic_enum::enum_cast<Color>("Blue"); // Color::Blue
```

The manual approach works fine until the enum gets large or you have many enums to reflect. At that point the `magic_enum` library - referenced in the header - handles it with zero boilerplate using compiler-specific introspection tricks.

---

## Notes

- The `ENABLE_BITMASK` macro approach is used by many codebases (Chromium, Qt).
- `magic_enum` provides compile-time reflection over enum values using compiler-specific tricks.
- C++23's `std::to_underlying()` replaces `static_cast<underlying_type_t<E>>(e)` - use it to reduce the cast noise in the operator implementations.
- Always use `enum class` over `enum` - the type safety is worth the extra operators.
