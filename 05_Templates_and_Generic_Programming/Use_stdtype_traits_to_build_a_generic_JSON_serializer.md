# Use `std::type_traits` to Build a Generic JSON Serializer

**Category:** Templates and Generic Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/header/type_traits>  

---

## Topic Overview

### Building a Type-Trait-Based JSON Serializer

The goal is a `to_json(T)` function that automatically serializes any supported type to a JSON string, using type traits and `if constexpr` to select the encoding. This is a practical example where you can see type introspection directly solving a real problem:

```cpp
to_json(42)          -> "42"
to_json(3.14)        -> "3.14"
to_json(true)        -> "true"
to_json("hello")     -> "\"hello\""
to_json(vector{1,2}) -> "[1,2]"
to_json(MyType{})    -> calls MyType::to_json() if it exists
```

### Type Detection Strategy

| JSON Type | C++ Type Detection |
| --- | --- |
| `null` | `std::is_null_pointer_v<T>` |
| `boolean` | `std::is_same_v<T, bool>` |
| `number` (int) | `std::is_integral_v<T> && !is_same<T,bool>` |
| `number` (float) | `std::is_floating_point_v<T>` |
| `string` | `std::is_convertible_v<T, std::string>` or `std::string`/`string_view` |
| `array` | Has `.begin()`, `.end()`, `.size()` (range-like) |
| `object` | Custom - user-defined `to_json()` method |

---

## Self-Assessment

### Q1: Detect array, object, string, number, and bool types using type traits in a `serialize<T>` function

The important ordering point here is that `bool` is an integral type in C++, so if you check `is_integral` before `is_same<T, bool>` you will serialize `true` as `"1"` instead of `"true"`. The `if constexpr` chain must handle the more specific case first.

```cpp
#include <iostream>
#include <string>
#include <sstream>
#include <vector>
#include <map>
#include <type_traits>
#include <concepts>

// === Type detection traits ===

// Is it a string-like type?
template <typename T>
concept StringLike = std::is_convertible_v<T, std::string_view>;

// Is it a range/container (has begin/end)?
template <typename T>
concept RangeLike = requires(T t) {
    t.begin();
    t.end();
} && !StringLike<T>;  // strings have begin/end but should be treated as strings

// Has a custom to_json() member?
template <typename T>
concept HasToJson = requires(const T& t) {
    { t.to_json() } -> std::convertible_to<std::string>;
};

// Is it a map-like container (has key_type and mapped_type)?
template <typename T>
concept MapLike = requires {
    typename T::key_type;
    typename T::mapped_type;
} && RangeLike<T>;

// === Forward declare ===
template <typename T>
std::string to_json(const T& value);

// === The serializer ===
template <typename T>
std::string to_json(const T& value) {
    if constexpr (std::is_null_pointer_v<T>) {
        return "null";
    }
    else if constexpr (std::is_same_v<std::decay_t<T>, bool>) {
        return value ? "true" : "false";
    }
    else if constexpr (std::is_integral_v<T>) {
        return std::to_string(value);
    }
    else if constexpr (std::is_floating_point_v<T>) {
        std::ostringstream oss;
        oss << value;
        return oss.str();
    }
    else if constexpr (StringLike<T>) {
        return "\"" + std::string(value) + "\"";
    }
    else if constexpr (MapLike<T>) {
        std::string result = "{";
        bool first = true;
        for (const auto& [k, v] : value) {
            if (!first) result += ",";
            result += to_json(k) + ":" + to_json(v);
            first = false;
        }
        return result + "}";
    }
    else if constexpr (RangeLike<T>) {
        std::string result = "[";
        bool first = true;
        for (const auto& elem : value) {
            if (!first) result += ",";
            result += to_json(elem);
            first = false;
        }
        return result + "]";
    }
    else if constexpr (HasToJson<T>) {
        return value.to_json();
    }
    else {
        static_assert(!sizeof(T*), "Unsupported type for JSON serialization");
    }
}

int main() {
    std::cout << "=== Primitive types ===\n";
    std::cout << "null:   " << to_json(nullptr) << "\n";
    std::cout << "bool:   " << to_json(true) << "\n";
    std::cout << "int:    " << to_json(42) << "\n";
    std::cout << "double: " << to_json(3.14) << "\n";
    std::cout << "string: " << to_json(std::string("hello")) << "\n";

    std::cout << "\n=== Containers ===\n";
    std::vector<int> nums = {1, 2, 3};
    std::cout << "vector: " << to_json(nums) << "\n";

    std::vector<std::string> words = {"a", "b"};
    std::cout << "strings: " << to_json(words) << "\n";

    std::map<std::string, int> obj = {{"x", 1}, {"y", 2}};
    std::cout << "map:    " << to_json(obj) << "\n";

    std::cout << "\n=== Nested ===\n";
    std::vector<std::vector<int>> matrix = {{1,2},{3,4}};
    std::cout << "nested: " << to_json(matrix) << "\n";

    return 0;
}
// Expected output:
//   null:   null
//   bool:   true
//   int:    42
//   double: 3.14
//   string: "hello"
//   vector: [1,2,3]
//   strings: ["a","b"]
//   map:    {"x":1,"y":2}
//   nested: [[1,2],[3,4]]
```

The `MapLike` concept is checked before `RangeLike` because a `std::map` satisfies both - the more specific one must win.

### Q2: Use `if constexpr` chains to select the correct JSON encoding for each detected category

The key insight is that **`if constexpr` is evaluated at compile time** - only the matching branch is compiled for any given type. This matters because some branches contain code that would not even compile for the wrong type, such as calling `std::to_string` on a `std::string`.

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <vector>
#include <sstream>

// Simplified serializer showing if constexpr branching
template <typename T>
std::string json_encode(const T& value) {
    // Order matters! Check bool BEFORE integral (bool is integral)
    if constexpr (std::is_same_v<std::decay_t<T>, bool>) {
        // Branch 1: bool -> "true"/"false"
        return value ? "true" : "false";
    }
    else if constexpr (std::is_integral_v<T>) {
        // Branch 2: integers -> digits
        return std::to_string(value);
    }
    else if constexpr (std::is_floating_point_v<T>) {
        // Branch 3: floating point -> decimal
        std::ostringstream oss;
        oss << value;
        return oss.str();
    }
    else if constexpr (std::is_convertible_v<T, std::string_view>) {
        // Branch 4: string-like -> quoted
        return "\"" + std::string(value) + "\"";
    }
    else {
        // Branch 5: unsupported -> compile-time error
        static_assert(!sizeof(T*), "Type not supported");
    }
}

// Why order matters:
// bool is_integral -> true!  So we MUST check bool first
// const char* is_convertible_to<string_view> -> true

int main() {
    std::cout << "=== if constexpr branching ===\n";
    std::cout << json_encode(true) << "\n";          // "true" (not "1"!)
    std::cout << json_encode(42) << "\n";             // "42"
    std::cout << json_encode(3.14) << "\n";           // "3.14"
    std::cout << json_encode(std::string("hi")) << "\n"; // "\"hi\""

    std::cout << "\n=== Why if constexpr, not regular if ===\n";
    std::cout << "Regular if: ALL branches must compile for ALL types\n";
    std::cout << "if constexpr: only the MATCHING branch is compiled\n";
    std::cout << "-> to_string(string) would fail in regular if, but\n";
    std::cout << "  with if constexpr, it's never instantiated for strings\n";

    return 0;
}
```

### Q3: Add a custom `to_json(T)` detection concept to allow opt-in user-defined serialization

This is the extensibility story. Rather than the serializer knowing every type up front, you let user types opt in either by providing a `.to_json()` member or a free function discoverable via ADL.

```cpp
#include <iostream>
#include <string>
#include <sstream>
#include <type_traits>
#include <concepts>
#include <vector>

// === Concept: detects if T has a to_json() member ===
template <typename T>
concept CustomJsonSerializable = requires(const T& t) {
    { t.to_json() } -> std::convertible_to<std::string>;
};

// === Concept: detects if there's a free function to_json(T) via ADL ===
template <typename T>
concept AdlJsonSerializable = requires(const T& t) {
    { to_json_adl(t) } -> std::convertible_to<std::string>;
};

// === Forward declare the main serializer ===
template <typename T>
std::string to_json(const T& value);

// === User types that opt-in ===
struct Point {
    double x, y;
    // Member function opt-in
    std::string to_json() const {
        std::ostringstream oss;
        oss << "{\"x\":" << x << ",\"y\":" << y << "}";
        return oss.str();
    }
};

struct Color {
    int r, g, b;
    // Friend function opt-in (ADL-discoverable)
    friend std::string to_json_adl(const Color& c) {
        return "{\"r\":" + std::to_string(c.r) +
               ",\"g\":" + std::to_string(c.g) +
               ",\"b\":" + std::to_string(c.b) + "}";
    }
};

struct Rect {
    Point origin;
    double width, height;
    std::string to_json() const {
        return "{\"origin\":" + origin.to_json() +
               ",\"w\":" + std::to_string(width) +
               ",\"h\":" + std::to_string(height) + "}";
    }
};

// === The serializer with user-type support ===
template <typename T>
std::string to_json(const T& value) {
    if constexpr (std::is_same_v<std::decay_t<T>, bool>) {
        return value ? "true" : "false";
    }
    else if constexpr (std::is_integral_v<T>) {
        return std::to_string(value);
    }
    else if constexpr (std::is_floating_point_v<T>) {
        std::ostringstream oss;
        oss << value;
        return oss.str();
    }
    else if constexpr (std::is_convertible_v<T, std::string_view>) {
        return "\"" + std::string(value) + "\"";
    }
    else if constexpr (CustomJsonSerializable<T>) {
        // User type with .to_json() member
        return value.to_json();
    }
    else if constexpr (AdlJsonSerializable<T>) {
        // User type with free function to_json_adl()
        return to_json_adl(value);
    }
    else if constexpr (requires { value.begin(); value.end(); }) {
        std::string result = "[";
        bool first = true;
        for (const auto& elem : value) {
            if (!first) result += ",";
            result += to_json(elem);
            first = false;
        }
        return result + "]";
    }
    else {
        static_assert(!sizeof(T*), "Type not supported for JSON serialization");
    }
}

int main() {
    std::cout << "=== Custom types with to_json() member ===\n";
    Point p{1.5, 2.5};
    std::cout << "Point: " << to_json(p) << "\n";

    Rect r{Point{0, 0}, 100, 50};
    std::cout << "Rect:  " << to_json(r) << "\n";

    std::cout << "\n=== Custom types with ADL friend ===\n";
    Color c{255, 128, 0};
    std::cout << "Color: " << to_json(c) << "\n";

    std::cout << "\n=== Mixed: vector of custom types ===\n";
    std::vector<Point> points = {{0,0}, {1,1}, {2,3}};
    std::cout << "Points: " << to_json(points) << "\n";

    std::cout << "\n=== Concept checks ===\n";
    std::cout << "Point is CustomJsonSerializable: "
              << CustomJsonSerializable<Point> << "\n";        // 1
    std::cout << "int is CustomJsonSerializable: "
              << CustomJsonSerializable<int> << "\n";           // 0
    std::cout << "Color is AdlJsonSerializable: "
              << AdlJsonSerializable<Color> << "\n";            // 1

    return 0;
}
// Expected output:
//   Point: {"x":1.5,"y":2.5}
//   Rect:  {"origin":{"x":0,"y":0},"w":100.000000,"h":50.000000}
//   Color: {"r":255,"g":128,"b":0}
//   Points: [{"x":0,"y":0},{"x":1,"y":1},{"x":2,"y":3}]
```

The two opt-in patterns - member function and ADL friend - give users flexibility. If the type is in third-party code you cannot modify, you can provide a `to_json_adl` free function in the same namespace. If it is your own type, the member function is usually cleaner.

---

## Notes

- `if constexpr` selects the serialization branch at compile time - non-matching branches aren't compiled.
- Check `bool` before `integral` - `bool` satisfies `is_integral`!
- Use concepts (`CustomJsonSerializable`) or SFINAE to detect user-defined serialization hooks.
- Two opt-in patterns: **member function** `T::to_json()` and **ADL friend** `to_json_adl(T)`.
- `static_assert(!sizeof(T*), "msg")` in the `else` branch gives a readable error for unsupported types.
- For production use, consider nlohmann/json which uses similar trait-based detection internally.
