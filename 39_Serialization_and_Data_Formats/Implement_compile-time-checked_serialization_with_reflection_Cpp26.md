# Implement compile-time-checked serialization with reflection (C++26)

**Category:** Serialization and Data Formats  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2996r0.html>  

---

## Topic Overview

C++26 static reflection (`^` operator and `std::meta` namespace) enables automatic serialization by enumerating struct members at compile time - no macros, no code generation, no boilerplate.

The big deal here is that before reflection arrived, every serializable type required either a macro full of hidden gotchas, a code generator you had to run before building, or hand-written boilerplate that silently went stale whenever you renamed a field. Reflection solves all three problems at once by letting the compiler itself tell you what members a type has.

### Current State (Pre-Reflection): Macros

Before C++26, the typical approach looked like this - you had to manually list every field in a macro invocation:

```cpp
// Pre-C++26: macros or code generators
#define SERIALIZABLE_FIELDS(Type, ...) \
    template<> struct Serializer<Type> { \
        static json to_json(const Type& obj) { /* ... */ } \
    };

// Boilerplate grows linearly with the number of types.
```

The problem is obvious once your codebase grows: that list of fields is a second definition of your struct that you maintain by hand. Add a field to the struct, forget to add it to the macro, and your serializer silently drops data.

### C++26 Reflection Approach

With reflection, you write the serializer once and it works for every type automatically. The `template for` loop iterates over members at compile time - the compiler unrolls it into exactly the right code for each type. The `^T` operator produces a compile-time representation of the type `T`, and `obj.[:member:]` is the splice syntax that accesses the actual member at runtime:

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <string>
#include <iostream>
#include <sstream>

// Automatic JSON serializer using C++26 reflection
template<typename T>
std::string to_json(const T& obj) {
    std::ostringstream oss;
    oss << "{";
    bool first = true;

    // Iterate over all non-static data members at compile time
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            if (!first) oss << ",";
            first = false;

            // Get member name as string
            oss << "\"" << std::meta::identifier_of(member) << "\":";

            // Access the member value
            const auto& value = obj.[:member:];

            // Type-based formatting
            if constexpr (std::is_same_v<std::remove_cvref_t<decltype(value)>, std::string>) {
                oss << "\"" << value << "\"";
            } else if constexpr (std::is_same_v<std::remove_cvref_t<decltype(value)>, bool>) {
                oss << (value ? "true" : "false");
            } else {
                oss << value;
            }
        }
    }
    oss << "}";
    return oss.str();
}

// Usage - zero boilerplate per type!
struct Employee {
    std::string name;
    int age;
    double salary;
    bool active;
};

int main() {
    Employee emp{"Alice", 30, 85000.0, true};
    std::cout << to_json(emp) << "\n";
    // {"name":"Alice","age":30,"salary":85000,"active":true}
}
```

Notice that `Employee` has no special annotations, no macros, and no inherited base class. The `to_json` template handles it purely by inspecting `Employee`'s members at compile time. If you rename `age` to `years_old`, the JSON key automatically becomes `"years_old"` - the serializer and the struct can never drift apart.

### Compile-Time Validation

One of the best things about compile-time reflection is that you can also validate types at compile time. This `consteval` function walks the members and rejects any type that has a member we don't know how to serialize - turning a subtle runtime surprise into a hard build error:

```cpp
// Reflection enables compile-time checks on serializable types
template<typename T>
consteval bool is_serializable() {
    for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            using MemberType = typename[:std::meta::type_of(member):];
            if constexpr (!std::is_arithmetic_v<MemberType> &&
                          !std::is_same_v<MemberType, std::string>) {
                return false;  // Unsupported member type
            }
        }
    }
    return true;
}

static_assert(is_serializable<Employee>(), "Employee must be serializable");
```

The `static_assert` fires at compile time if you accidentally add, say, a `std::vector<Tag>` field to `Employee` before teaching the serializer how to handle it. You get a clear diagnostic immediately rather than discovering the problem at runtime when a field silently disappears from your output.

---

## Self-Assessment

### Q1: Write a reflection-based deserializer from JSON

The deserializer mirrors the serializer: iterate members at compile time, look up each name in the JSON object, and splice-assign the value back into the struct. The reason this is interesting is that you still only write it once, and it works for any serializable type:

```cpp
// Conceptual C++26 code
template<typename T>
T from_json(const json_object& j) {
    T obj{};
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            constexpr auto name = std::meta::identifier_of(member);
            using MemberType = typename[:std::meta::type_of(member):];

            if (j.contains(name)) {
                obj.[:member:] = j[name].template get<MemberType>();
            }
            // Missing fields: keep default-initialized value
        }
    }
    return obj;
}
```

### Q2: Generate a schema description from a struct using reflection

You can also use reflection to produce a human-readable schema dump - useful for documentation generation, validation tools, or debugging what the serializer sees:

```cpp
template<typename T>
void print_schema() {
    std::cout << "Schema for " << std::meta::identifier_of(^T) << ":\n";
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            std::cout << "  " << std::meta::identifier_of(member)
                      << " : " << std::meta::identifier_of(std::meta::type_of(member))
                      << "\n";
        }
    }
}
// print_schema<Employee>();
// Schema for Employee:
//   name : string
//   age : int
//   salary : double
//   active : bool
```

### Q3: Why is reflection-based serialization safer than macro-based approaches

Here is why the reflection approach wins on every dimension that matters for long-term maintainability:

1. **Type-checked at compile time**: the splice operator `[:member:]` verifies the member exists
2. **Refactoring-safe**: renaming a field automatically updates the serialized name
3. **No macro hygiene issues**: no hidden name collisions or expansion pitfalls
4. **Complete coverage**: reflection enumerates ALL members; macros require manual listing (easy to miss a field)
5. **Supports consteval validation**: you can check at compile time that all member types are serializable

---

## Notes

- C++26 reflection is not yet available in production compilers as of early 2026 - EDG and experimental Clang branches have partial support.
- Libraries like **glaze** and **Boost.Describe** provide similar functionality today using C++20 techniques.
- Reflection-based serialization generates optimal code: the compiler unrolls the member loop at compile time.
