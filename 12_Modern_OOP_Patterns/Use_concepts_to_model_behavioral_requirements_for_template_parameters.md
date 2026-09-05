# Use concepts to model behavioral requirements for template parameters

**Category:** Modern OOP Patterns  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/concepts>  

---

## Topic Overview

**Concepts** (C++20) let you express *what operations a type must support* directly in the template signature. Before concepts, the only way to do this was with SFINAE tricks that were painful to write and produced nearly unreadable error messages. Concepts give you clear, compile-time "interface contracts" for generic code - think of them as a virtual interface that costs nothing at runtime.

### Concept = Compile-Time Interface

Here is a side-by-side comparison of the two approaches to say "this type must support serialization":

```cpp
Virtual Interface (runtime):         Concept (compile-time):
┌────────────────────────┐          template <typename T>
│ class ISerializable {   │          concept Serializable = requires(T t) {
│   virtual bytes         │              { t.to_bytes() } -> same_as<vector<uint8_t>>;
│     to_bytes() = 0;     │              { T::from_bytes(span<uint8_t>{}) } ->
│   virtual void          │                                        same_as<T>;
│     from_bytes(span) =0;│          };
│ };                       │
└────────────────────────┘
Must inherit from ISerializable      Just needs the right methods.
Runtime vtable overhead.             Zero overhead. Checked at compile time.
```

The virtual interface forces types to explicitly opt in by inheriting. The concept just checks that the right methods exist - any type that happens to have them qualifies automatically.

---

## Self-Assessment

### Q1: Define a `Serializable` concept requiring `.to_bytes() -> std::vector<uint8_t>` and `.from_bytes(span)`

The `requires` expression is the heart of a concept. It lets you spell out exactly what expressions must be valid and what they must return. Here is how to write the `Serializable` concept and verify it at compile time:

```cpp
#include <iostream>
#include <vector>
#include <span>
#include <concepts>
#include <cstring>
#include <string>

// Define the concept
template <typename T>
concept Serializable = requires(const T& ct, std::span<const uint8_t> bytes) {
    { ct.to_bytes() } -> std::same_as<std::vector<uint8_t>>;
    { T::from_bytes(bytes) } -> std::same_as<T>;
};

// A type that satisfies Serializable
struct Point {
    float x, y;

    std::vector<uint8_t> to_bytes() const {
        std::vector<uint8_t> result(sizeof(float) * 2);
        std::memcpy(result.data(), &x, sizeof(float));
        std::memcpy(result.data() + sizeof(float), &y, sizeof(float));
        return result;
    }

    static Point from_bytes(std::span<const uint8_t> data) {
        Point p{};
        std::memcpy(&p.x, data.data(), sizeof(float));
        std::memcpy(&p.y, data.data() + sizeof(float), sizeof(float));
        return p;
    }
};

// A type that does NOT satisfy Serializable
struct Color {
    int r, g, b;
    // No to_bytes() or from_bytes() -- fails concept check
};

// Compile-time verification
static_assert(Serializable<Point>, "Point must be Serializable");
// static_assert(Serializable<Color>);  // COMPILE ERROR

int main() {
    Point p{3.14f, 2.72f};
    auto bytes = p.to_bytes();
    std::cout << "Serialized " << bytes.size() << " bytes\n";

    auto p2 = Point::from_bytes(bytes);
    std::cout << "Deserialized: (" << p2.x << ", " << p2.y << ")\n";
}
// Expected output:
//   Serialized 8 bytes
//   Deserialized: (3.14, 2.72)
```

The `static_assert` lines are your friend here - they turn concept checks into documentation that the compiler enforces. If `Point` ever loses one of those methods, you get an error immediately.

---

### Q2: Constrain a generic `serialize<T>` function using the `Serializable` concept

Once you have defined a concept, constraining a function template is just a matter of replacing `typename` with the concept name. There are four equivalent syntaxes - the comments show all of them:

```cpp
#include <iostream>
#include <vector>
#include <span>
#include <concepts>
#include <cstring>
#include <fstream>
#include <string>

template <typename T>
concept Serializable = requires(const T& ct, std::span<const uint8_t> bytes) {
    { ct.to_bytes() } -> std::same_as<std::vector<uint8_t>>;
    { T::from_bytes(bytes) } -> std::same_as<T>;
};

// Constrained function -- only accepts Serializable types
template <Serializable T>
void save_to_file(const T& obj, const std::string& filename) {
    auto bytes = obj.to_bytes();
    std::ofstream file(filename, std::ios::binary);
    file.write(reinterpret_cast<const char*>(bytes.data()), bytes.size());
    std::cout << "Saved " << bytes.size() << " bytes to " << filename << "\n";
}

template <Serializable T>
T load_from_file(const std::string& filename) {
    std::ifstream file(filename, std::ios::binary);
    std::vector<uint8_t> bytes(std::istreambuf_iterator<char>(file), {});
    return T::from_bytes(bytes);
}

// Alternative syntax options (all equivalent):
// 1. template <Serializable T> void f(T obj);
// 2. void f(Serializable auto obj);
// 3. template <typename T> requires Serializable<T> void f(T obj);
// 4. template <typename T> void f(T obj) requires Serializable<T>;

struct Config {
    int width, height;

    std::vector<uint8_t> to_bytes() const {
        std::vector<uint8_t> result(sizeof(int) * 2);
        std::memcpy(result.data(), &width, sizeof(int));
        std::memcpy(result.data() + sizeof(int), &height, sizeof(int));
        return result;
    }

    static Config from_bytes(std::span<const uint8_t> data) {
        Config c{};
        std::memcpy(&c.width, data.data(), sizeof(int));
        std::memcpy(&c.height, data.data() + sizeof(int), sizeof(int));
        return c;
    }
};

struct NotSerializable {
    int value;
};

int main() {
    Config cfg{1920, 1080};
    save_to_file(cfg, "config.bin");
    auto loaded = load_from_file<Config>("config.bin");
    std::cout << "Loaded: " << loaded.width << "x" << loaded.height << "\n";

    // save_to_file(NotSerializable{42}, "bad.bin");
    // COMPILE ERROR: constraints not satisfied
}
// Expected output:
//   Saved 8 bytes to config.bin
//   Loaded: 1920x1080
```

Pick whichever syntax reads most naturally for your situation. The `Serializable auto` form in syntax 2 is especially tidy for simple one-parameter functions.

---

### Q3: Show the improved error message when passing a non-Serializable type vs a raw template

This is one of the biggest practical wins from concepts. Without them, a mismatch produces a wall of template instantiation noise. With them, the error points right at the problem. Here are both:

```cpp
// Without concepts (raw template):
template <typename T>
void save_raw(const T& obj, const std::string& filename) {
    auto bytes = obj.to_bytes();  // Error deep inside template body
    // ...
}

struct Bad {};
// save_raw(Bad{}, "file.bin");

// ERROR (GCC, no concepts):
//   error: 'struct Bad' has no member named 'to_bytes'
//   note: in instantiation of function template 'save_raw<Bad>'
//   note: ... 20 lines of template instantiation backtrace ...
```

```cpp
// WITH concepts:
template <Serializable T>
void save_concept(const T& obj, const std::string& filename) {
    auto bytes = obj.to_bytes();
    // ...
}

// save_concept(Bad{}, "file.bin");

// ERROR (GCC, with concepts):
//   error: use of function 'save_concept<T>' with unsatisfied constraints
//   note: because 'Bad' does not satisfy 'Serializable'
//   note: because 'ct.to_bytes()' would be invalid: no member named 'to_bytes'
```

The concept error tells you *what is missing*, not just *where it broke*. That is a meaningful difference when you are debugging someone else's type.

**Error message comparison:**

| Aspect | Raw Template | With Concept |
| --- | --- | --- |
| Error location | Deep inside template body | At the call site |
| Message clarity | "no member named..." (generic) | "'Bad' does not satisfy 'Serializable'" |
| Tells you what's needed? | No - just what failed | Yes - names the requirement |
| Backtrace depth | 10-50 lines of instantiation notes | 2-3 lines |

You can make the messages even more targeted by splitting a compound concept into named pieces. That way the error can tell you exactly *which* requirement failed:

```cpp
// Composing concepts for even better messages:
#include <concepts>

template <typename T>
concept HasToBytes = requires(const T& t) {
    { t.to_bytes() } -> std::same_as<std::vector<uint8_t>>;
};

template <typename T>
concept HasFromBytes = requires(std::span<const uint8_t> data) {
    { T::from_bytes(data) } -> std::same_as<T>;
};

template <typename T>
concept Serializable2 = HasToBytes<T> && HasFromBytes<T>;

// Now errors pinpoint EXACTLY which part is missing:
// "does not satisfy HasToBytes" or "does not satisfy HasFromBytes"
```

---

## Notes

- **Concepts replace SFINAE** for constraining templates - much cleaner and more maintainable.
- **Standard concepts** in `<concepts>`: `std::integral`, `std::floating_point`, `std::movable`, `std::copyable`, `std::regular`, `std::invocable`, etc.
- **Range concepts** in `<ranges>`: `std::ranges::range`, `std::ranges::input_range`, `std::ranges::random_access_range`, etc.
- **Concepts don't add runtime cost** - they're purely a compile-time mechanism. The generated code is identical to unconstrained templates.
- **Subsumption:** If two overloads match, the compiler prefers the MORE constrained one. This replaces complex SFINAE priority ordering.
- **Concepts vs virtual interfaces:** Concepts check structural conformance (duck typing), while virtual interfaces require explicit inheritance. Concepts are zero-overhead but compile-time only.
