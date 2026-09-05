# Use strong typedefs (phantom types) to prevent unit confusion

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://www.fluentcpp.com/2016/12/08/strong-types-for-strong-interfaces/>  

---

## Topic Overview

**Strong typedefs** wrap primitive types so the compiler distinguishes between values with different meanings (meters vs seconds, dollars vs euros). A regular `typedef` or `using` is just an alias - it does not create a new type, so the compiler cannot tell them apart and will not warn you when you mix them up.

```cpp
using Meters = double;   // WEAK: Meters and Seconds are both double
using Seconds = double;  // distance / time compiles with NO warning!

// Strong typedef: compiler REJECTS mixing
struct Meters  { double value; };
struct Seconds { double value; };
// Meters m{5.0}; Seconds s{3.0};
// m.value / s.value;  // forces explicit extraction
```

The cost is essentially zero at runtime - the compiler optimizes the wrapper away entirely. What you gain is that an entire class of unit-confusion bugs becomes a compile error.

---

## Self-Assessment

### Q1: Implement Meters and Seconds as strong typedefs

The key technique is using an empty tag struct as a type parameter. `StrongType<MetersTag>` and `StrongType<SecondsTag>` are completely different types even though they have the same internal structure, because `MetersTag` and `SecondsTag` are different. Same-type arithmetic works naturally; cross-type arithmetic is a compile error.

```cpp
#include <iostream>

// Strong typedef template
template<typename Tag, typename T = double>
struct StrongType {
    T value;

    explicit constexpr StrongType(T v) : value(v) {}

    // Same-type arithmetic
    constexpr StrongType operator+(StrongType other) const {
        return StrongType(value + other.value);
    }
    constexpr StrongType operator-(StrongType other) const {
        return StrongType(value - other.value);
    }
    constexpr bool operator==(StrongType other) const {
        return value == other.value;
    }
    constexpr bool operator<(StrongType other) const {
        return value < other.value;
    }
};

// Distinct types via unique tags
struct MetersTag {};
struct SecondsTag {};
struct KilogramsTag {};

using Meters   = StrongType<MetersTag>;
using Seconds  = StrongType<SecondsTag>;
using Kilograms = StrongType<KilogramsTag>;

int main() {
    Meters distance(100.0);
    Seconds time(9.58);

    // GOOD: same-type operations work
    Meters total = distance + Meters(50.0);
    std::cout << "Total distance: " << total.value << " m\n";

    // COMPILE ERROR: can't add Meters + Seconds!
    // Meters bad = distance + time;   // ERROR!
    // Meters bad2 = distance + Kilograms(5.0);  // ERROR!

    // Must explicitly extract values for cross-type math:
    double speed = distance.value / time.value;
    std::cout << "Speed: " << speed << " m/s\n";
}
// Expected output:
// Total distance: 150 m
// Speed: 10.4384 m/s
```

The explicit `.value` extraction for cross-type math is intentional - it forces the programmer to acknowledge that they are combining quantities of different dimensions and take responsibility for the correctness of that combination.

### Q2: Generic zero-overhead strong type wrapper

This version uses `operator<=>` for comparison and inline tag struct definitions, which keeps the code compact. Notice that `sizeof(Width) == sizeof(int)` - there is genuinely no overhead, the wrapper exists only at the type system level.

```cpp
#include <iostream>
#include <type_traits>

// Full-featured strong type (zero overhead: same layout as T)
template<typename T, typename Phantom>
class NamedType {
    T value_;
public:
    constexpr explicit NamedType(T val) : value_(val) {}
    constexpr T& get() { return value_; }
    constexpr const T& get() const { return value_; }

    // Comparison operators
    constexpr auto operator<=>(const NamedType&) const = default;
};

// Usage: each using creates a UNIQUE type
using Width  = NamedType<int, struct WidthTag>;
using Height = NamedType<int, struct HeightTag>;
using Depth  = NamedType<int, struct DepthTag>;

// Functions are now type-safe!
void create_box(Width w, Height h, Depth d) {
    std::cout << "Box: " << w.get() << "x" << h.get() << "x" << d.get() << '\n';
}

int main() {
    Width w(10);
    Height h(20);
    Depth d(30);

    create_box(w, h, d);              // OK
    // create_box(h, w, d);           // ERROR: wrong order!
    // create_box(Width(10), Width(20), Width(30));  // ERROR: types must match

    // Zero overhead: same size as underlying type
    static_assert(sizeof(Width) == sizeof(int));
    std::cout << "sizeof(Width) = " << sizeof(Width) << '\n';
}
// Expected output:
// Box: 10x20x30
// sizeof(Width) = 4
```

The `create_box(h, w, d)` error is especially valuable: function argument order mistakes are among the hardest bugs to spot in code review, but the compiler catches them instantly when arguments have distinct types.

### Q3: A real bug strong types prevent (Mars Climate Orbiter)

This is not a hypothetical. In 1999 a $327.6 million spacecraft was lost because one engineering team provided thruster data in pound-force-seconds while another team's software expected newton-seconds. Both values were just `double`. Nobody noticed.

```cpp
#include <iostream>

// ================================================
// THE BUG: Mars Climate Orbiter (1999)
// One team used pound-force-seconds.
// Another team used newton-seconds.
// Both were just `double`. Nobody noticed.
// The spacecraft burned up in Mars' atmosphere.
// Cost: $327.6 million
// ================================================

// BAD: both are just double - easy to mix up
void adjust_trajectory_bad(double impulse_ns) {
    // Assumes newton-seconds!
    double force_n = impulse_ns / 10.0;
    std::cout << "[BAD] Force: " << force_n << " N\n";
}

// GOOD: strong types prevent the bug
template<typename Tag>
struct Impulse {
    double value;
    explicit constexpr Impulse(double v) : value(v) {}
};

using NewtonSeconds = Impulse<struct NSTag>;
using PoundForceSeconds = Impulse<struct LbfTag>;

// Conversion function
constexpr NewtonSeconds to_newton_seconds(PoundForceSeconds pfs) {
    return NewtonSeconds(pfs.value * 4.44822);  // 1 lbf·s = 4.448 N·s
}

void adjust_trajectory_good(NewtonSeconds impulse) {
    double force_n = impulse.value / 10.0;
    std::cout << "[GOOD] Force: " << force_n << " N\n";
}

int main() {
    // BAD: silent unit confusion
    double lbf_s = 100.0;  // pound-force-seconds
    adjust_trajectory_bad(lbf_s);  // OOPS! Treated as newton-seconds!

    // GOOD: compiler catches the mistake
    PoundForceSeconds measured(100.0);
    // adjust_trajectory_good(measured);  // ERROR: wrong type!

    // Must convert explicitly:
    NewtonSeconds converted = to_newton_seconds(measured);
    adjust_trajectory_good(converted);  // Correct!
}
// Expected output:
// [BAD] Force: 10 N
// [GOOD] Force: 44.4822 N
```

With strong types, the line `adjust_trajectory_good(measured)` would not compile - the programmer is forced to write the explicit conversion and, in doing so, is forced to think about whether the conversion is correct.

---

## Notes

- Strong typedefs have zero runtime overhead - the wrapper is optimized away.
- Libraries: Boost.StrongTypedef, `type_safe`, `NamedType`.
- C++26 may standardize strong typedefs natively.
- Use for: units, IDs, indices, coordinates, currency, timestamps.
- Combine with user-defined literals: `auto d = 100.0_meters;`
