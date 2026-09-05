# Use std::strong_ordering, weak_ordering, partial_ordering as <=> return types

**Category:** Standard Library Utilities  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/compare/strong_ordering>  

---

## Topic Overview

C++20's three-way comparison operator `<=>` (spaceship) returns one of three comparison category types. The return type tells the compiler **which comparison guarantees your type provides**, and it automatically synthesizes all six relational operators from one `<=>` definition.

The reason there are three categories - not just one - is that "equal" means different things to different types. For `int`, equal really means identical and substitutable. For a case-insensitive string, "Hello" and "HELLO" might sort the same but aren't interchangeable. For `double`, two values might not be orderable at all because of NaN. The three categories let you express exactly which guarantee you're making.

### Three Comparison Categories

```cpp
                ┌──────────────────────┐
                │  partial_ordering    │   Can be "unordered" (NaN != NaN)
                └─────────┬────────────┘
                          │ (refines)
                ┌─────────▼────────────┐
                │   weak_ordering      │   Equal != identical (e.g., "abc" == "ABC")
                └─────────┬────────────┘
                          │ (refines)
                ┌─────────▼────────────┐
                │  strong_ordering     │   Equal => identical (substitutable)
                └──────────────────────┘
```

| Category | Named values | Unordered? | Equal => identical? | Use case |
| --- | --- | --- | --- | --- |
| `strong_ordering` | `less`, `equal`, `greater` | No | Yes | `int`, `std::string`, IDs |
| `weak_ordering` | `less`, `equivalent`, `greater` | No | No | Case-insensitive strings |
| `partial_ordering` | `less`, `equivalent`, `greater`, `unordered` | Yes | No | `double` (NaN), intervals |

### Implicit Conversions

`strong_ordering` -> `weak_ordering` -> `partial_ordering` (widening is implicit, narrowing requires explicit cast).

### Core Example

Here are all three categories in one file. Notice the different named values: `strong_ordering` uses `equal`, while `weak_ordering` and `partial_ordering` use `equivalent` to signal that equivalence does not imply identity.

```cpp
#include <compare>
#include <iostream>
#include <string>
#include <cmath>

// === strong_ordering: int-like types ===
struct StudentID {
    int id;
    auto operator<=>(const StudentID&) const = default;
    // Returns strong_ordering because int's <=> is strong_ordering.
    // Compiler generates: ==, !=, <, >, <=, >= automatically!
};

// === weak_ordering: case-insensitive string ===
struct CIString {
    std::string data;

    std::weak_ordering operator<=>(const CIString& other) const {
        auto to_lower = [](char c) { return static_cast<char>(std::tolower(c)); };
        auto it1 = data.begin(), it2 = other.data.begin();
        for (; it1 != data.end() && it2 != other.data.end(); ++it1, ++it2) {
            if (to_lower(*it1) < to_lower(*it2)) return std::weak_ordering::less;
            if (to_lower(*it1) > to_lower(*it2)) return std::weak_ordering::greater;
        }
        if (data.size() < other.data.size()) return std::weak_ordering::less;
        if (data.size() > other.data.size()) return std::weak_ordering::greater;
        return std::weak_ordering::equivalent; // "abc" == "ABC" but not identical
    }

    bool operator==(const CIString& other) const {
        return (*this <=> other) == 0;
    }
};

// === partial_ordering: type with NaN ===
struct Measurement {
    double value; // may be NaN (sensor failure)

    std::partial_ordering operator<=>(const Measurement& other) const {
        return value <=> other.value; // double's <=> returns partial_ordering
    }
    bool operator==(const Measurement& other) const = default;
};

int main() {
    // strong_ordering - all six operators work
    StudentID a{1}, b{2};
    std::cout << std::boolalpha;
    std::cout << (a < b) << "\n";   // true
    std::cout << (a == b) << "\n";  // false
    std::cout << (a >= b) << "\n";  // false

    // weak_ordering - "Hello" == "hello"
    CIString s1{"Hello"}, s2{"hello"};
    std::cout << (s1 == s2) << "\n";  // true  ("equivalent" but different bytes)
    std::cout << (s1 < s2) << "\n";   // false

    // partial_ordering - NaN is unordered
    Measurement m1{3.14}, m2{NAN};
    std::cout << (m1 < m2) << "\n";   // false
    std::cout << (m1 > m2) << "\n";   // false
    std::cout << (m1 == m2) << "\n";  // false - NaN is unordered with everything
}
```

The `Measurement` example illustrates why NaN forces `partial_ordering`: NaN is not less than, greater than, or equal to anything - including itself. `partial_ordering` has a fourth named value `unordered` precisely for these situations.

---

## Self-Assessment

### Q1: Return std::partial_ordering from <=> for a type containing a double to handle NaN correctly

**Answer:**

Any struct that holds a `double` must use `partial_ordering` if it defaults its `<=>` operator - the compiler won't let you claim `strong_ordering` when a member's comparison is weaker.

```cpp
#include <compare>
#include <iostream>
#include <cmath>
#include <limits>

struct Temperature {
    double kelvin; // NaN means "sensor error"

    // Since double has NaN, we MUST use partial_ordering
    std::partial_ordering operator<=>(const Temperature& other) const {
        // double's built-in <=> returns std::partial_ordering
        return kelvin <=> other.kelvin;
    }

    bool operator==(const Temperature& other) const {
        return kelvin == other.kelvin;
    }

    bool is_valid() const {
        return !std::isnan(kelvin);
    }
};

int main() {
    Temperature boiling{373.15};
    Temperature freezing{273.15};
    Temperature error{std::numeric_limits<double>::quiet_NaN()};

    std::cout << std::boolalpha;

    // Normal comparison works:
    std::cout << (boiling > freezing) << "\n";  // true
    std::cout << (freezing < boiling) << "\n";  // true

    // NaN produces "unordered" for ALL comparisons:
    std::cout << (error < boiling) << "\n";     // false
    std::cout << (error > boiling) << "\n";     // false
    std::cout << (error == boiling) << "\n";    // false
    std::cout << (error == error) << "\n";      // false! NaN != NaN

    // Check the comparison category directly:
    auto result = error <=> boiling;
    if (result == std::partial_ordering::unordered) {
        std::cout << "Result is unordered (NaN involved)\n";
    }
    // Output: Result is unordered (NaN involved)

    // WRONG: using strong_ordering for a type with double
    // The compiler would reject: auto operator<=>(const T&) const = default;
    // ...if any member's <=> returns a weaker category than strong_ordering.
    // Since double <=> returns partial_ordering, the defaulted <=> also returns
    // partial_ordering. You cannot force strong_ordering on a type with double members.
}
```

`double`'s `<=>` inherently returns `std::partial_ordering` because NaN creates values that are incomparable with everything (including themselves). If your struct contains a `double`, the defaulted `<=>` will automatically return `partial_ordering`. You cannot "upgrade" it to `strong_ordering` - that would incorrectly claim all values are comparable.

### Q2: Explain when weak_ordering is appropriate: equality that does not imply identity (e.g., case-insensitive string)

**Answer:**

The key distinction is between "equivalent for ordering purposes" and "identical in every observable way." `weak_ordering` covers the first case - two values compare as equal but are not substitutable.

```cpp
#include <compare>
#include <iostream>
#include <string>
#include <algorithm>
#include <cctype>

// weak_ordering means: two values can be "equivalent" for ordering
// purposes but are NOT identical / substitutable.

// Example: Case-insensitive string comparison
struct CaseInsensitive {
    std::string text;

    std::weak_ordering operator<=>(const CaseInsensitive& other) const {
        // Compare character by character, ignoring case
        auto lower = [](unsigned char c) { return std::tolower(c); };

        auto it1 = text.begin(), it2 = other.text.begin();
        while (it1 != text.end() && it2 != other.text.end()) {
            int c1 = lower(*it1++), c2 = lower(*it2++);
            if (c1 < c2) return std::weak_ordering::less;
            if (c1 > c2) return std::weak_ordering::greater;
        }
        if (text.size() < other.text.size()) return std::weak_ordering::less;
        if (text.size() > other.text.size()) return std::weak_ordering::greater;
        return std::weak_ordering::equivalent;
        //      ^^^^^^^^^^^^^^^^^^^^^^^^^^
        // NOT "equal"! "Hello" and "HELLO" are equivalent but not identical.
    }

    // Must define operator== separately for weak_ordering types
    bool operator==(const CaseInsensitive& other) const {
        return (*this <=> other) == 0;
    }
};

// Why NOT strong_ordering?
// strong_ordering::equal means the two values are substitutable.
// "Hello" and "HELLO" are NOT substitutable - they print differently,
// hash differently, and have different byte representations.
// Claiming strong_ordering would be a LIE about your type's semantics.

int main() {
    CaseInsensitive a{"Hello"}, b{"HELLO"}, c{"World"};

    std::cout << std::boolalpha;
    std::cout << (a == b) << "\n";  // true  (equivalent, not identical)
    std::cout << (a < c) << "\n";   // true  ('h' < 'w')
    std::cout << (b < c) << "\n";   // true  ('h' < 'w')

    // Practical consequence: in a std::set<CaseInsensitive>,
    // "Hello" and "HELLO" would occupy the same slot.

    // WHEN TO USE EACH:
    //
    // strong_ordering:  a == b => f(a) == f(b) for ALL observable functions
    //   Examples: int, char, std::string (exact), unique identifiers
    //
    // weak_ordering:    a == b => a and b are in the same "equivalence class"
    //   Examples: case-insensitive strings, priority (same priority != same task)
    //
    // partial_ordering: some pairs of values are incomparable
    //   Examples: double (NaN), sets (subset ordering), intervals
}
```

`weak_ordering` is the correct choice when your comparison groups values into equivalence classes but equivalent values are not interchangeable. The term `equivalent` (used by `weak_ordering`) differs from `equal` (used by `strong_ordering`). If you used `strong_ordering`, you'd be promising that equal values are identical in every observable way - a promise that case-insensitive comparison cannot keep.

### Q3: Show how the standard library derives all six comparison operators from <=> returning strong_ordering

**Answer:**

With `= default`, you get all six relational operators for free. Here is what the compiler actually generates, and how `std::set` and `std::sort` benefit from it automatically.

```cpp
#include <compare>
#include <iostream>
#include <set>
#include <algorithm>
#include <vector>

struct Point {
    int x, y, z;

    // ONE line generates ALL SIX operators: ==, !=, <, >, <=, >=
    auto operator<=>(const Point&) const = default;
    // Return type is std::strong_ordering (because int <=> int is strong_ordering)
};

// What the compiler generates (conceptually):
//
// std::strong_ordering operator<=>(const Point& other) const {
//     if (auto cmp = x <=> other.x; cmp != 0) return cmp;
//     if (auto cmp = y <=> other.y; cmp != 0) return cmp;
//     return z <=> other.z;
// }
// bool operator==(const Point& other) const {
//     return x == other.x && y == other.y && z == other.z;
// }
//
// From these two, the compiler synthesizes:
//   operator!=  ->  !(a == b)
//   operator<   ->  (a <=> b) < 0
//   operator>   ->  (a <=> b) > 0
//   operator<=  ->  (a <=> b) <= 0
//   operator>=  ->  (a <=> b) >= 0

int main() {
    Point a{1, 2, 3}, b{1, 2, 4}, c{1, 2, 3};

    std::cout << std::boolalpha;

    // All six operators work:
    std::cout << "a == c: " << (a == c) << "\n";  // true
    std::cout << "a != b: " << (a != b) << "\n";  // true
    std::cout << "a <  b: " << (a < b)  << "\n";  // true  (z: 3 < 4)
    std::cout << "b >  a: " << (b > a)  << "\n";  // true
    std::cout << "a <= c: " << (a <= c) << "\n";  // true
    std::cout << "b >= a: " << (b >= a) << "\n";  // true

    // Works with standard library algorithms and containers:
    std::set<Point> points{{3,2,1}, {1,2,3}, {1,2,4}};
    for (auto& [x, y, z] : points)
        std::cout << "(" << x << "," << y << "," << z << ") ";
    std::cout << "\n";
    // Output: (1,2,3) (1,2,4) (3,2,1) - sorted lexicographically

    std::vector<Point> v{{5,1,0}, {1,2,3}, {3,3,3}};
    std::sort(v.begin(), v.end()); // uses operator<
    // v is now: {1,2,3}, {3,3,3}, {5,1,0}

    // Lexicographic member comparison order:
    // First compares x. If equal, compares y. If equal, compares z.
    Point p1{1, 5, 0}, p2{2, 0, 0};
    std::cout << "p1 < p2: " << (p1 < p2) << "\n"; // true (1 < 2, y and z irrelevant)
}
```

With `= default`, the compiler generates `<=>` that compares members in declaration order (lexicographic). From the `<=>` and `==` pair, the compiler synthesizes all remaining operators. The return type is the common comparison category of all members - if all members are `int`, the result is `strong_ordering`. If any member is `double`, the result weakens to `partial_ordering`.

---

## Notes

- **Return type deduction:** `auto operator<=>(const T&) const = default;` deduces the return type as the common comparison category of all members. Explicitly specifying the type (e.g., `std::strong_ordering`) causes a compilation error if any member doesn't support that category.
- **`==` optimization:** The compiler generates `operator==` separately from `<=>` because for types like `std::string`, equality can short-circuit on length before comparing characters.
- **Heterogeneous comparison:** You can define `<=>` against other types (e.g., `Point <=> int`) for mixed comparisons.
- **`std::is_eq`, `std::is_lt`, etc.:** Named comparison functions in `<compare>` convert any ordering result to `bool`.
- Compile with `-std=c++20 -Wall -Wextra`.
