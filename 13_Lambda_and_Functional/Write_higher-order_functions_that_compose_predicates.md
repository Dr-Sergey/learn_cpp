# Write higher-order functions that compose predicates

**Category:** Lambda and Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional>  

---

## Topic Overview

A **higher-order function** is a function that either takes other functions as arguments or returns a function as its result. **Predicate composition** is the practical payoff: instead of writing a new lambda every time you need a slightly different filtering rule, you build simple predicates once (`is_even`, `is_positive`) and then combine them into complex ones (`is_even AND is_positive`) using combinators.

The idea is that small, named predicates are easier to read, test, and reuse individually. Composition lets you snap them together without duplicating logic:

```cpp
auto is_even     = [](int x) { return x % 2 == 0; };
auto is_positive = [](int x) { return x > 0; };

// Composed predicate:
auto is_even_and_positive = logical_and(is_even, is_positive);

is_even_and_positive(4);   // true
is_even_and_positive(-2);  // false
is_even_and_positive(3);   // false
```

---

## Self-Assessment

### Q1: Implement `logical_and(p1, p2)` that returns a lambda combining two predicates

The implementation is straightforward: capture both predicates inside a returned lambda and apply them with `&&`. The real value shows up when you start chaining compositions - `logical_and(even_and_pos, is_small)` works because the composed predicate is itself just another callable.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

// Higher-order function: returns a lambda that ANDs two predicates
template <typename P1, typename P2>
auto logical_and(P1 p1, P2 p2) {
    return [p1, p2](const auto& x) {
        return p1(x) && p2(x);
    };
}

// Also: logical_or and logical_not
template <typename P1, typename P2>
auto logical_or(P1 p1, P2 p2) {
    return [p1, p2](const auto& x) {
        return p1(x) || p2(x);
    };
}

template <typename P>
auto logical_not(P p) {
    return [p](const auto& x) {
        return !p(x);
    };
}

int main() {
    auto is_even     = [](int x) { return x % 2 == 0; };
    auto is_positive = [](int x) { return x > 0; };
    auto is_small    = [](int x) { return x < 10; };

    // Compose: even AND positive
    auto even_and_pos = logical_and(is_even, is_positive);

    // Compose: (even AND positive) AND small
    auto small_even_pos = logical_and(even_and_pos, is_small);

    // Compose: even OR positive
    auto even_or_pos = logical_or(is_even, is_positive);

    std::vector<int> nums = {-6, -3, -2, 0, 1, 2, 3, 4, 8, 12, 15};

    std::cout << "even AND positive: ";
    for (int x : nums)
        if (even_and_pos(x)) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "small even positive: ";
    for (int x : nums)
        if (small_even_pos(x)) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "even OR positive: ";
    for (int x : nums)
        if (even_or_pos(x)) std::cout << x << " ";
    std::cout << "\n";

    std::cout << "NOT even: ";
    for (int x : nums)
        if (logical_not(is_even)(x)) std::cout << x << " ";
    std::cout << "\n";
}
// Expected output:
//   even AND positive: 2 4 8 12
//   small even positive: 2 4 8
//   even OR positive: -6 -2 0 1 2 3 4 8 12 15
//   NOT even: -3 1 3 15
```

Notice that `small_even_pos` is built by composing an already-composed predicate - `even_and_pos` is just another callable, so it works as an argument to `logical_and` without any special handling.

---

### Q2: Build a pipeline of transformations using function composition

Predicate composition is the filtering side of the story. Function composition is the transformation side. Here we build `compose(f, g)` which produces a new function `h` where `h(x) = f(g(x))`, and `pipe(f, g)` which runs left-to-right instead. The important thing to watch in the variadic `pipe_all` is how the recursion builds the chain entirely at compile time - by the time you call `pipeline(5)`, the compiler has already unrolled the whole chain.

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <cctype>

// compose(f, g) returns a function h where h(x) = f(g(x))
template <typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto&&... args) {
        return f(g(std::forward<decltype(args)>(args)...));
    };
}

// pipe(f, g) returns h where h(x) = g(f(x))  (left-to-right)
template <typename F, typename G>
auto pipe(F f, G g) {
    return [f, g](auto&&... args) {
        return g(f(std::forward<decltype(args)>(args)...));
    };
}

// Variadic pipe: pipe(f1, f2, f3, ...) = f3(f2(f1(x)))
template <typename F>
auto pipe_all(F f) { return f; }

template <typename F, typename... Fs>
auto pipe_all(F f, Fs... rest) {
    return pipe(f, pipe_all(rest...));
}

int main() {
    auto double_it = [](int x) { return x * 2; };
    auto add_one   = [](int x) { return x + 1; };
    auto negate    = [](int x) { return -x; };
    auto to_string = [](int x) { return "result: " + std::to_string(x); };

    // compose: right-to-left (mathematical style)
    auto f = compose(double_it, add_one);  // double(add_one(x))
    std::cout << "compose(double, add1)(5): " << f(5) << "\n";  // (5+1)*2 = 12

    // pipe: left-to-right (pipeline style)
    auto g = pipe(add_one, double_it);  // add_one then double
    std::cout << "pipe(add1, double)(5): " << g(5) << "\n";  // (5+1)*2 = 12

    // Variadic pipeline:
    auto pipeline = pipe_all(add_one, double_it, negate, to_string);
    std::cout << pipeline(5) << "\n";  // -((5+1)*2) = -12 -> "result: -12"
}
// Expected output:
//   compose(double, add1)(5): 12
//   pipe(add1, double)(5): 12
//   result: -12
```

`compose` and `pipe` produce the same result here because both apply `add_one` then `double_it` - the difference is just the order of arguments. That reversed argument order is what makes `compose` feel like the mathematical `f ∘ g` notation (right-to-left), while `pipe` feels like a shell pipeline (left-to-right).

---

### Q3: Show how `views::filter` with a composed predicate avoids double iteration

This example gets into a practical performance question. When you chain two separate `filter` views, the range library is lazy so you only traverse the container once - but each element still gets evaluated against both predicates separately. With a single composed predicate you get the same result with one predicate check per element, plus short-circuit evaluation: if `is_even` fails, `is_gt5` never runs at all.

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <algorithm>

namespace rv = std::ranges::views;

template <typename P1, typename P2>
auto both(P1 p1, P2 p2) {
    return [p1, p2](const auto& x) { return p1(x) && p2(x); };
}

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12};

    auto is_even = [](int x) { return x % 2 == 0; };
    auto is_gt5  = [](int x) { return x > 5; };

    // BAD: Two separate filters = TWO passes of predicate evaluation
    // (Actually ranges are lazy so it's still one traversal,
    //  but TWO predicate checks per element)
    int count1 = 0, count2 = 0;
    auto counted_even = [&](int x) { ++count1; return x % 2 == 0; };
    auto counted_gt5  = [&](int x) { ++count2; return x > 5; };

    std::cout << "Two filters:\n";
    for (int x : data | rv::filter(counted_even) | rv::filter(counted_gt5)) {
        std::cout << x << " ";
    }
    std::cout << "\n  predicate1 calls: " << count1
              << ", predicate2 calls: " << count2 << "\n";

    // GOOD: One composed predicate = ONE pass
    int count3 = 0;
    auto composed = [&](int x) {
        ++count3;
        return is_even(x) && is_gt5(x);
    };

    std::cout << "\nOne composed filter:\n";
    for (int x : data | rv::filter(composed)) {
        std::cout << x << " ";
    }
    std::cout << "\n  composed calls: " << count3 << "\n";

    // Using our both() combinator:
    std::cout << "\nWith both() combinator: ";
    for (int x : data | rv::filter(both(is_even, is_gt5))) {
        std::cout << x << " ";
    }
    std::cout << "\n";
}
// Expected output:
//   Two filters:
//   6 8 10 12
//     predicate1 calls: 12, predicate2 calls: 6
//
//   One composed filter:
//   6 8 10 12
//     composed calls: 12
//
//   With both() combinator: 6 8 10 12
```

**Why composed is better:**

- Two chained `filter` views: first filter runs on ALL 12 elements, second filter runs on the 6 that pass (18 total predicate calls)
- One composed filter: runs on ALL 12 elements once (12 total, with short-circuit `&&`)

---

## Notes

- **Short-circuit evaluation:** `logical_and` uses `&&`, so the second predicate isn't called if the first returns false.
- **Variadic composition:** `pipe_all(f1, f2, ..., fn)` builds a pipeline at compile time - zero overhead.
- **`std::not_fn`** is the standard library's built-in predicate negator (see separate topic).
- **C++ has no `|` operator for function composition** (unlike Haskell's `.` or F#'s `|>`), but ranges `|` provides similar pipeline syntax for range operations.
