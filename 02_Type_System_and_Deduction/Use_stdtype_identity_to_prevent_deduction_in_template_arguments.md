# Use `std::type_identity` to Prevent Deduction in Template Arguments

**Category:** Type System and Deduction  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/types/type_identity>  

---

## Topic Overview

### What Is `std::type_identity`

`std::type_identity` (C++20) is a simple identity metafunction that wraps a type without changing it:

```cpp
template<typename T>
struct type_identity { using type = T; };

template<typename T>
using type_identity_t = typename type_identity<T>::type;
```

It looks trivial, and it is - on purpose. Its entire value comes from one property: **it creates a non-deduced context** for template arguments. When a parameter uses `type_identity_t<T>`, the compiler will not try to deduce `T` from that parameter. `T` must be figured out from somewhere else.

### Why Prevent Deduction

Here's the problem this solves. Suppose you have two arguments of the same template type, but the caller passes mismatched types:

```cpp
// Problem: conflicting deduction
template<typename T>
T max(T a, T b) { return a > b ? a : b; }

max(1, 2.0);  // ERROR: T deduced as int from arg1, double from arg2
```

The compiler sees `int` from the first argument and `double` from the second. It can't pick one, so it gives up. The fix is to tell the compiler to deduce `T` from only one parameter and let the other convert:

```cpp
template<typename T>
T max(T a, std::type_identity_t<T> b) { return a > b ? a : b; }

max(1, 2.0);  // OK: T deduced as int from arg1; 2.0 converts to int
```

The reason this works is that `type_identity_t<T>` expands to `typename type_identity<T>::type`, which is a nested type accessed through a dependent name. The C++ standard says that nested types like this are **non-deduced contexts** - the compiler won't touch them during deduction. So `T` is pinned to `int` by the first argument, and `2.0` silently converts.

### How Non-Deduced Contexts Work

The C++ standard specifies that nested types like `typename X<T>::type` are **non-deduced contexts**. Since `type_identity_t<T>` is `typename type_identity<T>::type`, the compiler won't try to deduce `T` from that parameter. The reason this trips people up is that the wrapping looks like a no-op - `type_identity_t<T>` is literally just `T` after substitution - but the deduction machinery can't see through the dependent name, so it skips that parameter entirely during deduction.

### Common Use Cases

| Pattern | Purpose |
| --- | --- |
| `f(T x, type_identity_t<T> y)` | Deduce T from x, y just converts |
| `f(type_identity_t<T> x)` | Force explicit template specification |
| Default arguments with matching type | `f(T x, T y = T{})` fails - use `type_identity_t<T> y` |

---

## Self-Assessment

### Q1: Show that `void f(T x, type_identity_t<T> y)` forces T to be deduced from x only

Watch what happens to the second argument in each call - it converts to whatever `T` was pinned to by the first argument.

```cpp
#include <iostream>
#include <type_traits>

// T is deduced from first argument only
template<typename T>
void print_add(T x, std::type_identity_t<T> y) {
    std::cout << "  x=" << x << " y=" << y << " sum=" << (x + y)
              << " [T=" << typeid(T).name() << "]\n";
}

// Without type_identity: both params participate in deduction
template<typename T>
void print_add_both(T x, T y) {
    std::cout << "  x=" << x << " y=" << y << " sum=" << (x + y) << "\n";
}

// Practical: comparison function where the second argument matches the first
template<typename T>
bool in_range(T value, std::type_identity_t<T> low, std::type_identity_t<T> high) {
    return value >= low && value <= high;
}

int main() {
    std::cout << std::boolalpha;

    // === type_identity_t prevents deduction from y ===
    std::cout << "print_add (type_identity on y):\n";

    print_add(1, 2);       // T=int from x; y=2 (int->int, OK)
    print_add(1, 2.5);     // T=int from x; y=2.5 converts to int -> 2
    print_add(3.14, 2);    // T=double from x; y=2 converts to double -> 2.0

    // === Without type_identity: deduction conflict ===
    // print_add_both(1, 2.5);  // ERROR: T deduced as int AND double!

    // Works only when types match:
    std::cout << "\nprint_add_both (both params deduced):\n";
    print_add_both(1, 2);        // ok: both int
    print_add_both(1.5, 2.5);    // ok: both double

    // === Range check: T deduced from value only ===
    std::cout << "\nin_range (type_identity on low,high):\n";
    std::cout << "  42 in [0, 100]: " << in_range(42, 0, 100) << "\n";
    std::cout << "  3.14 in [0, 10]: " << in_range(3.14, 0, 10) << "\n";  // 0 and 10 convert to double
    // Without type_identity, in_range(3.14, 0, 10) would fail:
    // T deduced as double from 3.14, but int from 0 and 10

    return 0;
}
```

Notice `print_add(1, 2.5)` - `T` is locked to `int` by the first argument, so `2.5` is truncated to `2`. That truncation is exactly what you asked for when you said "deduce from the first argument only." If that's surprising, it's a good reminder to only use this pattern when conversion in the non-deduced parameter is genuinely acceptable.

**Output:**

```text
print_add (type_identity on y):
  x=1 y=2 sum=3 [T=int]
  x=1 y=2 sum=3 [T=int]
  x=3.14 y=2 sum=5.14 [T=double]

print_add_both (both params deduced):
  x=1 y=2 sum=3
  x=1.5 y=2.5 sum=4

in_range (type_identity on low,high):
  42 in [0, 100]: true
  3.14 in [0, 10]: true
```

### Q2: Explain why this is useful when one parameter should match a deduced type exactly

Here are three real scenarios where the non-deduced context earns its keep:

```cpp
#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// Use case 1: Default arguments that match the deduced type
template<typename T>
void fill_with(std::vector<T>& vec, std::type_identity_t<T> value = T{}) {
    for (auto& elem : vec) elem = value;
}
// Without type_identity_t, `T value = T{}` would need to be deducible
// from the default argument, which is impossible.

// Use case 2: Preventing narrowing via the "wrong" deduction
template<typename T>
void safe_assign(T& target, std::type_identity_t<T> source) {
    target = source;  // source is implicitly converted to T
}

// Use case 3: Callback type forced to match
template<typename T>
void transform_all(std::vector<T>& vec,
                    std::type_identity_t<T>(*func)(T)) {
    for (auto& elem : vec) elem = func(elem);
}

int double_it(int x) { return x * 2; }

int main() {
    // Use case 1: Default argument
    std::vector<int> v(5);
    fill_with(v);       // T=int deduced from vec; value defaults to 0
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    fill_with(v, 42);   // T=int from vec; value=42
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // Use case 2: safe_assign
    int x = 0;
    safe_assign(x, 3.14);  // T=int from target; 3.14 converts to int
    std::cout << "x = " << x << "\n";

    double d = 0.0;
    safe_assign(d, 42);    // T=double from target; 42 converts to double
    std::cout << "d = " << d << "\n";

    // Use case 3: Callback type matching
    std::vector<int> nums = {1, 2, 3, 4, 5};
    transform_all(nums, double_it);
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";

    return 0;
}
```

Use case 1 is particularly worth understanding: without `type_identity_t`, writing `T value = T{}` as a default argument puts `T` in a deduced context from the default itself, which doesn't work. Wrapping it in `type_identity_t` removes that parameter from deduction, so `T` gets pinned by the vector argument and the default just uses it.

**Output:**

```text
0 0 0 0 0
42 42 42 42 42
x = 3
d = 42
2 4 6 8 10
```

### Q3: Replace a non-deduced context workaround using `type_identity`

Before C++20, developers had to roll their own `non_deduced` wrapper. The new way is cleaner:

```cpp
#include <iostream>
#include <type_traits>
#include <functional>

// === OLD workaround (pre-C++20): manual non-deduced context wrapper ===
namespace old_way {
    template<typename T>
    struct non_deduced { using type = T; };

    template<typename T>
    using non_deduced_t = typename non_deduced<T>::type;

    template<typename T>
    T clamp(T val, non_deduced_t<T> lo, non_deduced_t<T> hi) {
        return val < lo ? lo : (val > hi ? hi : val);
    }
}

// === NEW way (C++20): just use std::type_identity_t ===
namespace new_way {
    template<typename T>
    T clamp(T val, std::type_identity_t<T> lo, std::type_identity_t<T> hi) {
        return val < lo ? lo : (val > hi ? hi : val);
    }
}

// === Another old workaround: enable_if + decltype ===
namespace old_callback {
    // Ugly SFINAE to force callback return type to match
    template<typename T, typename F,
             typename = std::enable_if_t<std::is_same_v<decltype(std::declval<F>()(std::declval<T>())), T>>>
    void apply_each(std::vector<T>& v, F func) {
        for (auto& e : v) e = func(e);
    }
}

namespace new_callback {
    // Clean: callback signature forced via type_identity
    template<typename T>
    void apply_each(std::vector<T>& v, std::type_identity_t<std::function<T(T)>> func) {
        for (auto& e : v) e = func(e);
    }
}

int main() {
    // Clamp: old vs new — same behavior
    std::cout << "old clamp(3.14, 0, 10): " << old_way::clamp(3.14, 0, 10) << "\n";
    std::cout << "new clamp(3.14, 0, 10): " << new_way::clamp(3.14, 0, 10) << "\n";

    std::cout << "old clamp(15, 0, 10): " << old_way::clamp(15, 0, 10) << "\n";
    std::cout << "new clamp(15, 0, 10): " << new_way::clamp(15, 0, 10) << "\n";

    // Callback: new way is much cleaner
    std::vector<int> nums = {1, 2, 3, 4, 5};
    new_callback::apply_each(nums, [](int x) { return x * x; });
    std::cout << "Squared: ";
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";

    return 0;
}
```

The behavior is identical in both cases. The gain is entirely on the readability and maintenance side - `std::type_identity_t` communicates intent clearly and removes the need to define your own helper.

**Output:**

```text
old clamp(3.14, 0, 10): 3.14
new clamp(3.14, 0, 10): 3.14
old clamp(15, 0, 10): 10
new clamp(15, 0, 10): 10
Squared: 1 4 9 16 25
```

---

## Notes

- `std::type_identity` was introduced in C++20. For earlier standards, write your own (it's two lines).
- This technique is used in the standard library itself - e.g., `std::clamp` uses non-deduced contexts for the comparison parameters.
- `type_identity_t<T>` is also useful in concepts/constraints to create a "barrier" that prevents concepts from accidentally deducing through an alias.
