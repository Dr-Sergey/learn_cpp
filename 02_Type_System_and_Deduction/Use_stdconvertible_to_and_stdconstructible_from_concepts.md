# Use `std::convertible_to` and `std::constructible_from` Concepts

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/concepts/convertible_to>  

---

## Topic Overview

### What Are These Concepts

`std::convertible_to<From, To>` and `std::constructible_from<T, Args...>` are C++20 concepts that check convertibility and constructibility relationships between types. They look similar at first glance but draw an important line: one requires that conversions happen implicitly, the other is happy with explicit constructors too.

### `std::convertible_to<From, To>`

This concept checks if `From` can be **implicitly AND explicitly** converted to `To`. Both paths must work:

```cpp
#include <concepts>

// Requires BOTH:
//   1. static_cast<To>(from)  works (explicit)
//   2. To t = from;           works (implicit)

static_assert(std::convertible_to<int, double>);     // int -> double: OK
static_assert(std::convertible_to<double, int>);      // double -> int: narrowing but allowed
static_assert(!std::convertible_to<int*, double*>);   // unrelated pointers: No
```

### `std::constructible_from<T, Args...>`

This concept checks if `T` can be **directly constructed** from `Args...`. Explicit constructors count:

```cpp
#include <concepts>
#include <string>

static_assert(std::constructible_from<std::string, const char*>);  // string("hello") OK
static_assert(std::constructible_from<std::string, 5, 'x'>);      // string(5, 'x') OK
static_assert(std::constructible_from<int>);                        // int() OK (default ctor)
```

### Key Difference: Implicit vs Explicit

This is the crux of it. The difference between these two concepts is whether an explicit constructor is sufficient:

| Conversion Type | `convertible_to` | `constructible_from` |
| --- | --- | --- |
| Implicit conversion (`To t = from;`) | Required | Not required |
| Explicit construction (`To t(from);`) | Required | Required |
| Explicit-only constructors | Rejects | Accepts |

Let's see that directly with two types - one with an implicit constructor and one with an explicit constructor:

```cpp
struct Explicit {
    explicit Explicit(int) {}   // explicit ctor
};

struct Implicit {
    Implicit(int) {}            // implicit (converting) ctor
};

// constructible_from: accepts both explicit and implicit
static_assert(std::constructible_from<Explicit, int>);  // OK: Explicit e(42)
static_assert(std::constructible_from<Implicit, int>);  // OK: Implicit i(42)

// convertible_to: requires implicit conversion
static_assert(!std::convertible_to<int, Explicit>);     // FAIL: Explicit e = 42; won't compile
static_assert(std::convertible_to<int, Implicit>);      // OK: Implicit i = 42; works
```

The key intuition is: `convertible_to` guards against accidental implicit conversions by refusing to model the relationship unless `To t = from` actually compiles. If you constrain a function parameter with `convertible_to`, callers can't accidentally sneak in an `explicit` constructor conversion.

### Related Concepts

Here is the broader family of construction-related concepts, so you can pick the right tool:

| Concept | Checks |
| --- | --- |
| `std::convertible_to<From, To>` | Implicit + explicit conversion |
| `std::constructible_from<T, Args...>` | Direct construction from Args |
| `std::default_initializable<T>` | `T()` and `T t;` and `T{}` |
| `std::move_constructible<T>` | `T(std::move(t))` |
| `std::copy_constructible<T>` | `T(t)` for const/non-const lvalue |

### How the Standard Library Uses These

```cpp
// std::convertible_to is used in:
// - Ranges sentinel concept: sentinel_for<S, I> requires convertible_to for comparison
// - Iterator concept: indirectly_readable requires convertible_to for reference types
// - Concept return type constraints: { expr } -> std::convertible_to<bool>

// std::constructible_from is used in:
// - std::optional::emplace, std::variant::emplace
// - Container emplace operations
// - allocator_traits::construct requirements
```

---

## Self-Assessment

### Q1: Write a `Numeric` concept requiring `constructible_from<int>` and `convertible_to<double>`

This example builds a concept from scratch using both constraints, then checks which types satisfy it:

```cpp
#include <iostream>
#include <concepts>
#include <type_traits>
#include <string>

// Numeric: can be built from int, can be implicitly used as double
template<typename T>
concept Numeric = std::constructible_from<T, int>
               && std::convertible_to<T, double>;

// Algorithm constrained with our concept
template<Numeric T>
double average(T a, T b) {
    // convertible_to<T, double> guarantees implicit conversion works
    double sum = a + b;  // T implicitly converts to double for arithmetic
    return sum / 2.0;
}

template<Numeric T>
T from_int(int value) {
    // constructible_from<T, int> guarantees this works
    return T(value);
}

// Types to test
struct MyNum {
    double val;
    MyNum(int v) : val(v) {}           // constructible from int
    operator double() const { return val; } // convertible to double
};

struct ExplicitNum {
    double val;
    explicit ExplicitNum(int v) : val(v) {}    // constructible from int (explicit)
    explicit operator double() const { return val; } // NOT implicitly convertible!
};

struct StringLike {
    std::string s;
    StringLike(int v) : s(std::to_string(v)) {} // constructible from int
    // NOT convertible to double
};

int main() {
    std::cout << std::boolalpha;

    // Verify concept satisfaction
    std::cout << "=== Concept checks ===\n";
    std::cout << "int:         " << Numeric<int> << "\n";          // true
    std::cout << "double:      " << Numeric<double> << "\n";       // true
    std::cout << "float:       " << Numeric<float> << "\n";        // true
    std::cout << "MyNum:       " << Numeric<MyNum> << "\n";        // true
    std::cout << "ExplicitNum: " << Numeric<ExplicitNum> << "\n";  // false (no implicit -> double)
    std::cout << "StringLike:  " << Numeric<StringLike> << "\n";   // false (not -> double)
    std::cout << "std::string: " << Numeric<std::string> << "\n";  // false

    // Use with satisfying types
    std::cout << "\n=== average() ===\n";
    std::cout << "average(3, 7) = " << average(3, 7) << "\n";
    std::cout << "average(2.5, 3.5) = " << average(2.5, 3.5) << "\n";
    std::cout << "average(MyNum(10), MyNum(20)) = " << average(MyNum(10), MyNum(20)) << "\n";

    std::cout << "\n=== from_int() ===\n";
    auto n1 = from_int<int>(42);
    auto n2 = from_int<double>(42);
    auto n3 = from_int<MyNum>(42);
    std::cout << "from_int<int>(42) = " << n1 << "\n";
    std::cout << "from_int<double>(42) = " << n2 << "\n";
    std::cout << "from_int<MyNum>(42) = " << static_cast<double>(n3) << "\n";

    // These would fail:
    // average(ExplicitNum(1), ExplicitNum(2));  // ERROR: Numeric<ExplicitNum> not satisfied
    // from_int<std::string>(42);                // ERROR: Numeric<std::string> not satisfied

    return 0;
}
```

**Output:**

```text
=== Concept checks ===
int:         true
double:      true
float:       true
MyNum:       true
ExplicitNum: false
StringLike:  false
std::string: false

=== average() ===
average(3, 7) = 5
average(2.5, 3.5) = 3
average(MyNum(10), MyNum(20)) = 15

=== from_int() ===
from_int<int>(42) = 42
from_int<double>(42) = 42
from_int<MyNum>(42) = 42
```

### Q2: Explain the difference between `convertible_to` (implicit) and `constructible_from` (explicit allowed)

The `unique_ptr` example here is one of the best illustrations because it's something you already know - `unique_ptr` has an explicit constructor on purpose:

```cpp
#include <iostream>
#include <concepts>
#include <string>
#include <memory>

// === The key difference: explicit vs implicit ===

struct Widget {
    int value;

    // Implicit converting constructor
    Widget(int v) : value(v) {}

    // Explicit conversion to string
    explicit operator std::string() const { return std::to_string(value); }

    // Implicit conversion to double
    operator double() const { return static_cast<double>(value); }
};

// std::unique_ptr has explicit constructor from raw pointer:
// explicit unique_ptr(pointer p);

int main() {
    std::cout << std::boolalpha;

    // === Widget ===
    std::cout << "=== Widget ===\n";

    // constructible_from: accepts explicit constructors
    std::cout << "constructible_from<Widget, int>: "
              << std::constructible_from<Widget, int> << "\n";  // true

    // convertible_to: requires implicit conversion
    std::cout << "convertible_to<int, Widget>: "
              << std::convertible_to<int, Widget> << "\n";    // true (implicit ctor)

    // Widget -> string: explicit only
    std::cout << "constructible_from<string, Widget>: "
              << std::constructible_from<std::string, Widget> << "\n";  // false (no suitable ctor)
    std::cout << "convertible_to<Widget, string>: "
              << std::convertible_to<Widget, std::string> << "\n";  // false (explicit operator)

    // Widget -> double: implicit conversion operator
    std::cout << "convertible_to<Widget, double>: "
              << std::convertible_to<Widget, double> << "\n";  // true

    // === unique_ptr: explicit constructor ===
    std::cout << "\n=== unique_ptr ===\n";
    std::cout << "constructible_from<unique_ptr<int>, int*>: "
              << std::constructible_from<std::unique_ptr<int>, int*> << "\n";  // true
    std::cout << "convertible_to<int*, unique_ptr<int>>: "
              << std::convertible_to<int*, std::unique_ptr<int>> << "\n";  // false!

    // This is why:
    // std::unique_ptr<int> p = new int(42);     // ERROR: explicit ctor
    // std::unique_ptr<int> p(new int(42));       // OK: direct construction

    // === Practical implications ===
    std::cout << "\n=== When to use each ===\n";

    // Use convertible_to when you need SAFE implicit conversions
    // (e.g., function parameters, return types)
    // void f(Widget w);  f(42);  <- needs convertible_to<int, Widget>

    // Use constructible_from when explicit construction is acceptable
    // (e.g., emplace, factory functions)
    // container.emplace(args...);  <- needs constructible_from<T, Args...>

    std::cout << "convertible_to: checks implicit (safe for APIs)\n";
    std::cout << "constructible_from: checks explicit (ok for construction sites)\n";

    return 0;
}
```

**Output:**

```text
=== Widget ===
constructible_from<Widget, int>: true
convertible_to<int, Widget>: true
constructible_from<string, Widget>: false
convertible_to<Widget, string>: false
convertible_to<Widget, double>: true

=== unique_ptr ===
constructible_from<unique_ptr<int>, int*>: true
convertible_to<int*, unique_ptr<int>>: false

=== When to use each ===
convertible_to: checks implicit (safe for APIs)
constructible_from: checks explicit (ok for construction sites)
```

**Key insight:** `convertible_to` is **stricter** than `constructible_from` - it requires that the conversion can happen implicitly (without a cast). `constructible_from` accepts both implicit and explicit constructions.

### Q3: Show how these concepts are used in the standard ranges iterator requirements

The ranges library leans on both of these concepts throughout its iterator hierarchy. This example builds a custom iterator and shows the verification in action:

```cpp
#include <iostream>
#include <concepts>
#include <ranges>
#include <iterator>
#include <vector>
#include <type_traits>

// The ranges library uses convertible_to and constructible_from extensively:

// 1. std::indirectly_readable requires:
//    common_reference_with<iter_reference_t<I>&&, iter_value_t<I>&>
//    which internally checks convertible_to

// 2. In requires-expressions, return type constraints use -> convertible_to<bool>:
//    { a == b } -> std::convertible_to<bool>;

// 3. sentinel_for concept:
//    requires(I i, S s) { {i == s} -> std::convertible_to<bool>; }

// Let's demonstrate:

// Custom iterator that models input_iterator
struct CountIterator {
    using value_type = int;
    using difference_type = std::ptrdiff_t;

    int current = 0;

    int operator*() const { return current; }
    CountIterator& operator++() { ++current; return *this; }
    CountIterator operator++(int) { auto tmp = *this; ++current; return tmp; }

    bool operator==(const CountIterator&) const = default;
};

// Custom sentinel
struct CountSentinel {
    int limit;

    // This must return something convertible_to<bool>
    bool operator==(const CountIterator& it) const {
        return it.current >= limit;
    }
};

// Verify concepts
static_assert(std::input_iterator<CountIterator>);
static_assert(std::sentinel_for<CountSentinel, CountIterator>);

// Using convertible_to in our own concept
template<typename T>
concept BoolTestable = std::convertible_to<T, bool> && requires(T t) {
    { !t } -> std::convertible_to<bool>;
};

// Using constructible_from in an emplace-like function
template<typename T, typename... Args>
    requires std::constructible_from<T, Args...>
T make(Args&&... args) {
    return T(std::forward<Args>(args)...);
}

int main() {
    std::cout << std::boolalpha;

    // Our custom iterator works with ranges
    std::cout << "=== Custom iterator with ranges ===\n";
    auto rng = std::ranges::subrange(CountIterator{0}, CountSentinel{5});
    for (int x : rng) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    // Verify convertible_to<bool> usage
    std::cout << "\n=== BoolTestable concept ===\n";
    std::cout << "int BoolTestable: " << BoolTestable<int> << "\n";           // true
    std::cout << "bool BoolTestable: " << BoolTestable<bool> << "\n";         // true
    std::cout << "string BoolTestable: " << BoolTestable<std::string> << "\n"; // false

    // make<T> uses constructible_from
    std::cout << "\n=== make<T> with constructible_from ===\n";
    auto s = make<std::string>(5, 'x');
    std::cout << "make<string>(5, 'x') = " << s << "\n";

    auto v = make<std::vector<int>>(std::initializer_list<int>{1,2,3});
    std::cout << "make<vector<int>>({1,2,3}) size = " << v.size() << "\n";

    // Standard library iterator concept usage
    std::cout << "\n=== Standard iterator concepts ===\n";
    using VIt = std::vector<int>::iterator;
    std::cout << "vector::iterator satisfies input_iterator: "
              << std::input_iterator<VIt> << "\n";

    // The sentinel concept checks == returns convertible_to<bool>
    std::cout << "vector::iterator sentinel_for itself: "
              << std::sentinel_for<VIt, VIt> << "\n";

    return 0;
}
```

**Output:**

```text
=== Custom iterator with ranges ===
0 1 2 3 4

=== BoolTestable concept ===
int BoolTestable: true
bool BoolTestable: true
string BoolTestable: false

=== make<T> with constructible_from ===
make<string>(5, 'x') = xxxxx
make<vector<int>>({1,2,3}) size = 3

=== Standard iterator concepts ===
vector::iterator satisfies input_iterator: true
vector::iterator sentinel_for itself: true
```

Notice how `convertible_to<bool>` in requires-expressions ensures comparison operators return boolean-like results, `sentinel_for<S, I>` checks that `i == s` returns something `convertible_to<bool>`, and `constructible_from` enables generic factory/emplace patterns that work with explicit constructors. These concepts are fundamental to the C++20 ranges iterator concept hierarchy.

---

## Notes

- `std::convertible_to<From, To>` checks **both** explicit and implicit conversion paths. This is stronger than just `std::is_convertible_v<From, To>` because the concept also tests the explicit cast.
- `std::constructible_from<T, Args...>` does **not** check that the construction is `noexcept`. Use `std::is_nothrow_constructible_v<T, Args...>` for that.
- In requires-expressions, `{ expr } -> std::convertible_to<T>` is the standard way to constrain return types. Don't use `std::same_as` unless you need exact type match (it's too strict for most cases).
- `std::convertible_to` is preferred in public API constraints because it prevents accidental implicit conversions from `explicit` constructors.
