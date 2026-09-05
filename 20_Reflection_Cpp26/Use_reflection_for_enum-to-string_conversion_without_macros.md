# Use reflection for enum-to-string conversion without macros

**Category:** Reflection Cpp26  
**Standard:** C++17  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

Before C++26, getting the string name of an enum value meant either writing a switch statement by hand, reaching for a macro trick, or depending on a library like `magic_enum`. All three options have real drawbacks. C++26 reflection changes this completely: `enumerators_of(^E)` gives you every enumerator of the enum at compile time, and `identifier_of` turns each one into its source-code name as a `string_view`. A single generic function covers every enum you will ever write, with no macros and no external dependencies.

Here is how the approaches compare:

| Approach | LOC | Portable | Enum limits | Maintenance |
| --- | --- | --- | --- | --- |
| Manual switch | N cases | Yes | None | Per-change |
| X-macros | ~5 | Yes | None | Fragile |
| `magic_enum` | 0 | Limited | ~256 values | Zero |
| **C++26 reflection** | **~5** | **Yes** | **None** | **Zero** |

The real win over `magic_enum` is the "None" in the enum limits column. `magic_enum` works by scanning a numeric range and checking whether the compiler's `__PRETTY_FUNCTION__` string contains a valid identifier - which is clever, but breaks for sparse or large enums. Reflection asks the compiler for its actual symbol table, so there is no range to worry about.

---

## Self-Assessment

### Q1: Iterate enum values and names with `enumerators_of`

This is the foundation everything else builds on. `enumerators_of` hands you a compile-time sequence of `std::meta::info` values, one per enumerator. The `template for` loop then expands over them - it is conceptually like a regular for loop, but each iteration is a separate compile-time step.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

enum class Weekday { Mon, Tue, Wed, Thu, Fri, Sat, Sun };

int main() {
    constexpr auto enumerators = std::meta::enumerators_of(^Weekday);
    static_assert(enumerators.size() == 7);

    std::cout << "Weekday enumerators:\n";
    template for (constexpr auto e : enumerators) {
        constexpr std::string_view name = std::meta::identifier_of(e);
        constexpr Weekday val = [:e:];
        std::cout << "  " << name << " = " << static_cast<int>(val) << '\n';
    }
    // Output:
    // Weekday enumerators:
    //   Mon = 0
    //   Tue = 1
    //   Wed = 2
    //   Thu = 3
    //   Fri = 4
    //   Sat = 5
    //   Sun = 6

    // Works with sparse enums too:
    enum class HttpCode { OK = 200, NotFound = 404, Error = 500 };
    template for (constexpr auto e : std::meta::enumerators_of(^HttpCode)) {
        std::cout << std::meta::identifier_of(e) << " = "
                  << static_cast<int>([:e:]) << '\n';
    }
}
```

Notice the `[:e:]` syntax - that is the "splice" operator, which takes a `std::meta::info` value and turns it back into the actual enumerator so you can compare or cast it. You will see this pattern throughout reflection code.

### Q2: `constexpr to_string()` without macros

With the basic iteration in place, the generic `to_string` is just a loop that checks each enumerator against the runtime value. What makes this elegant is that the same five-line template works for every enum in your codebase.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string_view>

enum class Season { Spring, Summer, Autumn, Winter };
enum class Priority { Low = 1, Medium = 5, High = 10, Critical = 100 };

// Generic enum_to_string - works for ANY enum:
template <typename E>
constexpr std::string_view to_string(E value) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) {
            return std::meta::identifier_of(e);
        }
    }
    return "<unknown>";
}

// Compile-time verification:
static_assert(to_string(Season::Spring) == "Spring");
static_assert(to_string(Season::Winter) == "Winter");
static_assert(to_string(Priority::Critical) == "Critical");

// Also: from_string:
template <typename E>
constexpr std::optional<E> from_string(std::string_view name) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (name == std::meta::identifier_of(e)) return [:e:];
    }
    return std::nullopt;
}

static_assert(from_string<Season>("Summer") == Season::Summer);

int main() {
    std::cout << to_string(Season::Autumn) << '\n';     // Autumn
    std::cout << to_string(Priority::High) << '\n';     // High

    auto s = from_string<Priority>("Medium");
    if (s) std::cout << "Parsed: " << static_cast<int>(*s) << '\n'; // 5
}
```

The `static_assert` lines at the top of `main` are not just documentation - they prove the conversion works correctly at compile time, before the program even runs. That is a level of confidence you cannot get from a hand-written switch.

### Q3: Comparison with `magic_enum` library

It is worth understanding exactly *why* `magic_enum` has range limits before you appreciate how reflection avoids them. This example walks through both approaches side by side.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>

enum class Color { Red, Green, Blue, Alpha };

// ===== magic_enum approach (C++17 library) =====
// #include <magic_enum.hpp>
// auto name = magic_enum::enum_name(Color::Red);    // "Red"
// auto val  = magic_enum::enum_cast<Color>("Blue"); // Color::Blue
//
// Limitations of magic_enum:
// 1. Default range: only [-128, 127] for enum values
//    enum class Big { X = 1000 }; // magic_enum CANNOT see this!
// 2. Relies on compiler-specific __PRETTY_FUNCTION__ / __FUNCSIG__
// 3. Increased compile time (instantiates for every value in range)
// 4. Not standard C++ - external dependency
// 5. Range must be customized per-enum for sparse/large enums

// ===== C++26 reflection approach =====
template <typename E>
constexpr std::string_view to_string(E value) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) return std::meta::identifier_of(e);
    }
    return "<unknown>";
}

// Advantages of reflection:
// 1. No range limit - works for ALL enum values, any underlying type
// 2. Standard C++ - no external library needed
// 3. Faster compilation - iterates actual enumerators, not a range
// 4. Works with ALL compilers that support C++26
// 5. Zero runtime overhead (compile-time only)

enum class BigEnum : long long { X = 1'000'000, Y = 2'000'000 };
// magic_enum: FAILS (out of default range)
// reflection: WORKS
static_assert(to_string(BigEnum::X) == "X");

int main() {
    std::cout << to_string(Color::Green) << '\n';  // Green
    std::cout << to_string(BigEnum::Y) << '\n';    // Y
}
```

The `BigEnum` example is the clearest demonstration of the fundamental difference. `magic_enum` cannot see `X = 1'000'000` because it was never designed to scan that far. Reflection just asks the compiler "what enumerators does this type have?" and gets back exactly the right answer, regardless of the numeric values.

---

## Notes

- `enumerators_of` returns all explicitly declared enumerators and nothing else - no synthetic or implicit values sneak in.
- Reflection-based conversion has no range limitations, unlike `magic_enum`, because it reads from the compiler's actual symbol table rather than probing a numeric range.
- The `template for` expansion generates code equivalent to a hand-written switch statement, so there is no runtime overhead over writing it by hand.
- This is the canonical first example for C++26 reflection - if someone asks "what is reflection good for?", this is usually the answer they get.
- Pairing `to_string` with `from_string` gives you complete bidirectional conversion for logging, config parsing, and serialization.
