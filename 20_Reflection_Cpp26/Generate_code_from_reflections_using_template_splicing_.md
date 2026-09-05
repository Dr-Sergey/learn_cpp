# Generate code from reflections using template splicing [:...]:

**Category:** Reflection Cpp26  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

The **splice operator** `[:r:]` is the other half of the reflection story. Reflection (`^`) takes a piece of code and gives you a compile-time handle (`meta::info`) to it. Splicing (`[:r:]`) does the reverse - it takes a `meta::info` handle and inserts the thing it represents back into the code at that point.

Think of it as a two-step pipeline: first you reflect something into a handle so you can inspect or transform it, then you splice it back into code to actually use it.

| Splice form | Produces |
| --- | --- |
| `[:type_reflection:]` | A type (usable in `using`, template args, etc.) |
| `[:value_reflection:]` | A value or expression |
| `obj.[:member_reflection:]` | Member access on a concrete object |
| `[:func_reflection:](args)` | A function call |

To see the round-trip clearly, here is the reflect-then-splice pattern in its simplest form:

```cpp
Reflect -> Transform -> Splice:

  int x = 42;
  ^x  -------> meta::info  ---> [:^x:] == 42
  ^int ------> meta::info  ---> using T = [:^int:]; // T is int
```

The reason this trips people up is that `[:..:]` does not return a value - it is a syntax transformation. When you write `using T = [:int_refl:]`, the compiler literally fills in `int` at that point, as if you had written `using T = int`. The splice is not a cast or a function call; it is a compile-time code substitution.

---

## Self-Assessment

### Q1: Splice a reflected type into a template argument

Here you can see the most fundamental use of splicing: taking a reflected type and putting it back into a `using` declaration or a template argument. Once you have a `meta::info` handle, you can pass it around like any other `constexpr` value and splice it wherever you need a type.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <vector>
#include <string>

template <typename T>
void print_type() {
    std::cout << std::meta::display_string_of(^T) << '\n';
}

int main() {
    // Reflect a type:
    constexpr auto int_refl = ^int;
    constexpr auto str_refl = ^std::string;

    // Splice back into code:
    using T1 = [:int_refl:];   // T1 is int
    using T2 = [:str_refl:];   // T2 is std::string

    T1 x = 42;
    T2 s = "hello";
    std::cout << x << ", " << s << '\n';  // 42, hello

    // Use in template arguments:
    std::vector<[:int_refl:]> v = {1, 2, 3};  // vector<int>

    // Build types from reflections dynamically:
    constexpr auto members = std::meta::nonstatic_data_members_of(^struct { int a; double b; });
    using FirstType = [:std::meta::type_of(members[0]):];
    static_assert(std::is_same_v<FirstType, int>);
}
```

The last part is particularly interesting: you reflect an anonymous struct inline, ask for its first member's type, and splice that type into a `using` declaration - all at compile time. This is the kind of computed-type pattern that used to require complex recursive template machinery.

### Q2: Call a function by its reflected name

Splicing is not limited to types - you can splice function reflections too, which lets you build name-based dispatch tables. The pattern here iterates the static members of a class by name and calls whichever one matches the string argument.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string_view>

int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }
int sub(int a, int b) { return a - b; }

// Dispatch table built from reflections:
struct MathOps {
    static int add(int a, int b) { return a + b; }
    static int mul(int a, int b) { return a * b; }
    static int sub(int a, int b) { return a - b; }
};

template <typename OpsType>
constexpr int dispatch(std::string_view name, int a, int b) {
    constexpr auto members = std::meta::members_of(^OpsType);
    template for (constexpr auto m : members) {
        if (std::meta::identifier_of(m) == name) {
            return [:m:](a, b);  // splice + call!
        }
    }
    return -1;  // not found
}

int main() {
    // Compile-time lookup, runtime dispatch:
    std::cout << dispatch<MathOps>("add", 10, 20) << '\n';  // 30
    std::cout << dispatch<MathOps>("mul", 10, 20) << '\n';  // 200
    std::cout << dispatch<MathOps>("sub", 10, 20) << '\n';  // -10
}
```

The line `return [:m:](a, b)` is where the splice does the work: `[:m:]` splices in the function itself (not a pointer to it, but the name of the function), and then you immediately call it with `(a, b)`. This compiles to a direct call - the reflection overhead is fully paid at compile time.

### Q3: Type-safe serializer using member iteration + type splicing

Here both kinds of splicing work together: `obj.[:m:]` splices a member access for the field value, and `using MType = [:std::meta::type_of(m):]` splices the member's type into a `using` declaration so you can branch on it with `if constexpr`. The recursive call handles nested structs automatically.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <string>
#include <sstream>
#include <type_traits>

// Serializer that handles types differently using spliced type info:
template <typename T>
std::string serialize(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << "{";
    bool first = true;

    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;

        // Splice the member type:
        using MType = [:std::meta::type_of(m):];

        oss << '"' << std::meta::identifier_of(m) << "\": ";

        if constexpr (std::is_arithmetic_v<MType>) {
            oss << obj.[:m:];
        } else if constexpr (std::is_same_v<MType, std::string>) {
            oss << '"' << obj.[:m:] << '"';
        } else if constexpr (std::is_same_v<MType, bool>) {
            oss << (obj.[:m:] ? "true" : "false");
        } else {
            // Recursively serialize nested structs:
            oss << serialize(obj.[:m:]);
        }
    }
    oss << "}";
    return oss.str();
}

struct Address {
    std::string city;
    int zip;
};

struct Person {
    std::string name;
    int age;
    Address address;
};

int main() {
    Person p{"Alice", 30, {"NYC", 10001}};
    std::cout << serialize(p) << '\n';
    // Output: {"name": "Alice", "age": 30, "address": {"city": "NYC", "zip": 10001}}
}
```

The recursive call `oss << serialize(obj.[:m:])` works because `[:m:]` splices in the `Address` sub-object, and `serialize` is a template that handles any struct. This is the entire basis of a generic JSON serializer - about twenty lines of code that scales to arbitrarily nested structs.

---

## Notes

- `[:r:]` requires `r` to be a `consteval`-computed `std::meta::info` - you cannot splice a runtime value.
- Splicing types - `[:type_info:]` - produces a type that is usable anywhere a type name is legal: `using`, template arguments, `sizeof`, `decltype`, etc.
- Splicing values - `[:value_info:]` - produces an expression that evaluates to the reflected value.
- Splicing members - `obj.[:member_info:]` - accesses the reflected field on `obj`, just as if you had written the field name literally.
- Splicing functions - `[:func_info:](args...)` - calls the reflected function directly.
- This mechanism replaces code generation tools and extensive template metaprogramming for structural dispatch patterns.
