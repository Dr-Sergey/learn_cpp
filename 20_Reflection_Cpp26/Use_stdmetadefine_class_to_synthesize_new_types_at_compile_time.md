# Use std::meta::define_aggregate to synthesize new types at compile time

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2996>  

---

## Topic Overview

`std::meta::define_aggregate` creates entirely new types at compile time from a compile-time description. This is the most powerful reflection feature - it generates types that do not exist anywhere in your source code, determined entirely by compile-time logic.

```cpp
Compile-time description             Generated type

fields = [{"x", ^^int},        ->    struct Generated {
          {"y", ^^int},                   int x;
          {"name", ^^string}]             int y;
                                         string name;
define_aggregate(^^S, fields)              };
```

This is a fundamentally different capability from everything else in C++. Templates let you select from types that already exist. `constexpr` computation lets you compute values at compile time. `define_aggregate` lets you create entirely new types at compile time - types whose fields are determined by logic, not by anything written in your source. The most direct use cases are ORMs (where a database schema drives the C++ type), protocol buffer generation (where a protocol spec drives the packet layout), and configuration structs (where the fields come from an external source of truth).

---

## Self-Assessment

### Q1: Generate a struct with fields from a compile-time description

The example declares `Point3D` without defining it, then lets a `consteval` block - code that runs entirely at compile time - supply the definition. `data_member_spec` describes one field, and `define_aggregate` assembles them into a real C++ struct. After the block, `Point3D` is a complete type that you use just like any other struct.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 reflection
#include <meta>
#include <iostream>

// define_aggregate COMPLETES a class that has been declared but not defined.
struct Point3D;   // incomplete on purpose

// It has to run inside a consteval block: that is the one place where a
// constant evaluation is allowed to inject a definition into the program.
consteval {
    std::meta::define_aggregate(^^Point3D, {
        std::meta::data_member_spec(^^float, {.name = "x"}),
        std::meta::data_member_spec(^^float, {.name = "y"}),
        std::meta::data_member_spec(^^float, {.name = "z"}),
    });
}
// From here on Point3D is complete, exactly as if we had written
//   struct Point3D { float x; float y; float z; };

int main() {
    Point3D p{};
    p.x = 1.0f;
    p.y = 2.0f;
    p.z = 3.0f;

    std::cout << p.x << ", " << p.y << ", " << p.z << '\n';
    // Output: 1, 2, 3

    // Verify the type has the expected layout:
    static_assert(sizeof(Point3D) == 3 * sizeof(float));

    static constexpr auto members = std::define_static_array(
        std::meta::nonstatic_data_members_of(^^Point3D, std::meta::access_context::current()));
    static_assert(members.size() == 3);
}
```

Once the type is created, it is a fully ordinary C++ struct. You can create instances of it, access its fields by name, take `sizeof` it, and even reflect back over it with `nonstatic_data_members_of`. The compile-time origin does not impose any restrictions on how you use it at runtime.

### Q2: Generate a POD layout struct from a protocol specification

This example is closer to a real use case: a network protocol is described as a compile-time list of field names and types, and a `consteval` block turns that description into the definition of `Packet`. The resulting `Packet` type has exactly the fields the spec calls for, with the correct types and names.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <cstdint>
#include <string_view>
#include <vector>

// One field of a protocol specification:
struct FieldDesc {
    std::string_view name;
    std::meta::info type;
};

struct Packet;   // declared here, completed from the specification below

consteval {
    // The specification is ordinary compile-time data...
    const FieldDesc spec[] = {
        {"header",     ^^std::uint32_t},
        {"sequence",   ^^std::uint16_t},
        {"flags",      ^^std::uint16_t},
        {"payload_sz", ^^std::uint32_t},
        {"checksum",   ^^std::uint32_t},
    };

    // ...and ordinary code turns it into member descriptions:
    std::vector<std::meta::info> members;
    for (const FieldDesc& f : spec) {
        members.push_back(std::meta::data_member_spec(f.type, {.name = f.name}));
    }
    std::meta::define_aggregate(^^Packet, members);
}

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
    template for (constexpr auto m : std::define_static_array(
            std::meta::nonstatic_data_members_of(^^Packet, std::meta::access_context::current()))) {
        std::cout << std::meta::identifier_of(m) << " = "
                  << pkt.[:m:] << '\n';
    }
}
```

The nice part here is that you can then use `nonstatic_data_members_of` on the generated type in the same loop that prints all the fields. The synthesized type is fully reflectable, so all the generic tools you have built with reflection work on it just as well as on hand-written structs.

### Q3: `define_aggregate` vs `constexpr` class generation

This example is important for understanding exactly what `define_aggregate` adds that did not exist before. The reason this trips people up is that C++ already had `constexpr` computation and template metaprogramming, so it is not immediately obvious what was missing. The key distinction is the difference between computing *values* at compile time and creating *types* at compile time.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>
#include <type_traits>

// ===== constexpr class generation (existing C++) =====
// You can compute VALUES at compile time:
constexpr int compute() { return 42; }
// But you CANNOT create new TYPES at compile time:
// constexpr struct ??? = make_struct(); // IMPOSSIBLE in C++23

// You can use template metaprogramming to select types:
template <bool B>
using Select = std::conditional_t<B, int, double>;
// But the types must already exist.

// ===== define_aggregate (C++26 reflection) =====
// Gives a body to a class whose members were never written in source code.

template <typename First, typename Second>
struct Pair;   // primary template: declared, deliberately never defined

// A reusable recipe: complete one specialization of Pair.
consteval void define_pair(std::meta::info pair,
                           std::meta::info first_type,
                           std::meta::info second_type) {
    std::meta::define_aggregate(pair, {
        std::meta::data_member_spec(first_type,  {.name = "first"}),
        std::meta::data_member_spec(second_type, {.name = "second"}),
    });
}

// Run the recipe for the specializations we need:
consteval {
    define_pair(^^Pair<int, std::string>, ^^int, ^^std::string);
    define_pair(^^Pair<double, double>, ^^double, ^^double);
}

using IntStrPair = Pair<int, std::string>;
using DblDblPair = Pair<double, double>;

// Key differences:
// 1. constexpr: computes values at compile time
//    define_aggregate: creates TYPE DEFINITIONS at compile time
//
// 2. Templates: stamp out one fixed shape for different types
//    define_aggregate: the shape itself (which members, how many) is computed
//
// 3. constexpr struct: must be fully written in source
//    define_aggregate: fields determined by compile-time logic

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
    static_assert(std::meta::nonstatic_data_members_of(
        ^^IntStrPair, std::meta::access_context::current()).size() == 2);
}
```

Think of it this way: before `define_aggregate`, compile-time programming could only influence what happened to existing types. With `define_aggregate`, it can *create* types. That gap is what opens up the ORM, protocol buffer, and config-struct use cases - because in those scenarios, you do not know the types ahead of time. The types should come from the schema or spec, not from what a programmer wrote on a particular day.

---

## Notes

- `define_aggregate` takes a reflection of an incomplete class and a range of `data_member_spec` values, one per field you want to add, and completes that class.
- It must be evaluated from a `consteval { }` block, the only context allowed to inject a definition. Declare the class first (`struct Packet;`), then define it from the block.
- `data_member_spec(type_info, {.name = "field_name"})` describes a single data member with the given type and name.
- The generated type is a fully real C++ type - you can use it anywhere: local variables, function parameters, containers, even further reflection.
- Primary use cases are ORM mapping (database schema drives the C++ struct), protocol buffer generation (wire format spec drives the packet type), and configuration structs (external source of truth drives the fields).
- Earlier drafts called this function `define_class`; it was renamed because it can only produce aggregates (public data members, no member functions).
- `define_aggregate` can only add data members to the generated type - you cannot inject member functions through it.
