# Use Named Constructor Idiom for expressive object construction

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://isocpp.org/wiki/faq/ctors#named-ctor-idiom>  

---

## Topic Overview

The **Named Constructor Idiom** uses static factory methods with descriptive names instead of overloaded constructors. This disambiguates constructors that would otherwise have identical parameter types.

The problem it solves is a classic: you want to construct an object in multiple ways, but some of those ways look identical at the call site. Two constructors that both take a single `double` cannot coexist. Even when they can coexist, callers have no way to know what the argument means just by reading the code.

```cpp
// BAD: which constructor does what?
Circle c1(5.0);   // radius? diameter? area?
Circle c2(78.5);  // radius? diameter? area?

// GOOD: self-documenting
auto c1 = Circle::fromRadius(5.0);
auto c2 = Circle::fromArea(78.5);
```

---

## Self-Assessment

### Q1: Implement `Circle::fromRadius()` and `Circle::fromArea()` as named constructors

The key move is making the real constructor `private`. That forces all callers through the named constructors, which also means the conversion logic lives in exactly one place per creation path.

```cpp
#include <cmath>
#include <iostream>
#include <ostream>
#include <numbers>

class Circle {
    double radius_;

    // Private constructor: can only be called by named constructors
    explicit Circle(double r) : radius_(r) {}

public:
    // Named constructors (static factory methods)
    static Circle fromRadius(double r) {
        return Circle(r);
    }

    static Circle fromDiameter(double d) {
        return Circle(d / 2.0);
    }

    static Circle fromArea(double area) {
        return Circle(std::sqrt(area / std::numbers::pi));
    }

    static Circle fromCircumference(double c) {
        return Circle(c / (2.0 * std::numbers::pi));
    }

    double radius() const { return radius_; }
    double area() const { return std::numbers::pi * radius_ * radius_; }
    double circumference() const { return 2.0 * std::numbers::pi * radius_; }

    friend std::ostream& operator<<(std::ostream& os, const Circle& c) {
        return os << "Circle(r=" << c.radius_ << ", area=" << c.area() << ")";
    }
};

int main() {
    auto c1 = Circle::fromRadius(5.0);
    auto c2 = Circle::fromDiameter(10.0);
    auto c3 = Circle::fromArea(78.5398);
    auto c4 = Circle::fromCircumference(31.4159);

    std::cout << c1 << '\n';
    std::cout << c2 << '\n';
    std::cout << c3 << '\n';
    std::cout << c4 << '\n';
}
// Expected output:
// Circle(r=5, area=78.5398)
// Circle(r=5, area=78.5398)
// Circle(r=5, area=78.5398)
// Circle(r=5, area=78.5398)
```

All four paths converge on the same internal representation (radius), but callers get to work in whatever units make sense for them.

### Q2: Explain why named constructors beat overloaded constructors with same parameter types

This is the situation where overloading simply cannot help you. Two constructors with identical signatures cannot both exist - the compiler has no way to choose between them. Named constructors sidestep that limitation entirely.

```cpp
#include <iostream>
#include <string>

// BAD: both constructors take a single double -- ambiguous!
class TemperatureBad {
    double kelvin_;
public:
    // Temperature(double celsius) ???
    // Temperature(double fahrenheit) ???
    // Can't have both! Same signature!
    TemperatureBad(double kelvin) : kelvin_(kelvin) {}
};

// GOOD: named constructors disambiguate
class Temperature {
    double kelvin_;
    explicit Temperature(double k) : kelvin_(k) {}

public:
    static Temperature fromKelvin(double k) { return Temperature(k); }
    static Temperature fromCelsius(double c) { return Temperature(c + 273.15); }
    static Temperature fromFahrenheit(double f) { return Temperature((f - 32.0) * 5.0/9.0 + 273.15); }

    double kelvin() const { return kelvin_; }
    double celsius() const { return kelvin_ - 273.15; }
    double fahrenheit() const { return (kelvin_ - 273.15) * 9.0/5.0 + 32.0; }
};

int main() {
    auto boiling = Temperature::fromCelsius(100.0);
    auto body = Temperature::fromFahrenheit(98.6);
    auto absolute_zero = Temperature::fromKelvin(0.0);

    std::cout << "Boiling: " << boiling.celsius() << " C, " << boiling.kelvin() << " K\n";
    std::cout << "Body: " << body.celsius() << " C, " << body.fahrenheit() << " F\n";
    std::cout << "Abs zero: " << absolute_zero.celsius() << " C\n";
}
// Expected output:
// Boiling: 100 C, 373.15 K
// Body: 37 C, 98.6 F
// Abs zero: -273.15 C
```

Notice that the internal representation is always Kelvin - the named constructors handle the unit conversion and the class never exposes that detail to callers.

### Q3: Show how Named Constructor Idiom interacts with CTAD and factory functions

Named constructors are not limited to returning a plain `T` by value. They can return `std::optional<T>` for fallible construction, `std::unique_ptr<T>` for heap allocation, or anything else that makes sense - regular constructors cannot do that.

```cpp
#include <iostream>
#include <memory>
#include <optional>
#include <string>

class Connection {
    std::string host_;
    int port_;
    bool ssl_;

    Connection(std::string host, int port, bool ssl)
        : host_(std::move(host)), port_(port), ssl_(ssl) {}

public:
    // Named constructors return by value (NRVO-friendly)
    static Connection http(const std::string& host, int port = 80) {
        return Connection(host, port, false);
    }

    static Connection https(const std::string& host, int port = 443) {
        return Connection(host, port, true);
    }

    // Can return optional for fallible construction
    static std::optional<Connection> fromUrl(const std::string& url) {
        if (url.starts_with("https://"))
            return https(url.substr(8));
        if (url.starts_with("http://"))
            return http(url.substr(7));
        return std::nullopt;  // invalid URL
    }

    // Can return unique_ptr for heap allocation
    static std::unique_ptr<Connection> make_shared_https(const std::string& host) {
        // Can't use make_unique (private ctor), so:
        return std::unique_ptr<Connection>(new Connection(host, 443, true));
    }

    friend std::ostream& operator<<(std::ostream& os, const Connection& c) {
        return os << (c.ssl_ ? "https" : "http") << "://" << c.host_ << ":" << c.port_;
    }
};

int main() {
    auto c1 = Connection::http("example.com");
    auto c2 = Connection::https("secure.com");
    auto c3 = Connection::fromUrl("https://api.example.com");

    std::cout << c1 << '\n';
    std::cout << c2 << '\n';
    if (c3) std::cout << *c3 << '\n';

    auto bad = Connection::fromUrl("ftp://invalid");
    std::cout << "ftp URL: " << (bad ? "valid" : "invalid") << '\n';
}
// Expected output:
// http://example.com:80
// https://secure.com:443
// https://api.example.com:443
// ftp URL: invalid
```

The `fromUrl` factory returns `std::nullopt` on bad input - something a constructor simply cannot do (it must either succeed or throw). This is one of the most practical advantages of the idiom.

---

## Notes

- Named constructors are NRVO-friendly - the return is typically elided.
- Make the actual constructor `private` to force use of named constructors.
- The standard library uses this pattern: `std::optional::value()`, `std::filesystem::path::string()`.
- `std::chrono` uses it extensively: `seconds(5)`, `milliseconds(100)`.
