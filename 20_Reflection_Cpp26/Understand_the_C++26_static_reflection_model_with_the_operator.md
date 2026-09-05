# Understand the C++26 static reflection model with the ^ operator

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

The `^` (caret / reflect) operator is the entry point to C++26 static reflection. Every time you write `^something`, the compiler produces a value of type `std::meta::info` - an opaque, compile-time handle that represents the entity you reflected. You can then pass that handle to the query functions in `std::meta` to learn everything about the entity at compile time.

Here is what you can reflect with `^`:

| Operand | Example | Reflects |
| --- | --- | --- |
| Type | `^int`, `^MyClass` | The type itself |
| Variable | `^myVar` | The variable declaration |
| Function | `^myFunc` | The function |
| Namespace | `^std` | The namespace |
| Enum | `^Color` | The enum type |
| Enumerator | `^Color::Red` | Individual enumerator |

---

## Self-Assessment

### Q1: Use `^int` to obtain a `std::meta::info` value

Here is the simplest possible introduction to the reflect operator. You write `^T` where `T` is any named entity, and you get a `meta::info` value back. You can then ask that value for a human-readable name using `display_string_of` or `identifier_of`.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>

int main() {
    // ^ operator produces a std::meta::info:
    constexpr std::meta::info int_info = ^int;
    constexpr std::meta::info double_info = ^double;
    constexpr std::meta::info string_info = ^std::string;

    // Query the display name:
    std::cout << std::meta::display_string_of(int_info) << '\n';    // int
    std::cout << std::meta::display_string_of(double_info) << '\n';  // double
    std::cout << std::meta::display_string_of(string_info) << '\n';  // std::string

    // Reflect user-defined types:
    struct Point { int x, y; };
    constexpr auto point_info = ^Point;
    std::cout << std::meta::identifier_of(point_info) << '\n';  // Point

    // Reflect a template:
    constexpr auto vec_info = ^std::vector;
    std::cout << std::meta::display_string_of(vec_info) << '\n';
    // std::vector
}
```

Notice that every one of these queries is a compile-time constant. There is nothing happening at runtime here - by the time `main` runs, the compiler already resolved all those names.

### Q2: Difference between a reflection (`meta::info`) and a type

This is the distinction that trips people up most often. A reflection is not a type - it is a compile-time value that *describes* a type. The difference matters because they live in completely different syntactic positions.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <type_traits>

// A reflection is NOT a type. It's a compile-time VALUE.
//
//   int            -> a type (used in declarations, sizeof, etc.)
//   ^int           -> a meta::info value (used in consteval queries)
//   [:^int:]       -> spliced back into a type

int main() {
    // ^int is a VALUE, not a type:
    constexpr auto r = ^int;           // OK: value
    // int x = r;                     // ERROR: r is meta::info, not int

    // You CANNOT use meta::info where a type is expected:
    // sizeof(^int);      // ERROR: ^int is a value, not a type
    // std::vector<^int>;  // ERROR

    // You CAN splice it back to a type:
    using T = [:r:];        // T is int
    static_assert(std::is_same_v<T, int>);

    T value = 42;           // int value = 42;
    std::cout << value << '\n';

    // Comparison: meta::info values compare by identity:
    static_assert(^int == ^int);           // same type -> equal
    static_assert(^int != ^double);        // different type -> not equal
    static_assert(^int != ^unsigned int);  // int != unsigned int

    // meta::info is a literal type:
    constexpr auto a = ^int;
    constexpr auto b = ^int;
    static_assert(a == b);  // OK, constexpr comparison
}
```

The round trip `^int` -> `meta::info` -> `[:r:]` -> `int` is the key. The reflect operator takes you into the reflection domain; the splice operator `[::]` brings you back out. You will see this pattern constantly in C++26 reflection code.

### Q3: Reflections as constant expressions passed to `consteval` functions

Once you can reflect an entity, you can write `consteval` functions that accept `meta::info` and answer compile-time questions about it. This is the foundation for all the interesting compile-time metaprogramming that reflection enables.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>

// consteval: MUST be evaluated at compile time
consteval std::size_t field_count(std::meta::info type) {
    return std::meta::nonstatic_data_members_of(type).size();
}

consteval bool is_empty_struct(std::meta::info type) {
    return std::meta::nonstatic_data_members_of(type).empty();
}

consteval bool has_field(std::meta::info type, std::string_view name) {
    for (auto m : std::meta::nonstatic_data_members_of(type)) {
        if (std::meta::identifier_of(m) == name) return true;
    }
    return false;
}

struct Empty {};
struct Pair { int first; int second; };
struct Config { std::string host; int port; bool debug; };

// All evaluated at compile time:
static_assert(field_count(^Empty) == 0);
static_assert(field_count(^Pair) == 2);
static_assert(field_count(^Config) == 3);

static_assert(is_empty_struct(^Empty));
static_assert(!is_empty_struct(^Pair));

static_assert(has_field(^Config, "port"));
static_assert(!has_field(^Config, "timeout"));

int main() {
    // Use compile-time reflection results at runtime:
    constexpr auto n = field_count(^Config);
    std::cout << "Config fields: " << n << '\n';   // 3

    // Conditional compilation based on reflection:
    if constexpr (has_field(^Config, "debug")) {
        std::cout << "Config has debug field\n";
    }
}
```

The `static_assert` calls at namespace scope verify everything purely at compile time. The `if constexpr` inside `main` is another natural fit - reflection gives you a genuine compile-time boolean, and `if constexpr` consumes it without generating dead code for the branch that does not apply.

---

## Notes

- `^` is a unary prefix operator with highest precedence.
- `^` can be applied to any named entity visible at the point of reflection.
- `std::meta::info` is a special type: literal, trivially copyable, but only meaningful at compile time.
- Reflections cannot be stored in runtime containers or passed across translation units.
- `consteval` functions accepting `meta::info` enable powerful compile-time metaprogramming.
