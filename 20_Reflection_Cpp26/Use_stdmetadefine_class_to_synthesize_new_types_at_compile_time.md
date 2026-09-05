# Use std::meta::define_class to synthesize new types at compile time

**Category:** Reflection Cpp26  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

`std::meta::define_class` creates entirely new types at compile time from a compile-time description. This is the most powerful reflection feature - it generates types that do not exist anywhere in your source code, determined entirely by compile-time logic.

```cpp
Compile-time description             Generated type

fields = [{"x", ^int},        ->    struct Generated {
          {"y", ^int},                   int x;
          {"name", ^string}]             int y;
                                         string name;
define_class(^S, fields)              };
```

This is a fundamentally different capability from everything else in C++. Templates let you select from types that already exist. `constexpr` computation lets you compute values at compile time. `define_class` lets you create entirely new types at compile time - types whose fields are determined by logic, not by anything written in your source. The most direct use cases are ORMs (where a database schema drives the C++ type), protocol buffer generation (where a protocol spec drives the packet layout), and configuration structs (where the fields come from an external source of truth).

---

## Self-Assessment

### Q1: Generate a struct with fields from a compile-time description

The `make_point_type` function below is a `consteval` function - it runs entirely at compile time - and its job is to build the description of a new type. `data_member_spec` describes one field, and `define_class` assembles them into a real C++ struct. The `using Point3D = [:make_point_type():]` line then splices the resulting type into the normal type system so you can use it just like any other struct.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string>
#include <vector>

// define_class creates a new type from a list of member descriptions:
consteval std::meta::info make_point_type() {
    // Start with an incomplete class:
    // define_class takes a reflection of an incomplete class and
    // a vector of data_member_spec:
    return std::meta::define_class(^struct Point3D, {
        std::meta::data_member_spec(^float, {.name = "x"}),
        std::meta::data_member_spec(^float, {.name = "y"}),
        std::meta::data_member_spec(^float, {.name = "z"}),
    });
}

// The type is created at compile time:
using Point3D = [:make_point_type():];

int main() {
    Point3D p{};
    p.x = 1.0f;
    p.y = 2.0f;
    p.z = 3.0f;

    std::cout << p.x << ", " << p.y << ", " << p.z << '\n';
    // Output: 1, 2, 3

    // Verify the type has the expected layout:
    static_assert(sizeof(Point3D) == 3 * sizeof(float));

    constexpr auto members = std::meta::nonstatic_data_members_of(^Point3D);
    static_assert(members.size() == 3);
}
```

Once the type is created, it is a fully ordinary C++ struct. You can create instances of it, access its fields by name, take `sizeof` it, and even reflect back over it with `nonstatic_data_members_of`. The compile-time origin does not impose any restrictions on how you use it at runtime.

### Q2: Generate a POD layout struct from a protocol specification

This example is closer to a real use case: a network protocol is described as a compile-time list of field names and types, and `make_packet_type` turns that description into a concrete struct. The resulting `Packet` type has exactly the fields the spec calls for, with the correct types and names.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <cstdint>
#include <string_view>
#include <vector>

// Protocol specification as compile-time data:
struct FieldDesc {
    std::string_view name;
    std::meta::info type;
};

// Build a struct from a protocol spec:
consteval std::meta::info make_packet_type(
    std::vector<FieldDesc> fields) {
    std::vector<std::meta::info> member_specs;
    for (auto& f : fields) {
        member_specs.push_back(
            std::meta::data_member_spec(f.type, {.name = f.name}));
    }
    return std::meta::define_class(^struct Packet, member_specs);
}

// Define a network packet from specification:
constexpr auto packet_spec = std::vector<FieldDesc>{
    {"header",    ^uint32_t},
    {"sequence",  ^uint16_t},
    {"flags",     ^uint16_t},
    {"payload_sz",^uint32_t},
    {"checksum",  ^uint32_t},
};

using Packet = [:make_packet_type(packet_spec):];

int main() {
    Packet pkt{};
    pkt.header = 0xDEADBEEF;
    pkt.sequence = 42;
    pkt.flags = 0x01;
    pkt.payload_sz = 1024;
    pkt.checksum = 0xABCD;

    std::cout << "Header: 0x" << std::hex << pkt.header << '\n';
    std::cout << "Seq: " << std::dec << pkt.sequence << '\n';
    std::cout << "Size: " << sizeof(Packet) << " bytes\n";

    // Print all fields generically:
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^Packet)) {
        std::cout << std::meta::identifier_of(m) << " = "
                  << pkt.[:m:] << '\n';
    }
}
```

The nice part here is that you can then use `nonstatic_data_members_of` on the generated type in the same loop that prints all the fields. The synthesized type is fully reflectable, so all the generic tools you have built with reflection work on it just as well as on hand-written structs.

### Q3: `define_class` vs `constexpr` class generation

This example is important for understanding exactly what `define_class` adds that did not exist before. The reason this trips people up is that C++ already had `constexpr` computation and template metaprogramming, so it is not immediately obvious what was missing. The key distinction is the difference between computing *values* at compile time and creating *types* at compile time.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>

// ===== constexpr class generation (existing C++) =====
// You can compute VALUES at compile time:
constexpr int compute() { return 42; }
// But you CANNOT create new TYPES at compile time:
// constexpr struct ??? = make_struct(); // IMPOSSIBLE in C++23

// You can use template metaprogramming to select types:
template <bool B>
using Select = std::conditional_t<B, int, double>;
// But the types must already exist.

// ===== define_class (C++26 reflection) =====
// Creates types that DON'T EXIST in source code:

consteval std::meta::info make_pair_type(
    std::meta::info first_type,
    std::meta::info second_type) {
    return std::meta::define_class(^struct Pair, {
        std::meta::data_member_spec(first_type, {.name = "first"}),
        std::meta::data_member_spec(second_type, {.name = "second"}),
    });
}

// Generate a type based on compile-time logic:
using IntStrPair = [:make_pair_type(^int, ^std::string):];
using DblDblPair = [:make_pair_type(^double, ^double):];

// Key differences:
// 1. constexpr: computes values at compile time
//    define_class: creates TYPES at compile time
//
// 2. Templates: select from existing types
//    define_class: generate entirely new types
//
// 3. constexpr struct: must be fully written in source
//    define_class: fields determined by compile-time logic

int main() {
    IntStrPair p1;
    p1.first = 42;
    p1.second = "hello";
    std::cout << p1.first << ", " << p1.second << '\n';
    // Output: 42, hello

    DblDblPair p2{3.14, 2.72};
    std::cout << p2.first << ", " << p2.second << '\n';
    // Output: 3.14, 2.72

    // The struct layout was computed, not written:
    static_assert(std::meta::nonstatic_data_members_of(^IntStrPair).size() == 2);
}
```

Think of it this way: before `define_class`, compile-time programming could only influence what happened to existing types. With `define_class`, it can *create* types. That gap is what opens up the ORM, protocol buffer, and config-struct use cases - because in those scenarios, you do not know the types ahead of time. The types should come from the schema or spec, not from what a programmer wrote on a particular day.

---

## Notes

- `define_class` takes a reflection of an incomplete class and a vector of `data_member_spec` values, one per field you want to add.
- `data_member_spec(type_info, {.name = "field_name"})` describes a single data member with the given type and name.
- The generated type is a fully real C++ type - you can use it anywhere: local variables, function parameters, containers, even further reflection.
- Primary use cases are ORM mapping (database schema drives the C++ struct), protocol buffer generation (wire format spec drives the packet type), and configuration structs (external source of truth drives the fields).
- This is the most advanced P2996 feature and some compilers may not support it in their initial C++26 releases.
- `define_class` can only add data members to the generated type - you cannot inject member functions through it.
