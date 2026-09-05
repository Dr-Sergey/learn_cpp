# Implement compile-time struct serialization using reflection

**Category:** Reflection Cpp26  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

Serialization is one of the most repeated pieces of boilerplate in any C++ codebase. Before reflection, every approach required you to either write the serializer by hand for each type, register the fields explicitly with a macro, or use an external code generator. Reflection makes all of that unnecessary - you write the serializer once and it works for every struct automatically.

The table below shows just how much work reflection eliminates compared to older approaches:

| Approach | LOC per type | Maintenance | Type safety |
| --- | --- | --- | --- |
| Hand-written `toJson()` | ~10-30 | Manual per field | Compile-time |
| X-Macros | ~5 (macro) | Fragile | Limited |
| `BOOST_DESCRIBE` | ~3 (registration) | Per-type registration | Good |
| **Reflection** | **0** | **Zero - automatic** | **Full** |

The "0 LOC per type" row is the key. You define your struct once, and every reflection-based utility you write - JSON serializers, binary packers, debug printers - picks it up automatically.

---

## Self-Assessment

### Q1: `consteval` JSON-like serializer for any aggregate

Here is the fundamental pattern: iterate the members, check each member's type at compile time with `if constexpr`, and format the value appropriately. The recursive call at the end handles nested structs by applying the same function to the sub-object.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <string>
#include <sstream>
#include <iostream>

// Generic JSON serializer using reflection:
template <typename T>
std::string to_json(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << "{";
    bool first = true;

    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;

        oss << '"' << std::meta::identifier_of(m) << "\": ";

        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>) {
            oss << '"' << obj.[:m:] << '"';
        } else if constexpr (std::is_same_v<MType, bool>) {
            oss << (obj.[:m:] ? "true" : "false");
        } else if constexpr (std::is_arithmetic_v<MType>) {
            oss << obj.[:m:];
        } else {
            // Recursively serialize nested structs:
            oss << to_json(obj.[:m:]);
        }
    }
    oss << "}";
    return oss.str();
}

struct Address { std::string city; int zip; };
struct Person { std::string name; int age; Address addr; };

int main() {
    Person p{"Alice", 30, {"NYC", 10001}};
    std::cout << to_json(p) << '\n';
    // Output: {"name": "Alice", "age": 30, "addr": {"city": "NYC", "zip": 10001}}

    Address a{"London", 12345};
    std::cout << to_json(a) << '\n';
    // Output: {"city": "London", "zip": 12345}
}
```

The `using MType = [:std::meta::type_of(m):]` line is the key to the type dispatch. You reflect the member's type into a handle, splice it into a `using` declaration, and then use that type alias in `if constexpr` checks. The compiler evaluates each branch at compile time - only the matching branch ends up in the generated code for each member type.

### Q2: Binary serialization layout using `meta::offset_of` and `meta::size_of`

For binary protocols and network serialization you often need to know the exact byte layout. Reflection gives you `offset_of` and `size_of` at compile time, so you can build a layout descriptor that is guaranteed to match the actual struct layout.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <cstring>
#include <vector>
#include <iostream>
#include <cstdint>

struct Packet {
    uint32_t id;
    uint16_t type;
    uint16_t flags;
    double payload;
};

// Compile-time layout descriptor:
template <typename T>
struct LayoutInfo {
    struct Field {
        std::string_view name;
        std::size_t offset;
        std::size_t size;
    };

    static constexpr auto fields = []{
        constexpr auto members = std::meta::nonstatic_data_members_of(^T);
        std::array<Field, members.size()> result{};
        std::size_t i = 0;
        template for (constexpr auto m : members) {
            result[i++] = {
                std::meta::identifier_of(m),
                std::meta::offset_of(m),
                std::meta::size_of(std::meta::type_of(m))
            };
        }
        return result;
    }();
};

// Binary serialize using layout info:
template <typename T>
std::vector<std::byte> serialize_binary(const T& obj) {
    std::vector<std::byte> buffer(sizeof(T));
    // Copy field by field using reflection-derived offsets:
    for (const auto& f : LayoutInfo<T>::fields) {
        std::memcpy(
            buffer.data() + f.offset,
            reinterpret_cast<const std::byte*>(&obj) + f.offset,
            f.size);
    }
    return buffer;
}

template <typename T>
T deserialize_binary(const std::vector<std::byte>& buffer) {
    T obj;
    std::memcpy(&obj, buffer.data(), sizeof(T));
    return obj;
}

int main() {
    Packet p{42, 1, 0x0F, 3.14};
    auto bytes = serialize_binary(p);
    auto p2 = deserialize_binary<Packet>(bytes);

    std::cout << "id=" << p2.id << " type=" << p2.type
              << " payload=" << p2.payload << '\n';
    // Output: id=42 type=1 payload=3.14

    // Print layout:
    for (const auto& f : LayoutInfo<Packet>::fields) {
        std::cout << f.name << ": offset=" << f.offset
                  << " size=" << f.size << '\n';
    }
}
```

`LayoutInfo<T>::fields` is a `static constexpr` array built inside a lambda - the entire thing is evaluated at compile time and lives in read-only memory. The serialize/deserialize functions then use that pre-computed layout at runtime, but they are doing nothing more than `memcpy` to the right offsets.

### Q3: Reflection vs X-macros vs `BOOST_DESCRIBE`

It is worth seeing the three approaches side by side to understand what reflection replaces and why. The comparison uses comments to show the old approaches because they do not interact cleanly with the new reflection-based `print_all_fields`.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>

// ========== OLD WAY 1: X-Macros ==========
// #define PERSON_FIELDS(X) X(std::string, name) X(int, age) X(double, height)
// struct PersonXMacro {
//     #define DECLARE(type, name) type name;
//     PERSON_FIELDS(DECLARE)
//     #undef DECLARE
// };
// void print_xmacro(const PersonXMacro& p) {
//     #define PRINT(type, name) std::cout << #name << "=" << p.name << '\n';
//     PERSON_FIELDS(PRINT)
//     #undef PRINT
// }
// Problems: fragile, no type safety, hard to debug, preprocessor pollution

// ========== OLD WAY 2: BOOST_DESCRIBE ==========
// struct PersonBoost { std::string name; int age; double height; };
// BOOST_DESCRIBE_STRUCT(PersonBoost, (), (name, age, height))
// // Must manually register each field name -- easy to forget one

// ========== NEW WAY: C++26 Reflection ==========
struct Person {
    std::string name;
    int age;
    double height;
};
// NOTHING ELSE NEEDED. No macros, no registration.

template <typename T>
void print_all_fields(const T& obj) {
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        std::cout << std::meta::identifier_of(m) << " = ";
        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            std::cout << '"' << obj.[:m:] << '"';
        else
            std::cout << obj.[:m:];
        std::cout << '\n';
    }
}

int main() {
    Person p{"Alice", 30, 5.6};
    print_all_fields(p);  // Works for ANY struct, zero registration
    // Output:
    // name = "Alice"
    // age = 30
    // height = 5.6

    // Add a new field to Person -> print_all_fields automatically picks it up.
    // With X-macros or BOOST_DESCRIBE: must also update the macro/registration.
}
```

The practical difference shows up when you add a field. With X-macros you update both the struct definition and the macro list - and if you forget the macro list, nothing warns you. With `BOOST_DESCRIBE` you update the struct and the `BOOST_DESCRIBE_STRUCT` call. With reflection you update only the struct, and every utility that uses it automatically includes the new field.

---

## Notes

- `meta::offset_of(m)` returns the byte offset of a member within its containing struct - the same value `offsetof` gives, but for any member via reflection.
- `meta::size_of(info)` returns the size in bytes of a reflected type - equivalent to `sizeof` but applicable to a `meta::info` handle.
- Reflection serialization is inherently type-safe - accessing a member through `obj.[:m:]` and splicing its type with `[:type_of(m):]` are both fully type-checked at compile time.
- For binary serialization, always consider endianness and alignment - `memcpy` preserves the in-memory layout, which may not be portable across platforms with different byte orders.
- Nested struct serialization works naturally by making the serializer template recursive - the same `to_json` or `serialize` function handles both top-level and nested structs.
