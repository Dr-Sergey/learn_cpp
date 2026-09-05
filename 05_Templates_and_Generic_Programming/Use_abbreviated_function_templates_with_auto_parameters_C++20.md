# Use Abbreviated Function Templates with `auto` Parameters (C++20)

**Category:** Templates and Generic Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/function_template#Abbreviated_function_template>  

---

## Topic Overview

### What Are Abbreviated Function Templates

C++20 lets you write **function templates without the `template<...>` syntax** by using `auto` (or constrained `auto`) in the parameter list. The result is a real function template - it's just written in a more concise form:

```cpp
// Traditional template syntax
template <typename T, typename U>
void print_pair(T a, U b);

// Abbreviated function template (C++20) — equivalent!
void print_pair(auto a, auto b);
```

Each `auto` parameter introduces an **independent implicit template parameter**. The compiler treats the abbreviated form exactly as if you had written the explicit `template<typename T, typename U>` version.

### Equivalence Rules

The table below maps each abbreviated form to its traditional equivalent. Pay attention to the third row - two `auto` parameters do not force the same type, they each get their own independent type parameter:

| Abbreviated | Equivalent Traditional |
| --- | --- |
| `void f(auto x)` | `template<typename T> void f(T x)` |
| `void f(auto x, auto y)` | `template<typename T, typename U> void f(T x, U y)` |
| `void f(auto x, auto y)` | NOT `template<typename T> void f(T x, T y)` - T and U are **independent** |
| `void f(std::integral auto x)` | `template<std::integral T> void f(T x)` |
| `void f(const auto& x)` | `template<typename T> void f(const T& x)` |

### Constrained `auto`

You can prefix `auto` with a concept to constrain the deduced type. This gives you concise overloading without any `enable_if` machinery:

```cpp
void process(std::integral auto x);     // x must satisfy std::integral
void process(std::floating_point auto x); // overload for floating point
```

---

## Self-Assessment

### Q1: Write `void f(auto x, auto y)` and explain it is equivalent to `template<typename T, typename U>`

The most important thing to internalize here is that each `auto` is independently deduced. If you need two parameters to be the same type, you still have to use the traditional `template<typename T>` syntax:

```cpp
#include <iostream>
#include <string>
#include <typeinfo>

// === Abbreviated form (C++20) ===
void f(auto x, auto y) {
    std::cout << "f(" << x << ", " << y << ")\n";
    std::cout << "  x type: " << typeid(x).name()
              << ", y type: " << typeid(y).name() << "\n";
}

// === Equivalent traditional form ===
// template <typename T, typename U>
// void f(T x, U y) {
//     std::cout << "f(" << x << ", " << y << ")\n";
// }

// KEY INSIGHT: each `auto` is a DIFFERENT template parameter
// So auto x and auto y can be DIFFERENT types

// This is NOT the same as:
// template <typename T>
// void f(T x, T y);    // <- forces SAME type for both params

// To force the same type with abbreviated syntax, you can't — use traditional syntax:
template <typename T>
void same_type(T x, T y) {
    std::cout << "same_type(" << x << ", " << y << ")\n";
}

int main() {
    std::cout << "=== Abbreviated function template ===\n";

    // Different types — works because auto -> independent template params
    f(42, 3.14);                     // T=int, U=double
    f(std::string("hello"), 100);    // T=string, U=int
    f('A', 'B');                     // T=char, U=char (same, but independently deduced)

    std::cout << "\n=== Same vs different types ===\n";

    // f(42, "hello");     // Works — auto allows mixed types
    same_type(1, 2);       // OK: both int
    // same_type(1, 2.0);  // ERROR: T deduced as int AND double — ambiguous

    // Abbreviated in lambdas (also C++20):
    auto lambda = [](auto a, auto b) {
        return a + b;  // each auto is independent
    };
    std::cout << "\nLambda: " << lambda(10, 20) << "\n";       // 30
    std::cout << "Lambda: " << lambda(1.5, 2.5) << "\n";       // 4.0

    return 0;
}
// Expected output:
//   f(42, 3.14)
//     x type: i, y type: d
//   f(hello, 100)
//     x type: ...string..., y type: i
//   f(A, B)
//     x type: c, y type: c
//   same_type(1, 2)
//   Lambda: 30
//   Lambda: 4
```

### Q2: Constrain an abbreviated template parameter: `void f(std::integral auto x)`

Constrained `auto` is where abbreviated templates really shine for readability. You get overload sets based on concepts with almost no boilerplate, and the syntax reads almost like English:

```cpp
#include <iostream>
#include <concepts>
#include <string>

// Constrained abbreviated function templates
// Syntax: void f(ConceptName auto param)

void process(std::integral auto x) {
    std::cout << "  integral: " << x << " (size=" << sizeof(x) << ")\n";
}

void process(std::floating_point auto x) {
    std::cout << "  floating: " << x << " (size=" << sizeof(x) << ")\n";
}

// You can combine constraints with qualifiers
void inspect(const std::integral auto& x) {
    std::cout << "  const ref to integral: " << x << "\n";
}

// Multiple constrained parameters
void combine(std::integral auto a, std::floating_point auto b) {
    std::cout << "  combine: " << a << " + " << b << " = " << (a + b) << "\n";
}

// Custom concept with abbreviated syntax
template <typename T>
concept Printable = requires(T t, std::ostream& os) {
    { os << t } -> std::same_as<std::ostream&>;
};

void display(Printable auto const& item) {
    std::cout << "  display: " << item << "\n";
}

int main() {
    std::cout << "=== Constrained auto parameters ===\n";
    process(42);         // matches integral overload
    process(3.14);       // matches floating_point overload
    process(100L);       // long -> integral
    process(2.718f);     // float -> floating_point

    std::cout << "\n=== Const ref to integral ===\n";
    int val = 99;
    inspect(val);
    inspect(42);

    std::cout << "\n=== Multiple constrained params ===\n";
    combine(10, 3.14);   // integral + floating_point
    // combine(1.0, 2.0); // ERROR: 1.0 is not integral

    std::cout << "\n=== Custom concept ===\n";
    display(42);
    display("hello");
    display(std::string("world"));

    // Compile-time error if constraint not satisfied:
    // process(std::string("hi"));  // ERROR: string is not integral or floating_point

    return 0;
}
// Expected output:
//   integral: 42 (size=4)
//   floating: 3.14 (size=8)
//   integral: 100 (size=4 or 8)
//   floating: 2.718 (size=4)
//   const ref to integral: 99
//   const ref to integral: 42
//   combine: 10 + 3.14 = 13.14
//   display: 42
//   display: hello
//   display: world
```

### Q3: Show the interaction between abbreviated templates and explicit template argument lists

This is a subtle but important point: even though the template parameters are "invisible" in abbreviated syntax, they still exist as real template parameters, and you can supply explicit template arguments for them. The rule is that any explicitly declared parameters come first, and the invented `auto` parameters come after in left-to-right order:

```cpp
#include <iostream>
#include <concepts>
#include <string>
#include <vector>

// === Key rule: abbreviated templates CAN have explicit template args ===
// But the template parameters are "invisible" — they're invented.

// Each `auto` becomes an invented template parameter, appended in order:
void show(auto x, auto y) {
    std::cout << "  x=" << x << " y=" << y << "\n";
}
// Equivalent to:
// template <typename _T1, typename _T2>
// void show(_T1 x, _T2 y);

// So you CAN call with explicit template arguments:
// show<int, double>(42, 3.14);

// === Mixing explicit template params with auto ===
template <typename Result>
Result convert(auto input) {
    // Result is explicit, input's type is deduced
    return static_cast<Result>(input);
}
// Equivalent to:
// template <typename Result, typename _T1>
// Result convert(_T1 input);

// Explicit params come FIRST, invented (auto) params come AFTER

// === Constrained auto with explicit args ===
template <int N>
void repeat(std::integral auto val) {
    for (int i = 0; i < N; ++i)
        std::cout << val << " ";
    std::cout << "\n";
}
// Equivalent to:
// template <int N, std::integral _T1>
// void repeat(_T1 val);

int main() {
    std::cout << "=== Explicit args with abbreviated templates ===\n";

    // Let the compiler deduce
    show(42, 3.14);           // T1=int, T2=double

    // Provide explicit template args for the invented parameters
    show<double, int>(42, 3); // T1=double (x=42.0), T2=int (y=3)

    std::cout << "\n=== Mixing explicit + auto params ===\n";

    // Result=int is explicit, input type is deduced from argument
    auto a = convert<int>(3.14);     // deduces _T1=double
    auto b = convert<double>(42);    // deduces _T1=int
    std::cout << "  convert<int>(3.14) = " << a << "\n";    // 3
    std::cout << "  convert<double>(42) = " << b << "\n";   // 42.0

    std::cout << "\n=== Non-type + auto params ===\n";
    repeat<3>(42);         // N=3, val=42 (int)
    repeat<5>(short(7));   // N=5, val=7 (short)

    std::cout << "\n=== Order of template parameters ===\n";
    std::cout << "Rule: explicit params first, then invented params (left-to-right)\n";
    std::cout << "  template<typename R> R convert(auto input)\n";
    std::cout << "  -> template<typename R, typename _T1> R convert(_T1 input)\n";
    std::cout << "  Call: convert<int>(3.14) -> R=int, _T1=double\n";

    return 0;
}
// Expected output:
//   x=42 y=3.14
//   x=42 y=3
//   convert<int>(3.14) = 3
//   convert<double>(42) = 42
//   42 42 42
//   7 7 7 7 7
//   Rule: explicit params first, then invented params (left-to-right)
//   ...
```

---

## Notes

- `auto` in function parameters (C++20) makes each `auto` an independent invented template parameter.
- `ConceptName auto x` constrains the deduced type - equivalent to `template<ConceptName T> void f(T x)`.
- Each `auto` is a **different** type parameter - two `auto` params can be different types.
- Explicit template arguments can be mixed: declared params first, then invented params in left-to-right order.
- Abbreviated syntax works for lambdas too: `[](auto x, std::integral auto y) { ... }`.
- Cannot force two `auto` params to be the same type - use traditional `template<typename T>` for that.
