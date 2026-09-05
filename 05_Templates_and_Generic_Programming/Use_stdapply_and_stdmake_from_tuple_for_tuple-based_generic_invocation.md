# Use `std::apply` and `std::make_from_tuple` for Tuple-Based Generic Invocation

**Category:** Templates and Generic Programming  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/apply>  

---

## Topic Overview

### What Are `std::apply` and `std::make_from_tuple`

These two utilities solve the same core problem from different angles: you have a tuple holding some values, and you want to use those values as separate arguments - either to call a function or to construct an object. Without these tools you'd need to unpack the tuple manually using `std::get<0>`, `std::get<1>`, and so on, which doesn't generalize across tuple sizes:

```cpp
auto args = std::make_tuple(1, 3.14, "hello");

// Call a function with tuple elements as separate arguments
std::apply(some_function, args);  // -> some_function(1, 3.14, "hello")

// Construct an object from tuple elements
auto obj = std::make_from_tuple<MyClass>(args);  // -> MyClass(1, 3.14, "hello")
```

Both tools are available since C++17 and work with any tuple-like type (`std::tuple`, `std::pair`, `std::array`).

### Key Signatures

```cpp
// C++17
template <class F, class Tuple>
constexpr decltype(auto) apply(F&& f, Tuple&& t);

// C++17
template <class T, class Tuple>
constexpr T make_from_tuple(Tuple&& t);
```

### When to Use

| Use Case | Tool |
| --- | --- |
| Call function with tuple args | `std::apply(f, tuple)` |
| Construct object from tuple | `std::make_from_tuple<T>(tuple)` |
| Iterate over tuple elements | `std::apply` with a lambda |
| Generic dispatch / RPC | Store args in tuple, apply later |

---

## Self-Assessment

### Q1: Use `std::apply` to invoke a function with arguments stored in a `std::tuple`

The power of `std::apply` is that it works with any callable - free functions, lambdas, function objects, and (via a wrapping lambda) member functions. The tuple unpacking is completely generic across sizes:

```cpp
#include <iostream>
#include <tuple>
#include <string>
#include <functional>

// Free function
int add(int a, int b) {
    return a + b;
}

// Multi-type function
void greet(const std::string& name, int age, double height) {
    std::cout << "  Hello " << name << ", age " << age
              << ", height " << height << "m\n";
}

// A class with operator()
struct Multiplier {
    double factor;
    double operator()(double x) const { return x * factor; }
};

int main() {
    std::cout << "=== std::apply with free function ===\n";
    auto args1 = std::make_tuple(3, 7);
    int result = std::apply(add, args1);
    std::cout << "  add(3, 7) = " << result << "\n";  // 10

    std::cout << "\n=== std::apply with multi-type args ===\n";
    auto args2 = std::make_tuple(std::string("Alice"), 30, 1.75);
    std::apply(greet, args2);  // Hello Alice, age 30, height 1.75m

    std::cout << "\n=== std::apply with lambda ===\n";
    auto triple_sum = [](auto a, auto b, auto c) {
        return a + b + c;
    };
    auto args3 = std::make_tuple(1.0, 2.5, 3.5);
    std::cout << "  sum = " << std::apply(triple_sum, args3) << "\n";  // 7.0

    std::cout << "\n=== std::apply with function object ===\n";
    auto args4 = std::make_tuple(5.0);
    Multiplier mul{3.0};
    std::cout << "  5.0 * 3.0 = " << std::apply(mul, args4) << "\n";  // 15.0

    std::cout << "\n=== std::apply with member function ===\n";
    auto str_args = std::make_tuple(std::string("hello world"), 6);
    auto sub = std::apply([](const std::string& s, size_t pos) {
        return s.substr(pos);
    }, str_args);
    std::cout << "  substr(6) = " << sub << "\n";  // "world"

    std::cout << "\n=== Iterate over tuple with apply ===\n";
    auto values = std::make_tuple(42, 3.14, "text");
    std::apply([](auto&&... args) {
        ((std::cout << "  " << args << "\n"), ...);
    }, values);

    return 0;
}
```

### Q2: Construct a class object from a tuple of constructor arguments using `make_from_tuple`

`std::make_from_tuple<T>` is the construction counterpart to `std::apply`. It's especially useful when you want to store constructor arguments and defer the actual construction until later - the `DeferredFactory` pattern at the end illustrates this:

```cpp
#include <iostream>
#include <tuple>
#include <string>
#include <memory>

struct Person {
    std::string name;
    int age;
    double salary;

    Person(std::string n, int a, double s)
        : name(std::move(n)), age(a), salary(s) {}

    void print() const {
        std::cout << "  Person{" << name << ", " << age
                  << ", $" << salary << "}\n";
    }
};

struct Point3D {
    double x, y, z;
    // Aggregate — no constructor needed
    void print() const {
        std::cout << "  Point3D{" << x << ", " << y << ", " << z << "}\n";
    }
};

// Generic factory: create any T from a tuple of args
template <typename T, typename Tuple>
T create(Tuple&& args) {
    return std::make_from_tuple<T>(std::forward<Tuple>(args));
}

// Store constructor args for deferred construction
template <typename T>
class DeferredFactory {
    using ArgsTuple = std::tuple<std::string, int, double>;
    ArgsTuple args_;
public:
    DeferredFactory(ArgsTuple args) : args_(std::move(args)) {}

    T build() const {
        return std::make_from_tuple<T>(args_);
    }
};

int main() {
    std::cout << "=== make_from_tuple with constructor ===\n";
    auto person_args = std::make_tuple(std::string("Bob"), 25, 75000.0);
    Person p = std::make_from_tuple<Person>(person_args);
    p.print();  // Person{Bob, 25, $75000}

    std::cout << "\n=== make_from_tuple with aggregate ===\n";
    auto point_args = std::make_tuple(1.0, 2.0, 3.0);
    Point3D pt = std::make_from_tuple<Point3D>(point_args);
    pt.print();  // Point3D{1, 2, 3}

    std::cout << "\n=== Generic factory ===\n";
    auto p2 = create<Person>(std::make_tuple(std::string("Carol"), 30, 90000.0));
    p2.print();

    std::cout << "\n=== Deferred construction ===\n";
    DeferredFactory<Person> factory(std::make_tuple(std::string("Dave"), 40, 120000.0));
    Person p3 = factory.build();
    p3.print();

    return 0;
}
```

### Q3: Show the equivalence between `std::apply` and a hand-rolled `index_sequence` expansion

If you've ever wondered how `std::apply` actually works internally, here it is. The trick is `std::index_sequence` - it generates a compile-time sequence of integers `0, 1, 2, ...` that are then used as pack-expansion indices into the tuple. This is the standard pattern for unpacking tuple-like types in generic code:

```cpp
#include <iostream>
#include <tuple>
#include <utility>
#include <string>

// === Hand-rolled apply using index_sequence ===
namespace detail {
    template <typename F, typename Tuple, std::size_t... I>
    constexpr decltype(auto) apply_impl(F&& f, Tuple&& t,
                                         std::index_sequence<I...>) {
        return std::invoke(std::forward<F>(f),
                           std::get<I>(std::forward<Tuple>(t))...);
        // std::get<0>(t), std::get<1>(t), std::get<2>(t), ...
    }
}

template <typename F, typename Tuple>
constexpr decltype(auto) my_apply(F&& f, Tuple&& t) {
    return detail::apply_impl(
        std::forward<F>(f),
        std::forward<Tuple>(t),
        std::make_index_sequence<
            std::tuple_size_v<std::remove_reference_t<Tuple>>>{});
}

// === Hand-rolled make_from_tuple ===
namespace detail {
    template <typename T, typename Tuple, std::size_t... I>
    constexpr T make_from_tuple_impl(Tuple&& t, std::index_sequence<I...>) {
        return T(std::get<I>(std::forward<Tuple>(t))...);
    }
}

template <typename T, typename Tuple>
constexpr T my_make_from_tuple(Tuple&& t) {
    return detail::make_from_tuple_impl<T>(
        std::forward<Tuple>(t),
        std::make_index_sequence<
            std::tuple_size_v<std::remove_reference_t<Tuple>>>{});
}

// Test struct
struct Widget {
    std::string name;
    int value;
    Widget(std::string n, int v) : name(std::move(n)), value(v) {}
};

int main() {
    auto args = std::make_tuple(10, 20, 30);

    auto sum = [](int a, int b, int c) { return a + b + c; };

    std::cout << "=== Equivalence: std::apply vs hand-rolled ===\n";
    int r1 = std::apply(sum, args);
    int r2 = my_apply(sum, args);
    std::cout << "std::apply:  " << r1 << "\n";   // 60
    std::cout << "my_apply:    " << r2 << "\n";    // 60

    std::cout << "\n=== Equivalence: make_from_tuple ===\n";
    auto widget_args = std::make_tuple(std::string("gadget"), 42);
    Widget w1 = std::make_from_tuple<Widget>(widget_args);
    Widget w2 = my_make_from_tuple<Widget>(widget_args);
    std::cout << "std: " << w1.name << " " << w1.value << "\n";  // gadget 42
    std::cout << "mine: " << w2.name << " " << w2.value << "\n"; // gadget 42

    std::cout << "\n=== How index_sequence expansion works ===\n";
    std::cout << "Given tuple<int,double,string> t and index_sequence<0,1,2>:\n";
    std::cout << "  std::get<I>(t)... expands to:\n";
    std::cout << "  std::get<0>(t), std::get<1>(t), std::get<2>(t)\n";
    std::cout << "These become the function call arguments.\n";

    return 0;
}
```

---

## Notes

- `std::apply(f, tuple)` - unpacks tuple elements as function arguments. Available since C++17.
- `std::make_from_tuple<T>(tuple)` - constructs `T` from tuple elements. Available since C++17.
- Internally both use `std::index_sequence` to expand the pack: `std::get<0>(t), std::get<1>(t), ...`
- Works with any tuple-like type (`std::tuple`, `std::pair`, `std::array`).
- `std::apply` uses `std::invoke` internally, so it works with member function pointers too.
- Useful for generic dispatch, deferred construction, and RPC-style argument storage.
