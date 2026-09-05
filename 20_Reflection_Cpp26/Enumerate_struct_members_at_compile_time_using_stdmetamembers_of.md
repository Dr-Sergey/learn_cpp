# Enumerate struct members at compile time using std::meta::members_of

**Category:** Reflection Cpp26  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

`std::meta::members_of` and its companion `nonstatic_data_members_of` are the workhorses of struct introspection in C++26 reflection. They give you a compile-time sequence of member handles that you can iterate, inspect by name and type, and splice back into code to access the actual field values on real objects.

These are the queries you will reach for most often when writing generic struct utilities:

| Query | Returns |
| --- | --- |
| `members_of(^S)` | All members (data, static, functions) |
| `nonstatic_data_members_of(^S)` | Only instance data members |
| `identifier_of(m)` | Member name as `string_view` |
| `type_of(m)` | Member type as `meta::info` |
| `display_string_of(info)` | Human-readable type name (implementation-defined format) |

---

## Self-Assessment

### Q1: Iterate non-static data members of a struct

This example shows the full workflow: reflect the struct, walk its members, and then print each field value from a live object. Pay attention to the `if constexpr` blocks inside the loop - because each member has a different type, you need to branch at compile time to handle strings, booleans, and numeric types differently.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>

struct Config {
    std::string host;
    int port;
    bool verbose;
    double timeout;
};

int main() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^Config);

    std::cout << "Config has " << members.size() << " fields:\n";
    template for (constexpr auto m : members) {
        std::cout << "  " << std::meta::identifier_of(m) << '\n';
    }

    // Access on an instance:
    Config cfg{"localhost", 8080, true, 30.0};
    template for (constexpr auto m : members) {
        std::cout << std::meta::identifier_of(m) << " = ";
        // Use if constexpr to handle types that need different printing:
        using MemberType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MemberType, std::string>) {
            std::cout << '"' << cfg.[:m:] << '"';
        } else if constexpr (std::is_same_v<MemberType, bool>) {
            std::cout << (cfg.[:m:] ? "true" : "false");
        } else {
            std::cout << cfg.[:m:];
        }
        std::cout << '\n';
    }
}
// Output:
// Config has 4 fields:
//   host
//   port
//   verbose
//   timeout
// host = "localhost"
// port = 8080
// verbose = true
// timeout = 30
```

The line `using MemberType = [:std::meta::type_of(m):]` is how you get the actual C++ type of a reflected member so you can use it in `if constexpr` checks. You reflect `m` to a type info, then splice that type info back into a `using` declaration - giving you a real type alias to work with.

### Q2: Print name and type of each member

You can also query layout information - byte offset and size - directly from reflection. This is useful for debugging memory layouts, writing binary serializers, or just understanding how a struct is laid out without looking it up in a debugger.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string_view>

struct Particle {
    float x, y, z;
    float velocity;
    int id;
    bool active;
};

template <typename T>
void describe_struct() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    constexpr std::string_view type_name = std::meta::identifier_of(^T);

    std::cout << "struct " << type_name << " {\n";
    template for (constexpr auto m : members) {
        constexpr auto name = std::meta::identifier_of(m);
        constexpr auto type = std::meta::display_string_of(std::meta::type_of(m));
        constexpr auto offset = std::meta::offset_of(m);
        constexpr auto size = std::meta::size_of(std::meta::type_of(m));

        std::cout << "    " << type << " " << name
                  << ";  // offset=" << offset
                  << " size=" << size << '\n';
    }
    std::cout << "};  // total size=" << sizeof(T) << '\n';
}

int main() {
    describe_struct<Particle>();
}
// Output:
// struct Particle {
//     float x;  // offset=0 size=4
//     float y;  // offset=4 size=4
//     float z;  // offset=8 size=4
//     float velocity;  // offset=12 size=4
//     int id;  // offset=16 size=4
//     bool active;  // offset=20 size=1
// };  // total size=24
```

Notice that all four queries - `identifier_of`, `display_string_of`, `offset_of`, and `size_of` - are all `constexpr`. The entire describe output could be computed at compile time if needed. This is an automated struct introspection tool written in about fifteen lines.

### Q3: Generic `to_string()` for any aggregate

Putting it all together: a fully generic `to_string` that handles any struct you throw at it. The function uses `identifier_of(^T)` to get the type's own name for the prefix, then iterates members with the same `if constexpr` type-dispatch pattern from Q1.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>
#include <sstream>

template <typename T>
std::string generic_to_string(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;

    oss << std::meta::identifier_of(^T) << "{";
    bool first = true;
    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;
        oss << std::meta::identifier_of(m) << "=";

        using MemberType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MemberType, std::string>) {
            oss << '"' << obj.[:m:] << '"';
        } else if constexpr (std::is_same_v<MemberType, bool>) {
            oss << (obj.[:m:] ? "true" : "false");
        } else {
            oss << obj.[:m:];
        }
    }
    oss << "}";
    return oss.str();
}

struct Employee {
    std::string name;
    int id;
    double salary;
    bool active;
};

struct Vec2 { float x, y; };

int main() {
    Employee e{"Alice", 42, 95000.0, true};
    std::cout << generic_to_string(e) << '\n';
    // Output: Employee{name="Alice", id=42, salary=95000, active=true}

    Vec2 v{1.5f, 2.5f};
    std::cout << generic_to_string(v) << '\n';
    // Output: Vec2{x=1.5, y=2.5}
}
```

The key thing to appreciate here is that `generic_to_string` works for both `Employee` and `Vec2` without any modification. Add a field to either struct and the output automatically includes it. That zero-maintenance property is the whole point of reflection-based generic utilities.

---

## Notes

- `nonstatic_data_members_of` is the right choice for data-only iteration - prefer it over `members_of` unless you specifically need static members or member functions too.
- `identifier_of(^T)` returns the type's own name as a string - useful for generating output prefixes or diagnostic messages.
- `display_string_of` gives a human-readable type name, but the format is implementation-defined and may not be stable across compilers.
- `offset_of` and `size_of` enable precise layout introspection without `offsetof` macros or platform-specific tricks.
- Reflection works with inheritance - `bases_of(^Derived)` gives you the base classes so you can walk the full hierarchy if needed.
