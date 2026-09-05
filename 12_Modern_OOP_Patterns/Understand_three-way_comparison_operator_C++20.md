# Understand three-way comparison operator <=> (C++20)

**Category:** Modern OOP Patterns  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/operator_comparison>  

---

## Topic Overview

The **spaceship operator** `<=>` (three-way comparison) compares two objects and returns one of three results: less, equal, or greater. From a single `operator<=>`, the compiler can synthesize all six comparison operators: `==`, `!=`, `<`, `>`, `<=`, `>=`.

Before C++20, you had to write all six manually - or use the clunky `std::rel_ops` workaround. The spaceship operator eliminates that entirely. One line does the job.

### Ordering Categories

There are three ordering categories, and picking the right one matters for correctness. The main question is: if two objects compare equal, are they truly interchangeable?

```cpp
strong_ordering          weak_ordering            partial_ordering
───────────────          ──────────────           ─────────────────
::less                   ::less                   ::less
::equal                  ::equivalent             ::equivalent
::greater                ::greater                ::greater
                                                  ::unordered    <- NaN!

Substitutability:        No substitutability:     May be incomparable:
  a == b implies          a == b does NOT imply    a <=> b might be
  f(a) == f(b)            f(a) == f(b)             "unordered"

Example: int             Example: case-insensitive Example: float (NaN)
                          string comparison
```

### What `= default` Generates

When all your members already support `<=>`, you can ask the compiler to generate the comparison for you. It compares members in declaration order, stopping at the first pair that differs.

```cpp
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;
    // Compiler generates memberwise <=> comparison:
    //   1. Compare x <=> other.x
    //   2. If equal, compare y <=> other.y
    //   3. Returns std::strong_ordering (because int is strong)
    //
    // ALSO generates: ==, !=, <, >, <=, >=
};
```

---

## Self-Assessment

### Q1: Write a class with `operator<=> = default` and show all six comparison operators are generated

This is the simplest and most common case. You define `Version` with three integer members, add one line for the spaceship operator, and every comparison you could want just works - including sorting with `std::sort`.

```cpp
#include <iostream>
#include <compare>
#include <string>
#include <vector>
#include <algorithm>

struct Version {
    int major;
    int minor;
    int patch;

    // Single line generates ALL comparisons!
    auto operator<=>(const Version&) const = default;
};

int main() {
    Version v1{1, 2, 3};
    Version v2{1, 3, 0};
    Version v3{1, 2, 3};

    // All six operators work:
    std::cout << std::boolalpha;
    std::cout << "v1 == v3: " << (v1 == v3) << "\n";  // true
    std::cout << "v1 != v2: " << (v1 != v2) << "\n";  // true
    std::cout << "v1 <  v2: " << (v1 <  v2) << "\n";  // true (1.2 < 1.3)
    std::cout << "v1 >  v2: " << (v1 >  v2) << "\n";  // false
    std::cout << "v1 <= v3: " << (v1 <= v3) << "\n";  // true (equal)
    std::cout << "v2 >= v1: " << (v2 >= v1) << "\n";  // true

    // Works with standard algorithms:
    std::vector<Version> versions = {{2,0,0}, {1,0,0}, {1,5,3}, {1,5,2}};
    std::sort(versions.begin(), versions.end());

    std::cout << "\nSorted versions:\n";
    for (const auto& v : versions)
        std::cout << "  " << v.major << "." << v.minor << "." << v.patch << "\n";

    // Three-way comparison result:
    auto result = v1 <=> v2;
    if (result < 0)      std::cout << "\nv1 < v2\n";
    else if (result > 0) std::cout << "\nv1 > v2\n";
    else                 std::cout << "\nv1 == v2\n";
}
// Expected output:
//   v1 == v3: true
//   v1 != v2: true
//   v1 <  v2: true
//   v1 >  v2: false
//   v1 <= v3: true
//   v2 >= v1: true
//
//   Sorted versions:
//     1.0.0
//     1.5.2
//     1.5.3
//     2.0.0
//
//   v1 < v2
```

Notice that `std::sort` just works with no extra configuration. Before C++20 you'd need at least `operator<` for this; with the spaceship operator, the default generates everything the algorithm needs.

---

### Q2: Implement a custom `<=>` that returns `std::partial_ordering` for a float-containing type

When your type contains a `double` or `float`, you can't use `= default` for `<=>` and get `strong_ordering`, because floating-point has NaN - a value that is not less than, equal to, or greater than anything else. You need to handle that case yourself and return `partial_ordering`.

```cpp
#include <iostream>
#include <compare>
#include <cmath>

class Temperature {
    double kelvin_;
public:
    explicit Temperature(double k) : kelvin_(k) {}

    double kelvin() const { return kelvin_; }

    // Custom <=> returning partial_ordering (because double can be NaN)
    std::partial_ordering operator<=>(const Temperature& other) const {
        // NaN handling: if either is NaN, result is unordered
        if (std::isnan(kelvin_) || std::isnan(other.kelvin_))
            return std::partial_ordering::unordered;

        if (kelvin_ < other.kelvin_) return std::partial_ordering::less;
        if (kelvin_ > other.kelvin_) return std::partial_ordering::greater;
        return std::partial_ordering::equivalent;
    }

    // Must define == separately when <=> is custom (not defaulted)
    bool operator==(const Temperature& other) const {
        if (std::isnan(kelvin_) || std::isnan(other.kelvin_))
            return false;  // NaN != NaN
        return kelvin_ == other.kelvin_;
    }
};

int main() {
    Temperature boiling(373.15);
    Temperature freezing(273.15);
    Temperature nan_temp(std::nan(""));

    std::cout << std::boolalpha;
    std::cout << "boiling > freezing: " << (boiling > freezing) << "\n";
    std::cout << "freezing < boiling: " << (freezing < boiling) << "\n";
    std::cout << "boiling == boiling: " << (boiling == boiling) << "\n";

    // NaN comparisons - all false!
    std::cout << "\nNaN comparisons:\n";
    std::cout << "nan > freezing:  " << (nan_temp > freezing) << "\n";
    std::cout << "nan < freezing:  " << (nan_temp < freezing) << "\n";
    std::cout << "nan == nan:      " << (nan_temp == nan_temp) << "\n";
    std::cout << "nan != nan:      " << (nan_temp != nan_temp) << "\n";

    // Check ordering category
    auto result = nan_temp <=> freezing;
    if (result == std::partial_ordering::unordered)
        std::cout << "nan <=> freezing: UNORDERED\n";
}
// Expected output:
//   boiling > freezing: true
//   freezing < boiling: true
//   boiling == boiling: true
//
//   NaN comparisons:
//   nan > freezing:  false
//   nan < freezing:  false
//   nan == nan:      false
//   nan != nan:      true
//   nan <=> freezing: UNORDERED
```

The important rule to remember here: when you write a custom `<=>` (rather than `= default`), the compiler does **not** automatically generate `operator==` for you. You must define it separately.

---

### Q3: Explain the three ordering types: `strong_ordering`, `weak_ordering`, `partial_ordering`

The choice of ordering type is a semantic claim about your type, not just a technical choice. `strong_ordering` says "if two objects compare equal, they are fully interchangeable for any purpose." `weak_ordering` says "equivalent objects may still differ in some observable way." `partial_ordering` adds "some pairs of objects can't be compared at all."

| Ordering | Values | Substitutability | Example Types |
| --- | --- | --- | --- |
| **`strong_ordering`** | `less`, `equal`, `greater` | Yes: `a == b` implies `f(a) == f(b)` for ANY `f` | `int`, `char`, `std::string` |
| **`weak_ordering`** | `less`, `equivalent`, `greater` | No: equivalent objects may differ | Case-insensitive string, `std::weak_order` |
| **`partial_ordering`** | `less`, `equivalent`, `greater`, **`unordered`** | No + may be incomparable | `float` (NaN), custom math objects |

The ordering categories form a hierarchy - you can implicitly convert from a stricter to a more permissive one, but not the other direction:

```cpp
Implicit conversions (narrowing):
  strong_ordering -> weak_ordering -> partial_ordering
  (strong is the most refined, partial is the most general)
```

Here's a concrete demonstration of all three:

```cpp
#include <iostream>
#include <compare>
#include <string>
#include <algorithm>

// strong_ordering: int
void demo_strong() {
    auto r = 3 <=> 5;  // std::strong_ordering::less
    static_assert(std::is_same_v<decltype(r), std::strong_ordering>);
    std::cout << "3 <=> 5: strong_ordering::less\n";
}

// weak_ordering: case-insensitive string
struct CIString {
    std::string data;

    std::weak_ordering operator<=>(const CIString& other) const {
        auto to_lower = [](const std::string& s) {
            std::string result = s;
            for (auto& c : result) c = static_cast<char>(std::tolower(c));
            return result;
        };
        auto l = to_lower(data);
        auto r = to_lower(other.data);
        if (l < r) return std::weak_ordering::less;
        if (l > r) return std::weak_ordering::greater;
        return std::weak_ordering::equivalent;
        // "Hello" == "HELLO" (equivalent but NOT identical - not substitutable)
    }

    bool operator==(const CIString& other) const {
        return (*this <=> other) == 0;
    }
};

void demo_weak() {
    CIString a{"Hello"}, b{"HELLO"};
    std::cout << "\"Hello\" == \"HELLO\": " << (a == b) << "\n";  // true (equivalent)
    // But a.data != b.data - not substitutable!
}

// partial_ordering: float
void demo_partial() {
    float x = 1.0f, nan = std::numeric_limits<float>::quiet_NaN();
    auto r = nan <=> x;  // partial_ordering::unordered
    std::cout << "NaN <=> 1.0: unordered? " << (r == std::partial_ordering::unordered) << "\n";
}

int main() {
    std::cout << std::boolalpha;
    demo_strong();
    demo_weak();
    demo_partial();
}
// Expected output:
//   3 <=> 5: strong_ordering::less
//   "Hello" == "HELLO": true
//   NaN <=> 1.0: unordered? true
```

The `CIString` case is worth pausing on. "Hello" and "HELLO" are *equivalent* under case-insensitive comparison - they sort the same way. But they're not *equal* in the `strong_ordering` sense, because `f("Hello") != f("HELLO")` if `f` is something like "print the string." That's exactly why it returns `weak_ordering::equivalent` rather than `strong_ordering::equal`.

---

## Notes

- **`= default` spaceship** does memberwise comparison in declaration order - first member that differs determines the result.
- **`operator==`** is NOT generated from `operator<=>` unless `<=>` is defaulted. For custom `<=>`, you must also define `operator==`.
- **Heterogeneous comparison:** You can define `operator<=>(const MyType&, int)` for comparing different types.
- **`std::strong_order`, `std::weak_order`, `std::partial_order`** are customization-point objects that provide total ordering even for types like `float` (where `-0.0 == +0.0` but `total_order` distinguishes them).
- **Before C++20:** You needed to write all 6 operators manually (or use `std::rel_ops`, which was a poor solution). The spaceship operator eliminates this boilerplate entirely.
- **Performance:** `<=>` can be more efficient than separate `<` and `==` because it computes the relationship in one pass.
