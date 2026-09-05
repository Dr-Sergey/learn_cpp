# Use reflection to implement automatic JSON serialization

**Category:** Reflection Cpp26  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

Reflection enables writing a single `to_json()` function that works for **any** aggregate type, handling nested structs recursively - with zero boilerplate per type.

```cpp
struct Person {            to_json(person)
  string name;       ->    {"name": "Alice", "age": 30,
  int age;                  "addr": {"city": "NYC", "zip": 10001}}
  Address addr;
};
// Zero per-type code needed!
```

Before reflection, JSON serialization in C++ meant choosing between writing a `toJson()` method on every struct, registering each field with a macro like `NLOHMANN_DEFINE_TYPE_INTRUSIVE`, or maintaining a separate mapping file. All of those require you to touch the serialization layer every time you change a struct. With reflection, the serializer iterates the struct's fields at compile time - so adding a new field to `Person` automatically makes it appear in the JSON output, with no other changes needed.

---

## Self-Assessment

### Q1: Generic `serialize(T)` that emits JSON key/value pairs

The function below uses `nonstatic_data_members_of` to get every field of `T`, then uses `identifier_of` for the JSON key name and `obj.[:m:]` to access the runtime value. The `constexpr if` chain handles each type category - string, bool, arithmetic, and nested struct. Notice the recursive call for nested structs: reflection naturally handles arbitrary depth.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string>
#include <sstream>
#include <type_traits>
#include <vector>

// Forward declaration for recursive use:
template <typename T>
std::string to_json(const T& obj);

// Helper: is_aggregate concept for recursion detection
template <typename T>
constexpr bool is_reflectable_struct =
    std::is_class_v<T> && !std::is_same_v<T, std::string>;

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
        } else if constexpr (is_reflectable_struct<MType>) {
            oss << to_json(obj.[:m:]);  // recurse!
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
    // {"name": "Alice", "age": 30, "addr": {"city": "NYC", "zip": 10001}}

    Address a{"London", 12345};
    std::cout << to_json(a) << '\n';
    // {"city": "London", "zip": 12345}
}
```

The recursion through `is_reflectable_struct` is what makes deep nesting work automatically. When `to_json` encounters `addr`, it checks that `Address` is a class that is not `std::string`, and calls `to_json` again on it. The compiler resolves this recursion at compile time, so there is no runtime overhead from the nesting.

### Q2: Handle nested structs recursively with `constexpr if`

This version factors the type dispatch into a separate `to_json_value` helper, which makes the main `to_json` loop cleaner and makes it easier to extend later. The multi-level trip example shows three levels of nesting working transparently.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>
#include <sstream>
#include <vector>

// Extended serializer with array support:
template <typename T> std::string to_json(const T&);

template <typename T>
std::string to_json_value(const T& val) {
    if constexpr (std::is_same_v<T, std::string>) {
        return '"' + val + '"';
    } else if constexpr (std::is_same_v<T, bool>) {
        return val ? "true" : "false";
    } else if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(val);
    } else if constexpr (requires { std::meta::nonstatic_data_members_of(^T); }) {
        return to_json(val);  // nested struct: recurse
    } else {
        return "null";
    }
}

template <typename T>
std::string to_json(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << "{";
    bool first = true;
    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;
        oss << '"' << std::meta::identifier_of(m) << "\": "
            << to_json_value(obj.[:m:]);
    }
    oss << "}";
    return oss.str();
}

// Multi-level nesting:
struct GPS { double lat; double lon; };
struct Location { std::string name; GPS coords; };
struct Trip { std::string title; Location from; Location to; int km; };

int main() {
    Trip t{
        "Road Trip",
        {"Paris", {48.8566, 2.3522}},
        {"Berlin", {52.5200, 13.4050}},
        1050
    };
    std::cout << to_json(t) << '\n';
    // {"title": "Road Trip",
    //  "from": {"name": "Paris", "coords": {"lat": 48.856600, "lon": 2.352200}},
    //  "to": {"name": "Berlin", "coords": {"lat": 52.520000, "lon": 13.405000}},
    //  "km": 1050}
}
```

The `requires { std::meta::nonstatic_data_members_of(^T); }` check in `to_json_value` is what decides whether to recurse - if the type has reflectable members, recurse into it; otherwise fall back to `"null"`. That single check handles arbitrarily deep struct nesting without any per-type registration.

### Q3: Reflection eliminates manual `REFLECT_FIELD` macros

This comparison makes the before/after contrast concrete. The old macro approach required every field to be explicitly listed - which was easy to get wrong, painful to maintain, and opaque to IDEs and static analyzers.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>

// ===== OLD WAY: REFLECT_FIELD macros =====
// #define REFLECT_FIELD(type, name) type name;
// #define REFLECT_FIELDS(cls, ...) \
//     struct cls { __VA_ARGS__ };
//     void serialize(const cls& obj, ostream& os) { ... }
//
// Usage:
// REFLECT_FIELDS(Person,
//     REFLECT_FIELD(std::string, name)
//     REFLECT_FIELD(int, age)
// )
//
// Problems:
// 1. Macro magic: hard to debug, no IDE support
// 2. Must register EVERY field explicitly
// 3. Adding a field requires updating macro invocation
// 4. No type safety in macro expansion
// 5. Cannot handle nested types cleanly

// ===== NEW WAY: C++26 Reflection =====
struct Person {
    std::string name;
    int age;
    double height;
    // Just add fields. That's it.
};

template <typename T>
void serialize(const T& obj, std::ostream& os) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    os << "{";
    bool first = true;
    template for (constexpr auto m : members) {
        if (!first) os << ", ";
        first = false;
        os << '"' << std::meta::identifier_of(m) << "\": ";
        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            os << '"' << obj.[:m:] << '"';
        else
            os << obj.[:m:];
    }
    os << "}";
}

int main() {
    Person p{"Bob", 25, 5.11};
    serialize(p, std::cout);  // {"name": "Bob", "age": 25, "height": 5.11}
    std::cout << '\n';

    // Adding a new field `bool active` to Person:
    // - Old way: update REFLECT_FIELDS macro call (easy to forget)
    // - New way: just add the field. Serialization auto-updates.

    // No macros. No registration. No boilerplate.
}
```

The comments in `main` capture the practical difference well. With the macro approach, adding a field to a struct meant hunting down every registration site across the codebase and updating it. With reflection, the struct definition *is* the registration - there is no separate step that can get out of sync.

---

## Notes

- Use `constexpr if` for type dispatch to handle strings, bools, arithmetic, and nested structs correctly.
- Recursive `to_json` naturally handles arbitrarily deep nesting without any extra per-type setup.
- The generic serializer is around 20 lines and works for every aggregate type in your codebase.
- This approach replaces `REFLECT_FIELD`, `BOOST_DESCRIBE`, and `nlohmann::NLOHMANN_DEFINE_TYPE_INTRUSIVE` with zero per-type code.
- For production use, you will want to add array/vector support and null handling as extensions to the same pattern.
