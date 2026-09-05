# Know the rules of structured bindings (C++17)

**Category:** Core Language Fundamentals  
**Standard:** C++17  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP17.md#structured-bindings>  

---

## Topic Overview

**Structured bindings** (C++17) let you unpack objects into individual named variables
in a single declaration. Think of it as the language finally giving you a clean way to name
the pieces of a pair, tuple, or struct without writing `.first`, `.second`, or `get<0>()` everywhere.

### Unpacking Pairs and Tuples

The most common place you'll reach for structured bindings is when dealing with `std::pair`
and `std::tuple`. Here's what the three most common patterns look like together:

```cpp
#include <tuple>
#include <map>

// From std::pair:
std::pair<std::string, int> get_person() { return {"Alice", 30}; }
auto [name, age] = get_person();
std::cout << name << " is " << age << "\n"; // Alice is 30

// From std::tuple:
std::tuple<int, double, std::string> get_data() { return {42, 3.14, "hello"}; }
auto [id, value, label] = get_data();

// From std::map insert (pair<iterator, bool>):
std::map<std::string, int> scores;
auto [it, inserted] = scores.insert({"Alice", 95});
if (inserted) {
    std::cout << "Added: " << it->first << " = " << it->second << "\n";
}
```

Notice that the map insert case is particularly useful - you get both the iterator and the
success flag in one clean line instead of having to call `.first` and `.second`.

### Unpacking Structs (Aggregates)

Structured bindings work on plain aggregate structs too, with no extra machinery needed:

```cpp
struct Point { double x; double y; double z; };

Point get_origin() { return {0.0, 0.0, 0.0}; }
auto [x, y, z] = get_origin();
std::cout << x << ", " << y << ", " << z; // 0, 0, 0
```

The binding names map to the struct members in declaration order - so make sure your binding
names match the mental model of what those members mean.

### Reference Bindings

You control whether the binding copies or aliases the source. Here's the difference side by side:

```cpp
std::pair<std::string, int> person{"Alice", 30};

// By value - copies the pair's elements:
auto [name1, age1] = person;
name1 = "Bob"; // does NOT modify person

// By reference - aliases the pair's elements:
auto& [name2, age2] = person;
name2 = "Charlie"; // modifies person.first!
std::cout << person.first; // "Charlie"

// By const reference - read-only aliases:
const auto& [name3, age3] = person;
// name3 = "Dave";  // ERROR: name3 is const
```

The rule of thumb: use `const auto&` when reading, `auto&` when you need to modify through
the binding, and plain `auto` when you want an independent copy.

### In Range-For Loops (Most Common Use)

This is where structured bindings really shine in everyday code. Iterating over a map used
to require `.first` and `.second` everywhere:

```cpp
std::map<std::string, std::vector<int>> grades = {
    {"Alice", {90, 85, 92}},
    {"Bob",   {78, 82, 88}},
};

for (const auto& [name, scores] : grades) {
    double avg = 0;
    for (int s : scores) avg += s;
    avg /= scores.size();
    std::cout << name << ": " << avg << "\n";
}
```

Using `const auto&` here avoids copying the vector of scores on each iteration and makes the
intent obvious: we're reading, not modifying.

### With Arrays

Structured bindings also work on C-style arrays, as long as the number of names matches
the array size exactly:

```cpp
int arr[3] = {10, 20, 30};
auto [a, b, c] = arr;  // a=10, b=20, c=30

// Number of bindings must match:
// auto [x, y] = arr;  // ERROR: 3 elements but only 2 bindings
```

### Custom Types

To make your own class work with structured bindings, you need to teach the compiler three
things: how many elements there are, what type each one has, and how to get each one.
Here's the complete pattern:

```cpp
#include <tuple>

class Color {
    uint8_t r_, g_, b_;
public:
    Color(uint8_t r, uint8_t g, uint8_t b) : r_(r), g_(g), b_(b) {}
    uint8_t r() const { return r_; }
    uint8_t g() const { return g_; }
    uint8_t b() const { return b_; }
};

// Specialize these in namespace std:
template<> struct std::tuple_size<Color> : std::integral_constant<size_t, 3> {};
template<size_t I> struct std::tuple_element<I, Color> { using type = uint8_t; };

template<size_t I>
uint8_t get(const Color& c) {
    if constexpr (I == 0) return c.r();
    else if constexpr (I == 1) return c.g();
    else return c.b();
}

// Now you can use structured bindings:
Color red{255, 0, 0};
auto [r, g, b] = red;  // r=255, g=0, b=0
```

Once you provide those three hooks, the compiler treats your type just like a tuple.

---

## Self-Assessment

### Q1: Use structured bindings to destructure a `std::pair`, `std::tuple`, and a custom struct

This example walks through all the major cases together so you can see how consistent the
syntax is across different source types:

```cpp
#include <iostream>
#include <string>
#include <tuple>
#include <utility>
#include <map>

struct Point3D { double x, y, z; };

std::pair<std::string, int> get_person() { return {"Alice", 30}; }
std::tuple<int, double, std::string> get_record() { return {1, 99.5, "pass"}; }
Point3D get_origin() { return {0.0, 0.0, 0.0}; }

int main() {
    // 1. std::pair
    auto [name, age] = get_person();
    std::cout << name << " is " << age << "\n";  // Alice is 30

    // 2. std::tuple
    auto [id, score, status] = get_record();
    std::cout << "ID=" << id << " Score=" << score << " Status=" << status << "\n";

    // 3. Custom struct (aggregate)
    auto [x, y, z] = get_origin();
    std::cout << "Origin: " << x << ", " << y << ", " << z << "\n";

    // 4. In map iteration (pair<const Key, Value>)
    std::map<std::string, int> scores = {{"Alice", 95}, {"Bob", 87}};
    for (const auto& [student, grade] : scores) {
        std::cout << student << ": " << grade << "\n";
    }

    // 5. With map insert (pair<iterator, bool>)
    auto [it, inserted] = scores.emplace("Charlie", 91);
    std::cout << (inserted ? "Added" : "Exists") << ": " << it->first << "\n";

    return 0;
}
```

### Q2: Explain why `auto& [a, b] = pair;` makes `a` and `b` references to the elements

The key is understanding that the compiler creates a hidden intermediate variable. Here's
what's actually happening under the hood, and then a proof by mutation:

```cpp
#include <iostream>
#include <utility>

int main() {
    std::pair<std::string, int> person{"Alice", 30};

    // auto& [name, age] = person;
    // The compiler generates something like:
    //   auto& __hidden = person;        // __hidden is a reference to person
    //   decltype(auto) name = __hidden.first;   // reference to person.first
    //   decltype(auto) age  = __hidden.second;  // reference to person.second

    // BY REFERENCE:
    auto& [name_ref, age_ref] = person;
    name_ref = "Bob";    // Modifies person.first!
    age_ref = 25;        // Modifies person.second!
    std::cout << person.first << " " << person.second << "\n";  // Bob 25

    // BY VALUE (copy):
    auto [name_copy, age_copy] = person;
    name_copy = "Charlie";  // Does NOT modify person
    std::cout << person.first << "\n";  // Still "Bob"

    // The key insight:
    // - `auto [a,b] = expr;`  -> hidden variable is a COPY of expr
    // - `auto& [a,b] = expr;` -> hidden variable is a REFERENCE to expr
    // - `auto&& [a,b] = expr;`-> hidden variable uses forwarding reference rules
    // The bindings (a, b) always refer to the HIDDEN variable's members.

    // Proof: modifying through by-value binding doesn't affect original
    std::pair p{1, 2};
    auto [x, y] = p;
    x = 100;
    std::cout << p.first << "\n";  // 1 (unchanged)

    auto& [rx, ry] = p;
    rx = 100;
    std::cout << p.first << "\n";  // 100 (changed!)

    return 0;
}
```

### Q3: Show how structured bindings interact with `const`

`const` on a structured binding applies to the hidden variable, and that `const`-ness
propagates through to all the individual bindings. Watch how this plays out in practice:

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <map>

int main() {
    std::pair<std::string, int> person{"Alice", 30};

    // const auto& - read-only references
    const auto& [name1, age1] = person;
    // name1 = "Bob";  // ERROR: name1 is const std::string&
    // age1 = 25;      // ERROR: age1 is const int&
    std::cout << name1 << "\n";  // OK to read

    // const auto - const copy
    const auto [name2, age2] = person;
    // name2 = "Bob";  // ERROR: name2 is const
    // But person is unaffected regardless (it's a copy)

    // auto& with a const pair - bindings inherit const
    const std::pair<std::string, int> const_pair{"Bob", 40};
    auto& [n, a] = const_pair;
    // n = "X";  // ERROR: n is const std::string& (because const_pair is const)

    // Mutable reference to non-const pair
    auto& [m_name, m_age] = person;
    m_name = "Modified";  // OK
    std::cout << person.first << "\n";  // "Modified"

    // In range-for: prefer const auto& for read-only access
    std::map<std::string, int> m = {{"A", 1}, {"B", 2}};
    for (const auto& [key, val] : m) {
        // key and val are const references - cannot modify
        std::cout << key << "=" << val << "\n";
    }

    // Mutable iteration:
    for (auto& [key, val] : m) {
        // key is still const (map keys are always const)
        // val is mutable
        val *= 10;
    }
    // m is now {{"A", 10}, {"B", 20}}

    return 0;
}
```

**Key rules:**

- `const auto& [a, b]` - both bindings are const references (cannot modify).
- `auto& [a, b]` - bindings are mutable references (if the source isn't const).
- In a `std::map`, the key is always `const Key` even with `auto&`.
- `const` on structured bindings applies to the hidden variable, which propagates to all bindings.

---

## Notes

- Structured bindings work with: pairs, tuples, arrays, and aggregate structs.
- For non-aggregate classes, specialize `std::tuple_size`, `std::tuple_element`, and provide `get<I>()`.
- Use `const auto&` in range-for for read-only access, `auto&` for mutation.
- The number of binding names must exactly match the number of elements.
- C++20 allows structured bindings in lambdas: `[&a = x.first]` etc.
