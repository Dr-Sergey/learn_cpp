# Understand token injection and code injection proposals

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2996> <https://wg21.link/P3294>  

---

## Topic Overview

### Beyond Introspection: Code Generation

C++26 static reflection (P2996) enables **introspection** - querying properties of types, members, and enumerators at compile time. But the complementary feature is **code injection** - the ability to generate new code based on reflection results. Think of introspection as reading the structure of your program, and injection as writing back into it based on what you read.

### Token Injection (P3294)

The **token injection** proposal allows injecting tokens (new declarations, statements, expressions) into the program based on compile-time computations. The syntax is still evolving, but the idea is that a `consteval` function can compute what code should exist and then `queue_injection` it to be emitted into the right scope.

```cpp
#include <meta>

// Conceptual syntax (subject to evolution):
struct Point {
    int x, y, z;
};

// Generate operator== automatically using reflection + injection
consteval void generate_equality(std::meta::info type) {
    auto members = std::meta::nonstatic_data_members_of(type);
    // Inject: friend bool operator==(const T& a, const T& b) {
    //     return a.x == b.x && a.y == b.y && a.z == b.z;
    // }
    queue_injection(^^{
        friend bool operator==(const [:\(type):] & a, const [:\(type):] & b) {
            return [:\(fold_and(members, [](auto m) {
                return ^^{ a.[:$(m):] == b.[:$(m):] };
            })):];
        }
    });
}
```

This is what "fully automated code generation" looks like. The `generate_equality` function never has to know how many fields `Point` has or what they are named - reflection supplies that at compile time, and injection writes the resulting operator into the type.

### The Splice Operator `[: ... :]`

The splice operator takes a reflection value and injects it back into code. You have seen this in the earlier topics, but it is worth seeing again in context with injection:

```cpp
constexpr auto refl = ^^int;   // reflect on 'int'
[:refl:] x = 42;               // splice: declares 'int x = 42'

constexpr auto member = /* reflection of a data member */;
obj.[:member:] = 10;           // splice: accesses that member on obj
```

The splice is the lower-level building block; injection is the higher-level operation that creates whole new declarations.

### Token Sequences

Token sequences (`^^{ ... }`) represent fragments of code that can be manipulated at compile time. You can think of them as structured, type-safe code templates - unlike macros, they are parsed by the compiler and can contain splices and interpolations.

```cpp
consteval auto make_getter(std::meta::info member) {
    auto name = std::meta::identifier_of(member);
    auto type = std::meta::type_of(member);

    return ^^{
        constexpr auto get_[:\(name):]() const -> const [:\(type):]& {
            return this->[:\(member):];
        }
    };
}
```

This `make_getter` function takes a reflected member and returns a token sequence representing a getter function for it. The compiler parses and type-checks the token sequence when it is injected, not when it is created - so errors are reported in context.

### Practical Example: Auto-Generating Struct Serialization

Here is a more complete example showing the injection pattern applied to a real use case - automatic JSON serialization:

```cpp
#include <meta>
#include <string>

template<typename T>
consteval void inject_to_json() {
    auto members = std::meta::nonstatic_data_members_of(^^T);

    queue_injection(^^{
        friend std::string to_json(const T& obj) {
            std::string result = "{";
            bool first = true;
            template for (constexpr auto m : members) {
                if (!first) result += ", ";
                result += "\"";
                result += std::meta::identifier_of(m);
                result += "\": ";
                result += std::to_string(obj.[:m:]);
                first = false;
            }
            result += "}";
            return result;
        }
    });
}

struct Config {
    int width = 1920;
    int height = 1080;
    int fps = 60;
    consteval { inject_to_json<Config>(); }
};

// Now Config has: friend std::string to_json(const Config& obj);
```

The `consteval { ... }` block inside the class body is an injection site - the compiler evaluates the `consteval` expression at class-definition time and emits whatever code was injected. The result is that `Config` automatically gets a `to_json` function without the developer writing a single field-specific line.

### Code Injection vs. Macros vs. Templates

If you are trying to decide how injection fits into the landscape of C++ code generation techniques, here is the comparison:

| Approach | Introspection | Type-safe | Debugging | Power |
| --- | :---: | :---: | :---: | :---: |
| Macros | No | No | Poor | High (textual) |
| Templates/SFINAE | Limited | Yes | OK | Medium |
| Concepts | Type constraints | Yes | Good | Medium |
| Reflection + Injection | Full | Yes | TBD | Very High |

The key column is "Introspection" - macros and templates cannot look at the structure of a type and react to it. Reflection can, and injection lets you act on what you find.

---

## Self-Assessment

### Q1: What is the difference between reflection and injection

- **Reflection** (`^^` operator): Queries compile-time properties of existing code - member names, types, attributes.
- **Injection** (`queue_injection`, token sequences): Generates **new** code based on reflection results.

Reflection reads; injection writes. Together, they enable fully automated code generation. You would use reflection alone when you only need to inspect a type (for logging, serialization, equality comparison). You add injection when you need to synthesize new declarations or member functions that become part of the type itself.

### Q2: Show how to enumerate struct members and generate getters

```cpp
struct Player {
    std::string name;
    int health;
    float speed;

    // Inject getters for all members
    consteval {
        for (auto m : std::meta::nonstatic_data_members_of(^^Player)) {
            queue_injection(make_getter(m));
        }
    }
};

// Generated:
// const std::string& get_name() const;
// const int& get_health() const;
// const float& get_speed() const;
```

The `consteval { ... }` block runs at the point where `Player` is being defined. It iterates over the reflected members and injects a getter for each one. If you later add a `double stamina` field to `Player`, a `get_stamina()` getter appears automatically - no other change needed.

### Q3: Why are token sequences better than string-based code generation

Token sequences (`^^{ ... }`) are:

1. **Parsed by the compiler** - syntax errors are caught immediately, not at the injection site.
2. **Type-checked** - spliced values must match their contexts; you cannot inject a type where a value is expected.
3. **Hygienic** - names are resolved in the correct scope, so there is no risk of accidentally capturing or shadowing an unrelated name.
4. **Debuggable** - the compiler can map generated code back to the injection site, making error messages meaningful.

String-based generation (like macros) has none of these properties. A macro expands to text that is then re-parsed; if the result is invalid C++, you get an error message pointing at the expansion site with no indication of what the macro author intended. Token sequences are structured, not textual, so the compiler has full understanding of what is being injected.

---

## Notes

- Token injection proposals are still evolving - syntax may change before C++26 is finalized.
- The `queue_injection()` function adds code to be injected at the end of the current evaluation context.
- Injection works in `consteval` blocks inside class definitions.
- This is the most powerful compile-time code generation feature C++ has ever had.
