# Use std::bind_front (C++20) for partial application

**Category:** Lambda and Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/bind_front>  

---

## Topic Overview

`std::bind_front` (C++20) binds the **first N arguments** of a callable and returns a new callable that accepts the remaining arguments. Think of it as a clean, modern replacement for `std::bind` in the common case where you just want to lock down the leading arguments. The original `std::bind` works, but it forces you to write out placeholders like `_1`, `_2`, and the intent gets buried in the syntax.

Here's a direct comparison:

```cpp
void print(const std::string& prefix, int x) { std::cout << prefix << x; }

// std::bind (C++11) - verbose, error-prone
auto f1 = std::bind(print, "Value: ", std::placeholders::_1);

// std::bind_front (C++20) - clean!
auto f2 = std::bind_front(print, "Value: ");

f1(42);  // "Value: 42"
f2(42);  // "Value: 42"
```

Both produce the same result, but `bind_front` makes the intent obvious at a glance: fix the prefix, pass the rest later.

### bind vs bind_front

The table below captures the key differences. The practical takeaway is that `bind_front` is strictly better for the "bind the first N arguments" use case, and you only need `std::bind` when you specifically need argument reordering.

| Feature | `std::bind` | `std::bind_front` |
| --- | --- | --- |
| Placeholders | Required (`_1, _2...`) | **Not needed** |
| Argument reorder | Yes | **No** |
| Perfect forwarding | No (copies by default) | **Yes** |
| Nested bind | Special rules | No special rules |
| Readability | Low | **High** |

---

## Self-Assessment

### Q1: Replace a `std::bind` call with `std::bind_front` and show the readability improvement

Notice how the `bind_front` versions are shorter and tell you immediately what's happening - there's no mental effort spent parsing placeholder indices.

```cpp
#include <iostream>
#include <functional>
#include <string>
#include <algorithm>
#include <vector>

struct Logger {
    void log(const std::string& level, const std::string& msg) {
        std::cout << "[" << level << "] " << msg << "\n";
    }
};

int multiply(int a, int b) { return a * b; }

int main() {
    Logger logger;

    // std::bind (C++11) - verbose and confusing
    using namespace std::placeholders;
    auto log_error_bind = std::bind(&Logger::log, &logger, "ERROR", _1);
    log_error_bind("disk full");

    auto times3_bind = std::bind(multiply, _1, 3);
    std::cout << "bind: " << times3_bind(7) << "\n";

    // std::bind_front (C++20) - clear and simple
    auto log_error = std::bind_front(&Logger::log, &logger, "ERROR");
    log_error("disk full");

    auto times3 = std::bind_front(multiply, 3);
    // Note: binds FIRST arg, so this is multiply(3, x)
    std::cout << "bind_front: " << times3(7) << "\n";

    // Practical use: sort with custom comparator
    auto greater_than = std::bind_front(std::greater<int>{});
    std::vector<int> nums = {5, 2, 8, 1, 9};
    std::sort(nums.begin(), nums.end(), greater_than);
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";
}
// Expected output:
//   [ERROR] disk full
//   bind: 21
//   [ERROR] disk full
//   bind_front: 21
//   9 8 5 2 1
```

---

### Q2: Show that `std::bind_front` does not support placeholder arguments and what the alternative is

This is the one thing `bind_front` deliberately does not do: reorder arguments. It binds from the front and that's it. When you need to fix a middle argument or swap argument order, a lambda is the right tool.

```cpp
#include <iostream>
#include <functional>

void report(int a, int b, int c) {
    std::cout << "a=" << a << " b=" << b << " c=" << c << "\n";
}

int main() {
    // std::bind can REORDER arguments:
    using namespace std::placeholders;
    auto reordered = std::bind(report, _3, _1, _2);  // swap args!
    reordered(10, 20, 30);  // calls report(30, 10, 20)

    // std::bind_front CANNOT reorder - only binds from the front:
    auto front_bound = std::bind_front(report, 100);
    front_bound(200, 300);  // calls report(100, 200, 300)

    // For middle/back binding or reordering, use a LAMBDA:
    auto bind_middle = [](int a, int c) {
        report(a, 999, c);  // b is fixed to 999
    };
    bind_middle(1, 3);  // calls report(1, 999, 3)

    // Lambda for reordering:
    auto swap_args = [](int x, int y, int z) {
        report(z, x, y);  // same as bind's _3, _1, _2
    };
    swap_args(10, 20, 30);  // calls report(30, 10, 20)

    // C++23: std::bind_back - binds from the BACK
    // auto back_bound = std::bind_back(report, 300);
    // back_bound(100, 200);  // calls report(100, 200, 300)
}
// Expected output:
//   a=30 b=10 c=20
//   a=100 b=200 c=300
//   a=1 b=999 c=3
//   a=30 b=10 c=20
```

---

### Q3: Implement partial application manually using a lambda and compare with `bind_front`

Both approaches work, but `bind_front` is noticeably more concise when all you want to do is supply the first argument of a member function. When you're doing anything more elaborate, a lambda wins because the intent is explicit.

```cpp
#include <iostream>
#include <functional>
#include <string>
#include <vector>
#include <algorithm>

struct Sensor {
    std::string name;
    double read_value(double offset) const {
        return 42.0 + offset;
    }
};

int main() {
    Sensor s{"Temperature"};

    // Manual lambda partial application
    auto read_lambda = [&s](double offset) {
        return s.read_value(offset);
    };
    std::cout << "Lambda: " << read_lambda(0.5) << "\n";

    // std::bind_front
    auto read_bind = std::bind_front(&Sensor::read_value, &s);
    std::cout << "bind_front: " << read_bind(0.5) << "\n";

    // Comparison:
    // Lambda:     captures &s, explicit body
    // bind_front: one line, perfect forwarding built-in

    // Practical: partial application in algorithms
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Lambda version:
    auto is_greater_lambda = [](int threshold, int x) { return x > threshold; };
    auto count_l = std::count_if(data.begin(), data.end(),
        [&](int x) { return is_greater_lambda(5, x); });

    // bind_front version - cleaner!
    auto is_greater = [](int threshold, int x) { return x > threshold; };
    auto count_b = std::count_if(data.begin(), data.end(),
        std::bind_front(is_greater, 5));

    std::cout << "Lambda count: " << count_l << "\n";
    std::cout << "bind_front count: " << count_b << "\n";
}
// Expected output:
//   Lambda: 42.5
//   bind_front: 42.5
//   Lambda count: 5
//   bind_front count: 5
```

---

## Notes

- **Perfect forwarding:** `bind_front` perfectly forwards remaining arguments. `std::bind` copies by default (you need `std::ref` to pass by reference).
- **`std::bind_back` (C++23):** Binds the **last** N arguments. Complements `bind_front` for the opposite end.
- **Member functions:** `std::bind_front(&Class::method, &obj)` is a clean one-liner for creating a callable from a member function.
- **Prefer lambdas** for anything more complex than binding the first few arguments - the intent stays clearer.
- **`std::bind` is not deprecated** but is rarely needed now that `bind_front` and lambdas are available.
