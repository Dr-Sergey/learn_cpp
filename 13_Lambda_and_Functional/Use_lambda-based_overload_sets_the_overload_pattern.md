# Use lambda-based overload sets (the overload pattern)

**Category:** Lambda and Functional  
**Standard:** C++17 (using-pack expansion)  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant/visit>  

---

## Topic Overview

The **overload pattern** creates a single callable object from multiple lambdas, each handling a different type. It's the standard way to use `std::visit` with `std::variant` in idiomatic C++17, and it's one of those patterns that looks like magic until you see the mechanics behind it.

The complete setup is just a few lines:

```cpp
// The overload helper (C++17)
template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;  // deduction guide

// Usage with std::visit
std::visit(overloaded{
    [](int i)           { std::cout << "int: " << i; },
    [](double d)        { std::cout << "double: " << d; },
    [](const std::string& s) { std::cout << "string: " << s; }
}, my_variant);
```

### Mechanism

The key is that `overloaded` inherits from every lambda type and then brings all their `operator()` overloads into a single scope. Here's the inheritance picture:

```cpp
// overloaded<Lambda1, Lambda2, Lambda3>
//     |          |          |
//     +--inherits Lambda1 (has operator()(int))
//     +--inherits Lambda2 (has operator()(double))
//     +--inherits Lambda3 (has operator()(const string&))
//
//     using Lambda1::operator()...  <- brings ALL into scope
//     using Lambda2::operator()...
//     using Lambda3::operator()...
```

Without the `using` declarations, those inherited `operator()` overloads would hide each other - you'd get an ambiguity error whenever you tried to call the object. The `using Ts::operator()...` pack expansion is the crucial piece that makes it work.

---

## Self-Assessment

### Q1: Write the overloaded struct and use it with `std::visit`

This is the pattern in its most common form. The `std::visit` call dispatches to whichever lambda matches the type currently stored in the variant - at runtime, with no `if`/`else` chains or `std::get` needed:

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <vector>

// The overloaded helper
template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;  // CTAD deduction guide

using JsonValue = std::variant<int, double, bool, std::string>;

int main() {
    std::vector<JsonValue> values = {
        42,
        3.14,
        true,
        std::string{"hello"}
    };

    for (const auto& v : values) {
        std::visit(overloaded{
            [](int i)                { std::cout << "int: " << i << "\n"; },
            [](double d)             { std::cout << "double: " << d << "\n"; },
            [](bool b)               { std::cout << "bool: " << (b ? "true" : "false") << "\n"; },
            [](const std::string& s) { std::cout << "string: \"" << s << "\"\n"; }
        }, v);
    }
}
// Expected output:
//   int: 42
//   double: 3.14
//   bool: true
//   string: "hello"
```

Each call to `std::visit` checks what's in `v` at runtime and routes to the right lambda. The `overloaded` object presents a single callable interface with an overload for every alternative.

---

### Q2: Show how the overload pattern creates a callable that handles multiple types with lambdas

You're not limited to using `overloaded` only inside `std::visit`. You can construct an `overloaded` object and call it directly - it's a regular callable that just happens to accept multiple argument types. This is useful for building type-aware dispatch utilities:

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <cmath>

template <class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

struct Circle    { double radius; };
struct Rectangle { double w, h; };
struct Triangle  { double base, height; };

using Shape = std::variant<Circle, Rectangle, Triangle>;

int main() {
    // Create a single callable that handles THREE different types
    auto area = overloaded{
        [](const Circle& c)    { return 3.14159 * c.radius * c.radius; },
        [](const Rectangle& r) { return r.w * r.h; },
        [](const Triangle& t)  { return 0.5 * t.base * t.height; }
    };

    // The overloaded object can be called with ANY of the three types:
    std::cout << "Circle:    " << area(Circle{5.0}) << "\n";
    std::cout << "Rectangle: " << area(Rectangle{3.0, 4.0}) << "\n";
    std::cout << "Triangle:  " << area(Triangle{6.0, 3.0}) << "\n";

    // Use with std::visit on a variant:
    Shape s = Triangle{10.0, 5.0};
    double a = std::visit(area, s);
    std::cout << "Variant:   " << a << "\n";

    // Combine specific handlers with a generic catch-all:
    auto describe = overloaded{
        [](int i)    { return "integer: " + std::to_string(i); },
        [](double d) { return "double: " + std::to_string(d); },
        [](auto)     { return std::string{"unknown type"}; }  // catch-all
    };

    using Value = std::variant<int, double, std::string>;
    Value v1 = 42;
    Value v2 = std::string{"text"};
    std::cout << std::visit(describe, v1) << "\n";
    std::cout << std::visit(describe, v2) << "\n";
}
// Expected output:
//   Circle:    78.5398
//   Rectangle: 12
//   Triangle:  9
//   Variant:   25
//   integer: 42
//   unknown type
```

The `auto` catch-all at the end is particularly useful. It lets you handle a specific set of types precisely and fall back gracefully for anything else.

---

### Q3: Explain the `using Ts::operator()...` expansion and why it's needed

This is the part that makes the whole pattern work, and it's worth understanding deeply. The reason this trips people up is that C++ inheritance name lookup doesn't work the way you might expect when multiple bases define the same name.

The example below starts broken, then fixes it step by step:

```cpp
#include <iostream>

// WITHOUT using-declaration: base class operator() is HIDDEN!
struct LambdaA {
    void operator()(int x) const { std::cout << "int: " << x << "\n"; }
};

struct LambdaB {
    void operator()(double x) const { std::cout << "double: " << x << "\n"; }
};

// BROKEN: no using-declaration
struct BrokenOverload : LambdaA, LambdaB {
    // Both bases have operator(), but they HIDE each other!
    // BrokenOverload{}(42);    // ERROR: ambiguous!
    // BrokenOverload{}(3.14);  // ERROR: ambiguous!
};

// FIXED: using-declaration brings both into scope
struct FixedOverload : LambdaA, LambdaB {
    using LambdaA::operator();
    using LambdaB::operator();
    // Now both are visible - overload resolution works normally!
};

// The pattern generalizes with pack expansion
template <class... Ts>
struct overloaded : Ts... {
    using Ts::operator()...;  // using-declaration for EACH base!
    // Expands to:
    //   using T1::operator();
    //   using T2::operator();
    //   using T3::operator();
    //   ...
};
template <class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

int main() {
    // BrokenOverload b;
    // b(42);      // ERROR: ambiguous
    // b(3.14);    // ERROR: ambiguous

    FixedOverload f;
    f(42);      // calls LambdaA::operator()(int)
    f(3.14);    // calls LambdaB::operator()(double)

    // Variadic version:
    auto ov = overloaded{
        [](int x)    { std::cout << "int: " << x << "\n"; },
        [](double x) { std::cout << "double: " << x << "\n"; },
        [](auto x)   { std::cout << "other: " << x << "\n"; }
    };
    ov(42);
    ov(3.14);
    ov("hello");
}
// Expected output:
//   int: 42
//   double: 3.14
//   int: 42
//   double: 3.14
//   other: hello
```

**Why `using Ts::operator()...` is required:**

1. When a class inherits from multiple bases, name lookup finds a name in one base and **stops** - other bases are hidden.
2. Without `using`, calling `overloaded{}(42)` is ambiguous because the compiler finds `operator()` in multiple bases and can't choose.
3. The `using`-declaration brings ALL `operator()` overloads into the derived class scope, enabling normal overload resolution to pick the best match.
4. The `...` expands the pack, generating one `using` for each lambda base class.

This is a rare case where the pack expansion syntax does something genuinely elegant: a one-liner that scales to any number of bases.

---

## Notes

- **C++20 simplification:** The deduction guide `overloaded(Ts...) -> overloaded<Ts...>;` can be omitted in C++20 (automatic CTAD for aggregates).
- **Order matters:** If multiple lambdas can match (e.g., `auto` catch-all), normal C++ overload resolution applies - more specific matches win over generic ones.
- **Performance:** Zero overhead - the `overloaded` struct is typically optimized away entirely by the compiler.
- **Alternative spelling:** Some codebases name it `overload`, `match`, or `visitor` instead of `overloaded` - the implementation is the same.
- **Multi-variant visit:** `std::visit(overloaded{...}, v1, v2)` dispatches on both variants simultaneously, giving you a 2D type dispatch table.
