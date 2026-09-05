# Use `constexpr` Lambdas as Compile-Time Predicates

**Category:** Compile Time Programming  
**Standard:** C++17, C++20 (constexpr lambdas)  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

### `constexpr` Lambdas

Since C++17, lambdas are **implicitly `constexpr`** if they satisfy the requirements for a `constexpr` function - no annotation needed. Since C++20, you can also explicitly mark a lambda `consteval` to guarantee it only ever runs at compile time.

```cpp
// C++17: implicitly constexpr
auto square = [](int n) { return n * n; };
static_assert(square(5) == 25);  // Works: lambda is constexpr

// C++20: explicitly consteval
auto must_ct = [](int n) consteval { return n * n; };
```

The implicit `constexpr` behavior means that in practice, most simple lambdas you write will just work in compile-time contexts without any changes.

### Lambda as Compile-Time Predicate

A **predicate** is a function that returns `bool`. A **compile-time predicate** is a `constexpr` lambda that you use in `static_assert`, `if constexpr`, or as a template argument to test conditions during compilation rather than at runtime.

This lets you move invariant checks out of runtime assertions and into the build step, where they have zero cost if they pass and catch bugs immediately if they fail:

```cpp
constexpr auto is_even = [](int n) { return n % 2 == 0; };
static_assert(is_even(4));       // Compile-time predicate check
static_assert(!is_even(7));
```

### Key Points

| Feature | C++17 | C++20 |
| --- | --- | --- |
| Implicit `constexpr` lambda | Yes | Yes |
| Lambda in `static_assert` | Yes | Yes |
| Lambda as NTTP (template argument) | No | Yes |
| `consteval` lambda | No | Yes |
| Lambda with captures in `constexpr` | Value only | Value only |

The NTTP column is the big C++20 addition. Before C++20, you couldn't pass a lambda as a non-type template parameter. C++20 makes stateless (capture-free) lambdas usable as template arguments, which opens up a powerful policy pattern.

---

## Self-Assessment

### Q1: Write a `constexpr` lambda used inside a `static_assert` to validate a compile-time condition

Here's a set of predicates that check properties you'd typically want to verify at compile time for configuration data - primality, power-of-two checks, sortedness, and port validity. All of the `static_assert`s below run during compilation:

```cpp
#include <iostream>
#include <array>
#include <type_traits>

// === Basic compile-time predicates ===
constexpr auto is_prime = [](int n) -> bool {
    if (n < 2) return false;
    for (int i = 2; i * i <= n; ++i) {
        if (n % i == 0) return false;
    }
    return true;
};

constexpr auto is_power_of_two = [](unsigned n) -> bool {
    return n > 0 && (n & (n - 1)) == 0;
};

constexpr auto all_positive = [](const auto& arr) -> bool {
    for (const auto& x : arr) {
        if (x <= 0) return false;
    }
    return true;
};

constexpr auto is_sorted = [](const auto& arr) -> bool {
    for (std::size_t i = 1; i < arr.size(); ++i) {
        if (arr[i] < arr[i - 1]) return false;
    }
    return true;
};

// === Use in static_assert ===
static_assert(is_prime(17), "17 must be prime");
static_assert(!is_prime(15), "15 is not prime");
static_assert(is_power_of_two(256), "256 is 2^8");
static_assert(!is_power_of_two(100), "100 is not a power of 2");

constexpr std::array<int, 5> config = {1, 5, 10, 20, 50};
static_assert(all_positive(config), "Config values must be positive");
static_assert(is_sorted(config), "Config values must be sorted");

// === Compound predicate ===
constexpr auto is_valid_port = [](int port) {
    return port > 0 && port < 65536;
};

constexpr auto is_valid_config = [](int port, int timeout_ms) {
    return is_valid_port(port) && timeout_ms > 0 && timeout_ms <= 30000;
};

static_assert(is_valid_config(8080, 5000));
static_assert(!is_valid_config(0, 5000));
static_assert(!is_valid_config(8080, -1));

int main() {
    std::cout << "=== Compile-Time Predicate Checks ===\n";
    std::cout << "is_prime(17): " << is_prime(17) << "\n";
    std::cout << "is_prime(15): " << is_prime(15) << "\n";
    std::cout << "is_power_of_two(256): " << is_power_of_two(256) << "\n";
    std::cout << "all_positive(config): " << all_positive(config) << "\n";
    std::cout << "is_sorted(config): " << is_sorted(config) << "\n";
    std::cout << "is_valid_config(8080, 5000): " << is_valid_config(8080, 5000) << "\n";

    std::cout << "\nAll static_asserts passed at compile time!\n";
    return 0;
}
```

**Expected output:**

```text
=== Compile-Time Predicate Checks ===
is_prime(17): 1
is_prime(15): 0
is_power_of_two(256): 1
all_positive(config): 1
is_sorted(config): 1
is_valid_config(8080, 5000): 1

All static_asserts passed at compile time!
```

### Q2: Show that `constexpr` lambdas can be used in template arguments via NTTP

C++20 allows stateless lambdas (no captures) to be used as non-type template parameters. This enables a clean policy pattern where you parameterize a function or class by behavior - comparator, predicate, transform - all at compile time.

The key point is that the lambda becomes part of the type: `Processor<[](int n) { return n * 2; }>` and `Processor<[](int n) { return n * n; }>` are two distinct types, each optimized for their specific operation.

```cpp
#include <iostream>
#include <array>
#include <algorithm>

// === C++20: Lambda as Non-Type Template Parameter (NTTP) ===
// Stateless lambdas (no captures) can be used as NTTP since C++20

// A sorted array type parameterized by a comparator lambda
template <auto Comparator>
consteval auto sort_array(auto arr) {
    std::sort(arr.begin(), arr.end(), Comparator);
    return arr;
}

// A filtered count parameterized by a predicate lambda
template <auto Predicate, typename Container>
consteval std::size_t count_matching(const Container& c) {
    std::size_t count = 0;
    for (const auto& x : c) {
        if (Predicate(x)) ++count;
    }
    return count;
}

// === Compile-time policy via lambda NTTP ===
template <auto Transform>
struct Processor {
    static constexpr auto process(auto value) {
        return Transform(value);
    }
};

int main() {
    // Sort with different comparators at compile time
    constexpr std::array<int, 5> data = {5, 3, 1, 4, 2};

    constexpr auto ascending = sort_array<[](int a, int b) { return a < b; }>(data);
    constexpr auto descending = sort_array<[](int a, int b) { return a > b; }>(data);

    static_assert(ascending[0] == 1 && ascending[4] == 5);
    static_assert(descending[0] == 5 && descending[4] == 1);

    std::cout << "Ascending:  ";
    for (int x : ascending) std::cout << x << " ";
    std::cout << "\nDescending: ";
    for (int x : descending) std::cout << x << " ";
    std::cout << "\n";

    // Count with different predicates
    constexpr auto evens = count_matching<[](int n) { return n % 2 == 0; }>(data);
    constexpr auto odds = count_matching<[](int n) { return n % 2 != 0; }>(data);
    static_assert(evens == 2);  // 4, 2
    static_assert(odds == 3);   // 5, 3, 1

    std::cout << "Even count: " << evens << "\n";
    std::cout << "Odd count:  " << odds << "\n";

    // Lambda NTTP as policy
    using Doubler = Processor<[](int n) { return n * 2; }>;
    using Squarer = Processor<[](int n) { return n * n; }>;

    std::cout << "Doubler(5): " << Doubler::process(5) << "\n";   // 10
    std::cout << "Squarer(5): " << Squarer::process(5) << "\n";   // 25

    return 0;
}
```

**Expected output:**

```text
Ascending:  1 2 3 4 5
Descending: 5 4 3 2 1
Even count: 2
Odd count:  3
Doubler(5): 10
Squarer(5): 25
```

### Q3: Demonstrate a `constexpr` lambda that captures a `constexpr` variable

Value captures embed the captured value directly into the closure type. Since the value is known at compile time and no address is involved, it's a valid constant expression. Init captures let you go further and compute a derived value at capture time, as long as the initializer is itself a constant expression.

The comment in the example explains why reference captures don't work - it's the address problem again, same as in the previous topic.

```cpp
#include <iostream>
#include <array>

// === constexpr lambda with value capture ===
// Lambdas can capture constexpr variables by VALUE in constant expressions.
// The captured value becomes part of the closure and is available at compile time.

constexpr int SCALE_FACTOR = 10;
constexpr int OFFSET = 5;

// Capture by value - works in constexpr context
constexpr auto scaled = [factor = SCALE_FACTOR](int x) {
    return x * factor;
};

constexpr auto transformed = [s = SCALE_FACTOR, o = OFFSET](int x) {
    return x * s + o;
};

static_assert(scaled(3) == 30);
static_assert(transformed(3) == 35);

// === Init capture with computation ===
constexpr auto half_scale = [h = SCALE_FACTOR / 2](int x) {
    return x * h;
};

static_assert(half_scale(4) == 20);

// === Capture array by value ===
constexpr std::array<int, 4> coefficients = {1, 2, 3, 4};

constexpr auto dot_with_coefficients = [c = coefficients](const std::array<int, 4>& v) {
    int sum = 0;
    for (std::size_t i = 0; i < 4; ++i) {
        sum += c[i] * v[i];
    }
    return sum;
};

constexpr std::array<int, 4> values = {10, 20, 30, 40};
static_assert(dot_with_coefficients(values) == 1*10 + 2*20 + 3*30 + 4*40);  // 300

// === WHY reference capture doesn't work in constexpr ===
// constexpr auto bad = [&SCALE_FACTOR](int x) { return x * SCALE_FACTOR; };
// ERROR: reference to 'SCALE_FACTOR' is not a constant expression
//
// Reason: A reference is essentially a pointer, and pointers to local/global
// objects are not valid constant expressions (addresses are runtime values).

int main() {
    std::cout << "=== constexpr Lambda with Captures ===\n";
    std::cout << "scaled(3) = " << scaled(3) << "\n";               // 30
    std::cout << "transformed(3) = " << transformed(3) << "\n";     // 35
    std::cout << "half_scale(4) = " << half_scale(4) << "\n";       // 20

    std::cout << "\n=== Capture by Value ===\n";
    std::cout << "dot_with_coefficients = " << dot_with_coefficients(values) << "\n";  // 300

    std::cout << "\n=== Rules ===\n";
    std::cout << "By value [x = expr]:  OK in constexpr - value is embedded\n";
    std::cout << "By reference [&x]:    ERROR - address is runtime, not constant\n";
    std::cout << "Init capture [y=f()]: OK if f() is constexpr\n";

    return 0;
}
```

**Expected output:**

```text
=== constexpr Lambda with Captures ===
scaled(3) = 30
transformed(3) = 35
half_scale(4) = 20

=== Capture by Value ===
dot_with_coefficients = 300

=== Rules ===
By value [x = expr]:  OK in constexpr - value is embedded
By reference [&x]:    ERROR - address is runtime, not constant
Init capture [y=f()]: OK if f() is constexpr
```

---

## Notes

- Since C++17, lambdas are **implicitly constexpr** if their body qualifies.
- `constexpr` lambdas can be used in `static_assert`, `if constexpr`, and template arguments (NTTP in C++20).
- Stateless lambdas (no captures) can be NTTP since C++20.
- Captures by **value** work in `constexpr`; captures by **reference** do not (addresses aren't constant expressions).
- Init captures (`[x = expr]`) work if the initializer is a constant expression.
- `consteval` lambdas (C++20) guarantee compile-time-only execution.
