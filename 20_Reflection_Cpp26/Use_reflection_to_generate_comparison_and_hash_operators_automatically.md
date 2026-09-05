# Use reflection to generate comparison and hash operators automatically

**Category:** Reflection Cpp26  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

Every time you add a new field to a struct, you have a maintenance tax: update `operator==`, update `operator<=>`, update the `std::hash` specialization. Miss one and you have a silent bug where two objects compare equal even though they differ on the new field. Reflection eliminates that tax entirely. By iterating over all data members at compile time with `nonstatic_data_members_of`, a single generic function can handle equality, ordering, and hashing for any aggregate, and it automatically picks up new fields the moment you add them.

Here is what this means for the boilerplate count:

| Generated | Traditional LOC | Reflection LOC | Maintenance |
| --- | --- | --- | --- |
| `operator==` | N comparisons | 1 generic function | Zero |
| `operator<=>` | N comparisons | 1 generic function | Zero |
| `std::hash<T>` | N hash combines | 1 generic specialization | Zero |

---

## Self-Assessment

### Q1: Generic `operator==` that compares all members field-by-field

The `reflected_equal` function below walks every non-static data member of type `T` and compares the corresponding fields in `a` and `b`. The key ingredient is `obj.[:m:]` - that splice expression takes the compile-time member description `m` and turns it into a real member access at runtime.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
// C++26 with P2996 reflection
#include <meta>
#include <iostream>

// Generic equality - works for ANY aggregate:
template <typename T>
bool reflected_equal(const T& a, const T& b) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    bool result = true;
    template for (constexpr auto m : members) {
        if (a.[:m:] != b.[:m:]) result = false;
    }
    return result;
}

struct Point { int x, y, z; };
struct Person { std::string name; int age; double height; };

// Inject operator== via friend:
struct Color {
    int r, g, b, a;
    friend bool operator==(const Color& lhs, const Color& rhs) {
        return reflected_equal(lhs, rhs);
    }
};

int main() {
    Point a{1, 2, 3}, b{1, 2, 3}, c{1, 0, 3};
    std::cout << reflected_equal(a, b) << '\n'; // 1
    std::cout << reflected_equal(a, c) << '\n'; // 0

    Person p1{"Alice", 30, 5.6}, p2{"Alice", 30, 5.6};
    std::cout << reflected_equal(p1, p2) << '\n'; // 1

    Color red{255, 0, 0, 255}, also_red{255, 0, 0, 255};
    std::cout << (red == also_red) << '\n';  // 1 (uses operator==)
}
```

The `Color` struct shows the usual pattern for injecting a reflected operator: put a hidden friend `operator==` that delegates to the generic helper. This way `Color` gets a proper `==` operator that participates in normal overload resolution, and it automatically covers any future fields you add.

### Q2: Generic `std::hash<T>` for any aggregate via reflection

Getting a type into an `unordered_set` or `unordered_map` requires both `operator==` and a hash function. Writing a correct `std::hash` specialization by hand - combining each field's hash without introducing bias - is tedious and error-prone. The `ReflectedHash` template below does it generically using the same member iteration pattern.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <functional>
#include <iostream>
#include <unordered_set>
#include <string>

// Hash combine utility:
inline std::size_t hash_combine(std::size_t seed, std::size_t h) {
    return seed ^ (h + 0x9e3779b9 + (seed << 6) + (seed >> 2));
}

// Generic hash for any aggregate:
template <typename T>
struct ReflectedHash {
    std::size_t operator()(const T& obj) const {
        constexpr auto members = std::meta::nonstatic_data_members_of(^T);
        std::size_t seed = 0;
        template for (constexpr auto m : members) {
            using MType = [:std::meta::type_of(m):];
            seed = hash_combine(seed, std::hash<MType>{}(obj.[:m:]));
        }
        return seed;
    }
};

struct Point {
    int x, y;
    bool operator==(const Point&) const = default;
};

struct Employee {
    std::string name;
    int id;
    bool operator==(const Employee&) const = default;
};

int main() {
    // Use in unordered containers:
    std::unordered_set<Point, ReflectedHash<Point>> points;
    points.insert({1, 2});
    points.insert({3, 4});
    points.insert({1, 2}); // duplicate, not inserted
    std::cout << "Points: " << points.size() << '\n'; // 2

    std::unordered_set<Employee, ReflectedHash<Employee>> employees;
    employees.insert({"Alice", 1});
    employees.insert({"Bob", 2});
    std::cout << "Employees: " << employees.size() << '\n'; // 2

    // Verify hash consistency:
    ReflectedHash<Point> h;
    std::cout << (h({1,2}) == h({1,2})) << '\n'; // 1 (same input = same hash)
    std::cout << (h({1,2}) == h({2,1})) << '\n'; // 0 (usually different)
}
```

The `hash_combine` formula using `0x9e3779b9` is the golden ratio hash from Boost - it is a well-tested way to mix hash values without too many collisions, and it is fine to reuse in your own code.

### Q3: Compile-time cost: reflection-generated vs handwritten

A common concern when using reflection is whether the generated code is as fast as hand-written code, and whether the compile time overhead is acceptable. This example compares both directly.

<!-- compile: needs `meta` (not in the CI standard library yet) -->
```cpp
#include <meta>
#include <iostream>
#include <compare>

struct Vec3 { float x, y, z; };

// ===== Handwritten =====
bool manual_equal(const Vec3& a, const Vec3& b) {
    return a.x == b.x && a.y == b.y && a.z == b.z;
}

auto manual_compare(const Vec3& a, const Vec3& b) {
    if (auto c = a.x <=> b.x; c != 0) return c;
    if (auto c = a.y <=> b.y; c != 0) return c;
    return a.z <=> b.z;
}

// ===== Reflection-generated =====
template <typename T>
bool refl_equal(const T& a, const T& b) {
    bool eq = true;
    template for (constexpr auto m : std::meta::nonstatic_data_members_of(^T)) {
        if (a.[:m:] != b.[:m:]) eq = false;
    }
    return eq;
}

template <typename T>
auto refl_compare(const T& a, const T& b) {
    template for (constexpr auto m : std::meta::nonstatic_data_members_of(^T)) {
        if (auto c = a.[:m:] <=> b.[:m:]; c != 0) return c;
    }
    return std::strong_ordering::equal;
}

int main() {
    Vec3 a{1.0f, 2.0f, 3.0f}, b{1.0f, 2.0f, 3.0f};

    // Both produce identical results:
    std::cout << manual_equal(a, b) << '\n';  // 1
    std::cout << refl_equal(a, b) << '\n';    // 1

    // Runtime cost: IDENTICAL.
    // The template for expansion generates the same machine code
    // as the handwritten version. The compiler sees:
    //   refl_equal<Vec3> -> a.x==b.x && a.y==b.y && a.z==b.z
    // which is exactly manual_equal.

    // Compile-time cost: slightly higher for reflection
    // (compiler must resolve meta queries), but usually <1% of
    // total build time. Pays off when you have 50+ types.
}
```

After instantiation, `refl_equal<Vec3>` and `manual_equal` produce identical machine code. The `template for` expansion happens entirely at compile time - the optimizer sees the same field comparisons either way. The compile-time cost of the meta queries is real but small, and it pays for itself quickly once you have more than a handful of types to cover.

---

## Notes

- `template for` expansion generates code identical to hand-written field-by-field operations, so there is no runtime cost over doing it manually.
- Runtime performance of reflection-generated code is the same as manual code after optimization.
- Compile-time cost is slightly higher than manual code, but negligible for most projects and well worth it once you have many types.
- C++20 `operator<=>` with `= default` handles simple cases well; use reflection when you need custom logic, filtering, or when you want a single generic helper to cover many types at once.
- The hash combine formula `0x9e3779b9` is the golden ratio hash from Boost and is a reliable choice for mixing field hashes.
