# Know std::in_place_t and std::in_place_type_t for in-place construction

**Category:** Type System and Deduction  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/in_place>  

---

## Topic Overview

The `std::in_place` family of tags tells containers to **construct objects directly in their storage** instead of creating a temporary and moving it in. These tags carry no data - they exist purely to select a different constructor overload that forwards arguments straight to the contained object.

### The Tag Types

| Tag | Purpose | Used With |
| --- | --- | --- |
| `std::in_place` / `std::in_place_t` | Construct in-place (no type disambiguation needed) | `std::optional`, `std::expected` |
| `std::in_place_type<T>` / `std::in_place_type_t<T>` | Specify which type to construct | `std::variant`, `std::any` |
| `std::in_place_index<I>` / `std::in_place_index_t<I>` | Specify which alternative by index | `std::variant` |

### Why In-Place Construction Matters

Without `in_place`, you're constructing a temporary and then moving it into the container's storage - that's two operations. With `in_place`, you forward the constructor arguments directly into the container's internal buffer:

```cpp
#include <optional>
#include <string>

struct Heavy {
    std::string data;
    Heavy(std::string d) : data(std::move(d)) {
        std::cout << "Constructed\n";
    }
    Heavy(const Heavy&) { std::cout << "Copied\n"; }
    Heavy(Heavy&&) { std::cout << "Moved\n"; }
};

// Without in_place: construct temporary, then move it in
std::optional<Heavy> opt1 = Heavy("hello");
// Output: Constructed, Moved (or copy-elided in C++17)

// With in_place: construct directly inside optional's storage
std::optional<Heavy> opt2(std::in_place, "hello");
// Output: Constructed (no move at all!)
```

The difference matters most for non-movable types, or any type with an expensive move. For trivially copyable types the compiler will usually optimize the move away anyway.

### With std::variant

`variant` needs the extra `in_place_type<T>` or `in_place_index<I>` tags because it can hold multiple alternative types and needs to know which one you're constructing:

```cpp
#include <variant>
#include <string>

// variant with ambiguous types - need to specify which one
std::variant<int, double, std::string> v1(std::in_place_type<std::string>, 5, 'A');
// Constructs string("AAAAA") directly in variant's storage

// By index instead of type:
std::variant<int, double, std::string> v2(std::in_place_index<2>, "hello");
// Index 2 = std::string
```

---

## Self-Assessment

### Q1: Construct a `std::optional<MyType>` in-place using `std::in_place`

Here you can see all three ways to get in-place construction with `optional` - the constructor with `std::in_place`, and the `emplace()` member called after construction. The important thing to notice is that the constructor arguments go directly after the tag:

```cpp
#include <iostream>
#include <optional>
#include <string>
#include <vector>

struct Connection {
    std::string host;
    int port;
    std::vector<std::string> options;

    Connection(std::string h, int p, std::initializer_list<std::string> opts)
        : host(std::move(h)), port(p), options(opts) {
        std::cout << "Connection constructed: " << host << ":" << port << "\n";
    }
};

int main() {
    // 1. Without in_place - temporary + move
    // std::optional<Connection> conn1 = Connection("localhost", 8080, {"opt1"});

    // 2. With in_place - construct directly inside optional
    std::optional<Connection> conn2(std::in_place, "localhost", 8080,
                                     std::initializer_list<std::string>{"ssl", "compress"});

    if (conn2) {
        std::cout << "Host: " << conn2->host << "\n";
        std::cout << "Port: " << conn2->port << "\n";
        std::cout << "Options: ";
        for (const auto& opt : conn2->options) std::cout << opt << " ";
        std::cout << "\n";
    }

    // 3. emplace() also constructs in-place (after construction)
    std::optional<Connection> conn3;
    conn3.emplace("example.com", 443, std::initializer_list<std::string>{"https"});

    // 4. No unnecessary copies or moves!
    // Output: only "Connection constructed" messages, no "Copied" or "Moved"
}
```

`std::in_place` is a tag that selects the in-place constructor overload. The arguments after `std::in_place` are forwarded directly to `Connection`'s constructor, and the object is constructed directly in `optional`'s internal aligned storage - zero copies, zero moves.

### Q2: Emplace into a `std::variant` using `std::in_place_type<T>`

`variant` requires the type tag because a bare argument like `42` would be ambiguous if the variant holds both `int` and `double`. With `in_place_type<double>`, you're unambiguous and also get direct construction:

```cpp
#include <iostream>
#include <variant>
#include <string>
#include <complex>

int main() {
    // Problem: variant<int, double> - passing 42 is ambiguous
    // (could be int or double)

    // Solution: use in_place_type to specify exactly which type
    std::variant<int, double, std::string> v1(std::in_place_type<double>, 42);
    std::cout << "v1 holds double: " << std::get<double>(v1) << "\n";  // 42.0

    // Construct a string with (count, char) constructor
    std::variant<int, double, std::string> v2(std::in_place_type<std::string>, 5, 'X');
    std::cout << "v2 holds string: " << std::get<std::string>(v2) << "\n";  // XXXXX

    // Using in_place_index instead (by position)
    std::variant<int, double, std::string> v3(std::in_place_index<0>, 99);
    std::cout << "v3 holds int: " << std::get<0>(v3) << "\n";  // 99

    // With std::any - in_place_type disambiguates and avoids moves
    // std::any a(std::in_place_type<std::complex<double>>, 3.0, 4.0);

    // Emplace after construction
    std::variant<int, double, std::string> v4;
    v4.emplace<std::string>("constructed in-place");
    std::cout << "v4: " << std::get<std::string>(v4) << "\n";

    // emplace by index
    v4.emplace<1>(3.14);  // index 1 = double
    std::cout << "v4 now holds double: " << std::get<double>(v4) << "\n";
}
```

`std::in_place_type<T>` tells the variant which alternative to activate and construct. Arguments are forwarded to `T`'s constructor, avoiding temporary creation. `std::in_place_index<I>` does the same but uses the position in the variant's type list.

### Q3: Explain why in-place construction avoids the move/copy of passing a temporary

This example uses a `Tracker` type that logs every constructor and destructor call. Run through the three blocks mentally before reading the output - the "without in_place" path has an extra move and an extra destruction:

```cpp
#include <iostream>
#include <optional>
#include <string>

struct Tracker {
    std::string name;

    Tracker(std::string n) : name(std::move(n)) {
        std::cout << "  Constructed: " << name << "\n";
    }
    Tracker(const Tracker& o) : name(o.name) {
        std::cout << "  Copied: " << name << "\n";
    }
    Tracker(Tracker&& o) noexcept : name(std::move(o.name)) {
        std::cout << "  Moved: " << name << "\n";
    }
    ~Tracker() {
        std::cout << "  Destroyed: " << name << "\n";
    }
};

int main() {
    std::cout << "=== Without in_place (temporary + move) ===\n";
    {
        // Step 1: Tracker("Alpha") creates a temporary
        // Step 2: optional's constructor moves it into internal storage
        // Step 3: temporary is destroyed
        std::optional<Tracker> opt = Tracker("Alpha");
        // Note: C++17 mandatory copy elision may optimize this,
        // but the standard only guarantees elision for prvalues
    }

    std::cout << "\n=== With in_place (direct construction) ===\n";
    {
        // Only ONE construction - directly inside optional's storage
        // No temporary, no move, no extra destruction
        std::optional<Tracker> opt(std::in_place, "Beta");
    }

    std::cout << "\n=== emplace (same as in_place, but after construction) ===\n";
    {
        std::optional<Tracker> opt;  // empty - no construction
        opt.emplace("Gamma");         // construct directly in storage
    }
}
```

**Output (conceptual):**

```text
=== Without in_place ===
  Constructed: Alpha
  Moved: Alpha          <- extra move!
  Destroyed:            <- temporary destroyed
  Destroyed: Alpha

=== With in_place ===
  Constructed: Beta     <- only this! No move.
  Destroyed: Beta

=== emplace ===
  Constructed: Gamma    <- only this!
  Destroyed: Gamma
```

`in_place` forwards constructor arguments through the container's constructor to construct the object directly in the container's internal buffer - no temporary object is ever created.

---

## Notes

- `emplace()` does the same thing as `in_place` but for assignment after construction.
- `std::in_place` is a `constexpr` variable of type `std::in_place_t` - it's just a tag, carries no data.
- `std::expected<T, E>` (C++23) also supports `std::in_place` for the value and `std::unexpect` for the error.
- `std::any` uses `std::in_place_type<T>` because it can hold any type and needs to know which one to construct.
- Performance benefit is greatest for non-movable types or types with expensive moves - for trivially copyable types, the compiler usually optimizes away the move anyway.
