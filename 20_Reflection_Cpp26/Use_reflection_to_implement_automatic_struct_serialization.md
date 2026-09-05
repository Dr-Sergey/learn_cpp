# Use reflection to implement automatic struct serialization

**Category:** Reflection Cpp26  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

Reflection makes struct serialization fully automatic. Adding or removing a field from a struct automatically updates ALL serialization and deserialization code - with no changes to any serialization logic required.

The reason this is such a big deal in practice is the maintenance cost of the old approach. Every time you change a struct, you have to find and update the serializer, the deserializer, and anything else that enumerates fields. It is easy to miss one, especially in large codebases. Reflection eliminates the whole category of "forgot to update the serializer" bugs.

| Feature | Macro-based | Reflection |
| --- | --- | --- |
| Per-type registration | Required | None |
| New field handling | Manual update | Automatic |
| Nested types | Complex macros | Simple recursion |
| Type safety | Limited | Full |
| IDE support | Poor (macros) | Full |

---

## Self-Assessment

### Q1: Iterate fields with `members_of` and generate a serializer

The `serialize` function below uses `nonstatic_data_members_of` to walk every field of `T` at compile time. For each member, `identifier_of` gives you the field name as a string, and `obj.[:m:]` gives you the runtime value. The `constexpr if` handles the string case to add quotes; everything else is printed directly.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string>
#include <sstream>

template <typename T>
std::string serialize(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << std::meta::identifier_of(^T) << "{";
    bool first = true;

    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;
        oss << std::meta::identifier_of(m) << "=";

        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            oss << '"' << obj.[:m:] << '"';
        else if constexpr (std::is_same_v<MType, bool>)
            oss << (obj.[:m:] ? "true" : "false");
        else
            oss << obj.[:m:];
    }
    oss << "}";
    return oss.str();
}

struct Config {
    std::string host;
    int port;
    bool debug;
    double timeout;
};

int main() {
    Config cfg{"localhost", 8080, true, 30.0};
    std::cout << serialize(cfg) << '\n';
    // Config{host="localhost", port=8080, debug=true, timeout=30}

    struct Vec3 { float x, y, z; };
    Vec3 v{1.0f, 2.0f, 3.0f};
    std::cout << serialize(v) << '\n';
    // Vec3{x=1, y=2, z=3}
}
```

Notice that `std::meta::identifier_of(^T)` at the top of the output loop gives you the struct's own name - so the output says `Config{...}` rather than just `{...}`. That is reflection on the type itself, not just its members.

### Q2: Adding a new field automatically updates serialization

This is the key proof of the "zero maintenance" claim. `UserV2` adds two fields compared to `UserV1`, and the `serialize` function handles both without any modification.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>
#include <sstream>

template <typename T>
std::string serialize(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << "{";
    bool first = true;
    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;
        oss << '"' << std::meta::identifier_of(m) << "\": ";
        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            oss << '"' << obj.[:m:] << '"';
        else
            oss << obj.[:m:];
    }
    oss << "}";
    return oss.str();
}

// Version 1:
struct UserV1 {
    std::string name;
    int age;
};

// Version 2 (added email and active):
struct UserV2 {
    std::string name;
    int age;
    std::string email;   // NEW
    bool active;          // NEW
};

int main() {
    UserV1 u1{"Alice", 30};
    std::cout << serialize(u1) << '\n';
    // {"name": "Alice", "age": 30}

    UserV2 u2{"Bob", 25, "bob@example.com", true};
    std::cout << serialize(u2) << '\n';
    // {"name": "Bob", "age": 25, "email": "bob@example.com", "active": 1}

    // The serialize() function is EXACTLY THE SAME for both.
    // Adding `email` and `active` requires ZERO changes to serialization code.

    // With manual approach, you'd need to update:
    // - toJson() method
    // - fromJson() method
    // - operator== (if applicable)
    // - hash function (if applicable)
    // - print/debug functions
    // = 5+ places to update per field change!
}
```

The comment about "5+ places to update" is not an exaggeration for a real codebase. Each of those update sites is an opportunity to introduce a bug - either by forgetting to update one, or by updating them inconsistently. Reflection collapses all of those into a single source of truth: the struct definition itself.

### Q3: Reflection vs macro-based approaches (Boost.Hana, REFLECT)

There have been several prior attempts to solve this problem in C++. Understanding their limitations helps you appreciate why language-level reflection is the right fix rather than another library.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>

// ===== Boost.Hana approach =====
// BOOST_HANA_DEFINE_STRUCT(Person,
//     (std::string, name),
//     (int, age)
// );
// // Intrusive: modifies struct definition with macro
// // Fields must be declared INSIDE the macro
// // IDE cannot parse the struct normally

// ===== REFLECT macro approach =====
// struct Person { std::string name; int age; };
// REFLECT(Person, name, age)  // separate registration
// // Must list every field name
// // Easy to miss a field or get order wrong
// // Macro runs at preprocessing time - no type checking

// ===== Boost.Describe approach =====
// struct Person { std::string name; int age; };
// BOOST_DESCRIBE_STRUCT(Person, (), (name, age))
// // Non-intrusive but still manual
// // Must update when fields change

// ===== C++26 Reflection =====
struct Person {
    std::string name;
    int age;
    double height;
};
// No macros. No registration. Just the struct.

template <typename T>
void print_serialized(const T& obj) {
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        std::cout << std::meta::identifier_of(m) << ": ";
        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            std::cout << obj.[:m:];
        else
            std::cout << obj.[:m:];
        std::cout << '\n';
    }
}

int main() {
    Person p{"Alice", 30, 5.6};
    print_serialized(p);
    // name: Alice
    // age: 30
    // height: 5.6

    // Comparison summary:
    // Boost.Hana:    intrusive macros, C++14, good metaprogramming
    // REFLECT macro: fragile, error-prone, preprocessor-level
    // Boost.Describe: non-intrusive, but manual registration
    // C++26 Reflection: zero registration, standard, type-safe
}
```

The progression from Boost.Hana to Boost.Describe to C++26 reflection shows a gradual improvement in this area. Hana required you to write the struct inside the macro. Describe let you write the struct normally but still required manual field listing. Reflection finally removes the registration step entirely. The struct definition *is* the reflection data.

---

## Notes

- Reflection-based serialization requires zero per-type setup - there is no registration step separate from the struct definition itself.
- Adding or removing fields is automatically reflected in all generic serialization code, eliminating an entire class of "forgot to update" bugs.
- For deserialization, the same iteration pattern works in reverse: loop over members and assign `obj.[:m:] = parsed_value` for each one.
- Combine with `type_of` dispatch to adapt the same generic serializer to different output formats like JSON, XML, or binary.
