# Know how operator overloading rules constrain which operators can be overloaded

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/operators>  

---

## Topic Overview

### What Is Operator Overloading

Operator overloading is C++ letting you teach operators like `+`, `<<`, or `[]` what they should mean for *your* types. It's a great feature, but it comes with guardrails: the language is picky about **which** operators you may overload, **how** you must declare each one, and **what** constraints apply. This page is mostly about those guardrails - knowing them up front saves you from confusing compiler errors later.

### Operators That CANNOT Be Overloaded

Some operators are off-limits, and there's a reason behind each. Mostly it's because they aren't really "runtime operations" you could redirect to a function:

| Operator | Name | Reason |
| --- | --- | --- |
| `::` | Scope resolution | Fundamental to name lookup, not a runtime operation |
| `.` | Member access | Must always mean "access member" - no alternative semantics |
| `.*` | Pointer-to-member access | Same reason as `.` |
| `?:` | Ternary conditional | Has unique short-circuit semantics that can't be preserved |
| `sizeof` | Size query | Compile-time operator, not a function call |
| `typeid` | Type identification | Compile-time/RTTI operator |
| `alignof` | Alignment query | Compile-time operator |
| `noexcept` | Noexcept check | Compile-time operator |

### Member vs. Free Function Rules

For the operators you *can* overload, the next question is *where*: must it be a member, or is it better as a free function? Here's the cheat sheet, and we'll unpack the two most important rows right after:

| Operator | Must be member? | Typically free? | Notes |
| --- | --- | --- | --- |
| `=` (assignment) | **Yes** | - | Prevents hijacking assignment semantics |
| `()` (call) | **Yes** | - | Defines function objects/functors |
| `[]` (subscript) | **Yes** | - | C++23 allows multiple parameters |
| `->` (member access) | **Yes** | - | Must return pointer or proxy |
| `->*` | No | No | Rarely overloaded |
| `<<`, `>>` (stream) | No | **Yes** | Left operand is `std::ostream&`, not your type |
| `+`, `-`, `*`, `/` | No | **Yes** | Free function enables symmetric conversions |
| `==`, `<=>` | **Yes** (C++20) | - | Compiler generates `!=`, `<`, `>`, etc. |
| Conversion `operator T()` | **Yes** | - | Implicit or explicit conversion |
| `new`, `delete` | May be member or global | - | Controls allocation |

### Why `operator=` Must Be a Member

The rule that `operator=` has to be a member isn't arbitrary - it's about keeping a class in control of its own assignment. If assignment could be defined from the outside, anyone could quietly rewire how your objects copy themselves:

```cpp
// If this were allowed (it's NOT):
MyClass& operator=(MyClass& lhs, const MyClass& rhs);  // ERROR: = must be member
```

By forcing it to be a member, the language guarantees the class author is the one deciding how invariants survive an assignment.

### Why `operator<<` Is Typically Free

Stream output flips the situation. The left operand of `std::cout << obj` is the *stream*, not your object - and you can't go adding member functions to `std::ostream`:

```cpp
struct Point {
    double x, y;

    // Can't make operator<< a member - the left operand is std::ostream, not Point!
    // std::ostream& operator<<(std::ostream& os) would mean: point << std::cout; // wrong order!
};

// Correct: free function (or hidden friend)
std::ostream& operator<<(std::ostream& os, const Point& p) {
    return os << "(" << p.x << ", " << p.y << ")";
}
// Usage: std::cout << point;  // natural syntax
```

So the left operand decides the form: when it's not your type, the operator has to live outside your class.

### The Canonical Operator Patterns

There are well-worn "canonical" shapes for these operators. Following them means you write the logic once and get correct, symmetric behavior for free.

**Arithmetic operators:** put the real work in `+=` (a member), then build `+` on top of it (a free function):

```cpp
struct Vec2 {
    double x, y;

    // Member: modifies this object
    Vec2& operator+=(const Vec2& rhs) {
        x += rhs.x;
        y += rhs.y;
        return *this;
    }

    // C++20 spaceship operator for all comparisons
    auto operator<=>(const Vec2&) const = default;
};

// Free function: creates a new object (enables implicit conversion on both sides)
Vec2 operator+(Vec2 lhs, const Vec2& rhs) {
    lhs += rhs;  // Reuses operator+= (lhs is a copy!)
    return lhs;
}
```

**Unary operators** are smaller but follow the same spirit - mutate-in-place ones return a reference, value-producing ones return a fresh object:

```cpp
struct Integer {
    int val;

    Integer  operator-() const { return Integer{-val}; }       // Unary minus
    Integer  operator+() const { return *this; }                // Unary plus
    Integer& operator++()      { ++val; return *this; }        // Pre-increment
    Integer  operator++(int)   { auto old = *this; ++val; return old; } // Post-increment
    bool     operator!() const { return val == 0; }             // Logical NOT
};
```

### C++20 Comparison Operators

Before C++20, writing all six comparison operators by hand was tedious and bug-prone. The spaceship operator `<=>` collapses that into a single line, and the compiler synthesizes the rest:

```cpp
#include <compare>

struct Point3D {
    double x, y, z;

    // Generates ==, !=, <, >, <=, >= automatically!
    auto operator<=>(const Point3D&) const = default;
};
```

When the default member-wise comparison isn't what you want, you write `<=>` yourself and pick the ordering category that fits:

```cpp
struct CaseInsensitiveString {
    std::string data;

    bool operator==(const CaseInsensitiveString& other) const {
        return std::ranges::equal(data, other.data, [](char a, char b) {
            return std::tolower(a) == std::tolower(b);
        });
    }

    std::weak_ordering operator<=>(const CaseInsensitiveString& other) const {
        // Custom comparison logic
        auto a = data, b = other.data;
        std::transform(a.begin(), a.end(), a.begin(), ::tolower);
        std::transform(b.begin(), b.end(), b.begin(), ::tolower);
        return a <=> b;
    }
};
```

(The ordering is `weak_ordering` here because two strings that differ only in case compare *equivalent* but aren't truly equal - a perfect fit for the weak category.)

### Overloading Pitfalls

A few things to never do, learned the hard way by generations of C++ programmers:

1. **Don't overload `&&`, `||`, or `,`** - overloaded versions become ordinary function calls and lose short-circuit / sequencing guarantees.
2. **Don't change what an operator means** - `operator+` should add. Surprising semantics are how you make code unreadable.
3. **Be consistent** - if you overload `==`, give it `!=` too (or just `= default` in C++20 and let the compiler do it).
4. **Mind your return types** - `operator+` returns by value, `operator+=` returns by reference.

---

## Self-Assessment

### Q1: List operators that cannot be overloaded

These are the operators C++ simply won't let you touch, with the reasoning for each:

| Operator | Symbol | Why |
| --- | --- | --- |
| Scope resolution | `::` | Compile-time name lookup, not a runtime operation |
| Member access | `.` | Must always access the actual member, no alternative meaning |
| Pointer-to-member | `.*` | Same constraint as `.` |
| Ternary conditional | `?:` | Unique short-circuit semantics can't be replicated in a function |
| `sizeof` | `sizeof` | Compile-time query, not a function |
| `typeid` | `typeid` | RTTI operator, not overloadable |
| `alignof` | `alignof` | Compile-time alignment query |
| `noexcept` | `noexcept` | Compile-time exception specification check |

And two more that aren't even real operators in the runtime sense: the preprocessor's `#` (stringize) and `##` (token paste).

### Q2: Explain why `operator=` must be a member and `operator<<` is typically free

**`operator=` must be a non-static member**, for three reinforcing reasons:

1. **Invariant protection:** only the class author should decide how assignment preserves things like resource ownership or reference counts.
2. **Default generation:** the compiler auto-generates `operator=` when you don't write one - and that machinery only works for members.
3. **Anti-hijacking:** if outsiders could define `operator=` for your type, they could silently break its semantics.

```cpp
struct Resource {
    int* ptr;
    // Only the class author should decide how assignment works
    Resource& operator=(const Resource& other) {
        if (this != &other) {
            delete ptr;
            ptr = new int(*other.ptr);
        }
        return *this;
    }
};
```

**`operator<<` is typically a free function (often a hidden friend)** for the mirror-image reason:

1. **The left operand isn't your type:** `std::cout << obj` is really `operator<<(std::cout, obj)`, and you can't add members to `std::ostream`.
2. **As a member it would read backwards:** `this` would be the left operand, forcing the unnatural `obj << std::cout`.
3. **Hidden friend** is the modern, tidy way to write it:

```cpp
struct Color {
    int r, g, b;
    friend std::ostream& operator<<(std::ostream& os, const Color& c) {
        return os << "rgb(" << c.r << "," << c.g << "," << c.b << ")";
    }
};
```

### Q3: Show the canonical form for overloading `operator+` as a free function using `operator+=`

This is the pattern worth memorizing. Notice that every binary arithmetic operator below is a two-line free function that delegates to its compound-assignment member - zero duplicated math:

```cpp
#include <iostream>
#include <cmath>

struct Complex {
    double real, imag;

    // Constructor
    Complex(double r = 0, double i = 0) : real(r), imag(i) {}

    // operator+= as MEMBER (modifies *this)
    Complex& operator+=(const Complex& rhs) {
        real += rhs.real;
        imag += rhs.imag;
        return *this;
    }

    Complex& operator-=(const Complex& rhs) {
        real -= rhs.real;
        imag -= rhs.imag;
        return *this;
    }

    Complex& operator*=(const Complex& rhs) {
        double new_real = real * rhs.real - imag * rhs.imag;
        double new_imag = real * rhs.imag + imag * rhs.real;
        real = new_real;
        imag = new_imag;
        return *this;
    }

    // Unary minus
    Complex operator-() const { return {-real, -imag}; }

    // Comparison (C++20)
    bool operator==(const Complex&) const = default;

    // Stream output as hidden friend
    friend std::ostream& operator<<(std::ostream& os, const Complex& c) {
        return os << c.real << (c.imag >= 0 ? "+" : "") << c.imag << "i";
    }
};

// operator+ as FREE FUNCTION - takes lhs BY VALUE (it's a copy!)
Complex operator+(Complex lhs, const Complex& rhs) {
    lhs += rhs;     // Reuse operator+=
    return lhs;     // Return the modified copy
}

Complex operator-(Complex lhs, const Complex& rhs) {
    lhs -= rhs;
    return lhs;
}

Complex operator*(Complex lhs, const Complex& rhs) {
    lhs *= rhs;
    return lhs;
}

int main() {
    Complex a{3, 4}, b{1, -2};

    std::cout << "a     = " << a << "\n";       // 3+4i
    std::cout << "b     = " << b << "\n";       // 1-2i
    std::cout << "a + b = " << (a + b) << "\n"; // 4+2i
    std::cout << "a - b = " << (a - b) << "\n"; // 2+6i
    std::cout << "a * b = " << (a * b) << "\n"; // 11-2i
    std::cout << "-a    = " << (-a) << "\n";    // -3-4i

    // Chaining works:
    Complex c = a + b + Complex{10, 0};
    std::cout << "a+b+10 = " << c << "\n";      // 14+2i

    return 0;
}
```

**Why this pattern is the canonical one:**

1. **`operator+=` is the member** - it mutates `*this` and returns a reference. Natural, intuitive.
2. **`operator+` is a free function** taking `lhs` **by value** (a copy you're free to modify), applying `+=`, then returning that copy.
3. **No duplicated logic** - the actual addition lives only in `operator+=`.
4. **Symmetric conversions** - because `+` is free, implicit conversions can fire on *either* operand, so `1.0 + c` works as well as `c + 1.0`.
5. **Still efficient** - that by-value copy is usually elided by NRVO, so you don't pay for it.

---

## Additional Examples

### Subscript Operator (C++23 Multidimensional)

`operator[]` used to take exactly one index. C++23 lifts that restriction, so a matrix can finally use `m[r, c]` instead of `m(r, c)`:

```cpp
#include <vector>
#include <stdexcept>
#include <iostream>

class Matrix {
    std::vector<double> data_;
    std::size_t rows_, cols_;
public:
    Matrix(std::size_t r, std::size_t c) : data_(r * c, 0.0), rows_(r), cols_(c) {}

    // C++23: multidimensional subscript
    double& operator[](std::size_t r, std::size_t c) {
        return data_[r * cols_ + c];
    }
    const double& operator[](std::size_t r, std::size_t c) const {
        return data_[r * cols_ + c];
    }

    // Pre-C++23 fallback: operator()(r, c)
    double& operator()(std::size_t r, std::size_t c) {
        return data_[r * cols_ + c];
    }
};

int main() {
    Matrix m(3, 3);
    m[1, 2] = 42.0;             // C++23 multidimensional subscript
    std::cout << m[1, 2] << "\n"; // 42
}
```

### Function Call Operator (Functor)

Overloading `operator()` turns an object into something callable - a *functor*. This is what lets you pass stateful "functions" to algorithms:

```cpp
#include <iostream>
#include <algorithm>
#include <vector>

struct Threshold {
    double limit;
    bool operator()(double val) const {  // Makes Threshold a functor
        return val > limit;
    }
};

int main() {
    std::vector<double> data = {1.5, 3.7, 2.1, 4.8, 0.3};
    auto count = std::count_if(data.begin(), data.end(), Threshold{3.0});
    std::cout << count << " values exceed 3.0\n";  // 2
}
```

---

## Notes

- Use the **canonical patterns**: `+=` as member, `+` as a free function that reuses `+=`.
- In C++20, prefer `operator<=>` with `= default` for comparisons - it writes the boilerplate for you.
- Use **hidden friends** for stream operators and any ADL-only operators.
- Never overload `&&`, `||`, or `,` - they quietly lose their special evaluation semantics.
- `operator[]` gained multi-argument support in C++23.
