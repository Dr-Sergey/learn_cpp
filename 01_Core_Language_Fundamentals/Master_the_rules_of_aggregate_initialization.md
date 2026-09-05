# Master the rules of aggregate initialization

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/aggregate_initialization>  

---

## Topic Overview

An **aggregate** is a class type (struct/class/union) or array that can be initialized with a
braced-init-list without calling a user-provided constructor. The tricky part is that the rules
for what qualifies as an aggregate have tightened with almost every standard version - so a struct
that was an aggregate in C++17 may not be one in C++20.

### Aggregate Conditions by Standard

Here's the full picture of how the rules evolved. If a row says "No" in a column, that
restriction was added or removed in that version:

| Condition | C++11 | C++14 | C++17 | C++20 |
| --- | --- | --- | --- | --- |
| No user-provided constructors | Yes | Yes | Yes | Yes |
| No user-declared constructors | - | - | - | Yes (stricter!) |
| No private/protected non-static members | Yes | Yes | Yes | Yes |
| No virtual functions | Yes | Yes | Yes | Yes |
| No virtual/private/protected base classes | Yes | Yes | Yes | Yes |
| Public bases allowed | - | - | Yes | Yes |
| Default member initializers allowed | - | Yes | Yes | Yes |
| `= default` / `= delete` ctors allowed | Yes | Yes | Yes | No |

The headline C++20 change: `T() = default;` or `T() = delete;` now disqualifies a struct
from being an aggregate. That's a breaking change if you relied on it.

### Basic Aggregate Initialization

When a type is an aggregate, brace initialization fills its members in declaration order.
Any members you leave out get value-initialized (zeroed for scalars):

```cpp
struct Point {
    double x;
    double y;
    double z;
};

Point p1 = {1.0, 2.0, 3.0};    // aggregate init
Point p2{4.0, 5.0};             // z is value-initialized to 0.0
Point p3{};                     // all zeros
```

### Designated Initializers (C++20)

C++20 lets you name the members you're initializing, which makes config-style structs
much more readable and immune to member-reorder bugs. One constraint: the order must
match the declaration order.

```cpp
struct Config {
    int width  = 800;
    int height = 600;
    bool fullscreen = false;
    int fps_cap = 60;
};

// Name the members explicitly - order must match declaration order
Config cfg{.width = 1920, .height = 1080, .fullscreen = true};
// cfg.fps_cap uses default: 60

// Config bad{.height = 1080, .width = 1920};  // ERROR: wrong order in C++
```

### Nested Aggregate Initialization

When an aggregate contains another aggregate, you can use explicit nesting or let the
compiler flatten the list for you via brace elision:

```cpp
struct Inner { int a, b; };
struct Outer { Inner in; int c; };

Outer o1 = {{1, 2}, 3};         // explicit nesting
Outer o2 = {1, 2, 3};           // brace elision - also valid
```

### Arrays Are Aggregates Too

Arrays follow the same rules - you can initialize them positionally, and any elements
you omit are zero-initialized:

```cpp
int arr[] = {1, 2, 3, 4, 5};            // size deduced as 5
int matrix[2][3] = {{1,2,3}, {4,5,6}};  // 2D aggregate init
```

---

## Self-Assessment

### Q1: List the conditions that make a type an aggregate in C++11, C++14, C++17, and C++20

**Answer:**

**C++11 aggregate requirements:**

- Array type, OR class type with:
  - No user-**provided** constructors (`= default` and `= delete` are OK)
  - No private or protected non-static data members
  - No base classes at all
  - No virtual functions
  - No default member initializers (brace-or-equal)

**C++14 relaxation:**

- Same as C++11 except: **default member initializers are now allowed**.

```cpp
// C++14: aggregate despite default member initializer
struct C14 {
    int x = 10;   // OK in C++14, NOT OK in C++11
    int y;
};
C14 c{1, 2};      // x=1, y=2 - initializer overrides default
```

**C++17 relaxation:**

- Same as C++14 except: **public base classes are now allowed**.
- Base class subobjects are initialized first in the braced list.

```cpp
struct Base { int a; };
struct C17 : Base {    // aggregate in C++17
    int b;
};
C17 obj{{10}, 20};     // Base::a=10, b=20
```

**C++20 tightening:**

- Same as C++17 except: **no user-declared constructors at all**.
- `T() = default;` and `T() = delete;` now disqualify.

```cpp
struct NotAggregate {
    NotAggregate() = default;   // user-declared -> not aggregate in C++20
    int x;
};
// NotAggregate na{42};  // ERROR in C++20, OK in C++17
```

### Q2: Demonstrate designated initializers (C++20) and show how they prevent reordering bugs

Without designated initializers it's easy to silently swap `age` and `salary`. With them,
the field names are right there in the initializer - the compiler enforces the correct order
and catches any mistakes:

```cpp
#include <iostream>
#include <string>

struct Employee {
    std::string name;
    int         age;
    double      salary;
    bool        active;
};

int main() {
    // Without designated initializers - easy to swap age and salary by accident:
    Employee e1{"Alice", 30, 75000.0, true};

    // With designated initializers - self-documenting and order-safe:
    Employee e2{
        .name   = "Bob",
        .age    = 25,
        .salary = 82000.0,
        .active = true
    };

    // Skip members - they use default initialization:
    Employee e3{.name = "Charlie", .active = false};
    // e3.age = 0, e3.salary = 0.0

    // Compiler REJECTS wrong order:
    // Employee bad{.salary = 50000.0, .name = "Eve"};  // ERROR

    std::cout << e2.name << " earns " << e2.salary << "\n";
    std::cout << e3.name << " age=" << e3.age << "\n";
}
```

**How it works:**

- Designated initializers name each member explicitly - can't accidentally swap `age` and `salary`.
- C++ (unlike C) requires designators in **declaration order** - the compiler rejects out-of-order designators.
- Unmentioned members are **value-initialized** (zero for scalars, default-constructed for class types).
- Designated initializers cannot be mixed with positional initializers.

### Q3: Show a case where adding a user-provided constructor breaks aggregate initialization

This is a common surprise when refactoring: you add a convenience constructor and suddenly
designated initializers stop compiling. Here's all three versions side by side:

```cpp
#include <iostream>
#include <type_traits>

// Version 1: aggregate
struct PointV1 {
    double x;
    double y;
};

// Version 2: add a constructor -> no longer aggregate
struct PointV2 {
    double x;
    double y;
    PointV2(double x, double y) : x(x), y(y) {}   // user-provided!
};

// Version 3: C++20 - even = default breaks it
struct PointV3 {
    double x;
    double y;
    PointV3() = default;   // user-declared -> not aggregate in C++20
};

int main() {
    // V1: aggregate init works
    PointV1 p1{1.0, 2.0};                // OK
    PointV1 p1b{.x = 1.0, .y = 2.0};    // designated init - OK

    // V2: aggregate init broken
    PointV2 p2{3.0, 4.0};               // calls the constructor, NOT aggregate init
    // PointV2 p2b{.x=3.0, .y=4.0};     // ERROR: designated init requires aggregate

    // V3: depends on standard version
    // In C++17: PointV3 is aggregate  -> PointV3{5.0, 6.0} works
    // In C++20: PointV3 is NOT aggregate -> PointV3{5.0, 6.0} ERROR

    std::cout << std::boolalpha;
    std::cout << "V1 is aggregate: " << std::is_aggregate_v<PointV1> << "\n";   // true
    std::cout << "V2 is aggregate: " << std::is_aggregate_v<PointV2> << "\n";   // false
    std::cout << "V3 is aggregate: " << std::is_aggregate_v<PointV3> << "\n";   // false in C++20
}
```

**How it works:**

- `PointV1` has no constructors - aggregate. Brace init fills members directly.
- `PointV2` has a user-provided constructor - not an aggregate. `{3.0, 4.0}` calls the constructor rather than aggregate-initializing. Designated initializers are rejected.
- `PointV3` in C++20: `= default` is user-**declared**, so the struct is no longer aggregate. This is a breaking change from C++17.
- Use `std::is_aggregate_v<T>` (C++17) to test whether a type is an aggregate.

---

## Notes

- **Brace elision** allows flattening nested aggregates: `Outer o = {1, 2, 3};` instead of `{{1, 2}, 3}`.
- Aggregate initialization guarantees **left-to-right evaluation** of initializers.
- For JSON-like configuration structs, aggregates + designated initializers are extremely convenient - they provide named parameters without any constructor boilerplate.
- `std::is_aggregate_v<T>` was added in C++17 and is the authoritative way to check.
- Union types can also be aggregates - designated initializers can select which union member to initialize.
