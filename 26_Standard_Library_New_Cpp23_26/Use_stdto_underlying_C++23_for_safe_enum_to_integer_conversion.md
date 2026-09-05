# Use `std::to_underlying` (C++23) for Safe Enum-to-Integer Conversion

**Category:** Standard Library New Cpp23 26  
**Standard:** C++23  
**Reference:** [cppreference — std::to_underlying](https://en.cppreference.com/w/cpp/utility/to_underlying)  

---

## Topic Overview

`std::to_underlying`, defined in `<utility>`, converts a scoped or unscoped enumeration value to its underlying integral type. Before C++23, the idiomatic way was `static_cast<std::underlying_type_t<E>>(e)` - verbose, error-prone if the target type is manually specified, and invisible to grep-based code audits. `std::to_underlying` replaces this pattern with a single, self-documenting call.

Before C++23, extracting the numeric value of an enum required either an explicit `static_cast` to the underlying type or a helper function that developers had to write themselves. Both approaches are error-prone: `static_cast` lets you cast to the *wrong* integer type silently, and hand-rolled helpers are inconsistent across codebases. `std::to_underlying` eliminates these problems by always deducing the correct underlying type.

| Approach | Type-safe? | Deduces underlying type? | Readable? |
| -------- | ---------- | ------------------------ | --------- |
| `static_cast<int>(e)` | No - allows wrong target type | No | Moderate |
| `static_cast<std::underlying_type_t<E>>(e)` | Yes | Yes | Poor - verbose |
| `std::to_underlying(e)` | Yes | Yes | Excellent |

The function is `constexpr` and `noexcept`, making it usable in compile-time contexts, template metaprogramming, serialization routines, logging, and hash functions. It works identically for scoped (`enum class`) and unscoped (`enum`) enumerations.

### Implementation (simplified)

The implementation is deliberately trivial - the value of this function is the name, not the mechanics:

```cpp
template <class Enum>
constexpr std::underlying_type_t<Enum> to_underlying(Enum e) noexcept {
    return static_cast<std::underlying_type_t<Enum>>(e);
}
```

The simplicity is the point: the standard provides a well-named, discoverable vocabulary function so that every codebase converges on one idiom instead of each team inventing their own `enum_cast` or `underlying_cast` helper.

---

## Self-Assessment

### Q1: How does `std::to_underlying` compare to `static_cast` in a serialization context, and what bugs does it prevent

The subtle danger with `static_cast<int>(e)` is that it hardcodes the destination type. If the enum's underlying type is `uint8_t` and you cast to `int`, the compiler won't warn you - it's a legal widening conversion. But you've now written 4 bytes to the wire instead of 1, silently breaking your protocol. `std::to_underlying` can't make that mistake because it always uses the exact declared underlying type:

```cpp
#include <cstdint>
#include <utility>
#include <iostream>
#include <vector>
#include <bit>

enum class Permission : std::uint8_t {
    Read    = 1,
    Write   = 2,
    Execute = 4,
    Admin   = 128
};

// BUG-PRONE: static_cast to the wrong type silently widens
void serialize_bad(Permission p, std::vector<std::byte>& buf) {
    // Developer guessed int (4 bytes) instead of uint8_t (1 byte).
    // Compiles fine, writes 4 bytes instead of 1.
    auto val = static_cast<int>(p);
    auto* raw = reinterpret_cast<const std::byte*>(&val);
    buf.insert(buf.end(), raw, raw + sizeof(val));
}

// CORRECT: std::to_underlying always yields uint8_t here
void serialize_good(Permission p, std::vector<std::byte>& buf) {
    auto val = std::to_underlying(p);  // deduced as uint8_t
    static_assert(std::same_as<decltype(val), std::uint8_t>);
    buf.push_back(static_cast<std::byte>(val));
}

int main() {
    std::vector<std::byte> buf;

    serialize_good(Permission::Admin, buf);
    std::cout << "Serialized size (good): " << buf.size() << " byte(s)\n";
    // Output: 1

    buf.clear();
    serialize_bad(Permission::Admin, buf);
    std::cout << "Serialized size (bad):  " << buf.size() << " byte(s)\n";
    // Output: 4 - silent data bloat and protocol mismatch
}
```

The key insight: `std::to_underlying` eliminates an entire class of serialization bugs where the developer casts to a mismatched integer width. When the underlying type changes (e.g., from `uint8_t` to `uint16_t`), every `std::to_underlying` call adapts automatically; every `static_cast<uint8_t>` site silently truncates.

---

### Q2: How do you use `std::to_underlying` inside a `constexpr` bitmask flag system

Bitmask enums are the most common place you'll reach for `to_underlying`. To implement `operator|` and `operator&` on an `enum class`, you need to extract the integer, do the bitwise operation, and cast back. `to_underlying` makes the intent clear at each step:

```cpp
#include <cstdint>
#include <utility>
#include <type_traits>
#include <iostream>
#include <format>

enum class Style : std::uint16_t {
    None      = 0,
    Bold      = 1 << 0,
    Italic    = 1 << 1,
    Underline = 1 << 2,
    Strike    = 1 << 3,
    Code      = 1 << 4,
};

// Overload bitwise operators using to_underlying for clarity
constexpr Style operator|(Style a, Style b) noexcept {
    return static_cast<Style>(
        std::to_underlying(a) | std::to_underlying(b));
}

constexpr Style operator&(Style a, Style b) noexcept {
    return static_cast<Style>(
        std::to_underlying(a) & std::to_underlying(b));
}

constexpr bool has_flag(Style set, Style flag) noexcept {
    return (set & flag) != Style::None;
}

// Compile-time validation
static_assert(std::to_underlying(Style::Bold) == 1);
static_assert(std::to_underlying(Style::Code) == 16);
static_assert(has_flag(Style::Bold | Style::Italic, Style::Italic));

// Logging / formatting helper
void print_styles(Style s) {
    auto raw = std::to_underlying(s);
    std::cout << std::format("Style mask: 0x{:04X} (decimal {})\n", raw, raw);
    if (has_flag(s, Style::Bold))      std::cout << "  + Bold\n";
    if (has_flag(s, Style::Italic))    std::cout << "  + Italic\n";
    if (has_flag(s, Style::Underline)) std::cout << "  + Underline\n";
    if (has_flag(s, Style::Strike))    std::cout << "  + Strike\n";
    if (has_flag(s, Style::Code))      std::cout << "  + Code\n";
}

int main() {
    constexpr auto heading = Style::Bold | Style::Underline;
    print_styles(heading);
    // Style mask: 0x0005 (decimal 5)
    //   + Bold
    //   + Underline
}
```

Notice that the `static_assert` lines at namespace scope verify the bit values at compile time - zero runtime cost, and they document the expected layout of the bit field right next to the enum definition.

---

### Q3: How do you use `std::to_underlying` as a hash or map key for enum-indexed data structures

Two common patterns here: using an enum value as an array index (with a `COUNT` sentinel), and using an enum as a key in an `unordered_map` (which needs a hash). `to_underlying` makes both clean:

```cpp
#include <utility>
#include <unordered_map>
#include <string>
#include <iostream>
#include <format>
#include <array>
#include <cstdint>

enum class LogLevel : std::uint8_t {
    Trace = 0,
    Debug = 1,
    Info  = 2,
    Warn  = 3,
    Error = 4,
    Fatal = 5,
    COUNT // sentinel for array sizing
};

// 1. Compile-time array indexed by enum via to_underlying
constexpr std::array<const char*, std::to_underlying(LogLevel::COUNT)>
    level_names = {"TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"};

constexpr const char* to_string(LogLevel lv) noexcept {
    auto idx = std::to_underlying(lv);
    if (idx < level_names.size()) return level_names[idx];
    return "UNKNOWN";
}

// 2. Hash functor using to_underlying - generic for any enum
struct EnumHash {
    template <typename E>
        requires std::is_enum_v<E>
    constexpr std::size_t operator()(E e) const noexcept {
        return std::hash<std::underlying_type_t<E>>{}(std::to_underlying(e));
    }
};

int main() {
    // Compile-time lookup
    static_assert(to_string(LogLevel::Warn)[0] == 'W');

    // Runtime: enum-keyed map with generic hash
    std::unordered_map<LogLevel, std::string, EnumHash> thresholds;
    thresholds[LogLevel::Error] = "page oncall";
    thresholds[LogLevel::Fatal] = "page oncall + manager";

    for (auto& [level, action] : thresholds) {
        std::cout << std::format("[{}] -> {}\n", to_string(level), action);
    }

    // Direct use in switch (showing exhaustive pattern)
    LogLevel current = LogLevel::Info;
    switch (std::to_underlying(current)) {
        case 0: case 1: std::cout << "verbose\n"; break;
        case 2: case 3: std::cout << "standard\n"; break;
        case 4: case 5: std::cout << "critical\n"; break;
    }
}
```

The `COUNT` sentinel pattern is worth understanding: by placing a `COUNT` enumerator at the end, `std::to_underlying(LogLevel::COUNT)` gives you the number of enumerators as a compile-time constant. That means the `level_names` array is automatically the right size - add a new log level, update `level_names`, and the array size stays in sync without any manual counting.

---

## Notes

- **Header:** `<utility>` - a common source of missing-include errors since you might expect it to live in `<type_traits>`.
- **`constexpr` + `noexcept`:** usable in every evaluation context including `static_assert`, template arguments, and `consteval` functions.
- **Works with unscoped enums too:** although less commonly needed since unscoped enums implicitly convert to integers, `to_underlying` still guarantees the *exact* underlying type rather than a potentially widened `int`.
- **Migration tip:** search your codebase for `static_cast<.*underlying_type` - every hit is a candidate for replacement with `std::to_underlying`.
- **Compiler support (as of 2025):** GCC 13+, Clang 16+, MSVC 19.34+ (VS 2022 17.4+).
- **Pair with `std::format`:** `std::to_underlying` feeds directly into format specifiers - `std::format("{}", std::to_underlying(e))` - useful for logging and diagnostics.
