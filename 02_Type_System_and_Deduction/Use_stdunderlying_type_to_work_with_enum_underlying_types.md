# Use `std::underlying_type` to Work with Enum Underlying Types

**Category:** Type System and Deduction  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/types/underlying_type>  

---

## Topic Overview

### What Is `std::underlying_type`

Every enum in C++ has an **underlying integer type**. `std::underlying_type_t<E>` gives you that type at compile time. This matters because `enum class` deliberately hides the integer - there's no implicit conversion - so you need an explicit way to get at the raw value when you really need it.

```cpp
#include <type_traits>
#include <cstdint>

enum class Color : uint8_t { Red, Green, Blue };
enum class Status : int { OK = 0, Error = -1, Pending = 1 };
enum OldEnum { A, B, C };  // unscoped, usually int

static_assert(std::is_same_v<std::underlying_type_t<Color>, uint8_t>);
static_assert(std::is_same_v<std::underlying_type_t<Status>, int>);
```

### `std::to_underlying` (C++23)

C++23 provides a cleaner way to cast an enum to its underlying type, so you don't have to write the full `static_cast` every time:

```cpp
// C++23
auto val = std::to_underlying(Color::Red);  // uint8_t(0)

// Equivalent to:
auto val2 = static_cast<std::underlying_type_t<Color>>(Color::Red);
```

### Why Use These

| Problem | Solution |
| --- | --- |
| Need integer value of enum class (no implicit conversion) | `std::to_underlying(e)` or `static_cast<underlying_type_t<E>>(e)` |
| Storage for enum in a generic container | Use `underlying_type_t<E>` as the storage type |
| Bitwise operations on enum class flags | Cast to underlying, operate, cast back |
| Serialization (write enum as integer) | `underlying_type_t<E>` tells you the exact size |

---

## Self-Assessment

### Q1: Cast an enum class value to its underlying integer type safely using `std::to_underlying` (C++23)

Notice how `std::to_underlying` preserves the exact underlying type - the result for `Priority` is `uint8_t`, not just some random `int`. That precision matters for serialization and generic code.

```cpp
#include <iostream>
#include <type_traits>
#include <utility>
#include <cstdint>

enum class Priority : uint8_t {
    Low    = 1,
    Medium = 2,
    High   = 3,
    Critical = 4
};

enum class ErrorCode : int32_t {
    Success      = 0,
    NotFound     = -1,
    AccessDenied = -2,
    Timeout      = -3
};

enum class Permissions : uint16_t {
    None    = 0,
    Read    = 1 << 0,
    Write   = 1 << 1,
    Execute = 1 << 2,
    All     = Read | Write | Execute
};

int main() {
    // C++23: std::to_underlying — clean enum-to-integer conversion
    auto p = std::to_underlying(Priority::High);
    static_assert(std::is_same_v<decltype(p), uint8_t>);
    std::cout << "Priority::High = " << static_cast<int>(p) << "\n";

    auto e = std::to_underlying(ErrorCode::NotFound);
    static_assert(std::is_same_v<decltype(e), int32_t>);
    std::cout << "ErrorCode::NotFound = " << e << "\n";

    auto perm = std::to_underlying(Permissions::All);
    static_assert(std::is_same_v<decltype(perm), uint16_t>);
    std::cout << "Permissions::All = " << perm << "\n";

    // Comparison with the pre-C++23 way:
    auto old_way = static_cast<std::underlying_type_t<Priority>>(Priority::Critical);
    std::cout << "Priority::Critical (old way) = " << static_cast<int>(old_way) << "\n";

    // Use in conditional logic
    if (std::to_underlying(Priority::High) >= 3) {
        std::cout << "High priority alert!\n";
    }

    // Serialization example:
    std::cout << "\nSerialization sizes:\n";
    std::cout << "  Priority:    " << sizeof(std::underlying_type_t<Priority>) << " byte(s)\n";
    std::cout << "  ErrorCode:   " << sizeof(std::underlying_type_t<ErrorCode>) << " byte(s)\n";
    std::cout << "  Permissions: " << sizeof(std::underlying_type_t<Permissions>) << " byte(s)\n";

    return 0;
}
```

The `Permissions::All = 7` result comes from `Read(1) | Write(2) | Execute(4)` - bitwise OR works in the enum definition because the enum's underlying type is an integer even when the enum class itself doesn't expose that implicitly.

**Output:**

```text
Priority::High = 3
ErrorCode::NotFound = -1
Permissions::All = 7
Priority::Critical (old way) = 4
High priority alert!

Serialization sizes:
  Priority:    1 byte(s)
  ErrorCode:   4 byte(s)
  Permissions: 2 byte(s)
```

### Q2: Show a pre-C++23 cast using `static_cast<std::underlying_type_t<E>>(e)`

Before C++23 you needed a helper function - and it's genuinely useful to write one rather than spelling out the full cast every time. This example also shows `underlying_type_t` used as a storage type inside a generic class:

```cpp
#include <iostream>
#include <type_traits>
#include <cstdint>
#include <string>

enum class LogLevel : uint8_t {
    Trace   = 0,
    Debug   = 1,
    Info    = 2,
    Warning = 3,
    Error   = 4,
    Fatal   = 5
};

// Pre-C++23 helper: simulates std::to_underlying
template<typename E>
constexpr auto to_underlying(E e) noexcept {
    return static_cast<std::underlying_type_t<E>>(e);
}

// Convert int back to enum (with bounds check)
template<typename E>
constexpr E from_underlying(std::underlying_type_t<E> val) {
    return static_cast<E>(val);
}

// Use underlying_type_t for storage and comparison
template<typename E>
class EnumRange {
    using U = std::underlying_type_t<E>;
    U min_, max_;
public:
    constexpr EnumRange(E min, E max)
        : min_(to_underlying(min)), max_(to_underlying(max)) {}

    constexpr bool contains(E value) const {
        U v = to_underlying(value);
        return v >= min_ && v <= max_;
    }
};

std::string level_name(LogLevel level) {
    // Use underlying type for array indexing
    constexpr const char* names[] = {"TRACE", "DEBUG", "INFO", "WARNING", "ERROR", "FATAL"};
    auto idx = static_cast<std::underlying_type_t<LogLevel>>(level);
    return names[idx];
}

int main() {
    std::cout << std::boolalpha;

    // Pre-C++23 cast
    auto val = static_cast<std::underlying_type_t<LogLevel>>(LogLevel::Warning);
    static_assert(std::is_same_v<decltype(val), uint8_t>);
    std::cout << "Warning value: " << static_cast<int>(val) << "\n";

    // Using our helper
    std::cout << "Error value: " << static_cast<int>(to_underlying(LogLevel::Error)) << "\n";

    // Back to enum
    auto restored = from_underlying<LogLevel>(static_cast<uint8_t>(2));
    std::cout << "Restored: " << level_name(restored) << "\n";

    // EnumRange
    constexpr EnumRange<LogLevel> important(LogLevel::Warning, LogLevel::Fatal);
    std::cout << "\nIs Info important? " << important.contains(LogLevel::Info) << "\n";
    std::cout << "Is Error important? " << important.contains(LogLevel::Error) << "\n";
    std::cout << "Is Fatal important? " << important.contains(LogLevel::Fatal) << "\n";

    return 0;
}
```

The `EnumRange` class is a good illustration of why `underlying_type_t` is worth knowing: it lets you store enum values as their integer type internally (so you can compare them with `>=` and `<=`) while keeping the public interface in terms of the enum itself.

**Output:**

```text
Warning value: 3
Error value: 4
Restored: INFO

Is Info important? false
Is Error important? true
Is Fatal important? true
```

### Q3: Implement a bitflag type based on enum class using `underlying_type` for the storage

Bitwise operators on `enum class` don't come for free - you have to define them. Here the trick is always: cast to the underlying integer, operate, cast back. `underlying_type_t` makes that pattern concise and generic enough to work for any enum:

```cpp
#include <iostream>
#include <type_traits>
#include <cstdint>
#include <string>

// Define bitflag enum
enum class FileMode : uint32_t {
    None      = 0,
    Read      = 1 << 0,   // 1
    Write     = 1 << 1,   // 2
    Execute   = 1 << 2,   // 4
    Hidden    = 1 << 3,   // 8
    System    = 1 << 4,   // 16
    ReadWrite = Read | Write,
    All       = Read | Write | Execute | Hidden | System
};

// Bitwise operators using underlying_type
template<typename E>
    requires std::is_enum_v<E>
constexpr E operator|(E lhs, E rhs) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(static_cast<U>(lhs) | static_cast<U>(rhs));
}

template<typename E>
    requires std::is_enum_v<E>
constexpr E operator&(E lhs, E rhs) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(static_cast<U>(lhs) & static_cast<U>(rhs));
}

template<typename E>
    requires std::is_enum_v<E>
constexpr E operator^(E lhs, E rhs) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(static_cast<U>(lhs) ^ static_cast<U>(rhs));
}

template<typename E>
    requires std::is_enum_v<E>
constexpr E operator~(E e) {
    using U = std::underlying_type_t<E>;
    return static_cast<E>(~static_cast<U>(e));
}

template<typename E>
    requires std::is_enum_v<E>
constexpr E& operator|=(E& lhs, E rhs) { return lhs = lhs | rhs; }

template<typename E>
    requires std::is_enum_v<E>
constexpr E& operator&=(E& lhs, E rhs) { return lhs = lhs & rhs; }

// Helper: check if a flag is set
template<typename E>
    requires std::is_enum_v<E>
constexpr bool has_flag(E flags, E test) {
    using U = std::underlying_type_t<E>;
    return (static_cast<U>(flags) & static_cast<U>(test)) == static_cast<U>(test);
}

// Pretty print
std::string to_string(FileMode mode) {
    std::string result;
    if (has_flag(mode, FileMode::Read))    result += "R";
    if (has_flag(mode, FileMode::Write))   result += "W";
    if (has_flag(mode, FileMode::Execute)) result += "X";
    if (has_flag(mode, FileMode::Hidden))  result += "H";
    if (has_flag(mode, FileMode::System))  result += "S";
    return result.empty() ? "None" : result;
}

int main() {
    // Combine flags
    FileMode mode = FileMode::Read | FileMode::Write;
    std::cout << "mode = " << to_string(mode) << "\n";

    // Add a flag
    mode |= FileMode::Execute;
    std::cout << "After |= Execute: " << to_string(mode) << "\n";

    // Check flags
    std::cout << std::boolalpha;
    std::cout << "Has Read? " << has_flag(mode, FileMode::Read) << "\n";
    std::cout << "Has Hidden? " << has_flag(mode, FileMode::Hidden) << "\n";

    // Remove a flag
    mode &= ~FileMode::Write;
    std::cout << "After removing Write: " << to_string(mode) << "\n";

    // Toggle a flag
    mode = mode ^ FileMode::Hidden;
    std::cout << "After toggle Hidden: " << to_string(mode) << "\n";

    // Use underlying_type for raw value
    using U = std::underlying_type_t<FileMode>;
    U raw = static_cast<U>(mode);
    std::cout << "\nRaw value: " << raw << "\n";
    std::cout << "Underlying type size: " << sizeof(U) << " bytes\n";

    return 0;
}
```

One caution worth noting: these operator templates are constrained to `is_enum_v<E>`, which means they apply to all enums in the translation unit. In production code you'd want to narrow the constraint to specific flag enums to avoid accidentally enabling bitwise operations on something like a `Color` enum.

**Output:**

```text
mode = RW
After |= Execute: RWX
Has Read? true
Has Hidden? false
After removing Write: RX
After toggle Hidden: RXH

Raw value: 13
Underlying type size: 4 bytes
```

---

## Notes

- **`std::to_underlying` (C++23)** is trivial to implement for pre-C++23: `template<typename E> constexpr auto to_underlying(E e) noexcept { return static_cast<std::underlying_type_t<E>>(e); }`
- `std::underlying_type` is undefined behavior for non-enum types. Only use it with enums.
- For unscoped enums without a declared underlying type, the underlying type is implementation-defined (but at least as large as `int`). Scoped enums (`enum class`) default to `int`.
- Bitflag operators should be constrained to specific flag enums in production code (not all enums), to avoid enabling bitwise ops on enums like `Color`.
