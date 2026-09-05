# Enumerate enum values and names at compile time using reflection

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

C++26 reflection lets you ask the compiler "what enumerators does this enum have?" at compile time using `std::meta::enumerators_of`. Combined with `identifier_of` (to get the name as a string) and splicing (to get the actual value), you can build enum-to-string and string-to-enum conversions that work for any enum type - no macros, no X-macros, no hand-written switch statements.

The key APIs are worth understanding together, since they form a pipeline: reflect the enum, walk its enumerators, then either splice a value back out or read a name:

| API | Purpose |
| --- | --- |
| `^MyEnum` | Reflect the enum type into a `meta::info` handle |
| `enumerators_of(^E)` | Get a `vector<info>` of all enumerators |
| `identifier_of(e)` | Get enumerator name as `string_view` |
| `[:e:]` | Splice back to a value |
| `value_of<E>(e)` | Get the underlying integer value |

To make the flow concrete, here is the pipeline in diagram form - you reflect a type into a handle, enumerate its enumerators, then convert each one either to its name string or back to the value:

```cpp
Reflection flow:

  enum Color { Red, Green, Blue };

  ^Color -> meta::info (reflection of Color)
    |
  enumerators_of(^Color) -> [info(Red), info(Green), info(Blue)]
    |
  identifier_of(info(Red)) -> "Red"
  [:info(Red):] -> Color::Red (value)
```

---

## Self-Assessment

### Q1: Use `enumerators_of` to list all enumerator values

The goal here is simple: print every enumerator's name and integer value at runtime by doing all the introspection at compile time. Notice the `template for` loop - this is the C++26 expansion statement that lets you iterate a compile-time sequence, unlike a regular `for` which requires a runtime range.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

enum class Color { Red = 1, Green = 2, Blue = 4 };

int main() {
    // Get all enumerators:
    constexpr auto enumerators = std::meta::enumerators_of(^Color);

    // Iterate at compile time (template for + constexpr):
    std::cout << "Color enumerators:\n";
    template for (constexpr auto e : enumerators) {
        // identifier_of: get the name as string_view
        constexpr std::string_view name = std::meta::identifier_of(e);
        // [:e:] splices back to the actual value
        constexpr Color val = [:e:];
        std::cout << "  " << name << " = "
                  << static_cast<int>(val) << '\n';
    }
}
// Output:
//   Color enumerators:
//   Red = 1
//   Green = 2
//   Blue = 4
```

Each iteration gives you a compile-time `meta::info` handle for one enumerator. You then call `identifier_of` to read its source name and `[:e:]` to splice it back into an actual `Color` value.

### Q2: Type-safe `enum_to_string()` without macros

Before reflection, converting an enum to a string meant either a giant hand-written switch or a brittle macro expansion. With reflection you write the function once and it works for every enum type automatically - the compiler walks the enumerators for you.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <string_view>
#include <stdexcept>

enum class HttpStatus { OK = 200, NotFound = 404, ServerError = 500 };

// Generic enum_to_string using reflection:
template <typename E>
constexpr std::string_view enum_to_string(E value) {
    // Iterate enumerators at compile time:
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) {
            return std::meta::identifier_of(e);
        }
    }
    throw std::invalid_argument("Unknown enum value");
}

// Works for ANY enum type without changes:
static_assert(enum_to_string(HttpStatus::OK) == "OK");
static_assert(enum_to_string(HttpStatus::NotFound) == "NotFound");

enum class Direction { North, South, East, West };
static_assert(enum_to_string(Direction::East) == "East");

int main() {
    std::cout << enum_to_string(HttpStatus::ServerError) << '\n';
    // Output: ServerError

    // Comparison with old approach:
    // OLD: X-macro or manual switch with 20+ cases
    // NEW: 5 lines of generic code, works for ALL enums
}
```

The `static_assert` lines at the bottom are not just a test - they prove this all runs at compile time. You get both compile-time verified results and a runtime-usable function from the same code.

### Q3: Compile-time `string_to_enum()` lookup table

Going the other direction (string to enum) is equally clean. The first approach below does a linear scan - fine for small enums. The second builds a sorted array at compile time so you can binary-search it at runtime, which matters when you're parsing many strings in a loop.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <string_view>
#include <optional>
#include <array>
#include <algorithm>

enum class LogLevel { Debug, Info, Warning, Error, Fatal };

// Generic string_to_enum using reflection:
template <typename E>
constexpr std::optional<E> string_to_enum(std::string_view name) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (name == std::meta::identifier_of(e)) {
            return [:e:];
        }
    }
    return std::nullopt;
}

// Or generate a sorted lookup table for O(log N) search:
template <typename E>
constexpr auto make_enum_lookup_table() {
    constexpr auto enums = std::meta::enumerators_of(^E);
    std::array<std::pair<std::string_view, E>, enums.size()> table;

    std::size_t i = 0;
    template for (constexpr auto e : enums) {
        table[i++] = { std::meta::identifier_of(e), [:e:] };
    }

    std::ranges::sort(table, {}, &std::pair<std::string_view, E>::first);
    return table;
}

constexpr auto log_level_table = make_enum_lookup_table<LogLevel>();

int main() {
    // Linear search:
    auto level = string_to_enum<LogLevel>("Warning");
    if (level) {
        std::cout << "Parsed: " << enum_to_string(*level) << '\n';
    }

    // Table lookup (binary search):
    auto it = std::ranges::lower_bound(
        log_level_table, "Error",
        {}, &decltype(log_level_table)::value_type::first);
    if (it != log_level_table.end() && it->first == "Error") {
        std::cout << "Found: " << static_cast<int>(it->second) << '\n';
    }
}
// Output:
// Parsed: Warning
// Found: 3
```

The sorted table is built entirely at compile time - `constexpr auto log_level_table = ...` means the array and the sort both happen before your program runs. At runtime you get a read-only, pre-sorted array with O(log N) lookup and no heap allocation.

---

## Notes

- `enumerators_of` works on both scoped (`enum class`) and unscoped (`enum`) enums - you do not need to change your enum definition.
- `template for` is the C++26 expansion statement for iterating compile-time ranges; it is different from a regular `for` loop because the body is instantiated separately for each element.
- All reflection happens at compile time - the lookup tables and name strings are baked in with zero runtime overhead.
- This replaces macro-heavy solutions like `BETTER_ENUMS`, `magic_enum`, and X-macros with a language-level mechanism that needs no external library.
- `identifier_of` returns the source name exactly as written (case-sensitive), so `"NotFound"` will not match `"notFound"`.
