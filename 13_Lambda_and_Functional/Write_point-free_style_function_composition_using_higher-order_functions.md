# Write point-free style function composition using higher-order functions

**Category:** Lambda and Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional>  

---

## Topic Overview

**Point-free style** (also called tacit programming) is a way of defining functions without ever explicitly naming their arguments. Instead of writing `[](int x) { return f(g(x)); }`, you write `compose(f, g)` and let the composition machinery handle passing the argument along. The name comes from mathematics, where a "point" means an argument value - so "point-free" means the definition never mentions a specific point.

This style is natural and built-in in languages like Haskell and F#. In C++ you have to build the combinators yourself, but the result is equally expressive and the compiler inlines everything to zero overhead.

```cpp
// Pointed style (mentions arguments):
auto process = [](int x) { return to_string(negate(double_it(x))); };

// Point-free style (no argument mentioned):
auto process = compose(to_string, compose(negate, double_it));
// or with pipeline:
auto process = pipeline(double_it, negate, to_string);
```

---

## Self-Assessment

### Q1: Implement `compose(f, g)` that returns `[](auto x) { return f(g(x)); }`

The `compose` function captures both callables and returns a new lambda that applies them in right-to-left order, matching the mathematical convention for function composition. The variadic `compose_all` extends this to any number of functions by recursing on the tail - it builds the composition chain at compile time, so there is no runtime overhead from the recursion.

```cpp
#include <iostream>
#include <string>
#include <cmath>

// compose(f, g)(x) = f(g(x))  - right-to-left (mathematical convention)
template <typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto&&... args) -> decltype(auto) {
        return f(g(std::forward<decltype(args)>(args)...));
    };
}

// Variadic compose: compose_all(f, g, h)(x) = f(g(h(x)))
template <typename F>
auto compose_all(F f) { return f; }

template <typename F, typename... Fs>
auto compose_all(F f, Fs... rest) {
    return compose(f, compose_all(rest...));
}

int main() {
    auto square    = [](double x) { return x * x; };
    auto add_one   = [](double x) { return x + 1; };
    auto to_string = [](double x) { return std::to_string(x); };
    auto prefix    = [](std::string s) { return "Result: " + s; };

    // Two-function composition:
    auto square_then_add = compose(add_one, square);  // add_one(square(x))
    std::cout << square_then_add(5) << "\n";          // 5^2 + 1 = 26

    // Multi-function composition (right-to-left):
    auto pipeline = compose_all(prefix, to_string, add_one, square);
    // pipeline(3) = prefix(to_string(add_one(square(3))))
    //             = prefix(to_string(add_one(9)))
    //             = prefix(to_string(10))
    //             = prefix("10.000000")
    //             = "Result: 10.000000"
    std::cout << pipeline(3) << "\n";

    // Point-free: no argument name appears in the definition!
    auto abs_squared = compose(square, [](double x) { return std::abs(x); });
    std::cout << abs_squared(-7) << "\n";  // 49
}
// Expected output:
//   26
//   Result: 10.000000
//   49
```

Notice that in the definition of `abs_squared`, no variable name like `x` appears on the left-hand side at all. You are describing the shape of the computation - "square after absolute value" - without saying what you are going to apply it to. That is the point-free style in practice.

---

### Q2: Use compose to build a string transformation pipeline without intermediate variables

Here the point-free style pays off in readability. A `format_tag` function that trims, uppercases, truncates, and wraps in brackets would normally be a multi-line function with intermediate variables. Written as a pipeline, the definition of `format_tag` reads almost like a sentence: trim leading whitespace, then uppercase, then truncate to 10, then add brackets.

```cpp
#include <iostream>
#include <string>
#include <algorithm>
#include <cctype>
#include <utility>

template <typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto&&... args) -> decltype(auto) {
        return f(g(std::forward<decltype(args)>(args)...));
    };
}

template <typename F>
auto pipeline(F f) { return f; }

template <typename F, typename... Fs>
auto pipeline(F f, Fs... rest) {
    return [f, ...rest_fns = rest](auto&&... args) -> decltype(auto) {
        return pipeline(rest...)(f(std::forward<decltype(args)>(args)...));
    };
}

int main() {
    // Point-free string transformations:
    auto trim_leading = [](std::string s) {
        s.erase(s.begin(), std::find_if(s.begin(), s.end(),
            [](char c) { return !std::isspace(c); }));
        return s;
    };

    auto to_upper = [](std::string s) {
        std::transform(s.begin(), s.end(), s.begin(), ::toupper);
        return s;
    };

    auto add_brackets = [](std::string s) {
        return "[" + s + "]";
    };

    auto truncate_10 = [](std::string s) {
        if (s.size() > 10) s.resize(10);
        return s;
    };

    // Build pipeline without mentioning arguments:
    auto format_tag = pipeline(trim_leading, to_upper, truncate_10, add_brackets);

    // Use:
    std::cout << format_tag("  hello world") << "\n";
    std::cout << format_tag("  short") << "\n";
    std::cout << format_tag("   this is a very long tag name") << "\n";
}
// Expected output:
//   [HELLO WORL]
//   [SHORT]
//   [THIS IS A ]
```

Each step is a small, independently testable function. The `format_tag` pipeline is simply the composition of those steps. If requirements change - say you need to strip trailing whitespace too - you add one more step to the pipeline without touching the others.

---

### Q3: Explain why C++ lacks built-in composition and what `ranges::views` fills

This question gets at a genuine limitation of C++ compared to functional languages, and it is worth understanding why the limitation exists rather than just accepting it. The short answer is that C++ functions are not first-class values in the same way objects are. The longer answer is in the comments.

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <algorithm>
#include <string>

namespace rv = std::ranges::views;

int main() {
    // C++ has NO built-in function composition operator.
    // Haskell:  f . g           -- composition
    // F#:       x |> f |> g     -- pipeline
    // C++:      ???             -- must build manually

    // WHY C++ lacks it:
    // 1. C++ functions aren't first-class values
    //    - Function templates can't be passed as arguments
    //    - Overload sets aren't objects
    //    - ADL complicates lookup
    // 2. No unified callable concept until C++20's std::invocable
    // 3. Type deduction challenges with overloaded/templated functions

    // WHAT ranges::views provides: PIPELINE composition via operator|
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // This IS point-free style for range operations!
    auto pipeline = rv::filter([](int x) { return x % 2 == 0; })
                  | rv::transform([](int x) { return x * x; })
                  | rv::take(3);

    std::cout << "Pipeline result: ";
    for (int x : data | pipeline) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    // ranges::views::transform is essentially "map" composition:
    auto names = std::vector<std::string>{"Alice", "Bob", "Charlie"};
    auto lengths = rv::transform(&std::string::size);  // point-free!

    std::cout << "Lengths: ";
    for (auto len : names | lengths) {
        std::cout << len << " ";
    }
    std::cout << "\n";

    // Summary table:
    std::cout << "\n";
    std::cout << "Language    Composition   Pipeline\n";
    std::cout << "---------- ------------- -----------\n";
    std::cout << "Haskell     f . g         x & f & g\n";
    std::cout << "F#          f >> g        x |> f |> g\n";
    std::cout << "C++ ranges  N/A           r | f | g\n";
    std::cout << "C++ manual  compose(f,g)  pipe(f,g)(x)\n";
}
// Expected output:
//   Pipeline result: 4 16 36
//   Lengths: 5 3 7
//
//   Language    Composition   Pipeline
//   ---------- ------------- -----------
//   Haskell     f . g         x & f & g
//   F#          f >> g        x |> f |> g
//   C++ ranges  N/A           r | f | g
//   C++ manual  compose(f,g)  pipe(f,g)(x)
```

The `rv::transform(&std::string::size)` line is a genuinely point-free expression: a member function pointer used as a projection, with no lambda argument in sight. The `|` operator for ranges is the closest thing C++ has to a built-in pipeline operator, and it only works because range adaptors were specifically designed as objects that overload `operator|`.

---

## Notes

- **`operator|` for ranges** is the closest C++ gets to built-in pipeline composition.
- **Boost.Hof** and **range-v3** provide `compose`, `partial`, and other combinators.
- **C++23 `std::bind_back`** helps with partial application, moving toward more point-free style.
- **Limitation:** C++ compose requires concrete callable types - you can't compose overload sets or function templates without wrapping them in lambdas first.
- **Performance:** Composition via lambdas is zero-overhead - the compiler inlines everything.
