# Use designated initializers (C++20) for self-documenting struct initialization

**Category:** Best Practices and Idioms  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/aggregate_initialization>  

---

## Topic Overview

**Designated initializers** (C++20) let you name the fields you're initializing when constructing an aggregate, so the initialization reads like a short description of what you're setting up. You also get a compile-time guarantee: if someone reorders the struct fields, or if you misspell a field name, the compiler tells you immediately rather than silently shuffling values into the wrong slots.

Here's the contrast at a glance:

```cpp
// Before (C++17): what does true, false, 42 mean?
Config c{true, false, 42, 3.14f};

// After (C++20): crystal clear!
Config c{.verbose = true, .debug = false, .max_retries = 42, .timeout = 3.14f};
```

The second version is self-documenting. A reader doesn't need to look up the struct definition to understand what's being set.

---

## Self-Assessment

### Q1: Rewrite positional initialization using designated initializers

Compare the two versions of `Config` initialization below. The positional form (`bad`) forces you to remember the exact order of all five fields. The designated form (`good`) makes the intent obvious and lets you skip fields you're happy leaving at their defaults.

```cpp
#include <iostream>
#include <string>

struct Config {
    bool verbose = false;
    bool debug = false;
    int max_retries = 3;
    float timeout = 5.0f;
    std::string host = "localhost";
};

struct NetworkRequest {
    std::string url;
    std::string method = "GET";
    int timeout_ms = 5000;
    bool follow_redirects = true;
    int max_retries = 3;
};

int main() {
    // BAD: positional - what does true, false, 42 mean?
    Config bad{true, false, 42, 3.14f, "server.com"};

    // GOOD: designated initializers - self-documenting!
    Config good{
        .verbose = true,
        .debug = false,
        .max_retries = 42,
        .timeout = 3.14f,
        .host = "server.com"
    };

    // Can skip fields (they use defaults):
    Config minimal{.verbose = true, .max_retries = 10};
    // .debug = false, .timeout = 5.0f, .host = "localhost" (defaults)

    // Great for complex structs:
    NetworkRequest req{
        .url = "https://api.example.com/data",
        .method = "POST",
        .timeout_ms = 3000,
        .max_retries = 5
    };

    std::cout << "Host: " << good.host << '\n';
    std::cout << "Retries: " << minimal.max_retries << '\n';
    std::cout << "URL: " << req.url << '\n';
}
// Expected output:
// Host: server.com
// Retries: 10
// URL: https://api.example.com/data
```

Notice that `minimal` only sets two fields and silently inherits the rest from the default member initializers in the struct. That combination - designated initializers plus default member initializers - is what gives you the "named parameter" pattern in C++.

### Q2: Show that designated initializers catch accidental field reordering

This is the safety win that makes designated initializers more than just a readability nicety. With positional initialization, swapping `x` and `y` compiles without any warning. With designated initializers, putting the names out of declaration order is a hard compile error.

```cpp
#include <iostream>

struct Point {
    int x;
    int y;
    int z;
};

int main() {
    // Positional: reordering x,y is silent bug
    Point p1{10, 20, 30};     // x=10, y=20, z=30
    Point p2{20, 10, 30};     // x=20, y=10 - swapped! No warning.

    // Designated: reordering is a COMPILE ERROR
    Point p3{.x = 10, .y = 20, .z = 30};  // OK

    // This would NOT compile (out of order):
    // Point p4{.y = 20, .x = 10, .z = 30};  // ERROR!
    // C++ requires designators in declaration order.

    // Also catches typos:
    // Point p5{.x = 10, .w = 20, .z = 30};  // ERROR: no member 'w'

    std::cout << "p3 = (" << p3.x << ", " << p3.y << ", " << p3.z << ")\n";
}
// Expected output:
// p3 = (10, 20, 30)
```

The "must be in declaration order" rule is stricter than C's version of designated initializers, but it's consistent and predictable. If you need `.y` before `.x`, C++ is telling you to look at the struct and decide which order actually makes sense.

### Q3: Explain restrictions on designated initializers

Designated initializers only work on **aggregate types** - structs (or classes) with no user-declared constructors, no private members, no virtual functions, and no base classes that would disqualify them as aggregates. If you add a constructor, you're no longer an aggregate, and designated initialization is off the table.

```cpp
#include <iostream>

// AGGREGATE: no user-declared constructors, no private members,
// no virtual functions, no base classes (relaxed in C++17)
struct Aggregate {
    int x;
    int y;
    int z = 0;
};

// NOT an aggregate: has constructor
struct NonAggregate {
    int x, y;
    NonAggregate(int a, int b) : x(a), y(b) {}
};

int main() {
    // OK: Aggregate type
    Aggregate a{.x = 1, .y = 2};

    // ERROR: NonAggregate can't use designated initializers
    // NonAggregate na{.x = 1, .y = 2};  // won't compile

    std::cout << a.x << ' ' << a.y << ' ' << a.z << '\n';
}
// Expected output:
// 1 2 0
```

If the table below feels like a lot, the one rule to remember is: C++ requires designators in declaration order and doesn't support nested or array designators. It's a deliberate subset of C's version.

**Restrictions (C++ vs C):**

| Rule | C++ | C |
| --- | --- | --- |
| Must be in declaration order | Yes | No |
| Can skip members | Yes (use defaults) | Yes |
| Nested designators (`.a.b = 1`) | No | Yes |
| Array designators (`[0] = 1`) | No | Yes |
| Mix designated + positional | No | Yes |
| Requires aggregate type | Yes | Yes |

---

## Notes

- Designated initializers are a subset of C's designated initializers - C++ is stricter, but that strictness prevents some subtle bugs.
- Combine with default member initializers for a "named parameter" effect: set only what you need and let everything else default.
- Great for configuration structs, option bags, and builder-like patterns where most fields have sensible defaults.
- Zero runtime overhead - the compiler generates identical code to positional initialization.
