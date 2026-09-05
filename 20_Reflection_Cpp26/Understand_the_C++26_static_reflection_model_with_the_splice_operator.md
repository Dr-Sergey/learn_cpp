# Understand the C++26 static reflection model with the ^ splice operator

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

C++26 static reflection centers on two operators that work as a pair. The `^` (reflect) operator takes a source-code entity and converts it into a `std::meta::info` value you can inspect at compile time. The `[:r:]` (splice) operator goes the other direction: it takes a `meta::info` value and injects it back into code as a type, an expression, or a member access. Together they form a complete round-trip between your source code and the compile-time meta domain.

A useful way to picture it:

```cpp
// Source entity  ---^entity--->  std::meta::info  ---[:info:]--->  Code again
//      |                                                           |
//      int  ------^int------->  constexpr auto r  ---[:r:]----->  int
```

Here is a reference for the different ways you can use both operators:

| Operator | Syntax | Produces |
| --- | --- | --- |
| Reflect | `^entity` | `std::meta::info` compile-time value |
| Splice (type) | `[:type_info:]` | A type |
| Splice (value) | `[:value_info:]` | An expression |
| Splice (member) | `obj.[:member_info:]` | Member access |
| Name query | `identifier_of(info)` | `string_view` |

---

## Self-Assessment

### Q1: The reflect-expression `^T` produces a `std::meta::info` value

Here you can see `^` applied to a variety of different kinds of entities all at once: built-in types, user-defined types, enums, namespaces, and variables. The result in every case is a `meta::info` value that you can compare, pass to query functions, or store in a `constexpr` variable.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

struct Point { int x, y; };
enum class Color { Red, Green, Blue };

int main() {
    // ^T reflects any named entity into a meta::info value:
    constexpr std::meta::info int_r    = ^int;       // built-in type
    constexpr std::meta::info point_r  = ^Point;     // user type
    constexpr std::meta::info color_r  = ^Color;     // enum
    constexpr std::meta::info std_r    = ^std;       // namespace
    constexpr std::meta::info cout_r   = ^std::cout; // variable

    // meta::info is a compile-time value, not a runtime object:
    static_assert(int_r == ^int);       // identity comparison
    static_assert(int_r != ^double);    // different types

    // Query properties:
    std::cout << std::meta::display_string_of(int_r) << '\n';      // int
    std::cout << std::meta::identifier_of(point_r) << '\n';         // Point
    std::cout << std::meta::identifier_of(color_r) << '\n';         // Color

    // Count members:
    constexpr auto members = std::meta::nonstatic_data_members_of(point_r);
    static_assert(members.size() == 2);  // x, y
}
```

The identity comparison `static_assert(int_r == ^int)` illustrates that two reflections of the same entity compare equal, regardless of when or where you wrote `^int`. This makes reflections safe to use as compile-time keys or cache values.

### Q2: Use `identifier_of(^T)` to retrieve a type's name

One of the most practical immediate benefits of reflection is getting a reliable, portable, human-readable name for a type - no more `typeid(T).name()` returning mangled garbage. Here, `type_name<T>()` returns exactly the identifier as written in the source.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string_view>

struct Widget { int id; std::string label; };
struct Gadget { double value; bool active; };

// Generic type name printer:
template <typename T>
constexpr std::string_view type_name() {
    return std::meta::identifier_of(^T);
}

static_assert(type_name<Widget>() == "Widget");
static_assert(type_name<Gadget>() == "Gadget");

// Use in logging:
template <typename T>
void log_type(const T& obj) {
    std::cout << "[" << type_name<T>() << "] object at "
              << &obj << '\n';
}

int main() {
    Widget w{1, "test"};
    Gadget g{3.14, true};

    log_type(w);  // [Widget] object at 0x...
    log_type(g);  // [Gadget] object at 0x...

    // display_string_of for fully-qualified names:
    std::cout << std::meta::display_string_of(^std::string) << '\n';
    // std::basic_string<char, ...> (implementation-defined)
}
```

Use `identifier_of` when you want the short, source-level name. Use `display_string_of` when you want a more complete representation, including template arguments - but keep in mind that the exact format of `display_string_of` is implementation-defined.

### Q3: The splice `[:r:]` materialises a reflected entity back into code

This is the other half of the story. `[:r:]` takes a `meta::info` value and injects the entity it represents directly into whatever syntactic position you put it. You can use it as a type (in a `using` declaration or a template argument), as a member access on an object, or to write through a reflected member.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <vector>

struct Sensor {
    std::string name;
    double reading;
    int unit_id;
};

int main() {
    // === Type splicing ===
    constexpr auto int_r = ^int;
    using T = [:int_r:];        // T is int
    T x = 42;
    static_assert(std::is_same_v<T, int>);

    // Use in container:
    std::vector<[:int_r:]> v = {1, 2, 3};  // vector<int>

    // === Member splicing ===
    constexpr auto members = std::meta::nonstatic_data_members_of(^Sensor);
    Sensor s{"Temp", 23.5, 7};

    // Access members by reflection:
    template for (constexpr auto m : members) {
        std::cout << std::meta::identifier_of(m) << " = ";
        // obj.[:m:] accesses the actual member:
        auto& val = s.[:m:];

        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            std::cout << '"' << val << '"';
        else
            std::cout << val;
        std::cout << '\n';
    }
    // Output:
    // name = "Temp"
    // reading = 23.5
    // unit_id = 7

    // === Value splicing ===
    // Modify through splice:
    constexpr auto reading_m = members[1];  // 'reading' member
    s.[:reading_m:] = 99.9;
    std::cout << "Updated reading: " << s.reading << '\n';  // 99.9
}
```

The reason `s.[:m:]` works is that the splice expands to the actual member name at compile time - the line `auto& val = s.[:m:]` is literally equivalent to writing `auto& val = s.name` (or `.reading`, or `.unit_id`) once the compiler processes the `template for` expansion. No runtime indirection, no pointer arithmetic - just normal member access generated for you.

---

## Notes

- `^` and `[::]` are inverses: `[:^T:]` gives back `T`.
- `identifier_of` returns the source-level name exactly as written.
- `display_string_of` returns a human-readable string (may include template args).
- Both operators work entirely at compile time - no runtime overhead.
- The splice operator can appear in type positions, expression positions, and template arguments.
