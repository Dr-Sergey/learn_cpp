# Understand the limits of C++26 reflection: what is and is not reflectable

**Category:** Reflection Cpp26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

C++26 static reflection can reflect many entities, but not everything. The rough mental model is: **if it has a name and a declaration that the compiler sees, it is probably reflectable**. Preprocessor macros are already gone before the compiler starts, so they are off limits. Local variables inside function bodies are also not reflectable in C++26. Everything at namespace or class scope - types, functions, enums, members - is fair game.

| Reflectable | Not Reflectable |
| --- | --- |
| Types (`^int`, `^MyClass`) | Local variables inside function bodies |
| Namespaces (`^std`) | Statement blocks / control flow |
| Enums and enumerators | Preprocessor macros (`#define`) |
| Non-static data members | Concepts (partially) |
| Static data members | Function bodies / implementations |
| Member functions | Expression results |
| Function declarations | Template parameter packs directly |
| Template specializations | Unnamed/anonymous types |
| Base classes | Attributes (partially) |

---

## Self-Assessment

### Q1: What CAN be reflected in C++26

This example covers the full range of entities you can successfully pass to `^`. It is worth running through all of them at least once mentally so you know where the language draws the lines.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>

// Types:
struct Widget {
    int id;
    std::string name;
    static int count;
    void reset() { id = 0; name.clear(); }
};
enum class Color { Red, Green, Blue };
namespace mylib { struct Foo {}; }

int global_var = 42;
void free_function(int x) {}

int main() {
    // 1. Types (built-in and user-defined):
    constexpr auto t1 = ^int;           // OK
    constexpr auto t2 = ^Widget;        // OK
    constexpr auto t3 = ^std::string;   // OK

    // 2. Namespaces:
    constexpr auto ns = ^std;           // OK
    constexpr auto ns2 = ^mylib;        // OK

    // 3. Enums and enumerators:
    constexpr auto e = ^Color;          // OK
    constexpr auto ev = ^Color::Red;    // OK (individual enumerator)

    // 4. Data members:
    constexpr auto members = std::meta::nonstatic_data_members_of(^Widget);
    static_assert(members.size() == 2);  // id, name

    // 5. Member functions:
    constexpr auto all = std::meta::members_of(^Widget);
    // Includes: id, name, count (static), reset (function)

    // 6. Base classes:
    struct Base {};
    struct Derived : Base { int x; };
    constexpr auto bases = std::meta::bases_of(^Derived);
    static_assert(bases.size() == 1);

    // 7. Functions:
    constexpr auto f = ^free_function;  // OK

    std::cout << "All reflections succeeded\n";
}
```

Everything at a named, declared scope works. Individual enumerators like `^Color::Red` are reflectable too, which is what lets you iterate through an enum's values by name - a feature that C++ programmers have wanted for decades.

### Q2: What CANNOT be reflected

Here you see the boundaries. The most important ones to internalize are: macros are not C++ entities (they are text - the compiler never even sees them), local variables are not reflectable, and expression results are not named entities.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>

#define MY_MACRO 42

int main() {
    int local_var = 10;

    // 1. Local variables: CANNOT reflect
    // constexpr auto r = ^local_var;  // ERROR: local variables not reflectable

    // 2. Preprocessor macros: CANNOT reflect
    // constexpr auto m = ^MY_MACRO;   // ERROR: macros are not C++ entities
    //    Macros are text substitution - they don't exist in the type system

    // 3. Statement bodies / control flow: CANNOT reflect
    // constexpr auto s = ^(if (true) {});  // ERROR: not an entity

    // 4. Expression results: CANNOT reflect
    // constexpr auto e = ^(1 + 2);    // ERROR: expression, not named entity

    // 5. Lambda types directly: limited
    auto lambda = [](int x) { return x * 2; };
    // constexpr auto l = ^lambda;     // ERROR: local variable
    // But you CAN reflect the lambda's type through decltype:
    // constexpr auto lt = ^decltype(lambda);  // OK in some contexts

    // 6. Unnamed types: limited
    // struct { int x; } anon;          // anonymous struct
    // ^decltype(anon) may work but identifier_of returns ""

    // Workaround for macros: use constexpr variables instead:
    constexpr int my_value = 42;  // This IS reflectable at namespace scope

    std::cout << "Reflection has clear boundaries\n";
    std::cout << "Key rule: only named, declared entities are reflectable\n";
}
```

The workaround note at the bottom is worth keeping in mind: if you have a value that you currently express as a macro because you want to use it as a constant, replacing it with a `constexpr` variable at namespace scope makes it reflectable. That is one more reason to prefer `constexpr` over `#define` for numeric constants.

### Q3: Compile-time reflection vs runtime RTTI

This example puts both mechanisms in the same file so you can compare them directly. The goal is to make the trade-offs concrete, not abstract.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <typeinfo>

struct Base { virtual ~Base() = default; };
struct Derived : Base { int data = 42; };

struct Plain { int x; double y; };  // non-polymorphic

int main() {
    // === RTTI (runtime) ===
    Base* ptr = new Derived();

    // typeid: requires polymorphic type (virtual function)
    std::cout << typeid(*ptr).name() << '\n';  // implementation-defined mangled name

    // dynamic_cast: runtime downcast
    if (auto* d = dynamic_cast<Derived*>(ptr)) {
        std::cout << "Downcast succeeded: " << d->data << '\n';
    }

    // RTTI limitations:
    // - Cannot enumerate members of Derived
    // - Cannot get member names
    // - Requires virtual functions (vtable)
    // - Runtime overhead (vtable pointer per object)
    // - typeid().name() is implementation-defined

    // === Static Reflection (compile-time) ===
    // Works with ANY type, no virtual needed:
    constexpr auto members = std::meta::nonstatic_data_members_of(^Plain);
    template for (constexpr auto m : members) {
        std::cout << std::meta::identifier_of(m) << ": "
                  << std::meta::display_string_of(std::meta::type_of(m)) << '\n';
    }
    // x: int
    // y: double

    // Reflection advantages over RTTI:
    // + Enumerate ALL members with names and types
    // + Zero runtime overhead
    // + Works with non-polymorphic types
    // + Reliable, portable names (not mangled)

    // RTTI advantages over Reflection:
    // + Runtime polymorphic dispatch (dynamic_cast)
    // + Works with types not known at compile time

    delete ptr;
}
```

The key takeaway from this comparison is that the two features solve different problems. RTTI answers "what type is this object right now, at runtime?" - which requires virtual dispatch and a vtable. Static reflection answers "what does this type look like structurally?" - which the compiler can answer at compile time from the class definition alone. You will often want both in the same program.

---

## Notes

- Rule of thumb: if it has a **name** and a **declaration**, it's probably reflectable.
- Preprocessor macros are replaced before compilation - the compiler never sees them.
- Local variables may become reflectable in future C++ revisions.
- RTTI and static reflection are complementary, not competing features.
- Use reflection for generic code generation; use RTTI for runtime polymorphic dispatch.
