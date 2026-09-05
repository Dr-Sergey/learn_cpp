# Understand the splice operator [: :] for injecting reflections back into code

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

The splice operator `[:r:]` takes a `std::meta::info` value and injects the entity it represents directly into code. It is the complement of the reflect operator `^`. If `^` takes you from code into the meta domain, `[::]` takes you back out from the meta domain into code - as a type, a value, or a member access depending on where you put it.

Here is a reference for the different positions where you can use the splice:

| Context | Splice form | Result |
| --- | --- | --- |
| Type position | `using T = [:info:];` | Declares `T` as the reflected type |
| Member access | `obj.[:member_info:]` | Accesses the reflected member |
| Assignment | `obj.[:m:] = val;` | Writes to the reflected member |
| Function call | `[:func_info:](args)` | Calls the reflected function |
| Template arg | `vector<[:info:]>` | Uses reflected type as template parameter |

---

## Self-Assessment

### Q1: Splice a reflected type into a type position

The most fundamental use of `[::]` is recovering a type from a reflection. You write `using T = [:r:]` and the compiler substitutes the actual type wherever `T` appears - including inside template arguments for containers.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <vector>
#include <type_traits>

int main() {
    // Reflect a type:
    constexpr auto int_r = ^int;
    constexpr auto dbl_r = ^double;
    constexpr auto str_r = ^std::string;

    // Splice into a type alias:
    using T1 = [:int_r:];      // T1 is int
    using T2 = [:dbl_r:];      // T2 is double
    using T3 = [:str_r:];      // T3 is std::string

    static_assert(std::is_same_v<T1, int>);
    static_assert(std::is_same_v<T2, double>);
    static_assert(std::is_same_v<T3, std::string>);

    // Splice in template arguments:
    std::vector<[:int_r:]> v = {1, 2, 3};     // vector<int>

    // Splice member types:
    struct Point { int x; double y; };
    constexpr auto members = std::meta::nonstatic_data_members_of(^Point);
    using FirstField = [:std::meta::type_of(members[0]):];
    static_assert(std::is_same_v<FirstField, int>);

    T1 a = 42;
    T2 b = 3.14;
    T3 c = "hello";
    std::cout << a << ", " << b << ", " << c << '\n';
    // Output: 42, 3.14, hello
}
```

Notice the `using FirstField = [:std::meta::type_of(members[0]):]` line - you can nest a query call directly inside the splice. This lets you programmatically extract the type of any member and use it as a first-class type alias.

### Q2: Splice a member reflection to access and modify fields

Member splicing is where things get interesting for generic programming. `obj.[:m:]` is literally the same as writing `obj.fieldname` - the compiler substitutes the actual member name at compile time. You can read through it, write through it, take a reference to it - anything you could do with a normal member access.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>

struct Player {
    std::string name;
    int hp;
    int level;
    double xp;
};

// Generic field setter using splice:
template <typename T>
void set_field(T& obj, std::string_view field_name, auto value) {
    template for (constexpr auto m : std::meta::nonstatic_data_members_of(^T)) {
        if (std::meta::identifier_of(m) == field_name) {
            using MType = [:std::meta::type_of(m):];
            if constexpr (std::is_convertible_v<decltype(value), MType>) {
                obj.[:m:] = static_cast<MType>(value);  // splice + assign!
            }
            return;
        }
    }
}

// Generic field printer:
template <typename T>
void print_fields(const T& obj) {
    template for (constexpr auto m : std::meta::nonstatic_data_members_of(^T)) {
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
    Player p{"Hero", 100, 1, 0.0};

    // Read via splice:
    print_fields(p);
    // name = "Hero"
    // hp = 100
    // level = 1
    // xp = 0

    // Write via splice:
    set_field(p, "hp", 200);
    set_field(p, "level", 10);
    set_field(p, "xp", 5432.5);

    std::cout << "\nAfter update:\n";
    print_fields(p);
    // hp = 200
    // level = 10
    // xp = 5432.5
}
```

The `set_field` function is worth pausing on. The `template for` loop expands at compile time, but the name comparison `identifier_of(m) == field_name` runs at runtime inside the expanded body. The `if constexpr` is needed because each member has a different type - without it the compiler would try to assign every type of value to every type of field, which would fail for incompatible pairs. This compile-time loop / runtime comparison combination is a common pattern when you need to bridge reflected structure to dynamic behavior.

### Q3: Generic code using reflected entities as first-class values

Here are two utilities - `transform_fields` and `struct_equal` - that show what becomes possible when you treat member reflections as first-class values you can store in arrays and index into.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <tuple>
#include <functional>

// Compile-time map: transform every field of a struct
template <typename T, typename F>
auto transform_fields(const T& obj, F&& func) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    return [&]<std::size_t... Is>(std::index_sequence<Is...>) {
        return std::make_tuple(func(obj.[:members[Is]:])...);
    }(std::make_index_sequence<members.size()>{});
}

// Compile-time struct equality (generic, no operator== needed):
template <typename T>
bool struct_equal(const T& a, const T& b) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    bool result = true;
    template for (constexpr auto m : members) {
        if (a.[:m:] != b.[:m:]) result = false;
    }
    return result;
}

struct Vec3 { float x, y, z; };

int main() {
    Vec3 v{1.0f, 2.0f, 3.0f};

    // Transform: double every field
    auto doubled = transform_fields(v, [](auto val) { return val * 2; });
    // doubled is tuple<float, float, float>{2.0, 4.0, 6.0}
    std::cout << std::get<0>(doubled) << ", "
              << std::get<1>(doubled) << ", "
              << std::get<2>(doubled) << '\n';
    // Output: 2, 4, 6

    // Generic equality:
    Vec3 a{1.0f, 2.0f, 3.0f};
    Vec3 b{1.0f, 2.0f, 3.0f};
    Vec3 c{1.0f, 0.0f, 3.0f};
    std::cout << struct_equal(a, b) << '\n';  // 1 (true)
    std::cout << struct_equal(a, c) << '\n';  // 0 (false)
}
```

The `struct_equal` function is especially satisfying: you get a correct field-by-field equality comparison for any aggregate type, with no `operator==` boilerplate, and it is fully type-safe. The reason this is better than a manual implementation is that it cannot go out of sync with the struct - if you add a field to `Vec3`, `struct_equal` automatically checks that field too.

---

## Notes

- `[:r:]` requires `r` to be a constant expression of type `std::meta::info`.
- Splice in type position: `using T = [:info:]` or `vector<[:info:]>`.
- Splice in member position: `obj.[:member_info:]` for both read and write.
- Splice in function position: `[:func_info:](args...)` calls the function.
- `^` and `[::]` are inverses: `[:^T:]` yields `T`, `^[:info:]` yields back `info`.
