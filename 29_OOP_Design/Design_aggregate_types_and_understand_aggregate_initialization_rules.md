# Design aggregate types and understand aggregate initialization rules

**Category:** OOP Design  
**Standard:** C++20  

---

## Topic Overview

An **aggregate** is a class or struct with no user-declared constructors, no private or protected non-static data members, no virtual functions, and no virtual base classes. What you get in exchange for those restrictions is the ability to initialize the type directly with braces - no constructor call, just values listed in order. This makes aggregates extremely clean for plain data types, configuration objects, and return types.

The rules have evolved across standard versions, so it's worth knowing what changed when:

### Aggregate rules by standard version

| Feature | C++11/14 | C++17 | C++20 |
| --- | :---: | :---: | :---: |
| User-declared constructors | Not allowed | Not allowed | Not allowed |
| Inherited constructors | Disqualify | OK | OK |
| Brace/designated init | `{a, b, c}` | `{a, b, c}` | `{.x=1, .y=2}` |
| Base classes | None allowed | Allowed | Allowed |
| Default member init | No | Yes | Yes |

The most impactful addition in C++20 is **designated initializers**, which let you name the fields you're setting. This makes initialization self-documenting and lets you skip fields that have defaults.

### Designated Initializers (C++20)

Here's what designated initialization looks like in practice. You only name the fields you care about; the rest use their default values.

```cpp
struct Config {
    std::string host = "localhost";
    int port = 8080;
    bool ssl = false;
};
Config c{.port = 443, .ssl = true};  // host uses default
```

---

## Self-Assessment

### Q1: Show aggregate initialization and its gotchas

Aggregate initialization is straightforward for the happy path, but there are two gotchas worth knowing before they bite you: the narrowing-conversion ban and the designated-initializer ordering rule.

**Answer:**

```cpp
#include <string>
#include <array>
#include <iostream>

struct Point { int x, y, z; };

struct ServerConfig {
    std::string host = "localhost";
    int port = 8080;
    int max_connections = 100;
    bool debug = false;
};

int main() {
    // Basic aggregate init
    Point p1{1, 2, 3};
    Point p2{1, 2};     // z = 0 (value-initialized)
    Point p3{};          // All zero

    // C++20 designated initializers
    ServerConfig cfg1{.port = 443, .debug = true};
    // host = "localhost" (default), max_connections = 100 (default)

    // GOTCHA: order must match declaration order
    // ServerConfig bad{.debug = true, .port = 443};  // ERROR!

    // GOTCHA: narrowing conversions are errors in braces
    // Point p4{1.5, 2.5, 3.5};  // ERROR: double -> int narrows

    // Nested aggregates
    struct Line { Point start, end; };
    Line l1{{0,0,0}, {1,1,1}};      // Nested brace init
    Line l2{.start{1,2,3}, .end{4,5,6}};  // C++20

    // std::array is an aggregate!
    std::array<int, 4> arr{1, 2, 3, 4};

    std::cout << cfg1.host << ":" << cfg1.port << "\n";
    return 0;
}
```

The ordering rule trips people up because it feels artificial. Designated initializers must appear in declaration order because C++ initializes members in declaration order - there's no reordering under the hood. If you reverse `.debug` and `.port` the compiler rejects it immediately.

### Q2: Show what disqualifies a type from being an aggregate

The reason to care about aggregate status is practical: you lose brace initialization the moment you declare a constructor, even `= default`. This is a breaking change introduced in C++20 that affects code that worked under C++17.

**Answer:**

```cpp
#include <iostream>

// Aggregate
struct Good {
    int x;
    int y = 10;  // Default member init OK since C++14
};

// NOT aggregate: user-declared constructor
struct Bad1 {
    int x;
    Bad1() = default;  // C++17/20: even "= default" disqualifies!
    // In C++11/14: "= default" was OK for aggregates
};

// NOT aggregate: private member
struct Bad2 {
private:
    int x;
public:
    int y;
};

// NOT aggregate: virtual function
struct Bad3 {
    virtual void foo() {}
    int x;
};

// Aggregate with base class (C++17+)
struct Base { int a; };
struct Derived : Base { int b; };
// Derived d{{42}, 99};  // C++17: base subobject in braces

// Check at compile time
#include <type_traits>
static_assert(std::is_aggregate_v<Good>);
// static_assert(std::is_aggregate_v<Bad1>);  // FAILS
static_assert(std::is_aggregate_v<Derived>);  // C++17+

int main() {
    Good g1{1, 2};     // Aggregate init
    Good g2{.x = 1};   // C++20 designated
    Derived d{{10}, 20};
    std::cout << d.a << " " << d.b << "\n";
    return 0;
}
```

The `= default` story in C++20 is the most surprising part here. In C++17, writing `MyClass() = default;` kept the type as an aggregate. In C++20, any user-declared constructor - even a defaulted one - disqualifies it. If you need both a user-declared constructor and aggregate-like initialization in C++20, you have to pick one or the other. Use `std::is_aggregate_v<T>` in a `static_assert` to catch problems at compile time rather than discovering them through mysterious initialization failures.

### Q3: Use aggregates for configuration and structured bindings

Aggregates shine as named-parameter objects (solving the problem that C++ doesn't have keyword arguments) and as lightweight multi-value return types paired with structured bindings.

**Answer:**

```cpp
#include <string>
#include <vector>
#include <iostream>
#include <optional>

// Aggregate as a named parameter object
struct ConnectionOpts {
    std::string host = "localhost";
    int port = 5432;
    std::string database = "mydb";
    std::string user = "postgres";
    std::optional<std::string> password;
    int timeout_ms = 5000;
    int max_retries = 3;
    bool ssl = false;
};

void connect(const ConnectionOpts& opts) {
    std::cout << "Connecting to " << opts.user << "@"
              << opts.host << ":" << opts.port
              << "/" << opts.database
              << " ssl=" << opts.ssl << "\n";
}

// Aggregate return + structured bindings
struct ParseResult {
    bool success;
    std::string value;
    int error_code;
};

ParseResult parse(const std::string& input) {
    if (input.empty())
        return {false, "", -1};
    return {true, input, 0};
}

int main() {
    // Designated initializers = readable configuration
    connect({
        .host = "db.production.com",
        .port = 5433,
        .database = "analytics",
        .user = "app_service",
        .password = "secret123",
        .ssl = true
    });

    // Structured bindings with aggregate return
    auto [ok, value, code] = parse("hello");
    if (ok) std::cout << "Parsed: " << value << "\n";
    return 0;
}
```

The `ConnectionOpts` pattern is a clean replacement for long argument lists. Callers see exactly which fields they're setting and can skip everything they want to leave at default. Compare this to `connect("db.production.com", 5433, "analytics", "app_service", "secret123", 5000, 3, true)` where the reader has no idea what position means what.

---

## Notes

- C++20 designated initializers are the best alternative to named parameters in C++.
- Aggregates are initialized **in declaration order** - designated init must follow this order too.
- `std::is_aggregate_v<T>` lets you check aggregate status at compile time.
- Brace initialization catches narrowing (`double` to `int`) - this is a feature, not a bug.
- In C++20, `= default` on a constructor **disqualifies** a type from being an aggregate (breaking change from C++17).
- Aggregates combined with structured bindings make excellent multi-return values.
