# Use `std::variant` and `std::visit` for Type-Safe Unions

**Category:** Type System and Deduction  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

### What Is `std::variant`

`std::variant<Types...>` is a type-safe union that holds exactly one value from a fixed set of types. Unlike C unions, it tracks which type is currently active and prevents type-punning. You can think of it as a tagged union where the tag is managed automatically and reading the wrong member is a compile or runtime error rather than silent undefined behavior.

### Core API

Here's a quick tour of the main operations - creation, access, safe checking, and querying which type is active:

```cpp
#include <variant>
#include <string>
#include <iostream>

using Value = std::variant<int, double, std::string>;

Value v = 42;                          // holds int
v = 3.14;                             // now holds double
v = std::string("hello");             // now holds string

// Access by type
std::string& s = std::get<std::string>(v);

// Access by index
auto& s2 = std::get<2>(v);            // index 2 = string

// Safe check
if (auto* p = std::get_if<int>(&v)) {
    std::cout << "int: " << *p;        // p is nullptr if wrong type
}

// Which alternative is active?
v.index();  // returns 2 (for string)
std::holds_alternative<std::string>(v);  // true
```

### `std::visit` - The Visitor Pattern

`std::visit` is how you write code that handles every possible type in the variant. Pass it a callable that accepts any of the alternatives and it dispatches to the right one at runtime:

```cpp
std::visit([](auto&& arg) {
    std::cout << arg << "\n";
}, v);

// With type dispatch:
std::visit([](auto&& arg) {
    using T = std::decay_t<decltype(arg)>;
    if constexpr (std::is_same_v<T, int>)
        std::cout << "int: " << arg;
    else if constexpr (std::is_same_v<T, double>)
        std::cout << "double: " << arg;
    else
        std::cout << "string: " << arg;
}, v);
```

### The Overload Pattern

Writing a single generic lambda with `if constexpr` works, but it gets verbose. A cleaner approach is to combine multiple lambdas using the overload pattern - a small helper that inherits all their `operator()` overloads into one type:

```cpp
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

// C++17 deduction guide (not needed in C++20):
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Usage:
std::visit(overloaded{
    [](int i)         { std::cout << "int: " << i; },
    [](double d)      { std::cout << "double: " << d; },
    [](const std::string& s) { std::cout << "string: " << s; }
}, v);
```

The nice part about the overload pattern is that if you forget to handle one of the variant's alternatives, the compiler tells you - it's a compile error, not a runtime surprise.

### `std::monostate`

If the first type in a variant has no default constructor, the variant itself can't be default-constructed. `std::monostate` is an empty marker type that solves this:

```cpp
struct NoDefault {
    NoDefault(int) {}  // no default ctor
};

// std::variant<NoDefault, int> v;  // ERROR: can't default-construct NoDefault
std::variant<std::monostate, NoDefault, int> v;  // OK: defaults to monostate
```

---

## Self-Assessment

### Q1: Replace a tagged union struct with `std::variant` and show how `std::visit` eliminates type-punning

The old-style tagged union requires you to keep the tag and the active union member in sync manually. `std::variant` does that bookkeeping for you and makes accessing the wrong member a well-defined error instead of UB.

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <cassert>

// === OLD WAY: manual tagged union (C-style) ===
struct OldValue {
    enum Tag { INT, DOUBLE, STRING } tag;
    union {
        int i;
        double d;
        // Can't have std::string in a union (pre-C++11)!
        char str_buf[64];  // ugly workaround
    };
};

void print_old(const OldValue& v) {
    switch (v.tag) {
        case OldValue::INT:    std::cout << "int: " << v.i; break;
        case OldValue::DOUBLE: std::cout << "double: " << v.d; break;
        case OldValue::STRING: std::cout << "string: " << v.str_buf; break;
        // No compiler check for missing cases!
        // Accessing wrong member = undefined behavior (type-punning)
    }
    std::cout << "\n";
}

// === NEW WAY: std::variant (C++17) ===
using NewValue = std::variant<int, double, std::string>;

void print_new(const NewValue& v) {
    std::visit([](const auto& arg) {
        std::cout << "value: " << arg << "\n";
    }, v);
}

int main() {
    // Old way: easy to misuse
    OldValue old;
    old.tag = OldValue::INT;
    old.i = 42;
    print_old(old);

    // Danger: nothing prevents accessing wrong member
    // double bad = old.d;  // UB! reading double from int member

    // New way: type-safe
    NewValue v = 42;
    print_new(v);              // "value: 42"

    v = 3.14;
    print_new(v);              // "value: 3.14"

    v = std::string("hello");
    print_new(v);              // "value: hello"

    // Safe access:
    std::cout << "\nSafe access:\n";
    std::cout << "holds int? " << std::holds_alternative<int>(v) << "\n";
    std::cout << "holds string? " << std::holds_alternative<std::string>(v) << "\n";

    // get_if returns nullptr for wrong type (no UB):
    if (auto* p = std::get_if<int>(&v)) {
        std::cout << "int: " << *p << "\n";
    } else {
        std::cout << "Not an int\n";
    }

    // std::get throws if wrong type:
    try {
        int i = std::get<int>(v);  // throws std::bad_variant_access
    } catch (const std::bad_variant_access& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    return 0;
}
```

Notice that the old code's `char str_buf[64]` hack is completely gone - `std::variant` can hold a real `std::string` with no workarounds. And the `std::get<int>` throw at the end shows that even "wrong type" accesses have defined behavior, rather than silently reading garbage bytes.

**Output:**

```text
int: 42
value: 42
value: 3.14
value: hello

Safe access:
holds int? 0
holds string? 1
Not an int
Caught: bad variant access
```

### Q2: Write an overloaded visitor using the overload pattern and verify all alternatives are handled

The overload pattern really shines when you want each type to have its own handler. Here it's used with a basic value set and also with a simple expression tree to show how `std::visit` can act as a dispatch table for AST nodes:

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <vector>
#include <cmath>

// The overload pattern
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

// C++17 deduction guide
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// A simple expression tree
struct Literal { double value; };
struct Variable { std::string name; };
struct Add {};
struct Multiply {};

using Expr = std::variant<Literal, Variable, Add, Multiply>;

int main() {
    // === Basic overloaded visitor ===
    using Value = std::variant<int, double, std::string, bool>;

    std::vector<Value> values = {42, 3.14, std::string("hello"), true};

    std::cout << "=== Basic overloaded visitor ===\n";
    for (const auto& v : values) {
        std::visit(overloaded{
            [](int i)                { std::cout << "int: " << i << "\n"; },
            [](double d)             { std::cout << "double: " << d << "\n"; },
            [](const std::string& s) { std::cout << "string: \"" << s << "\"\n"; },
            [](bool b)               { std::cout << "bool: " << std::boolalpha << b << "\n"; }
            // If any alternative is NOT handled, this is a COMPILE ERROR
        }, v);
    }

    // === Expression tree visitor ===
    std::cout << "\n=== Expression tree ===\n";
    std::vector<Expr> program = {
        Literal{3.14},
        Variable{"x"},
        Add{},
        Literal{2.0},
        Multiply{}
    };

    for (const auto& expr : program) {
        std::visit(overloaded{
            [](const Literal& lit)  { std::cout << "PUSH " << lit.value << "\n"; },
            [](const Variable& var) { std::cout << "LOAD " << var.name << "\n"; },
            [](const Add&)          { std::cout << "ADD\n"; },
            [](const Multiply&)     { std::cout << "MUL\n"; }
        }, expr);
    }

    // === Return values from visit ===
    std::cout << "\n=== visit with return ===\n";
    Value v = 42;
    std::string desc = std::visit(overloaded{
        [](int i)            { return "integer " + std::to_string(i); },
        [](double d)         { return "float " + std::to_string(d); },
        [](const std::string& s) { return "text \"" + s + "\""; },
        [](bool b)           { return std::string(b ? "true" : "false"); }
    }, v);
    std::cout << "Description: " << desc << "\n";

    // === Compile-time completeness check ===
    // Uncommenting the line below (removing bool handler) causes a compile error:
    // std::visit(overloaded{
    //     [](int i)            { std::cout << i; },
    //     [](double d)         { std::cout << d; },
    //     [](const std::string& s) { std::cout << s; }
    //     // Missing bool handler -> COMPILE ERROR
    // }, values[0]);

    return 0;
}
```

The expression tree example is worth studying: each node type in `Expr` maps to exactly one lambda, and `std::visit` dispatches to the right one at runtime. This is the standard "variant as algebraic data type" pattern.

**Output:**

```text
=== Basic overloaded visitor ===
int: 42
double: 3.14
string: "hello"
bool: true

=== Expression tree ===
PUSH 3.14
LOAD x
ADD
PUSH 2
MUL

=== visit with return ===
Description: integer 42
```

### Q3: Explain what `std::monostate` is for and demonstrate its use in a default-constructible variant

Here's the concrete problem: you want a `variant` that can start out "empty" before a real value is assigned, but none of the alternative types are default-constructible. `std::monostate` gives you a well-typed empty state to put first in the list.

```cpp
#include <iostream>
#include <variant>
#include <string>

// A type with no default constructor
struct DatabaseConnection {
    std::string host;
    int port;
    DatabaseConnection(std::string h, int p) : host(std::move(h)), port(p) {
        std::cout << "  Connected to " << host << ":" << port << "\n";
    }
    // No default constructor!
};

struct FileHandle {
    std::string path;
    FileHandle(std::string p) : path(std::move(p)) {
        std::cout << "  Opened " << path << "\n";
    }
    // No default constructor!
};

// This would NOT compile:
// std::variant<DatabaseConnection, FileHandle> resource;
// ERROR: first type (DatabaseConnection) has no default constructor

// Solution: use std::monostate as the first alternative
using Resource = std::variant<std::monostate, DatabaseConnection, FileHandle>;

// Overload pattern
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

void describe(const Resource& r) {
    std::visit(overloaded{
        [](std::monostate) {
            std::cout << "  Resource: [empty/uninitialized]\n";
        },
        [](const DatabaseConnection& db) {
            std::cout << "  Resource: DB " << db.host << ":" << db.port << "\n";
        },
        [](const FileHandle& fh) {
            std::cout << "  Resource: File " << fh.path << "\n";
        }
    }, r);
}

int main() {
    std::cout << std::boolalpha;

    // Default construction -> monostate (no resource allocated)
    Resource r;
    std::cout << "After default construction:\n";
    describe(r);
    std::cout << "holds monostate: " << std::holds_alternative<std::monostate>(r) << "\n";

    // Assign a database connection
    std::cout << "\nAssigning database connection:\n";
    r = DatabaseConnection{"localhost", 5432};
    describe(r);

    // Replace with file handle
    std::cout << "\nReplacing with file handle:\n";
    r = FileHandle{"/var/log/app.log"};
    describe(r);

    // Reset to empty state
    std::cout << "\nResetting to monostate:\n";
    r = std::monostate{};
    describe(r);

    // monostate properties:
    std::monostate a, b;
    std::cout << "\n=== monostate properties ===\n";
    std::cout << "a == b: " << (a == b) << "\n";  // always true
    std::cout << "a < b:  " << (a < b) << "\n";   // always false
    // All monostate values are equal — it's a unit type

    return 0;
}
```

`std::monostate` is a **unit type** - it has exactly one value, and all instances compare equal. This makes it a clean "nothing here yet" sentinel that plays nicely with `std::visit` (there's exactly one handler for it) and with ordered containers (it compares consistently).

**Output:**

```text
After default construction:
  Resource: [empty/uninitialized]
holds monostate: true

Assigning database connection:
  Connected to localhost:5432
  Resource: DB localhost:5432

Replacing with file handle:
  Opened /var/log/app.log
  Resource: File /var/log/app.log

Resetting to monostate:
  Resource: [empty/uninitialized]

=== monostate properties ===
a == b: true
a < b:  false
```

---

## Notes

- **variant vs any vs optional:** `variant` = "one of these specific types" (closed set), `any` = "any type" (open set, type-erased), `optional` = "T or nothing" (one type or empty).
- **Exception on assignment:** If a variant assignment throws, the variant may enter the **valueless by exception** state (`variant.valueless_by_exception()` returns true). This is the only way a variant can hold no value.
- **Performance:** `std::visit` with small variant types is typically compiled to a jump table or switch - zero overhead vs manual switch statements.
- **C++20:** `std::visit` can accept a return type explicitly: `std::visit<ReturnType>(visitor, variant)`.
