# Understand static reflection with the ^ operator and std::meta

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

C++26 introduces **static reflection** via the `^` (reflect) operator and the `std::meta` namespace. The key idea is simple: `^T` produces a value of type `std::meta::info` that represents the entity `T` at compile time. You can then pass that value to query functions in `std::meta` to ask questions about the entity - its name, its members, its type - all during compilation with zero runtime cost.

Here is a quick reference for the key pieces:

| Concept | Description |
| --- | --- |
| `^entity` | Reflect operator - turns any entity into a `meta::info` value |
| `std::meta::info` | Opaque compile-time type representing a reflected entity |
| `identifier_of(info)` | Get the source name as `string_view` |
| `type_of(info)` | Get the type of a reflected member |
| `members_of(info)` | Get all members of a reflected class/struct |
| `[:info:]` | Splice - convert back from reflection to code |

If you want a mental picture of the pipeline, here it is:

```cpp
// Reflection pipeline:
//
//   Source code entity  ---^entity--->  std::meta::info
//                                          |
//                                     query APIs
//                                     (name, type,
//                                      members...)
//                                          |
//   Generated code      --[:info:]-------->
```

The `^` operator takes you from code to a meta-value, and the splice operator `[::]` takes you back from a meta-value to code. Everything in between is compile-time querying.

---

## Self-Assessment

### Q1: Use `^T` to obtain a reflection and query its name

This example shows the most fundamental use: reflect several different kinds of entities and print their names. Notice that `^` works on types, built-ins, namespaces, and enums equally.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

struct Widget {
    int id;
    std::string name;
    double price;
};

int main() {
    // Reflect a type:
    constexpr std::meta::info widget_refl = ^Widget;

    // Query the name:
    constexpr std::string_view type_name = std::meta::identifier_of(widget_refl);
    std::cout << "Type: " << type_name << '\n';  // Type: Widget

    // Reflect built-in types:
    constexpr auto int_refl = ^int;
    std::cout << "Built-in: " << std::meta::display_string_of(int_refl) << '\n';
    // Built-in: int

    // Reflect a namespace:
    constexpr auto std_refl = ^std;
    std::cout << "Namespace: " << std::meta::identifier_of(std_refl) << '\n';
    // Namespace: std

    // Reflect an enum:
    enum Color { Red, Green, Blue };
    constexpr auto color_refl = ^Color;
    std::cout << "Enum: " << std::meta::identifier_of(color_refl) << '\n';
    // Enum: Color
}
```

Every call here is a compile-time operation. By the time this program runs, all those names are already baked in as constants - the program is just printing them.

### Q2: Enumerate data members of a struct using `members_of`

This is where reflection starts to get genuinely useful. The `template for` loop is a compile-time expansion - it unrolls once per member, even though each member may have a completely different type. Notice how the loop body uses both `type_of` and `identifier_of` to print a type-annotated field list for any struct you hand it.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>

struct Config {
    std::string host;
    int port;
    bool ssl;
};

template <typename T>
void print_members() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);

    std::cout << std::meta::identifier_of(^T) << " has "
              << members.size() << " members:\n";

    template for (constexpr auto m : members) {
        std::cout << "  "
                  << std::meta::display_string_of(std::meta::type_of(m))
                  << " " << std::meta::identifier_of(m) << '\n';
    }
}

int main() {
    print_members<Config>();
    // Output:
    // Config has 3 members:
    //   std::string host
    //   int port
    //   bool ssl

    // Works for any struct:
    struct Vec3 { float x, y, z; };
    print_members<Vec3>();
    // Vec3 has 3 members:
    //   float x
    //   float y
    //   float z
}
```

The `print_members` function is completely generic - you did not write a single line specific to `Config` or `Vec3`. Reflection supplied all the struct-specific knowledge at compile time.

### Q3: Reflections are compile-time values, not runtime objects

This example nails down the most important conceptual point: `std::meta::info` is a compile-time-only value. You cannot store it in a `std::vector` at runtime, and you cannot create one inside an ordinary (non-`consteval`) function that runs at runtime. The reason this trips people up is that `meta::info` looks like a normal type, but it only has meaning during compilation - it is more like a type in a template than a value you carry around at runtime.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <type_traits>

// Key insight: std::meta::info is a COMPILE-TIME value.
// It exists only during compilation - zero runtime overhead.

// 1. You can pass reflections to consteval functions:
consteval bool has_member_named(std::meta::info type, std::string_view name) {
    for (auto m : std::meta::nonstatic_data_members_of(type)) {
        if (std::meta::identifier_of(m) == name) return true;
    }
    return false;
}

struct Player { std::string name; int hp; int level; };

// 2. Use in static_assert (purely compile-time):
static_assert(has_member_named(^Player, "hp"));
static_assert(!has_member_named(^Player, "mana"));

// 3. Use in constexpr variables:
constexpr auto player_refl = ^Player;
constexpr auto member_count = std::meta::nonstatic_data_members_of(player_refl).size();
static_assert(member_count == 3);

// 4. You CANNOT store meta::info at runtime:
// std::meta::info runtime_var;  // ERROR: not a runtime type
// std::vector<std::meta::info> v; // ERROR: cannot store in runtime container

// 5. consteval function - MUST be evaluated at compile time:
consteval std::size_t count_fields(std::meta::info t) {
    return std::meta::nonstatic_data_members_of(t).size();
}

int main() {
    // Compile-time constant derived from reflection:
    constexpr auto n = count_fields(^Player);
    std::cout << "Player has " << n << " fields\n";  // 3

    // The reflection itself has zero runtime footprint.
    // sizeof(std::meta::info) is NOT meaningful at runtime.
    static_assert(std::is_trivially_copyable_v<std::meta::info>);
}
```

The `static_assert` calls and the `consteval` functions all run at compile time. By the time `main` executes, the answer `3` is already a known constant in the binary - no reflection machinery runs at runtime at all.

---

## Notes

- `^` can reflect: types, namespaces, enums, variables, functions, templates.
- `std::meta::info` is an opaque, literal type - only meaningful at compile time.
- All `std::meta::` query functions are `consteval`.
- Reflections have zero runtime cost - they exist only during compilation.
- P2996R5 is the latest revision of the static reflection proposal.
