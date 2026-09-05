# Understand template lambdas (C++20) with explicit template parameter lists

**Category:** Lambda and Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

C++20 allows lambdas to have **explicit template parameter lists**, written between `[]` and `()`. You've probably used C++14 generic lambdas with `auto` parameters, which already give you a form of genericity. Template lambdas go further by giving you an actual name for the type, which unlocks everything you'd normally do with a template function:

```cpp
auto f = []<typename T>(T x) { return x * 2; };
```

This is more powerful than C++14 generic lambdas (`auto` parameters) because you can:

- Name the type explicitly (use `T` in the body)
- Constrain with concepts
- Use non-type template parameters
- Apply `sizeof...(Args)`, partial specialization patterns, etc.

The reason `auto` parameters aren't always enough is that once the type is deduced but unnamed, you need `decltype` gymnastics to refer to it again. Template lambdas eliminate that noise.

### Comparison

| Feature | Generic lambda (`auto`) | Template lambda (`<typename T>`) |
| --- | --- | --- |
| Access to type name | No - need `decltype(x)` | Yes - `T` is available |
| Concept constraints | `void f(Sortable auto x)` | `requires Sortable<T>` |
| Non-type params | Not possible | `[]<int N>() { ... }` |
| Pack expansion | `(auto... args)` | `<typename... Ts>(Ts... args)` |
| SFINAE | Awkward | Natural |

---

## Self-Assessment

### Q1: Write `[]<typename T>(T x) { ... }` and show how it differs from a generic lambda

The core difference shows up when you need to use the type name inside the body. With a generic lambda, you'd have to write `typename std::decay_t<decltype(container)>::value_type` to get the element type - which works but reads like noise. With a template lambda, `T` is just `T`.

```cpp
#include <iostream>
#include <vector>
#include <type_traits>

int main() {
    // C++14 generic lambda - type is deduced but unnamed
    auto generic = [](const auto& container) {
        // To get the element type, need verbose decltype:
        using ValueType = typename std::decay_t<decltype(container)>::value_type;
        std::cout << "Generic: " << sizeof(ValueType) << " bytes per element, "
                  << container.size() << " elements\n";
    };

    // C++20 template lambda - type parameter is NAMED
    auto templated = []<typename T>(const std::vector<T>& container) {
        // T is directly available - no decltype needed!
        std::cout << "Template: " << sizeof(T) << " bytes per element, "
                  << container.size() << " elements\n";

        // Can use T for other purposes:
        if constexpr (std::is_floating_point_v<T>)
            std::cout << "  (floating point type)\n";
        else
            std::cout << "  (non-floating type)\n";
    };

    std::vector<int> ints = {1, 2, 3};
    std::vector<double> doubles = {1.1, 2.2};

    generic(ints);
    generic(doubles);

    std::cout << "---\n";

    templated(ints);     // T = int
    templated(doubles);  // T = double

    // Template lambda can also enforce exact type matching:
    auto same_type = []<typename T>(T a, T b) {
        return a + b;
    };
    std::cout << same_type(1, 2) << "\n";      // OK: both int
    // same_type(1, 2.0);  // ERROR: T deduced as int AND double - ambiguous!
}
// Expected output:
//   Generic: 4 bytes per element, 3 elements
//   Generic: 8 bytes per element, 2 elements
//   ---
//   Template: 4 bytes per element, 3 elements
//     (non-floating type)
//   Template: 8 bytes per element, 2 elements
//     (floating point type)
//   3
```

Notice the `same_type` lambda at the end - because `T` must be consistent, calling it with mixed types is a compile error. That's often exactly what you want for strong type checking.

---

### Q2: Use a template lambda to constrain the `auto` parameter to a specific concept

Template lambdas pair naturally with C++20 concepts. You just write the concept name in place of `typename`, and the compiler enforces the constraint at the call site with a clear error message:

```cpp
#include <iostream>
#include <concepts>
#include <string>
#include <vector>
#include <numeric>

// Define a concept
template <typename T>
concept Summable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
};

int main() {
    // C++20: constrain with concept in template lambda
    auto sum = []<Summable T>(const std::vector<T>& v) -> T {
        return std::accumulate(v.begin(), v.end(), T{});
    };

    std::vector<int> ints = {1, 2, 3, 4, 5};
    std::vector<double> doubles = {1.1, 2.2, 3.3};
    std::vector<std::string> strings = {"Hello", " ", "World"};

    std::cout << "int sum: " << sum(ints) << "\n";
    std::cout << "double sum: " << sum(doubles) << "\n";
    std::cout << "string sum: " << sum(strings) << "\n";

    // Compile error if concept not satisfied:
    // struct Foo {};
    // std::vector<Foo> foos;
    // sum(foos);  // ERROR: Foo does not satisfy Summable

    // Shorthand concept constraint (also C++20):
    auto print_integral = []<std::integral T>(T value) {
        std::cout << "Integral value: " << value << "\n";
    };

    print_integral(42);
    print_integral(short{10});
    // print_integral(3.14);  // ERROR: double does not satisfy std::integral
}
// Expected output:
//   int sum: 15
//   double sum: 6.6
//   string sum: Hello World
//   Integral value: 42
//   Integral value: 10
```

The `std::string` accumulation works because `std::string`'s `operator+` satisfies `Summable`. This is a pleasant property of concept-based constraints: they're structural, so types you didn't anticipate can still satisfy them.

---

### Q3: Show how template lambdas simplify code that previously needed a local struct with `operator()`

Before C++14 generic lambdas and C++20 template lambdas, if you needed a templated callable you had to write a named struct with a templated `operator()`. Template lambdas replace that boilerplate with inline code at the point of use:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <type_traits>
#include <array>
#include <string>

int main() {
    // BEFORE (C++11/14): local struct for complex generic logic
    struct TypePrinter {
        template <typename T>
        void operator()(const std::vector<T>& v) const {
            std::cout << "Vector of " << typeid(T).name()
                      << ", size=" << v.size();
            if constexpr (std::is_arithmetic_v<T>) {
                T total{};
                for (auto x : v) total += x;
                std::cout << ", sum=" << total;
            }
            std::cout << "\n";
        }
    };

    TypePrinter printer;
    printer(std::vector<int>{1, 2, 3});
    printer(std::vector<double>{1.1, 2.2});
    printer(std::vector<std::string>{"a", "b"});

    std::cout << "---\n";

    // AFTER (C++20): template lambda - inline, no separate struct
    auto print = []<typename T>(const std::vector<T>& v) {
        std::cout << "Vector of " << typeid(T).name()
                  << ", size=" << v.size();
        if constexpr (std::is_arithmetic_v<T>) {
            T total{};
            for (auto x : v) total += x;
            std::cout << ", sum=" << total;
        }
        std::cout << "\n";
    };

    print(std::vector<int>{1, 2, 3});
    print(std::vector<double>{1.1, 2.2});
    print(std::vector<std::string>{"a", "b"});

    // Non-type template parameter (also C++20):
    auto make_array = []<typename T, int N>() {
        std::array<T, N> arr{};
        for (int i = 0; i < N; ++i) arr[i] = T(i);
        return arr;
    };

    auto arr = make_array.template operator()<double, 5>();
    for (double x : arr) std::cout << x << " ";
    std::cout << "\n";
}
// Expected output (type names may vary by compiler):
//   Vector of i, size=3, sum=6
//   Vector of d, size=2, sum=3.3
//   Vector of NSt..., size=2
//   ---
//   Vector of i, size=3, sum=6
//   Vector of d, size=2, sum=3.3
//   Vector of NSt..., size=2
//   0 1 2 3 4
```

The `make_array` example also shows non-type template parameters in a lambda - a capability that simply didn't exist before C++20. The `.template operator()<double, 5>()` syntax is a little verbose, but you'd more commonly use this inside another template where the type is deduced automatically.

---

## Notes

- **Syntax:** `[]<typename T, typename U>(T a, U b) { ... }` - template params go between `[]` and `()`.
- **NTTP in lambda:** `[]<int N>() { return std::array<int, N>{}; }` - non-type template parameters are supported.
- **Concept shorthand:** `[]<std::integral T>(T x)` is equivalent to `[]<typename T>(T x) requires std::integral<T>`.
- **Deduction guides:** Template lambdas follow the same template argument deduction rules as function templates.
- **Perfect forwarding:** `[]<typename T>(T&& x) { return f(std::forward<T>(x)); }` works naturally and is cleaner than the `auto&&` + `decltype` dance you'd need with a generic lambda.
