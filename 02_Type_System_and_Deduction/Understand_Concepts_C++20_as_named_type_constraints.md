# Understand Concepts (C++20) as named type constraints

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP20.md#concepts>  

---

## Topic Overview

**Concepts** are named compile-time predicates on template parameters. They replace SFINAE with readable, composable constraints. The big win over `enable_if` is readability - both in the template declaration itself and in the error messages you get when a type doesn't satisfy the constraint.

### Defining a Concept

A concept is defined with `concept` and a `requires` expression that lists what the type must support. Each requirement can optionally specify the return type:

```cpp
#include <concepts>

template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;  // a + b must exist and return something convertible to T
};

template<typename T>
concept Printable = requires(std::ostream& os, T val) {
    { os << val } -> std::same_as<std::ostream&>;
};
```

### Four Ways to Apply a Concept

C++20 gives you four syntactic forms. They're semantically equivalent for single-concept constraints, but they read differently and have minor differences in how they interact with multiple parameters:

```cpp
// 1. Requires clause (trailing)
template<typename T>
    requires Addable<T>
T add(T a, T b) { return a + b; }

// 2. Constrained template parameter
template<Addable T>
T add2(T a, T b) { return a + b; }

// 3. Abbreviated function template (auto)
auto add3(Addable auto a, Addable auto b) { return a + b; }

// 4. Trailing requires after function signature
template<typename T>
T add4(T a, T b) requires Addable<T> { return a + b; }
```

### Standard Library Concepts

| Concept | Requires |
| --- | --- |
| `std::integral` | Integer type |
| `std::floating_point` | Float/double |
| `std::same_as<T, U>` | T and U are same type |
| `std::convertible_to<From, To>` | Implicit conversion |
| `std::derived_from<D, B>` | D derives from B |
| `std::regular` | Copyable + default_initializable + equality_comparable |
| `std::movable` | Move constructible + move assignable + swappable |
| `std::sortable<I>` | Iterator supports sorting |

### Concept Subsumption

When two overloads are both constrained, the **more constrained** one wins. This is the reason you can have multiple overloads differing only by concept without getting ambiguity errors. The key is that one concept must provably imply the other:

```cpp
template<typename T>
concept Animal = requires(T t) { t.speak(); };

template<typename T>
concept Pet = Animal<T> && requires(T t) { t.name(); };
// Pet subsumes Animal (Pet implies Animal)

void handle(Animal auto a) { std::cout << "Animal\n"; }
void handle(Pet auto p)    { std::cout << "Pet\n"; }

// For a type that satisfies Pet: Pet overload wins (more constrained)
// For a type that only satisfies Animal: Animal overload wins
```

---

## Self-Assessment

### Q1: Write a concept `Summable` that requires `operator+` and default constructibility

The concept combines a standard library concept (`std::default_initializable`) with a custom `requires` expression. The function body depends on both - it default-constructs the accumulator, then uses `operator+`. Both requirements need to be in the concept for the code to be well-formed:

```cpp
#include <iostream>
#include <concepts>
#include <string>

// Define the concept
template<typename T>
concept Summable = std::default_initializable<T> && requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
};

// Use the concept to constrain a function
template<Summable T>
T sum_all(std::initializer_list<T> values) {
    T result{};  // default construct (guaranteed by concept)
    for (const auto& v : values) {
        result = result + v;  // + operator (guaranteed by concept)
    }
    return result;
}

// Test types
struct NoPlus {
    int x;
};

struct NoCtor {
    int x;
    NoCtor(int v) : x(v) {}
    NoCtor operator+(const NoCtor& o) const { return NoCtor{x + o.x}; }
    // No default constructor!
};

int main() {
    // int: default constructible + supports +
    static_assert(Summable<int>);
    std::cout << sum_all({1, 2, 3, 4, 5}) << "\n";  // 15

    // double: OK
    static_assert(Summable<double>);
    std::cout << sum_all({1.1, 2.2, 3.3}) << "\n";   // 6.6

    // string: default constructible + supports +
    static_assert(Summable<std::string>);
    using namespace std::string_literals;
    std::cout << sum_all({"Hello"s, " "s, "World"s}) << "\n";  // Hello World

    // NoPlus: no operator+
    static_assert(!Summable<NoPlus>);

    // NoCtor: no default constructor
    static_assert(!Summable<NoCtor>);

    // sum_all(std::initializer_list<NoPlus>{});  // Compile error with clear message
}
```

### Q2: Show the four ways to apply a concept

All four forms are shown here using the same `Numeric` concept. Note the subtle difference with way 3: each `auto` parameter is its own template parameter, so `way3_add(1, 2.0)` compiles even though `1` is `int` and `2.0` is `double`:

```cpp
#include <iostream>
#include <concepts>
#include <type_traits>

template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// === Way 1: requires clause (before function body) ===
template<typename T>
    requires Numeric<T>
T way1_add(T a, T b) {
    return a + b;
}

// === Way 2: Constrained template parameter ===
template<Numeric T>
T way2_add(T a, T b) {
    return a + b;
}

// === Way 3: Abbreviated function template (constrained auto) ===
auto way3_add(Numeric auto a, Numeric auto b) {
    return a + b;
    // Note: a and b can be DIFFERENT Numeric types!
}

// === Way 4: Trailing requires clause ===
template<typename T>
T way4_add(T a, T b) requires Numeric<T> {
    return a + b;
}

// Constrained auto in other contexts:
Numeric auto get_value() { return 42; }       // constrains return type
void process(Numeric auto value) { }          // constrains parameter

int main() {
    std::cout << way1_add(1, 2) << "\n";        // 3
    std::cout << way2_add(1.5, 2.5) << "\n";    // 4.0
    std::cout << way3_add(1, 2.0) << "\n";      // 3.0 (mixed types OK!)
    std::cout << way4_add(10, 20) << "\n";       // 30

    // All four reject non-numeric types:
    // way1_add(std::string("a"), std::string("b"));  // Error
}
```

**When to use which:**

- Way 1 (`requires`): Most flexible - can express complex constraints.
- Way 2 (constrained param): Cleanest for simple single-concept constraints.
- Way 3 (`auto`): Most concise, but each parameter gets its own template parameter.
- Way 4 (trailing): Useful when the constraint depends on the function signature.

### Q3: Show how concept subsumption determines overload resolution

The reason there's no ambiguity error here is that `RandomAccessContainer` logically implies `Container` which logically implies `Sized` - the compiler can verify this by normalizing the concept definitions into atomic constraints. The most constrained overload always wins:

```cpp
#include <iostream>
#include <concepts>
#include <string>

// Base concept: any type with .size()
template<typename T>
concept Sized = requires(T t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

// Refined concept: Sized + iterable (subsumes Sized)
template<typename T>
concept Container = Sized<T> && requires(T t) {
    t.begin();
    t.end();
    typename T::value_type;
};

// Even more refined: Container + supports random access (subsumes Container)
template<typename T>
concept RandomAccessContainer = Container<T> && requires(T t, std::size_t i) {
    { t[i] } -> std::same_as<typename T::reference>;
};

// Three overloads with increasing constraints
void describe(Sized auto& x) {
    std::cout << "Sized: size=" << x.size() << "\n";
}

void describe(Container auto& x) {
    std::cout << "Container: size=" << x.size() << ", first=";
    if (x.begin() != x.end()) std::cout << *x.begin();
    std::cout << "\n";
}

void describe(RandomAccessContainer auto& x) {
    std::cout << "RandomAccess: size=" << x.size() << ", x[0]=" << x[0] << "\n";
}

int main() {
    // string satisfies all three - most constrained (RandomAccessContainer) wins
    std::string s = "hello";
    describe(s);  // "RandomAccess: size=5, x[0]=h"

    // Result of subsumption:
    // RandomAccessContainer is a superset of Container which is a superset of Sized
    // For a type satisfying all three, the compiler picks RandomAccessContainer
    // because it's the most constrained (most specific)

    // If we had a type satisfying only Sized (no begin/end):
    struct OnlySized {
        std::size_t size() const { return 42; }
    };
    // static_assert(Sized<OnlySized>);
    // static_assert(!Container<OnlySized>);
    // describe(OnlySized{});  // Would call Sized overload
}
```

The compiler normalizes concept constraints into conjunctions/disjunctions of atomic constraints. Concept A **subsumes** concept B if A's constraints logically imply B's constraints. The most constrained overload wins - no ambiguity error.

---

## Notes

- Concepts produce **much better error messages** than SFINAE - the compiler tells you which requirement failed.
- `requires requires` (double requires) is valid: the first is the requires-clause, the second starts a requires-expression.
- Concepts are not checked at instantiation - they're checked at the call site. This means the function body can still fail to compile if the concept is too weak.
- Concept satisfaction is cached per type - checking the same concept with the same type is O(1) after the first check.
- C++ Core Guidelines: T.10 - "Specify concepts for all template arguments."
