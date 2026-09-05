# Know how to correctly implement comparison operators with spaceship operator

**Category:** OOP Design  
**Standard:** C++20  

---

## Topic Overview

C++20's **spaceship operator** (`<=>`) generates all six comparison operators from a single definition. Before C++20, you had to write `operator==`, `operator!=`, `operator<`, `operator>`, `operator<=`, and `operator>=` separately - six functions, each of which could independently contain a bug. The spaceship operator collapses that to one, and for simple types it can be defaulted so the compiler writes it for you.

| Category | Operators | `<=>` return type |
| --- | --- | --- |
| Equality only | `==`, `!=` | `std::strong_equality` (prefer `==` alone) |
| Full ordering | `<`, `>`, `<=`, `>=`, `==`, `!=` | `std::strong_ordering` |
| Partial ordering | Same but allows unordered | `std::partial_ordering` |
| Weak ordering | Same, equivalents not identical | `std::weak_ordering` |

### What the compiler generates

When you write a defaulted spaceship operator, the compiler does the work for you:

```cpp
auto operator<=>(const T&) const = default;
// generates ->
// ==  !=  <  >  <=  >=   (all six)
// Member-wise comparison in declaration order.
```

The rewrite rule is what makes all the other operators work: `a < b` is rewritten as `(a <=> b) < 0`, so you only need the one definition. The categories (`strong_ordering`, `weak_ordering`, `partial_ordering`) exist because not all types have the same notion of equality - integers have a strict total order, floats have NaN which is unordered with everything, and case-insensitive strings have values that are "equivalent for ordering" but not "identical."

---

## Self-Assessment

### Q1: Use defaulted and custom spaceship operators

Here are two contrasting cases: a `Point` where the defaulted version is exactly right, and a `CIString` (case-insensitive string) where you need a custom implementation. Pay attention to how the defaulted version handles the simple case with zero boilerplate, while the custom version gives you full control over what "less than" means.

**Answer:**

```cpp
#include <compare>
#include <string>
#include <iostream>
#include <set>

// Simple case: default works perfectly
struct Point {
    int x, y, z;
    auto operator<=>(const Point&) const = default;
};

// Custom: case-insensitive string wrapper
class CIString {
    std::string data_;
public:
    explicit CIString(std::string s) : data_(std::move(s)) {}
    const std::string& str() const { return data_; }

    // Custom spaceship for case-insensitive ordering
    std::weak_ordering operator<=>(const CIString& other) const {
        auto to_lower = [](char c) -> char {
            return (c >= 'A' && c <= 'Z') ? c + 32 : c;
        };
        size_t len = std::min(data_.size(), other.data_.size());
        for (size_t i = 0; i < len; ++i) {
            char a = to_lower(data_[i]);
            char b = to_lower(other.data_[i]);
            if (a != b)
                return a < b ? std::weak_ordering::less
                             : std::weak_ordering::greater;
        }
        return data_.size() <=> other.data_.size();
    }

    // Must define == separately for optimized equality check
    bool operator==(const CIString& other) const {
        if (data_.size() != other.data_.size()) return false;
        return (*this <=> other) == 0;
    }
};

int main() {
    // Point uses default <=> - all operators work
    Point a{1, 2, 3}, b{1, 2, 4};
    std::cout << std::boolalpha;
    std::cout << (a < b) << "\n";   // true
    std::cout << (a == b) << "\n";  // false

    // CIString: case-insensitive ordering
    std::set<CIString> names;
    names.insert(CIString("Alice"));
    names.insert(CIString("alice"));  // Same as "Alice"!
    std::cout << names.size() << "\n";  // 1
    return 0;
}
```

The `CIString` uses `std::weak_ordering` rather than `std::strong_ordering` because "Alice" and "alice" are considered equivalent for ordering purposes but are not the same value - you could distinguish them if you wanted to. That's exactly what `weak_ordering` means: elements can be equivalent without being identical.

### Q2: Handle mixed-type comparisons and ordering categories

Sometimes you need to compare two objects of different types (like meters and feet), or handle values that have no defined order relative to each other (like floating-point NaN). This example covers both cases. The key insight is that you need to pick the right ordering category to accurately model your type's semantics.

**Answer:**

```cpp
#include <compare>
#include <cmath>
#include <iostream>

// Partial ordering: NaN is unordered
class SafeFloat {
    double val_;
public:
    explicit SafeFloat(double v) : val_(v) {}
    double value() const { return val_; }

    std::partial_ordering operator<=>(const SafeFloat& other) const {
        if (std::isnan(val_) || std::isnan(other.val_))
            return std::partial_ordering::unordered;
        if (val_ < other.val_) return std::partial_ordering::less;
        if (val_ > other.val_) return std::partial_ordering::greater;
        return std::partial_ordering::equivalent;
    }
    bool operator==(const SafeFloat& o) const {
        return !std::isnan(val_) && !std::isnan(o.val_) && val_ == o.val_;
    }
};

// Mixed-type comparison: Meters vs Feet
struct Meters {
    double value;
    std::partial_ordering operator<=>(const Meters& o) const = default;
};
struct Feet { double value; };

// Free function for cross-type comparison
std::partial_ordering operator<=>(const Meters& m, const Feet& f) {
    return m.value <=> (f.value * 0.3048);
}
bool operator==(const Meters& m, const Feet& f) {
    return (m <=> f) == 0;
}

int main() {
    SafeFloat a(1.0), nan(NAN);
    auto r = a <=> nan;
    std::cout << (r == std::partial_ordering::unordered) << "\n";  // true

    Meters m{1.0};
    Feet f{3.28084};
    std::cout << ((m <=> f) == std::partial_ordering::equivalent) << "\n"; // ~true
    return 0;
}
```

The `SafeFloat` class is a good illustration of why `partial_ordering` exists: IEEE 754 floating-point specifies that NaN is unordered with everything, including itself. Regular `double` comparisons just return false for NaN, which is often not what you want. Making the ordering category explicit in the type forces callers to handle the unordered case.

### Q3: Implement spaceship for a class with member ordering priority

Here's a realistic scenario: a `Version` struct where ordering should look at major, minor, and patch numbers in sequence, but equality should also consider the label field. This is a case where you need custom `<=>` *and* custom `==`, because you want them to have slightly different semantics. The reason you must define both is that the compiler won't generate `==` from a custom `<=>` - it only does that for defaulted ones.

**Answer:**

```cpp
#include <compare>
#include <string>
#include <iostream>

// Version: compare major, then minor, then patch
struct Version {
    int major, minor, patch;
    std::string label;  // "alpha", "beta", "" for release

    // Custom: ignore label for ordering, but include for equality
    std::strong_ordering operator<=>(const Version& o) const {
        if (auto cmp = major <=> o.major; cmp != 0) return cmp;
        if (auto cmp = minor <=> o.minor; cmp != 0) return cmp;
        return patch <=> o.patch;
    }

    // Equality includes label
    bool operator==(const Version& o) const {
        return major == o.major && minor == o.minor
            && patch == o.patch && label == o.label;
    }
};

int main() {
    Version v1{2, 1, 0, "alpha"};
    Version v2{2, 1, 0, "beta"};
    Version v3{2, 1, 1, ""};

    std::cout << std::boolalpha;
    std::cout << (v1 < v3) << "\n";   // true (2.1.0 < 2.1.1)
    std::cout << (v1 == v2) << "\n";  // false (different labels)
    std::cout << (v1 < v2) << "\n";   // false (same version number)
    std::cout << ((v1 <=> v2) == 0) << "\n";  // true (ordering ignores label)
    return 0;
}
```

Notice the deliberate asymmetry: `v1 < v2` is false (because `<=>` ignores the label and they have the same numbers), but `v1 == v2` is also false (because `==` includes the label). This might look contradictory, but it's a valid and useful design - you can sort versions by number while still distinguishing alpha from beta for purposes of exact equality.

---

## Notes

- Default `<=>` is almost always enough - it compares all members in declaration order, and the compiler generates all six operators.
- If you define custom `<=>`, you **must** also define `==` separately for optimization - the compiler won't auto-generate it from a user-provided spaceship.
- `strong_ordering`: equal values are identical (integers, strings).
- `weak_ordering`: equivalent but distinguishable (case-insensitive strings).
- `partial_ordering`: some values are unordered (floating-point NaN).
- Rewrite rule: `a < b` is rewritten as `(a <=> b) < 0` - you never need to write the individual relational operators when you have `<=>`.
