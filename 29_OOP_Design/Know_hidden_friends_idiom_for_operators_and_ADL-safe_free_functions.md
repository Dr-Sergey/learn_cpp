# Know hidden friends idiom for operators and ADL-safe free functions

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

A **hidden friend** is a `friend` function defined *inside* a class body. It's only found via **ADL** (Argument-Dependent Lookup), not by normal unqualified lookup. The name "hidden" is accurate: the function exists and is callable, but only when one of the arguments is of the class type - it's invisible to everything else.

```cpp
class MyType {
    friend bool operator==(const MyType& a, const MyType& b) { // Hidden friend
        return a.val_ == b.val_;
    }
    int val_;
};
```

| Property | Member operator | Free function | Hidden friend |
| --- | :---: | :---: | :---: |
| Symmetric operands | No (`a==b` != `b==a` for conversions) | Yes | Yes |
| Found by ADL only | N/A | No (found everywhere) | Yes |
| Reduces overload set | N/A | No | Yes (faster compile) |
| Access to private | Yes | Needs friend decl | Yes |

The key insight is that hiding friends in the class body is strictly better than putting operators as free functions in the namespace, in almost every case. You get private access, symmetric operand treatment, and a smaller overload set - with no downside.

---

## Self-Assessment

### Q1: Implement operators as hidden friends

The pattern is simple: declare the operator with `friend` inside the class body, define it right there too, and don't add any separate namespace-scope declaration. Every operator that should feel "attached" to the type is a good candidate. Notice how all the `Temperature` operators below live inside the class - the class body is both the declaration and the definition.

**Answer:**

```cpp
#include <iostream>
#include <string>
#include <compare>

class Temperature {
    double kelvin_;
public:
    explicit Temperature(double k) : kelvin_(k) {}
    double kelvin() const { return kelvin_; }

    // Hidden friends: found ONLY via ADL on Temperature
    friend auto operator<=>(const Temperature& a, const Temperature& b) = default;

    friend Temperature operator+(const Temperature& a, const Temperature& b) {
        return Temperature(a.kelvin_ + b.kelvin_);
    }

    friend Temperature operator-(const Temperature& a, const Temperature& b) {
        return Temperature(a.kelvin_ - b.kelvin_);
    }

    friend std::ostream& operator<<(std::ostream& os, const Temperature& t) {
        return os << t.kelvin_ << "K";
    }

    // Hidden friend swap - found by ADL, no std:: needed
    friend void swap(Temperature& a, Temperature& b) noexcept {
        using std::swap;
        swap(a.kelvin_, b.kelvin_);
    }
};

int main() {
    Temperature t1(273.15), t2(373.15);
    std::cout << (t1 < t2) << "\n";    // true
    std::cout << (t1 + t2) << "\n";    // 646.3K
    std::cout << t1 << "\n";           // 273.15K
    return 0;
}
```

### Q2: Show why hidden friends reduce compilation time and prevent surprises

This is the "why bother" explanation. When you put an operator as a free function in the namespace, the compiler considers it as a candidate for *any* overload resolution in code that's seen the namespace - even for completely unrelated types. Hidden friends are invisible to that search. In large codebases with many types, this adds up to a measurable difference in compile times, and it prevents the occasional bewildering "where did this candidate come from?" error.

**Answer:**

```cpp
#include <string>
#include <iostream>

namespace lib {
    class Widget {
        int id_;
    public:
        explicit Widget(int id) : id_(id) {}

        // BAD: Free function operator in namespace
        // Found during ANY lookup if namespace is pulled in
    };
    // Non-hidden: pollutes overload resolution
    // bool operator==(const Widget& a, const Widget& b) { return a.id_ == b.id_; }

    class BetterWidget {
        int id_;
    public:
        explicit BetterWidget(int id) : id_(id) {}

        // GOOD: Hidden friend - only found via ADL
        friend bool operator==(const BetterWidget& a, const BetterWidget& b) {
            return a.id_ == b.id_;
        }
    };
}

// The hidden friend for BetterWidget won't accidentally participate
// in overload resolution for unrelated types
struct Unrelated {};

int main() {
    lib::BetterWidget a(1), b(2);
    std::cout << (a == b) << "\n";  // OK: ADL finds operator==

    // Unrelated u1{}, u2{};
    // u1 == u2;  // BetterWidget's operator== is NOT even considered
    //            // (unlike a namespace-scope operator)
    return 0;
}
```

### Q3: Implement a full type with hidden friends for all common operations

Here's what a well-designed type looks like when you apply the hidden friends idiom consistently. The `Identifier` class gets comparison, ordering, streaming, hashing, and swapping - all as hidden friends. Notice that even the `std::hash` specialization can be given access through a `friend` declaration, so the implementation stays private.

**Answer:**

```cpp
#include <iostream>
#include <functional>
#include <string>

class Identifier {
    std::string prefix_;
    int id_;
public:
    Identifier(std::string prefix, int id)
        : prefix_(std::move(prefix)), id_(id) {}

    // All operators as hidden friends
    friend bool operator==(const Identifier& a, const Identifier& b) {
        return a.prefix_ == b.prefix_ && a.id_ == b.id_;
    }

    friend auto operator<=>(const Identifier& a, const Identifier& b) {
        if (auto cmp = a.prefix_ <=> b.prefix_; cmp != 0)
            return cmp;
        return a.id_ <=> b.id_;
    }

    friend std::ostream& operator<<(std::ostream& os, const Identifier& id) {
        return os << id.prefix_ << "-" << id.id_;
    }

    friend std::istream& operator>>(std::istream& is, Identifier& id) {
        char dash;
        return is >> id.prefix_ >> dash >> id.id_;
    }

    // Hidden friend hash support
    friend struct std::hash<Identifier>;

    friend void swap(Identifier& a, Identifier& b) noexcept {
        using std::swap;
        swap(a.prefix_, b.prefix_);
        swap(a.id_, b.id_);
    }
};

// Hash specialization using friend access
template<> struct std::hash<Identifier> {
    size_t operator()(const Identifier& id) const {
        return std::hash<std::string>{}(id.prefix_)
             ^ (std::hash<int>{}(id.id_) << 1);
    }
};

#include <set>
#include <unordered_set>

int main() {
    std::set<Identifier> ordered;
    ordered.emplace("USR", 1);
    ordered.emplace("USR", 2);
    ordered.emplace("ORD", 1);
    for (const auto& id : ordered) std::cout << id << " ";
    std::cout << "\n";  // ORD-1 USR-1 USR-2

    std::unordered_set<Identifier> hashed;
    hashed.emplace("USR", 1);
    return 0;
}
```

The `Identifier` type now works cleanly with `std::set` (needs ordering), `std::unordered_set` (needs hashing), streams, and swap - all thanks to hidden friends. Everything is self-contained in the class definition, and none of these operators will accidentally show up in overload resolution for unrelated types.

---

## Notes

- Default to hidden friends for all operators - it's the modern C++ best practice (endorsed by P1601).
- Hidden friends reduce overload set size, which means faster compilation in large codebases.
- They provide **symmetric** operator behavior - both operands are treated equally for implicit conversions, unlike member operators.
- `swap`, `operator<<`, `operator>>`, and comparison operators are prime candidates for the hidden friends idiom.
- The technique is especially important in library code where namespace pollution affects clients.
- `friend` defined inside the class = hidden friend; `friend` declared inside but defined outside = not hidden. The definition must be inside the class body.
