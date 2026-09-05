# Write generic lambdas and understand their template mechanics (C++14)

**Category:** Lambda and Functional  
**Standard:** C++14  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

C++14 **generic lambdas** let you write a single lambda that works with any type, just by using `auto` as the parameter type. Under the hood, the compiler generates a closure type with a **templated `operator()`** - each `auto` parameter becomes its own independent template parameter.

Here is the key mental model: when you write `[](auto x, auto y)`, you are not writing a regular function. You are writing a blueprint. Every time you call that lambda with different types, the compiler stamps out a new instantiation of `operator()` for that specific combination of types. It is exactly as if you had written a struct with a template call operator yourself:

```cpp
// This generic lambda:
auto add = [](auto x, auto y) { return x + y; };

// Is equivalent to this closure type:
struct __add_lambda {
    template <typename T, typename U>
    auto operator()(T x, U y) const { return x + y; }
};
```

Each `auto` is a **separate** template parameter, so `add(1, 2.5)` deduces `T=int, U=double`.

---

## Self-Assessment

### Q1: Write a generic lambda and explain the generated `operator()`

The interesting thing to watch here is not just that the lambda works for multiple types - it is that each `auto` parameter is truly independent. Two `auto` parameters mean two separate template parameters, so you can mix and match types freely in a single call.

```cpp
#include <iostream>
#include <string>
#include <type_traits>

int main() {
    // Generic lambda with TWO auto parameters
    auto add = [](auto x, auto y) { return x + y; };

    // The compiler generates:
    // struct __lambda {
    //     template <typename T, typename U>
    //     auto operator()(T x, U y) const { return x + y; }
    // };

    // Each call instantiates a DIFFERENT operator():
    std::cout << add(3, 4) << "\n";               // int + int = int
    std::cout << add(1.5, 2.5) << "\n";           // double + double = double
    std::cout << add(1, 2.5) << "\n";             // int + double = double (mixed!)
    std::cout << add(std::string{"hello "}, std::string{"world"}) << "\n";

    // Prove each auto is independent:
    auto show_types = [](auto x, auto y) {
        std::cout << "T1=" << typeid(x).name()
                  << " T2=" << typeid(y).name() << "\n";
    };
    show_types(42, 3.14);    // T1=int, T2=double
    show_types(3.14, 42);    // T1=double, T2=int (reversed!)

    // Single auto = all params share one type? NO!
    // Each auto is INDEPENDENT:
    auto mixed = [](auto a, auto b, auto c) {
        return std::to_string(a) + "-" + std::to_string(b);
    };
    std::cout << mixed(1, 2.5, 'x') << "\n";
}
// Expected output:
//   7
//   4
//   3.5
//   hello world
//   T1=i T2=d   (MSVC: T1=int T2=double)
//   T1=d T2=i
//   1-2.500000
```

Notice how `show_types(42, 3.14)` and `show_types(3.14, 42)` produce reversed type names - that is the independence at work. Each call site drives its own deduction completely from scratch.

---

### Q2: Show how a generic lambda differs from a function template in overload resolution

This is one of the subtler distinctions in C++. A function template and a generic lambda both achieve type-generic behavior, but they behave very differently when you try to use them as values or pass them around. The reason this trips people up is that a function template is not an object - it is a recipe for generating functions. A lambda, even a generic one, is a concrete object you can store, copy, and pass directly.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

// Function template:
template <typename T>
void process(T x) { std::cout << "template: " << x << "\n"; }

// Non-template overload:
void process(int x) { std::cout << "int overload: " << x << "\n"; }

int main() {
    // Function template: participates in overload resolution
    process(42);          // calls int overload (exact match preferred)
    process(3.14);        // calls template<double>
    process("hello");     // calls template<const char*>

    // Generic lambda: NOT a template in the classic sense
    auto generic = [](auto x) { std::cout << "lambda: " << x << "\n"; };
    generic(42);          // instantiates operator()(int)
    generic(3.14);        // instantiates operator()(double)

    // Key differences:
    // 1. Lambda has a SINGLE object - no overload set
    //    You can't add overloads to a lambda
    //    (but see overloaded pattern from earlier topic)

    // 2. Lambda can be stored in a variable and passed as argument:
    auto apply = [](auto fn, auto val) { return fn(val); };
    // Function templates CANNOT be passed as arguments without wrapping:
    // apply(process, 42);  // ERROR: which process?
    apply(generic, 42);     // OK: generic is a single object

    // 3. Lambda is a concrete object; function template is just a blueprint
    auto ptr = &decltype(generic)::operator()<int>;  // can take address
    // &process;  // ERROR without specifying template arg

    // 4. Generic lambda converts to function pointer (if stateless, single type):
    int (*fp)(int) = [](auto x) { return x * 2; };  // ERROR: which instantiation?
    // Must be non-generic for conversion:
    int (*fp2)(int) = [](int x) { return x * 2; };  // OK
    std::cout << "fp2(5): " << fp2(5) << "\n";
}
// Expected output:
//   int overload: 42
//   template: 3.14
//   template: hello
//   lambda: 42
//   lambda: 3.14
//   lambda: 42
//   fp2(5): 10
```

The bottom line is that a generic lambda gives you the type-flexibility of a template while still being a first-class object. That is exactly why you can pass `generic` directly to `apply`, but you cannot do the same with a bare function template name like `process`.

---

### Q3: Use a generic lambda as a comparator for `std::sort` with multiple element types

One place where generic lambdas shine in everyday code is comparators. You write one lambda, and it just works for `int`, `double`, `std::string`, or anything else that supports the relevant operator. This example shows a single `descending` comparator reused across multiple container types:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

int main() {
    // ONE generic comparator works for ANY type that supports <
    auto descending = [](const auto& a, const auto& b) { return a > b; };

    // Sort integers:
    std::vector<int> nums = {5, 2, 8, 1, 9, 3};
    std::sort(nums.begin(), nums.end(), descending);
    std::cout << "ints: ";
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";

    // Sort doubles:
    std::vector<double> vals = {3.14, 1.41, 2.72, 0.57};
    std::sort(vals.begin(), vals.end(), descending);
    std::cout << "doubles: ";
    for (double v : vals) std::cout << v << " ";
    std::cout << "\n";

    // Sort strings:
    std::vector<std::string> words = {"banana", "apple", "cherry", "date"};
    std::sort(words.begin(), words.end(), descending);
    std::cout << "strings: ";
    for (const auto& w : words) std::cout << w << " ";
    std::cout << "\n";

    // Generic comparator with projection:
    auto by_length_desc = [](const auto& a, const auto& b) {
        return a.size() > b.size();
    };
    std::sort(words.begin(), words.end(), by_length_desc);
    std::cout << "by length: ";
    for (const auto& w : words) std::cout << w << " ";
    std::cout << "\n";

    // Generic absolute-value comparator:
    auto by_abs = [](auto a, auto b) {
        return (a < 0 ? -a : a) < (b < 0 ? -b : b);
    };
    std::vector<int> mixed = {-5, 2, -8, 1, -3, 7};
    std::sort(mixed.begin(), mixed.end(), by_abs);
    std::cout << "by abs: ";
    for (int n : mixed) std::cout << n << " ";
    std::cout << "\n";
}
// Expected output:
//   ints: 9 8 5 3 2 1
//   doubles: 3.14 2.72 1.41 0.57
//   strings: date cherry banana apple
//   by length: cherry banana apple date
//   by abs: 1 2 -3 -5 7 -8
```

Notice that `const auto&` is the right choice when you want to avoid unnecessary copies - it becomes `template<class T> operator()(const T&, const T&) const`. The `by_abs` comparator uses plain `auto` (by value) since the elements are cheap to copy integers.

---

## Notes

- **Each `auto`** = independent template parameter. `[](auto a, auto b)` has a **two-parameter** template `operator()`.
- **`const auto&`** = `template<class T> operator()(const T&)` - avoids copies.
- **C++20 upgrade:** For constraints, use `[](std::integral auto x)` or template lambdas `[]<typename T>(T x) requires ...`.
- **Stateless generic lambdas** cannot convert to function pointers (because which instantiation?).
- **`decltype(lambda)`** is a unique unnamed type for each lambda expression, even identical-looking ones.
