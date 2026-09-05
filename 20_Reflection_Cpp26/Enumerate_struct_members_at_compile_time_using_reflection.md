# Enumerate struct members at compile time using reflection

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

C++26 reflection lets you iterate over all data members of a struct at compile time using `std::meta::nonstatic_data_members_of(^S)`. This is a big deal because it means you can write generic functions - serializers, converters, printers - that work for any struct without you manually listing the fields or using macros.

Here is a quick reference to the APIs you will use most often when working with struct members:

| API | Purpose |
| --- | --- |
| `^MyStruct` | Reflect the struct type into a `meta::info` handle |
| `nonstatic_data_members_of(^S)` | Get all non-static data members |
| `members_of(^S)` | Get ALL members (including static, functions) |
| `identifier_of(m)` | Get member name as a string |
| `type_of(m)` | Get member type as `meta::info` |
| `obj.[:m:]` | Access the member on a concrete object via splice |

The distinction between `nonstatic_data_members_of` and `members_of` matters: if you want to visit only the instance fields (the data you would serialize or compare), you almost always want `nonstatic_data_members_of`.

---

## Self-Assessment

### Q1: Iterate non-static data members with `members_of`

Here you can see the two main things you do with a reflected member: read its name with `identifier_of`, and access its value on a concrete object with `obj.[:m:]`. The splice syntax `p.[:m:]` is the key - it turns a compile-time `meta::info` handle into an actual member access at that point in the code.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

struct Point {
    double x;
    double y;
    double z;
};

int main() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^Point);

    std::cout << "Point has " << members.size() << " members:\n";
    template for (constexpr auto m : members) {
        std::cout << "  " << std::meta::identifier_of(m)
                  << " : " << std::meta::display_string_of(std::meta::type_of(m))
                  << '\n';
    }

    // Access members on an object:
    Point p{1.0, 2.0, 3.0};
    template for (constexpr auto m : members) {
        std::cout << std::meta::identifier_of(m)
                  << " = " << p.[:m:] << '\n';
    }
}
// Output:
// Point has 3 members:
//   x : double
//   y : double
//   z : double
// x = 1
// y = 2
// z = 3
```

Notice that you get the type names printed too via `display_string_of(type_of(m))` - handy for diagnostics and introspection tools. The second loop shows `p.[:m:]` in action: for each compile-time member handle `m`, the compiler fills in the actual field access at that point.

### Q2: Generic `to_tuple` using reflection

This one is genuinely clever and worth reading carefully. The goal is to pack all fields of any struct into a `std::tuple` without the caller listing them. The trick is using a lambda with an index sequence to build `make_tuple(obj.[:members[Is]:]...)` as a pack expansion - one tuple element per member index.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <tuple>
#include <iostream>
#include <string>

struct Person {
    std::string name;
    int age;
    double height;
};

// Generic to_tuple for ANY aggregate:
template <typename T>
auto to_tuple(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);

    // Build the tuple by splicing each member access:
    return [&]<std::size_t... Is>(std::index_sequence<Is...>) {
        return std::make_tuple(obj.[:members[Is]:]...);
    }(std::make_index_sequence<members.size()>{});
}

int main() {
    Person p{"Alice", 30, 5.6};
    auto t = to_tuple(p);

    std::cout << std::get<0>(t) << ", "
              << std::get<1>(t) << ", "
              << std::get<2>(t) << '\n';
    // Output: Alice, 30, 5.6

    // Works for ANY struct:
    struct Vec3 { float x, y, z; };
    Vec3 v{1.0f, 2.0f, 3.0f};
    auto vt = to_tuple(v);
    // vt is tuple<float, float, float>{1.0f, 2.0f, 3.0f}
}
```

The reason this pattern uses index sequences rather than `template for` directly is that `make_tuple` needs a pack expansion to build the tuple in one shot. The index sequence lambda gives you `Is...` as a parameter pack so you can write `obj.[:members[Is]:]...` as a single variadic expression.

### Q3: Compile-time field counter without macros

One of the simpler but most immediately useful things you can do: get the number of fields in a struct as a `constexpr` value. This lets you write constraints and `static_assert`s that depend on struct shape - for example, a template that only accepts structs with eight fields or fewer.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>

struct Empty {};
struct One { int a; };
struct Three { int a; double b; char c; };
struct Complex {
    std::string name;
    int value;
    std::vector<int> data;
    double factor;
};

// Generic field counter:
template <typename T>
constexpr std::size_t field_count() {
    return std::meta::nonstatic_data_members_of(^T).size();
}

static_assert(field_count<Empty>() == 0);
static_assert(field_count<One>() == 1);
static_assert(field_count<Three>() == 3);
static_assert(field_count<Complex>() == 4);

// Practical use: compile-time validation
template <typename T>
    requires (field_count<T>() <= 8)
struct SmallStruct {
    T data;
};

// SmallStruct<Complex> ok;       // 4 fields, OK
// SmallStruct<HugeStruct> fail;  // >8 fields, compile error

int main() {
    std::cout << "Empty: " << field_count<Empty>() << '\n';    // 0
    std::cout << "Three: " << field_count<Three>() << '\n';    // 3
    std::cout << "Complex: " << field_count<Complex>() << '\n'; // 4
}
```

The `requires` clause on `SmallStruct` shows the practical payoff: you can gate template instantiation on structural properties of a type. Before reflection, getting field counts required hand-counted constants or fragile aggregate-initialization tricks. Now it is a one-liner.

---

## Notes

- `nonstatic_data_members_of` returns only instance fields - no static members, no member functions, no nested types.
- `members_of` returns everything including static members, member functions, and nested type declarations - use it when you need a broader picture.
- `template for` is the C++26 expansion statement for iterating compile-time sequences; it is not a runtime loop, and each iteration is a separate instantiation.
- Reflection replaces the C++17 structured bindings hacks that people used to count fields by testing which aggregate initializer arities compiled.
- All reflection queries are `consteval` - the introspection is a compile-time-only operation with zero runtime overhead.
