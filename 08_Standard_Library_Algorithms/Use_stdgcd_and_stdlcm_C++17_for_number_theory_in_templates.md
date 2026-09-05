# Use std::gcd and std::lcm (C++17) for number theory in templates

**Category:** Standard Library Algorithms  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/gcd>  

---

## Topic Overview

C++17 added `std::gcd` and `std::lcm` in `<numeric>`. Both are `constexpr` and work with any integer types, which makes them useful not just at runtime but as building blocks for compile-time math.

### Signatures

Here are the function signatures you need:

```cpp
#include <numeric>

constexpr auto std::gcd(M m, N n);  // greatest common divisor
constexpr auto std::lcm(M m, N n);  // least common multiple
```

If the table below feels like a lot, just remember: both functions always return a non-negative result, both handle zero gracefully, and both accept mixed integer types.

| Property | `std::gcd(a, b)` | `std::lcm(a, b)` |
| --- | --- | --- |
| Result | Largest positive divisor of both a and b | Smallest positive multiple of both a and b |
| `gcd(0, 0)` | 0 | - |
| `lcm(0, n)` | - | 0 |
| Sign | Always non-negative | Always non-negative |
| `constexpr` | Yes | Yes |
| Mixed types | Yes (`int` + `long` works) | Yes |

### Mathematical identity

$$\text{lcm}(a, b) = \frac{|a \times b|}{\gcd(a, b)}$$

---

## Self-Assessment

### Q1: Use std::gcd to simplify a rational number type's numerator and denominator

A rational number like `6/8` should automatically reduce to `3/4`. The trick is computing `gcd(numerator, denominator)` and dividing both by the result. The addition operator uses `std::lcm` to find a common denominator, and multiplication uses `gcd` a second time to cross-cancel before multiplying - that cross-cancellation is what prevents intermediate overflow on larger values.

```cpp
#include <iostream>
#include <numeric>
#include <cstdlib>

class Rational {
    int num_, den_;

    void simplify() {
        if (den_ < 0) { num_ = -num_; den_ = -den_; }
        int g = std::gcd(std::abs(num_), den_);
        if (g > 0) { num_ /= g; den_ /= g; }
    }

public:
    constexpr Rational(int num = 0, int den = 1)
        : num_(num), den_(den) {
        if (den_ == 0) throw std::invalid_argument("zero denominator");
        // Can't call simplify() in constexpr pre-C++14, but fine in C++17+
        if (den_ < 0) { num_ = -num_; den_ = -den_; }
        int g = std::gcd(std::abs(num_), den_);
        if (g > 0) { num_ /= g; den_ /= g; }
    }

    int num() const { return num_; }
    int den() const { return den_; }

    Rational operator+(const Rational& rhs) const {
        int common_den = std::lcm(den_, rhs.den_);
        int new_num = num_ * (common_den / den_) + rhs.num_ * (common_den / rhs.den_);
        return Rational(new_num, common_den);
    }

    Rational operator*(const Rational& rhs) const {
        // Cross-simplify to avoid overflow
        int g1 = std::gcd(std::abs(num_), rhs.den_);
        int g2 = std::gcd(std::abs(rhs.num_), den_);
        return Rational((num_ / g1) * (rhs.num_ / g2),
                        (den_ / g2) * (rhs.den_ / g1));
    }

    friend std::ostream& operator<<(std::ostream& os, const Rational& r) {
        os << r.num_;
        if (r.den_ != 1) os << "/" << r.den_;
        return os;
    }
};

int main() {
    Rational a(6, 8);     // automatically simplified to 3/4
    Rational b(10, -15);  // simplified to -2/3

    std::cout << "a = " << a << "\n";     // 3/4
    std::cout << "b = " << b << "\n";     // -2/3

    std::cout << "a + b = " << (a + b) << "\n";   // 3/4 + (-2/3) = 1/12
    std::cout << "a * b = " << (a * b) << "\n";   // 3/4 * (-2/3) = -1/2

    // Verify gcd simplification
    Rational c(100, 250);
    std::cout << "100/250 = " << c << "\n";  // 2/5

    return 0;
}
```

Notice how `100/250` comes out as `2/5` automatically - the constructor does the reduction before you ever see the value.

### Q2: Show that std::gcd works at compile time in a constexpr context

Because `std::gcd` and `std::lcm` are `constexpr`, you can use them in `static_assert` and in compile-time computations. The code below puts that to work - all the `static_assert` lines are checked by the compiler at build time, so if any of them are wrong you get a compile error instead of a runtime surprise.

```cpp
#include <iostream>
#include <numeric>
#include <array>

// === constexpr gcd/lcm ===
constexpr int simplified_num(int n, int d) {
    return n / std::gcd(n, d);
}
constexpr int simplified_den(int n, int d) {
    return d / std::gcd(n, d);
}

// Compile-time verification
static_assert(std::gcd(12, 8) == 4);
static_assert(std::gcd(0, 5) == 5);
static_assert(std::gcd(7, 0) == 7);
static_assert(std::gcd(0, 0) == 0);

static_assert(std::lcm(4, 6) == 12);
static_assert(std::lcm(0, 5) == 0);

static_assert(simplified_num(6, 8) == 3);
static_assert(simplified_den(6, 8) == 4);

// === constexpr with mixed types ===
static_assert(std::gcd(12L, 8) == 4);  // long + int -> common type

// === Compile-time array of reduced fractions ===
constexpr auto make_reduced_fractions() {
    struct Frac { int n, d; };
    std::array<Frac, 4> fracs = {{{2,4}, {3,9}, {10,15}, {7,21}}};
    for (auto& f : fracs) {
        int g = std::gcd(f.n, f.d);
        f.n /= g;
        f.d /= g;
    }
    return fracs;
}

constexpr auto fracs = make_reduced_fractions();
static_assert(fracs[0].n == 1 && fracs[0].d == 2);  // 2/4 -> 1/2
static_assert(fracs[1].n == 1 && fracs[1].d == 3);  // 3/9 -> 1/3
static_assert(fracs[2].n == 2 && fracs[2].d == 3);  // 10/15 -> 2/3
static_assert(fracs[3].n == 1 && fracs[3].d == 3);  // 7/21 -> 1/3

int main() {
    std::cout << "All static_asserts passed — gcd/lcm work at compile time!\n";

    // Runtime demonstration
    for (const auto& f : fracs) {
        std::cout << f.n << "/" << f.d << "\n";
    }
    // 1/2
    // 1/3
    // 2/3
    // 1/3

    return 0;
}
```

The `make_reduced_fractions` function runs entirely at compile time because it is marked `constexpr` and used in a `constexpr` context. The resulting `fracs` array lives in read-only data with no runtime computation needed.

### Q3: Implement a template rational type using NTTP numerator/denominator with gcd reduction

This is where things get interesting. Instead of a runtime class, you can encode rationals directly as template parameters - `Rational<3, 4>` is a different type from `Rational<1, 2>`. The `std::gcd` call inside the template immediately reduces those parameters at instantiation time, so `Rational<6, 8>` and `Rational<3, 4>` end up with identical `num` and `den` constants. The `Reduced` alias normalizes any fraction to its canonical form, which is how the type-level arithmetic stays manageable.

```cpp
#include <iostream>
#include <numeric>
#include <cstdlib>

// === Compile-time rational type using Non-Type Template Parameters ===

template <int Num, int Den>
    requires (Den != 0)
struct Rational {
    // Reduce at compile time
    // std::abs is not constexpr until C++23, so fold the sign by hand here.
    static constexpr int G = std::gcd(Num < 0 ? -Num : Num, Den < 0 ? -Den : Den);
    static constexpr int Sign = (Den < 0) ? -1 : 1;
    static constexpr int num = Sign * Num / G;
    static constexpr int den = Sign * Den / G;
};

// Type alias for the reduced form
template <int N, int D>
using Reduced = Rational<Rational<N, D>::num, Rational<N, D>::den>;

// === Arithmetic on compile-time rationals ===

// Addition: a/b + c/d = (a*d + c*b) / (b*d), then reduce
template <int N1, int D1, int N2, int D2>
using RatAdd = Reduced<N1 * D2 + N2 * D1, D1 * D2>;

// Multiplication: a/b * c/d = (a*c) / (b*d), then reduce
template <int N1, int D1, int N2, int D2>
using RatMul = Reduced<N1 * N2, D1 * D2>;

// Equality check
template <int N1, int D1, int N2, int D2>
constexpr bool rat_equal = (Rational<N1,D1>::num == Rational<N2,D2>::num) &&
                           (Rational<N1,D1>::den == Rational<N2,D2>::den);

// Print helper
template <int N, int D>
void print_rational() {
    using R = Rational<N, D>;
    std::cout << R::num;
    if constexpr (R::den != 1) std::cout << "/" << R::den;
    std::cout << "\n";
}

int main() {
    // All reduction happens at compile time!
    print_rational<6, 8>();      // 3/4
    print_rational<-10, 15>();   // -2/3
    print_rational<10, -15>();   // -2/3  (sign normalized)
    print_rational<7, 1>();      // 7

    // Compile-time arithmetic
    using Sum = RatAdd<1, 3, 1, 6>;  // 1/3 + 1/6 = 1/2
    std::cout << "1/3 + 1/6 = ";
    print_rational<Sum::num, Sum::den>();  // 1/2

    using Prod = RatMul<2, 3, 3, 4>;  // 2/3 * 3/4 = 1/2
    std::cout << "2/3 * 3/4 = ";
    print_rational<Prod::num, Prod::den>();  // 1/2

    // Static verification
    static_assert(Rational<6, 8>::num == 3);
    static_assert(Rational<6, 8>::den == 4);
    static_assert(rat_equal<1, 2, 2, 4>);   // 1/2 == 2/4
    static_assert(rat_equal<Sum::num, Sum::den, 1, 2>);  // 1/3 + 1/6 == 1/2

    return 0;
}
```

This is similar in spirit to how `std::ratio` works inside `<chrono>` - the standard library's duration types like `std::chrono::seconds` and `std::chrono::milliseconds` are built on exactly this kind of compile-time fraction arithmetic.

---

## Notes

- **`std::gcd`/`std::lcm` only work with integer types.** Passing floating-point types is a compile error.
- **Negative arguments are handled:** `gcd(-12, 8) == 4`. The result is always non-negative.
- **Overflow risk with `lcm`:** `lcm(a, b) = |a| / gcd(a,b) * |b|` can overflow if a*b exceeds the integer range. The standard computes it as `(a/gcd) * b` to minimize overflow risk, but large values can still overflow.
- **`std::ratio`** in `<ratio>` is the standard library's compile-time rational type (used by `<chrono>`). It uses similar GCD reduction but predates C++17's `std::gcd`.
- **Before C++17:** You'd write your own `constexpr gcd` using Euclid's algorithm. Now just use `std::gcd`.
