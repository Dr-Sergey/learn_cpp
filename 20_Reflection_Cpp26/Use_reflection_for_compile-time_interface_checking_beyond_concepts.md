# Use reflection for compile-time interface checking beyond concepts

**Category:** Reflection Cpp26  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

C++20 concepts are great, but they only check syntax - they can verify that a type has a callable `.begin()` or that a function exists, but they cannot inspect the *structure* of a type. Reflection gives you a completely different level of introspection: you can look at member names by string, check exact field types, count how many data members a struct has, or verify that every member satisfies some property.

Here is the practical comparison between what each tool can do:

| Check Type | Concepts | Reflection |
| --- | --- | --- |
| "Has `.begin()` and `.end()`" | Yes | Yes |
| "Has a member named `serialize`" | Indirect (SFINAE) | Direct |
| "Has exactly 3 `int` fields" | No | Yes |
| "All members are trivially copyable" | No | Yes |
| Diagnostic quality | Template errors | Custom `static_assert` messages |

If the table feels like a lot, the key insight is: concepts check *what you can do* with a type; reflection checks *what the type is made of*. They complement each other - you will often use both.

---

## Self-Assessment

### Q1: Verify a type has specific member functions with correct signatures

The two helper functions below are the building blocks for structural checking. `has_method` walks the member list and matches by name and return type; `has_field` does the same for data members. Notice that both run entirely at compile time via `consteval`.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

// Check if type T has a member function with given name and return type:
consteval bool has_method(std::meta::info type,
                          std::string_view name,
                          std::meta::info return_type) {
    for (auto m : std::meta::members_of(type)) {
        if (std::meta::is_function(m) &&
            std::meta::identifier_of(m) == name &&
            std::meta::type_of(m) == return_type) {
            return true;
        }
    }
    return false;
}

// Check if type has a specific data member:
consteval bool has_field(std::meta::info type,
                         std::string_view name) {
    for (auto m : std::meta::nonstatic_data_members_of(type)) {
        if (std::meta::identifier_of(m) == name) return true;
    }
    return false;
}

struct Serializable {
    int id;
    std::string data;
    std::string serialize() const { return data; }
    void deserialize(const std::string& s) { data = s; }
};

struct NotSerializable {
    int value;
};

// Compile-time structural checks:
static_assert(has_field(^Serializable, "id"));
static_assert(has_field(^Serializable, "data"));
static_assert(!has_field(^NotSerializable, "data"));

int main() {
    std::cout << "Serializable has 'id': "
              << has_field(^Serializable, "id") << '\n';
    std::cout << "NotSerializable has 'data': "
              << has_field(^NotSerializable, "data") << '\n';
}
```

The `static_assert` calls at the bottom are what make this useful in practice - they turn structural mismatches into hard compile-time errors before any runtime code runs.

### Q2: Reflection-based checks produce better diagnostics

One of the most practical wins from reflection over raw concepts is the quality of error messages. When a concept is not satisfied deep in a template, the error is often a wall of "constraints not satisfied" text. With reflection, you write the error message yourself and it fires exactly where you want it.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <type_traits>

// With concepts: cryptic error deep in template instantiation
// template <typename T> concept Drawable = requires(T t) { t.draw(); };
// Error: "constraints not satisfied" (unhelpful)

// With reflection: custom, descriptive error messages:
consteval void validate_drawable(std::meta::info type) {
    bool found_draw = false;
    for (auto m : std::meta::members_of(type)) {
        if (std::meta::is_function(m) &&
            std::meta::identifier_of(m) == "draw") {
            found_draw = true;
        }
    }
    if (!found_draw) {
        // This produces a clear compile error with YOUR message:
        // "Type 'BadWidget' is missing required 'draw()' method"
        static_assert(false, "Missing required 'draw()' member function");
    }
}

// Also check field requirements:
consteval void validate_entity(std::meta::info type) {
    constexpr const char* required_fields[] = {"id", "name", "position"};
    for (auto field_name : required_fields) {
        bool found = false;
        for (auto m : std::meta::nonstatic_data_members_of(type)) {
            if (std::meta::identifier_of(m) == field_name) {
                found = true;
                break;
            }
        }
        if (!found) {
            // Custom diagnostic: "Entity type missing required field: position"
        }
    }
}

struct GoodWidget {
    void draw() const {}
};

// validate_drawable(^GoodWidget);   // OK
// validate_drawable(^int);          // Clear compile error

int main() {
    std::cout << "Reflection gives custom compile-time diagnostics\n";
}
```

The reason this matters so much in a real codebase is that the person who writes the validator and the person who gets the error may be different people. Giving them a clear message - "Entity type missing required field: position" - saves a lot of head-scratching.

### Q3: Plugin system validator - check protocol conformance at compile time

Here is a concrete use case that shows why the structural checking approach pays off. A plugin system has a well-defined protocol - every plugin must implement `init`, `shutdown`, `name`, and `version`. With reflection you can enforce this at the point of instantiation, before any runtime registration happens.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>

// Define the plugin protocol:
//   - Must have: void init()
//   - Must have: void shutdown()
//   - Must have: std::string name() const
//   - Must have: int version() const

consteval bool has_member_function(std::meta::info type, std::string_view name) {
    for (auto m : std::meta::members_of(type)) {
        if (std::meta::is_function(m) &&
            std::meta::identifier_of(m) == name) {
            return true;
        }
    }
    return false;
}

consteval bool satisfies_plugin_protocol(std::meta::info type) {
    return has_member_function(type, "init") &&
           has_member_function(type, "shutdown") &&
           has_member_function(type, "name") &&
           has_member_function(type, "version");
}

struct AudioPlugin {
    void init() { /* load audio system */ }
    void shutdown() { /* cleanup */ }
    std::string name() const { return "AudioPlugin"; }
    int version() const { return 1; }
    void process() { /* plugin-specific */ }
};

struct BadPlugin {
    void init() {}
    // Missing: shutdown, name, version
};

static_assert(satisfies_plugin_protocol(^AudioPlugin),
              "AudioPlugin must satisfy plugin protocol");
// static_assert(satisfies_plugin_protocol(^BadPlugin));  // COMPILE ERROR!

// Generic plugin loader:
template <typename T>
    requires (satisfies_plugin_protocol(^T))
void load_plugin(T& plugin) {
    plugin.init();
    std::cout << "Loaded: " << plugin.name()
              << " v" << plugin.version() << '\n';
}

int main() {
    AudioPlugin ap;
    load_plugin(ap);  // Loaded: AudioPlugin v1

    // BadPlugin bp;
    // load_plugin(bp);  // COMPILE ERROR: does not satisfy protocol
}
```

Notice how `satisfies_plugin_protocol` is used both in a standalone `static_assert` and as a `requires` clause. You can mix the two styles freely - reflection-based checks compose naturally with C++20 concept syntax.

---

## Notes

- Concepts check syntactic requirements; reflection checks structural details like member names, field types, and member counts.
- Custom `static_assert` messages triggered from reflection give far clearer diagnostics than the template error walls you get from failed concept constraints.
- Combine the two tools: use `requires (satisfies_protocol(^T))` to get concept-style gating with reflection-powered checking logic underneath.
- This pattern is particularly valuable for plugin systems, serialization frameworks, and ORMs where you need to enforce non-trivial structural contracts across many types.
