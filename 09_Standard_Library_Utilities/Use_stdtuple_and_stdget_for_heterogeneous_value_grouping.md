# Use std::tuple and std::get for heterogeneous value grouping

**Category:** Standard Library Utilities  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/tuple>  

---

## Topic Overview

`std::tuple` (header `<tuple>`, C++11) is a fixed-size, heterogeneous container that groups values of different types. It generalizes `std::pair` to any number of elements. Combined with `std::get`, `std::apply`, structured bindings (C++17), and `std::make_from_tuple` (C++17), it forms a powerful toolkit for multi-value returns, generic programming, and heterogeneous data passing.

You'll reach for `tuple` most often when you want to return multiple values from a function without creating a dedicated struct. For quick one-off groupings - especially in generic code - it's often exactly the right tool.

### Key Operations

| Operation | Standard | Description |
| --- | --- | --- |
| `std::tuple<T1,T2,...>` | C++11 | Type definition |
| `std::make_tuple(a,b,c)` | C++11 | Create tuple (deduces types) |
| `std::get<N>(t)` | C++11 | Access Nth element (0-based) |
| `std::get<T>(t)` | C++14 | Access by type (must be unique) |
| `std::tie(a,b,c) = t` | C++11 | Unpack into existing vars |
| `auto [a,b,c] = t` | C++17 | Structured binding |
| `std::tuple_size_v<T>` | C++11 | Number of elements |
| `std::tuple_element_t<N,T>` | C++11 | Type of Nth element |
| `std::apply(f, t)` | C++17 | Call f(t0, t1, ..., tn) |
| `std::make_from_tuple<T>(t)` | C++17 | Construct T from tuple args |

### Core Example

Here is the basic vocabulary: creating a tuple, accessing by index and by type, and using structured bindings to name the elements.

```cpp
#include <tuple>
#include <string>
#include <iostream>

int main() {
    // Create a tuple
    auto t = std::make_tuple(42, 3.14, std::string("hello"));
    // Type: std::tuple<int, double, std::string>

    // Access by index
    std::cout << std::get<0>(t) << "\n"; // 42
    std::cout << std::get<1>(t) << "\n"; // 3.14
    std::cout << std::get<2>(t) << "\n"; // "hello"

    // Access by type (C++14, type must be unique)
    std::cout << std::get<double>(t) << "\n"; // 3.14

    // Structured bindings (C++17)
    auto [num, pi, greeting] = t;
    std::cout << num << ", " << pi << ", " << greeting << "\n";
    // Output: 42, 3.14, hello

    // Comparison (lexicographic)
    auto t1 = std::make_tuple(1, 2, 3);
    auto t2 = std::make_tuple(1, 2, 4);
    std::cout << std::boolalpha << (t1 < t2) << "\n"; // true
}
```

---

## Self-Assessment

### Q1: Write a function that returns multiple values using std::tuple and destructure with auto

**Answer:**

This is probably the most common use of `tuple` in modern C++. The trick is that C++17 structured bindings make the unpacking side look just as clean as the return side.

```cpp
#include <tuple>
#include <string>
#include <iostream>
#include <cmath>

// Function returns multiple heterogeneous values
std::tuple<bool, double, std::string> solve_quadratic(double a, double b, double c) {
    double discriminant = b * b - 4 * a * c;

    if (discriminant < 0) {
        return {false, 0.0, "no real roots"};
    }

    double root1 = (-b + std::sqrt(discriminant)) / (2 * a);
    double root2 = (-b - std::sqrt(discriminant)) / (2 * a);

    if (discriminant == 0) {
        return {true, root1, "one root (repeated)"};
    }
    return {true, root1, "two roots: x1=" + std::to_string(root1)
                          + " x2=" + std::to_string(root2)};
}

// Multiple return values for database lookups
std::tuple<int, std::string, bool> find_user(int id) {
    // Simulated lookup
    if (id == 1) return {1, "Alice", true};
    if (id == 2) return {2, "Bob", false};
    return {-1, "", false}; // not found
}

int main() {
    // === Structured bindings (C++17) - cleanest syntax ===
    auto [found, root, message] = solve_quadratic(1.0, -5.0, 6.0);
    std::cout << std::boolalpha;
    std::cout << "Found: " << found << ", root: " << root
              << ", msg: " << message << "\n";
    // Output: Found: true, root: 3, msg: two roots: x1=3.000000 x2=2.000000

    // === std::tie - unpack into existing variables ===
    int id;
    std::string name;
    bool active;
    std::tie(id, name, active) = find_user(1);
    std::cout << "User: " << id << ", " << name << ", active=" << active << "\n";
    // Output: User: 1, Alice, active=true

    // === std::ignore - skip unwanted fields ===
    std::tie(std::ignore, name, std::ignore) = find_user(2);
    std::cout << "Name only: " << name << "\n";
    // Output: Name only: Bob

    // === Direct std::get access ===
    auto result = solve_quadratic(1.0, 0.0, 1.0); // x^2+1=0, no real roots
    if (!std::get<0>(result)) {
        std::cout << std::get<2>(result) << "\n"; // "no real roots"
    }
}
```

`std::tuple` is ideal for returning multiple values when creating a dedicated struct would be overkill. C++17 structured bindings (`auto [a, b, c] = func()`) make the syntax clean. For pre-C++17 code, `std::tie` unpacks into existing variables. `std::get<N>` provides index-based access.

### Q2: Use std::apply to call a function with a tuple's elements as arguments

**Answer:**

`std::apply` is the key to making tuples useful for deferred calls and generic dispatch. It unpacks the tuple and passes each element as a separate argument to the callable.

```cpp
#include <tuple>
#include <iostream>
#include <string>
#include <functional>

// A regular function with multiple parameters
void log_event(int severity, const std::string& source, const std::string& message) {
    const char* levels[] = {"DEBUG", "INFO", "WARN", "ERROR"};
    std::cout << "[" << levels[severity] << "] " << source << ": " << message << "\n";
}

// Computing with numeric tuples
double compute_volume(double length, double width, double height) {
    return length * width * height;
}

int main() {
    // === std::apply: expand tuple into function arguments ===
    auto event = std::make_tuple(2, std::string("Network"), std::string("Connection lost"));
    std::apply(log_event, event);
    // Output: [WARN] Network: Connection lost

    auto dimensions = std::make_tuple(3.0, 4.0, 5.0);
    double vol = std::apply(compute_volume, dimensions);
    std::cout << "Volume: " << vol << "\n";
    // Output: Volume: 60

    // === With lambdas ===
    auto args = std::make_tuple(10, 20, 30);
    int sum = std::apply([](int a, int b, int c) { return a + b + c; }, args);
    std::cout << "Sum: " << sum << "\n";
    // Output: Sum: 60

    // === Generic tuple printer ===
    auto print_all = [](const auto&... args) {
        ((std::cout << args << " "), ...); // fold expression
        std::cout << "\n";
    };
    std::apply(print_all, std::make_tuple(1, "hello", 3.14, 'x'));
    // Output: 1 hello 3.14 x

    // === Practical: deferred function call ===
    // Store function + arguments, call later
    auto deferred = std::make_tuple(std::string("ALERT"), 42);
    // ... later ...
    std::apply([](const std::string& tag, int code) {
        std::cout << tag << ": error code " << code << "\n";
    }, deferred);
    // Output: ALERT: error code 42

    // HOW std::apply works (conceptually):
    // template<typename F, typename Tuple>
    // auto apply(F&& f, Tuple&& t) {
    //     return std::apply_impl(std::forward<F>(f), std::forward<Tuple>(t),
    //         std::make_index_sequence<std::tuple_size_v<std::decay_t<Tuple>>>{});
    // }
    // It uses index_sequence to expand get<0>(t), get<1>(t), ... as arguments.
}
```

`std::apply` (C++17) takes a callable and a tuple, then calls the function with the tuple elements unpacked as individual arguments. This is essential for generic programming where you have arguments stored in a tuple (e.g., deferred calls, serialized function parameters, variadic forwarding).

### Q3: Show how std::make_from_tuple constructs an object from a tuple of constructor arguments

**Answer:**

`std::make_from_tuple` is the construction counterpart to `std::apply`. Where `apply` calls a function, `make_from_tuple` constructs an object - both by unpacking the tuple into the call.

```cpp
#include <tuple>
#include <string>
#include <iostream>
#include <memory>
#include <vector>

struct Widget {
    int id;
    std::string name;
    double weight;

    Widget(int i, std::string n, double w)
        : id(i), name(std::move(n)), weight(w)
    {
        std::cout << "Widget(" << id << ", " << name << ", " << weight << ")\n";
    }
};

struct Point3D {
    double x, y, z;
    Point3D(double x, double y, double z) : x(x), y(y), z(z) {}
};

int main() {
    // === Construct Widget from a tuple of constructor args ===
    auto args = std::make_tuple(1, std::string("Gizmo"), 2.5);
    auto w = std::make_from_tuple<Widget>(std::move(args));
    // Output: Widget(1, Gizmo, 2.5)
    std::cout << "Created: " << w.name << "\n";

    // === Factory pattern: create objects from stored argument tuples ===
    std::vector<std::tuple<double, double, double>> point_specs{
        {1.0, 2.0, 3.0},
        {4.0, 5.0, 6.0},
        {7.0, 8.0, 9.0},
    };

    std::vector<Point3D> points;
    for (const auto& spec : point_specs) {
        points.push_back(std::make_from_tuple<Point3D>(spec));
    }

    for (const auto& p : points) {
        std::cout << "(" << p.x << ", " << p.y << ", " << p.z << ")\n";
    }
    // Output:
    // (1, 2, 3)
    // (4, 5, 6)
    // (7, 8, 9)

    // === With std::pair (also works) ===
    auto pair_args = std::make_tuple(std::string("key"), 42);
    auto p = std::make_from_tuple<std::pair<std::string, int>>(pair_args);
    std::cout << p.first << " = " << p.second << "\n";
    // Output: key = 42

    // === Why make_from_tuple matters: generic factory ===
    // In template code, you may receive constructor args as a tuple
    // (e.g., from deserialization or configuration). make_from_tuple
    // lets you construct any type without knowing how many args there are.

    // Conceptual implementation:
    // template<class T, class Tuple>
    // T make_from_tuple(Tuple&& t) {
    //     return std::apply([](auto&&... args) {
    //         return T(std::forward<decltype(args)>(args)...);
    //     }, std::forward<Tuple>(t));
    // }
}
```

`std::make_from_tuple<T>(tuple)` (C++17) constructs a `T` by unpacking the tuple as constructor arguments. It's the construction counterpart to `std::apply` (which calls functions). This enables generic factories, deserialization frameworks, and any pattern where constructor arguments are stored as data.

---

## Notes

- **`std::tuple` vs `struct`:** Prefer named structs for code clarity. Use tuples for quick multi-returns, template metaprogramming, and generic utilities.
- **`std::get<T>` (C++14):** Access by type (`std::get<double>(t)`) only works if the type appears exactly once in the tuple.
- **`std::tuple_cat`:** Concatenates multiple tuples: `auto t = std::tuple_cat(t1, t2, t3);`
- **Performance:** `std::tuple` is a value type with no overhead beyond its members. Compilers optimize away the tuple wrapper in most cases.
- **`std::pair` is a special case** of `std::tuple` with exactly 2 elements. They are interconvertible.
- Compile with `-std=c++17 -Wall -Wextra`.
