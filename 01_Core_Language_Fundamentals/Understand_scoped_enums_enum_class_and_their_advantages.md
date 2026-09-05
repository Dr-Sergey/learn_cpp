# Understand scoped enums (enum class) and their advantages

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP11.md#strongly-typed-enums>  

---

## Topic Overview

**Scoped enums** (`enum class`) were introduced in C++11 to fix three major problems with
traditional (unscoped) enums.

### Problem 1: Namespace Pollution

Unscoped enum names leak directly into the enclosing scope, making name collisions almost inevitable across different enums:

```cpp
// BAD: Unscoped enum - names leak into enclosing scope:
enum Color { Red, Green, Blue };
enum TrafficLight { Red, Yellow, Green };
// ERROR: 'Red' and 'Green' are already defined!

// GOOD: Scoped enum - names are contained:
enum class Color { Red, Green, Blue };
enum class TrafficLight { Red, Yellow, Green };
// No conflict! Must use Color::Red, TrafficLight::Red
```

### Problem 2: Implicit Integer Conversion

Unscoped enums silently convert to `int`, which opens the door to nonsensical arithmetic and accidental comparisons between unrelated enum types:

```cpp
// BAD: Unscoped enum - silently converts to int:
enum OldColor { Red, Green, Blue };
int x = Red;           // OK: implicit conversion to int
if (Red == 0) { ... }  // OK: comparing enum with int
int y = Red + Blue;    // OK: arithmetic on enums

// GOOD: Scoped enum - no implicit conversion:
enum class Color { Red, Green, Blue };
// int x = Color::Red;           // ERROR
int x = static_cast<int>(Color::Red);  // Must be explicit
// if (Color::Red == 0) { ... }  // ERROR
// auto y = Color::Red + Color::Blue; // ERROR
```

### Problem 3: No Control Over Underlying Type

With `enum class`, you can specify the exact storage type, which matters for serialization, bitfields, and memory layout:

```cpp
// GOOD: Scoped enum - specify the underlying type:
enum class Status : uint8_t  { OK = 0, Error = 1, Timeout = 2 };
enum class Flags  : uint32_t { None = 0, Read = 1, Write = 2, Exec = 4 };

// Check the size:
static_assert(sizeof(Status) == 1);  // guaranteed 1 byte
static_assert(sizeof(Flags) == 4);   // guaranteed 4 bytes
```

### Safe Conversion

When you do need the underlying integer value, you must be explicit about it. C++23 added `std::to_underlying` to make that a bit cleaner:

```cpp
enum class Color : int { Red = 0, Green = 1, Blue = 2 };

// Convert TO int (explicit):
int value = static_cast<int>(Color::Green);  // 1

// Convert FROM int (explicit, be careful):
Color c = static_cast<Color>(1);  // Color::Green

// C++23: std::to_underlying (cleaner syntax)
#include <utility>
auto value2 = std::to_underlying(Color::Blue);  // 2

// Pre-C++23 helper:
template<typename E>
constexpr auto to_underlying(E e) noexcept {
    return static_cast<std::underlying_type_t<E>>(e);
}
```

### Switch Completeness

One practical benefit of `enum class` is that the compiler can warn you when a switch is missing a case. Avoid `default:` when you want this protection:

```cpp
enum class Shape { Circle, Square, Triangle };

void draw(Shape s) {
    switch (s) {
        case Shape::Circle:   draw_circle();   break;
        case Shape::Square:   draw_square();   break;
        // Missing Shape::Triangle!
        // Compiler warns: "enumeration value 'Triangle' not handled in switch"
        // With -Wswitch-enum or -Wswitch flag
    }
}
```

### Bitwise Operations on Enum Class

`enum class` doesn't support bitwise ops by default - you need to define them. This is the standard boilerplate for flag-style enums:

```cpp
enum class Permissions : uint8_t {
    None    = 0,
    Read    = 1 << 0,
    Write   = 1 << 1,
    Execute = 1 << 2,
};

// Define bitwise operators:
constexpr Permissions operator|(Permissions a, Permissions b) {
    return static_cast<Permissions>(
        static_cast<uint8_t>(a) | static_cast<uint8_t>(b));
}
constexpr Permissions operator&(Permissions a, Permissions b) {
    return static_cast<Permissions>(
        static_cast<uint8_t>(a) & static_cast<uint8_t>(b));
}

// Usage:
auto perms = Permissions::Read | Permissions::Write;
if ((perms & Permissions::Read) != Permissions::None) {
    std::cout << "Has read permission\n";
}
```

---

## Self-Assessment

### Q1: Show how unscoped enum pollutes namespace and causes implicit int conversion bugs

This example collects all the unscoped enum failure modes in one place. The most alarming one is comparing `Red == Apple` - that compiles fine and evaluates to `true` because both happen to be zero:

```cpp
#include <iostream>

// BAD: Unscoped enum problems
enum Color { Red, Green, Blue };
// Red, Green, Blue leak into the global namespace!

// enum TrafficLight { Red, Yellow, Green };  // ERROR: Red and Green redefined

void unscoped_bugs() {
    Color c = Red;

    // Bug 1: Implicit conversion to int
    int x = c;                    // Silently converts - no warning
    std::cout << "x = " << x << "\n";  // prints 0

    // Bug 2: Arithmetic on enums (almost always a bug)
    int sum = Red + Blue;         // 0 + 2 = 2
    std::cout << "sum = " << sum << "\n";

    // Bug 3: Comparing different enums compiles fine
    enum Fruit { Apple, Banana };
    if (Red == Apple) {           // Compares Color with Fruit! Both are 0.
        std::cout << "Red == Apple?!\n";  // This prints!
    }

    // Bug 4: Can assign any int
    Color bad = static_cast<Color>(999);  // No compile error, undefined meaning
}

// GOOD: Scoped enum - all bugs prevented
enum class SafeColor { Red, Green, Blue };

void scoped_safe() {
    SafeColor c = SafeColor::Red;
    // int x = c;                   // ERROR: no implicit conversion
    // int sum = SafeColor::Red + SafeColor::Blue;  // ERROR
    // if (SafeColor::Red == 0) {}  // ERROR: can't compare with int
    int x = static_cast<int>(c);   // Must be explicit
    std::cout << "x = " << x << "\n";
}

int main() {
    unscoped_bugs();
    scoped_safe();
}
```

**How this works:**

- Unscoped enums inject their enumerators into the enclosing scope, causing name collisions.
- They implicitly convert to `int`, enabling nonsensical arithmetic and comparisons.
- Scoped enums (`enum class`) prevent all of these - names are scoped, conversions must be explicit.

### Q2: Demonstrate specifying the underlying type of an enum class and converting safely

Specifying the underlying type is especially useful when you need a guaranteed size (e.g., for serialized data or hardware registers). `std::underlying_type_t` lets you check or use that type at compile time:

```cpp
#include <iostream>
#include <cstdint>
#include <type_traits>

// Specify underlying type
enum class ErrorCode : uint16_t {
    OK           = 0,
    NotFound     = 404,
    ServerError  = 500,
    Timeout      = 504
};

enum class Flags : uint8_t {
    None  = 0,
    Read  = 1 << 0,  // 1
    Write = 1 << 1,  // 2
    Exec  = 1 << 2   // 4
};

// Safe conversion helper (pre-C++23)
template<typename E>
constexpr auto to_underlying(E e) noexcept {
    return static_cast<std::underlying_type_t<E>>(e);
}

int main() {
    // Size is guaranteed by the underlying type
    static_assert(sizeof(ErrorCode) == 2);  // uint16_t = 2 bytes
    static_assert(sizeof(Flags) == 1);      // uint8_t = 1 byte

    // Safe conversion to underlying type
    ErrorCode err = ErrorCode::NotFound;
    uint16_t code = to_underlying(err);     // 404
    std::cout << "Error code: " << code << "\n";

    // C++23: std::to_underlying (does the same thing)
    // auto code2 = std::to_underlying(err);

    // Convert from integer (must be explicit and careful)
    uint16_t raw = 500;
    ErrorCode e2 = static_cast<ErrorCode>(raw);  // OK: ErrorCode::ServerError

    // Verify underlying type at compile time
    static_assert(std::is_same_v<std::underlying_type_t<ErrorCode>, uint16_t>);
    static_assert(std::is_same_v<std::underlying_type_t<Flags>, uint8_t>);

    std::cout << "Flags::Read = " << to_underlying(Flags::Read) << "\n";  // 1
}
```

**How this works:**

- The `: uint16_t` suffix specifies the exact storage type, giving control over size and range.
- `std::underlying_type_t<E>` extracts the underlying type at compile time.
- `static_cast` is required for both directions (enum <-> integer).
- `std::to_underlying` (C++23) is the standard way to convert enum to its underlying integer.

### Q3: Write code comparing enum vs enum class in a switch statement with missing case detection

The critical habit to build here: don't add `default:` when switching on an `enum class`. Omitting it means the compiler will warn you when someone adds a new enumerator later.

```cpp
#include <iostream>

// Unscoped enum
enum Shape { Circle, Square, Triangle };

// Scoped enum
enum class Color { Red, Green, Blue, Yellow };

void draw_shape(Shape s) {
    switch (s) {
        case Circle:   std::cout << "Drawing circle\n"; break;
        case Square:   std::cout << "Drawing square\n"; break;
        // Missing Triangle! Compiler warns with -Wswitch
    }
}

void paint(Color c) {
    switch (c) {
        case Color::Red:    std::cout << "Painting red\n"; break;
        case Color::Green:  std::cout << "Painting green\n"; break;
        case Color::Blue:   std::cout << "Painting blue\n"; break;
        // Missing Color::Yellow! Compiler warns with -Wswitch
    }
}

// Best practice: no default case when switching on enum class
// This way the compiler can warn about unhandled enumerators.
// Adding `default:` suppresses the warning!

void best_practice(Color c) {
    switch (c) {
        case Color::Red:    std::cout << "R"; break;
        case Color::Green:  std::cout << "G"; break;
        case Color::Blue:   std::cout << "B"; break;
        case Color::Yellow: std::cout << "Y"; break;
        // All cases covered - compiler is happy
        // NO default: - if a new color is added, we get a warning
    }
}

int main() {
    draw_shape(Triangle);           // Falls through - no output, no warning at runtime
    paint(Color::Yellow);           // Falls through - no output, no warning at runtime

    // Key advantage of enum class: can't accidentally pass wrong type
    // draw_shape(Color::Red);      // ERROR with enum class
    // But with unscoped enum:
    // draw_shape(0);               // Compiles with unscoped! (0 = Circle)
}

// Compile: g++ -std=c++20 -Wall -Wswitch-enum file.cpp
// Both draw_shape and paint will produce -Wswitch warnings about missing cases.
```

**How this works:**

- Both enum and enum class trigger `-Wswitch` warnings for missing cases.
- **The critical difference:** enum class prevents implicit conversion from integers, so you can't accidentally pass `0` instead of `Color::Red`.
- **Best practice:** Don't add `default:` when switching on an enum class - omitting it lets the compiler warn when new enumerators are added later.
- `-Wswitch-enum` is stricter than `-Wswitch`: it warns even if a `default` case exists.

---

## Notes

- `enum class` is the **default choice** in modern C++. Use unscoped `enum` only for backward compatibility.
- C++20 added `using enum` to import scoped enum names locally: `using enum Color; auto c = Red;`
- For bitwise flags, define `operator|`, `operator&`, `operator^`, and `operator~` as free functions for your `enum class`.
- `std::to_underlying` (C++23) is cleaner than `static_cast<std::underlying_type_t<E>>(e)`.
- Forward-declaring an `enum class` is always valid: `enum class Color : int;` - useful for reducing header dependencies.
