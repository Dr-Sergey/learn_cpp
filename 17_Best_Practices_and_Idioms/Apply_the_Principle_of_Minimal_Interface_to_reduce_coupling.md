# Apply the Principle of Minimal Interface to reduce coupling

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-standalone>  

---

## Topic Overview

The **Principle of Minimal Interface** (Herb Sutter's GotW #84): *if a function can be written as a non-friend non-member using only the public interface, it should be.* This minimizes the number of functions that can break when the class internals change.

The intuition is straightforward. Every function with private access is a function that breaks if you refactor the private data. If `area()` can compute its result through the public `radius()` getter, there's no reason to put it inside the class and make it sensitive to implementation changes. Move it out, and you've reduced the blast radius of any future refactoring.

### Core Idea

The split looks like this. Only the functions that genuinely need private data stay inside the class:

```cpp
class Widget {                       // Member functions: MUST access privates
    int x_;
public:
    int x() const;                   // Getter: needs private access
    void set_x(int v);               // Setter: needs private access
    bool is_valid() const;           // Needs private state
};

// Non-member, non-friend: uses only public API
void print(const Widget& w) {        // Only calls w.x(), w.is_valid()
    // No access to w.x_ directly
}

bool operator==(const Widget& a, const Widget& b) {
    return a.x() == b.x();           // Only public API
}
```

### What to keep as member vs free function

| Keep as member | Make free function |
| --- | --- |
| Needs private/protected access | Can use public API only |
| Virtual functions (polymorphism) | Symmetric operators (==, <, +) |
| Constructors, destructor, assignment | Stream operators (<<, >>) |
| Conversion operators | Algorithm-like operations (sort, find) |

---

## Self-Assessment

### Q1: Factor out methods that can be non-member functions

The before/after below shows how `CircleBloated` stuffs five functions into the class that don't need private access. `Circle` strips it down to just the getter and setter - the only two functions that actually touch `radius_`:

```cpp
#include <cmath>
#include <iostream>
#include <string>
#include <ostream>
#include <numbers>

// BEFORE: bloated interface
class CircleBloated {
    double radius_;
public:
    explicit CircleBloated(double r) : radius_(r) {}
    double radius() const { return radius_; }
    void set_radius(double r) { radius_ = r; }

    // These DON'T need private access:
    double area() const { return 3.14159 * radius_ * radius_; }
    double circumference() const { return 2 * 3.14159 * radius_; }
    double diameter() const { return 2 * radius_; }
    bool is_larger_than(const CircleBloated& other) const {
        return radius_ > other.radius_;
    }
    std::string to_string() const {
        return "Circle(r=" + std::to_string(radius_) + ")";
    }
};

// AFTER: minimal member interface
class Circle {
    double radius_;
public:
    explicit Circle(double r) : radius_(r) {}
    double radius() const { return radius_; }
    void set_radius(double r) { radius_ = r; }
};

// Free functions: can change independently of Circle's internals
inline double area(const Circle& c)          { return std::numbers::pi * c.radius() * c.radius(); }
inline double circumference(const Circle& c) { return 2 * std::numbers::pi * c.radius(); }
inline double diameter(const Circle& c)      { return 2 * c.radius(); }

bool operator<(const Circle& a, const Circle& b) {
    return a.radius() < b.radius();
}

std::ostream& operator<<(std::ostream& os, const Circle& c) {
    return os << "Circle(r=" << c.radius() << ")";
}

int main() {
    Circle c(5.0);
    std::cout << c << '\n';
    std::cout << "Area: " << area(c) << '\n';
    std::cout << "Circumference: " << circumference(c) << '\n';

    Circle c2(3.0);
    std::cout << (c < c2 ? "c is smaller" : "c is larger") << '\n';
}
// Expected output:
// Circle(r=5)
// Area: 78.5398
// Circumference: 31.4159
// c is larger
```

Now if you decide to store `radius_` as a `float` instead of a `double`, you only need to update two functions - the getter and setter. All the computation in the free functions goes untouched.

### Q2: Explain why Herb Sutter argues that non-friend non-member functions are preferred

**Sutter's argument (GotW #84, "Monoliths Unstrung"):**

1. **Encapsulation:** Fewer functions with private access = stronger encapsulation. If you change a private member, only member/friend functions can break.

2. **Extensibility:** Non-member functions can be added in any namespace/header without modifying the class. Member functions require touching the class definition.

3. **ADL (Argument-Dependent Lookup):** Free functions in the same namespace are found via ADL, so `area(c)` works without qualification.

4. **Genericity:** Free functions can be templatized to work with any type that has `radius()`:

```cpp
#include <numbers>
template<typename Shape>
double area(const Shape& s) requires requires { s.radius(); } {
    return std::numbers::pi * s.radius() * s.radius();
}
// Works for Circle, Sphere, or any type with radius()
```

1. **Recompilation:** Adding a non-member function in a new header doesn't require recompiling users of the class.

Point 5 matters in large codebases. If every new utility function goes into the class definition, every file that includes the class header gets recompiled whenever someone adds one more helper. Moving those helpers to separate headers breaks that dependency.

### Q3: Show how a smaller member set enables extensibility without recompilation

Here the `Sensor` class is a stable library type - it has minimal members and nothing that isn't strictly necessary. All the application-specific logic (`is_valid_reading`, `normalize`, `print_report`) lives in separate translation units and can evolve freely without touching `Sensor` at all:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// Library header (stable, rarely changes)
class Sensor {
    double value_;
    bool calibrated_;
public:
    Sensor(double v, bool cal) : value_(v), calibrated_(cal) {}
    double value() const { return value_; }
    bool calibrated() const { return calibrated_; }
};

// ---- Application code (separate header/source) ----
// These can be added without touching Sensor or recompiling it:

bool is_valid_reading(const Sensor& s) {
    return s.calibrated() && s.value() >= 0.0;
}

double normalize(const Sensor& s, double min, double max) {
    return (s.value() - min) / (max - min);
}

void print_report(const std::vector<Sensor>& sensors) {
    for (const auto& s : sensors) {
        std::cout << "Value: " << s.value()
                  << " Valid: " << is_valid_reading(s) << '\n';
    }
}

int main() {
    std::vector<Sensor> sensors = {
        {25.0, true}, {-1.0, true}, {30.0, false}, {42.0, true}
    };
    print_report(sensors);

    auto count = std::count_if(sensors.begin(), sensors.end(), is_valid_reading);
    std::cout << "Valid: " << count << " / " << sensors.size() << '\n';
}
// Expected output:
// Value: 25 Valid: 1
// Value: -1 Valid: 0
// Value: 30 Valid: 0
// Value: 42 Valid: 1
// Valid: 2 / 4
```

Notice also that `is_valid_reading` works directly as a predicate to `std::count_if` - no wrapper needed. That's the ADL + free function story in practice: the function is in the same namespace as `Sensor`, callable without qualification, and compatible with standard algorithms.

---

## Notes

- The C++ Standard Library follows this: `std::sort`, `std::find`, `std::swap` are all free functions.
- ADL makes free functions as convenient as members: `sort(v.begin(), v.end())` finds `std::sort` automatically.
- Operators that should be free: `==`, `!=`, `<`, `+`, `-`, `<<`, `>>`.
- Members that must stay members: constructors, destructor, conversion operators, `operator=`, virtual functions.
