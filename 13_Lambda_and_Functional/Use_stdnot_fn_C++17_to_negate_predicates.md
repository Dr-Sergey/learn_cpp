# Use std::not_fn (C++17) to negate predicates

**Category:** Lambda and Functional  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/not_fn>  

---

## Topic Overview

`std::not_fn` creates a **negation wrapper** around any callable, returning a new callable that inverts the boolean result. It's the modern replacement for the old `std::not1`/`std::not2` helpers (deprecated in C++17, removed in C++20), and it eliminates the small but annoying boilerplate of writing a wrapper lambda just to add a `!`.

The basic idea is simple:

```cpp
auto is_even = [](int x) { return x % 2 == 0; };

// Without not_fn:
auto is_odd_verbose = [&](int x) { return !is_even(x); };

// With not_fn:
auto is_odd = std::not_fn(is_even);  // clean!

is_odd(3);  // true
is_odd(4);  // false
```

The wrapper lambda approach works fine, but it requires you to capture the original predicate and manually thread the arguments through. `std::not_fn` does all of that for you and handles cases you might not expect, like predicates that take multiple arguments or have both const and non-const overloads.

---

## Self-Assessment

### Q1: Replace a negation lambda with `std::not_fn(pred)`

Here you can see `not_fn` used with several different kinds of callables - a lambda, a member function pointer, and a function object. The key thing to notice is that you pass the predicate once and get a ready-to-use negated version back.

```cpp
#include <iostream>
#include <functional>
#include <algorithm>
#include <vector>
#include <string>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    auto is_even = [](int x) { return x % 2 == 0; };

    // BEFORE: manual negation lambda
    auto count_odd_v1 = std::count_if(nums.begin(), nums.end(),
        [&](int x) { return !is_even(x); });

    // AFTER: std::not_fn
    auto count_odd_v2 = std::count_if(nums.begin(), nums.end(),
        std::not_fn(is_even));

    std::cout << "count (lambda): " << count_odd_v1 << "\n";
    std::cout << "count (not_fn): " << count_odd_v2 << "\n";

    // Works with any predicate:
    auto is_empty = std::not_fn(&std::string::empty);
    std::vector<std::string> words = {"hello", "", "world", "", "test"};
    auto non_empty = std::count_if(words.begin(), words.end(), is_empty);
    std::cout << "non-empty strings: " << non_empty << "\n";

    // Works with function objects:
    auto not_greater = std::not_fn(std::greater<int>{});
    std::cout << "not_greater(5, 3): " << std::boolalpha << not_greater(5, 3) << "\n";
    std::cout << "not_greater(3, 5): " << not_greater(3, 5) << "\n";
}
// Expected output:
//   count (lambda): 5
//   count (not_fn): 5
//   non-empty strings: 3
//   not_greater(5, 3): false
//   not_greater(3, 5): true
```

---

### Q2: Show how `not_fn` composes with `std::copy_if` to create a "copy unless" operation

`std::copy_if` copies elements where the predicate returns true. Wrapping your predicate in `not_fn` lets you express "copy everything that does NOT match this condition" without needing a separate `remove_if` or a negating lambda. It reads like natural English once you get used to it.

```cpp
#include <iostream>
#include <functional>
#include <algorithm>
#include <vector>
#include <iterator>
#include <cctype>

int main() {
    // "Copy unless" pattern: copy elements that DON'T match
    std::vector<int> data = {1, -2, 3, -4, 5, -6, 7, -8};

    auto is_negative = [](int x) { return x < 0; };

    // copy_if copies elements where predicate is TRUE
    // not_fn(is_negative) = "copy unless negative" = copy non-negatives
    std::vector<int> positives;
    std::copy_if(data.begin(), data.end(),
                 std::back_inserter(positives),
                 std::not_fn(is_negative));

    std::cout << "positives: ";
    for (int x : positives) std::cout << x << " ";
    std::cout << "\n";

    // Partition with not_fn
    std::vector<int> nums = {10, 3, 7, 1, 8, 5, 2, 9};
    auto is_big = [](int x) { return x > 5; };

    // remove_if + not_fn = "keep unless"
    auto new_end = std::remove_if(nums.begin(), nums.end(),
                                  std::not_fn(is_big));
    nums.erase(new_end, nums.end());

    std::cout << "big numbers: ";
    for (int x : nums) std::cout << x << " ";
    std::cout << "\n";

    // String filtering with not_fn
    std::string input = "Hello, World! 123";
    std::string letters;
    std::copy_if(input.begin(), input.end(),
                 std::back_inserter(letters),
                 std::not_fn([](char c) { return std::isspace(c) || std::ispunct(c) || std::isdigit(c); }));
    std::cout << "letters only: " << letters << "\n";
}
// Expected output:
//   positives: 1 3 5 7
//   big numbers: 10 7 8 9
//   letters only: HelloWorld
```

---

### Q3: Explain that `not_fn` returns a perfect-forwarding wrapper, not just a bool negator

This is the part that trips people up. `std::not_fn` doesn't just slap a `!` on the return value - it builds a proper forwarding wrapper that passes all arguments through unchanged, preserves const-correctness, and handles any arity. The example below uses a type with both a `const` unary overload and a non-const binary overload to show that the wrapper respects both.

```cpp
#include <iostream>
#include <functional>
#include <string>

// not_fn wraps the callable and forwards ALL arguments perfectly.
// It's NOT just "return !pred(x)" - it handles:
//   - Multiple arguments
//   - Rvalue/lvalue forwarding
//   - const/non-const overloads
//   - Member function pointers

struct Validator {
    // const overload
    bool operator()(int x) const {
        std::cout << "  const overload: " << x << "\n";
        return x > 0;
    }

    // non-const overload
    bool operator()(int x, int y) {
        std::cout << "  non-const overload: " << x << ", " << y << "\n";
        return x > y;
    }
};

int main() {
    Validator v;

    // not_fn preserves forwarding for ALL overloads:
    auto not_v = std::not_fn(v);

    std::cout << "Single arg (const):\n";
    std::cout << "  result: " << std::boolalpha << std::as_const(not_v)(5) << "\n";

    std::cout << "Two args (non-const):\n";
    std::cout << "  result: " << not_v(3, 7) << "\n";

    // Multi-arg predicate negation:
    auto in_range = [](int val, int lo, int hi) { return val >= lo && val <= hi; };
    auto out_of_range = std::not_fn(in_range);

    std::cout << "in_range(5, 1, 10): " << in_range(5, 1, 10) << "\n";
    std::cout << "out_of_range(5, 1, 10): " << out_of_range(5, 1, 10) << "\n";
    std::cout << "out_of_range(15, 1, 10): " << out_of_range(15, 1, 10) << "\n";
}
// Expected output:
//   Single arg (const):
//     const overload: 5
//     result: false
//   Two args (non-const):
//     non-const overload: 3, 7
//     result: true
//   in_range(5, 1, 10): true
//   out_of_range(5, 1, 10): false
//   out_of_range(15, 1, 10): true
```

**Key properties of `std::not_fn`:**

1. **Perfect forwarding** - all arguments forwarded as-is to the wrapped callable
2. **Multi-argument** - works with any arity, not just unary predicates
3. **Overload-preserving** - const/non-const and different-arity overloads all work
4. **Stores a copy** - the callable is decay-copied into the wrapper (use `std::ref` if you need to avoid copying)

---

## Notes

- **Replaces `std::not1`/`std::not2`** which required `std::unary_function`/`std::binary_function` base classes (removed in C++20).
- **Cannot chain:** `std::not_fn(std::not_fn(pred))` compiles but is pointless - double negation gives you back the original.
- **With ranges (C++20):** You can use `not_fn` in range pipelines: `v | views::filter(std::not_fn(pred))`.
- **Alternative:** A simple `!` lambda is fine for one-off use: `[&](auto x) { return !pred(x); }`. Use `not_fn` when passing predicates as arguments to generic code where you want a clean, self-contained callable.
