# Use Template Specialization to Customize Standard Library Traits

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/partial_specialization>  

---

## Topic Overview

### Customizing Standard Library Traits

The C++ standard explicitly allows you to specialize certain standard library templates for your own types. This is the sanctioned way to integrate your types into the standard library ecosystem - making them work with hash maps, generic algorithms, or formatting facilities without any runtime overhead.

Here is the pattern at a glance:

```cpp
// Your type works with std::unordered_map after specializing std::hash:
template <>
struct std::hash<MyType> {
    size_t operator()(const MyType& t) const noexcept { ... }
};
```

### What Can Be Specialized

| Trait | Purpose | When to Specialize |
| --- | --- | --- |
| `std::hash<T>` | Enable use as `unordered_map`/`unordered_set` key | Custom types used in hash containers |
| `std::numeric_limits<T>` | Min/max/precision info for generic algorithms | Custom numeric types |
| `std::formatter<T>` (C++20) | Enable `std::format` / `std::print` | Custom types you want to format |
| `std::tuple_size<T>` / `std::tuple_element<I,T>` | Enable structured bindings | Types with fixed-size members |
| `std::iterator_traits<T>` | Iterator properties | Custom iterators |

### Rules for Specialization

- You may only specialize in **namespace `std`**
- The specialization must depend on at least one **user-defined type**
- You must **not** add new members beyond what the primary template defines
- All specializations in different TUs must be **the same** (ODR)

---

## Self-Assessment

### Q1: Specialize `std::hash<MyType>` to enable use of `MyType` as an `unordered_map` key

Without a `std::hash` specialization, trying to use your type as an `unordered_map` key gives a compile error buried inside the standard library headers - often with a confusing message. Once you provide the specialization, everything just works.

```cpp
#include <iostream>
#include <unordered_map>
#include <unordered_set>
#include <string>
#include <functional>

struct Point {
    int x, y;

    bool operator==(const Point& other) const = default;

    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        return os << "(" << p.x << "," << p.y << ")";
    }
};

// === Specialize std::hash for Point ===
template <>
struct std::hash<Point> {
    std::size_t operator()(const Point& p) const noexcept {
        // Combine hashes of members
        std::size_t h1 = std::hash<int>{}(p.x);
        std::size_t h2 = std::hash<int>{}(p.y);
        // Use a common combining technique
        return h1 ^ (h2 << 1);  // or boost::hash_combine style
    }
};

// === A more complex type ===
struct Employee {
    std::string name;
    int id;
    std::string department;

    bool operator==(const Employee& other) const = default;
};

template <>
struct std::hash<Employee> {
    std::size_t operator()(const Employee& e) const noexcept {
        std::size_t h1 = std::hash<std::string>{}(e.name);
        std::size_t h2 = std::hash<int>{}(e.id);
        std::size_t h3 = std::hash<std::string>{}(e.department);
        std::size_t result = h1;
        result ^= h2 + 0x9e3779b9 + (result << 6) + (result >> 2);
        result ^= h3 + 0x9e3779b9 + (result << 6) + (result >> 2);
        return result;
    }
};

int main() {
    std::cout << "=== Point in unordered_set ===\n";
    std::unordered_set<Point> points;
    points.insert({1, 2});
    points.insert({3, 4});
    points.insert({1, 2});  // duplicate — not inserted
    std::cout << "Set size: " << points.size() << "\n";  // 2
    for (const auto& p : points)
        std::cout << "  " << p << "\n";

    std::cout << "\n=== Point as unordered_map key ===\n";
    std::unordered_map<Point, std::string> labels;
    labels[{0, 0}] = "origin";
    labels[{1, 0}] = "right";
    labels[{0, 1}] = "up";
    for (const auto& [pt, label] : labels)
        std::cout << "  " << pt << " -> " << label << "\n";

    std::cout << "\n=== Employee in unordered_map ===\n";
    std::unordered_map<Employee, double> salaries;
    salaries[{"Alice", 1, "Eng"}] = 120000;
    salaries[{"Bob", 2, "Sales"}] = 95000;
    std::cout << "Employees: " << salaries.size() << "\n";

    return 0;
}
```

The hash combining formula for `Employee` is a common pattern borrowed from Boost. A naive XOR of all the hashes tends to produce poor distribution - the magic constant `0x9e3779b9` is derived from the golden ratio and helps spread bits around.

### Q2: Specialize `std::numeric_limits<MyDecimal>` to integrate a custom decimal type with generic algorithms

`std::numeric_limits` is what algorithms like `std::clamp`, `std::minmax_element`, and numeric code use to ask "what is the smallest or largest representable value?" Without a specialization for your type, those algorithms cannot work with it generically.

```cpp
#include <iostream>
#include <limits>
#include <cstdint>
#include <algorithm>
#include <vector>
#include <numeric>

// A fixed-point decimal: stores value * 100 internally
class Decimal {
    int64_t cents_;  // value * 100

public:
    constexpr Decimal() : cents_(0) {}
    constexpr explicit Decimal(double val) : cents_(static_cast<int64_t>(val * 100)) {}
    constexpr Decimal(int64_t whole, int64_t frac) : cents_(whole * 100 + frac) {}

    constexpr double to_double() const { return cents_ / 100.0; }
    constexpr int64_t raw() const { return cents_; }

    constexpr bool operator==(const Decimal&) const = default;
    constexpr auto operator<=>(const Decimal&) const = default;

    constexpr Decimal operator+(const Decimal& o) const { Decimal r; r.cents_ = cents_ + o.cents_; return r; }
    constexpr Decimal operator-(const Decimal& o) const { Decimal r; r.cents_ = cents_ - o.cents_; return r; }
    constexpr Decimal operator-() const { Decimal r; r.cents_ = -cents_; return r; }

    friend std::ostream& operator<<(std::ostream& os, const Decimal& d) {
        return os << d.to_double();
    }
};

// === Specialize std::numeric_limits for Decimal ===
template <>
struct std::numeric_limits<Decimal> {
    static constexpr bool is_specialized = true;
    static constexpr bool is_signed = true;
    static constexpr bool is_integer = false;
    static constexpr bool is_exact = true;  // fixed-point, not floating
    static constexpr bool has_infinity = false;
    static constexpr bool has_quiet_NaN = false;
    static constexpr int digits = 63;  // significant bits
    static constexpr int digits10 = 18;

    static constexpr Decimal min() noexcept { return Decimal(0, 1); }  // 0.01
    static constexpr Decimal max() noexcept {
        return Decimal(static_cast<double>(INT64_MAX) / 100.0);
    }
    static constexpr Decimal lowest() noexcept {
        return Decimal(static_cast<double>(INT64_MIN) / 100.0);
    }
    static constexpr Decimal epsilon() noexcept { return Decimal(0, 1); }  // 0.01
};

// === Generic algorithm that uses numeric_limits ===
template <typename T>
T clamp_to_range(T value) {
    return std::clamp(value,
                      std::numeric_limits<T>::lowest(),
                      std::numeric_limits<T>::max());
}

template <typename T>
void print_limits() {
    std::cout << "  min:     " << std::numeric_limits<T>::min() << "\n";
    std::cout << "  max:     " << std::numeric_limits<T>::max() << "\n";
    std::cout << "  epsilon: " << std::numeric_limits<T>::epsilon() << "\n";
    std::cout << "  signed:  " << std::numeric_limits<T>::is_signed << "\n";
}

int main() {
    std::cout << "=== numeric_limits<Decimal> ===\n";
    print_limits<Decimal>();

    std::cout << "\n=== Decimal with generic algorithms ===\n";
    std::vector<Decimal> prices = {
        Decimal(9, 99), Decimal(19, 50), Decimal(4, 75), Decimal(29, 99)
    };

    auto [mn, mx] = std::minmax_element(prices.begin(), prices.end());
    std::cout << "Min price: " << *mn << "\n";
    std::cout << "Max price: " << *mx << "\n";

    Decimal total = std::accumulate(prices.begin(), prices.end(), Decimal());
    std::cout << "Total: " << total << "\n";

    return 0;
}
```

### Q3: Explain the ODR rules when specializing standard library templates in different TUs

This is the part that catches people off guard. You can specialize `std::hash<Point>` in a header and include it in ten different translation units with no problem. But if you define two different specializations for the same type in two different `.cpp` files, you have undefined behavior - and the compiler is not required to warn you.

```cpp
                    TU 1 (file1.cpp)          TU 2 (file2.cpp)
                    ┌──────────────┐         ┌──────────────┐
                    │ template <>  │         │ template <>  │
                    │ struct hash  │    =    │ struct hash  │    <- MUST be identical!
                    │ <MyType> {   │         │ <MyType> {   │
                    │   ...        │         │   ...        │
                    │ };           │         │ };           │
                    └──────────────┘         └──────────────┘
```

**Rules:**

| Rule | Description |
| --- | --- |
| **Same definition** | If the same specialization appears in multiple TUs, it must be **token-for-token identical** |
| **Same meaning** | All names in the definition must resolve to the same entities |
| **Violation = UB** | Different definitions in different TUs -> **undefined behavior** (no diagnostic required!) |
| **Best practice** | Put specializations in a **header file** so all TUs see the same definition |

Here is the wrong and right way side by side:

```cpp
// === WRONG: Different definitions in different TUs ===
// file1.cpp:
// template <> struct std::hash<Point> {
//     size_t operator()(const Point& p) const { return p.x ^ p.y; }
// };

// file2.cpp:
// template <> struct std::hash<Point> {
//     size_t operator()(const Point& p) const { return p.x * 31 + p.y; }
// };
// -> ODR violation! Undefined behavior!

// === CORRECT: Single definition in header ===
// point.h:
// #pragma once
// struct Point { int x, y; };
// template <> struct std::hash<Point> {
//     size_t operator()(const Point& p) const noexcept {
//         return std::hash<int>{}(p.x) ^ (std::hash<int>{}(p.y) << 1);
//     }
// };
// All TUs #include "point.h" -> same definition everywhere
```

**Additional constraints:**

- You may only specialize for types that involve at least one **user-defined type** (not `std::hash<int>`)
- Partial specializations of standard library class templates are generally **not allowed** (only full)
- Adding entirely new templates or overloads to `namespace std` is **undefined behavior**

```cpp
#include <iostream>

int main() {
    std::cout << "=== ODR rules for std specializations ===\n";
    std::cout << "1. Put specialization in a header -> all TUs see the same def\n";
    std::cout << "2. Must specialize for user-defined types only\n";
    std::cout << "3. Full specialization only (usually)\n";
    std::cout << "4. Don't add new functions/templates to namespace std\n";
    std::cout << "5. Violation is UB — compiler may not warn you!\n";
    return 0;
}
```

---

## Notes

- Specialize `std::hash<T>` in `namespace std` to enable your type as an unordered container key.
- Specialize `std::numeric_limits<T>` to integrate custom numeric types with generic algorithms.
- Always define specializations in **header files** to satisfy ODR across translation units.
- Only specialize for types involving your user-defined types - never for `int`, `std::string`, etc.
- In C++20, prefer `std::formatter<T>` for custom formatting (replaces `operator<<` for `std::format`).
- Hash combining: use `h1 ^ (h2 + 0x9e3779b9 + (h1 << 6) + (h1 >> 2))` (Boost hash_combine pattern).
