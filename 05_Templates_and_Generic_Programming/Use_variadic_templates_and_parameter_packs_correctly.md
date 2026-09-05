# Use Variadic Templates and Parameter Packs Correctly

**Category:** Templates and Generic Programming  
**Standard:** C++11, C++17 (variadic templates)  
**Reference:** <https://en.cppreference.com/w/cpp/language/variadic_template>  

---

## Topic Overview

### What Are Variadic Templates

Variadic templates accept a variable number of template arguments via **parameter packs**. They are the foundation for `std::tuple`, `std::variant`, `std::make_shared`, and many other standard library facilities:

```cpp
template <typename... Args>   // Args is a template parameter pack
void func(Args... args) {     // args is a function parameter pack
    // sizeof...(Args) = number of types
    // sizeof...(args) = number of values (same number)
}
```

### Pack Expansion

A parameter pack is expanded by placing `...` after a pattern. The exact location of the `...` determines what gets repeated. This is the point that trips most people up, so it is worth reading carefully:

```cpp
template <typename... Args>
void f(Args... args) {
    g(args...);                // expands to: g(a1, a2, a3, ...)
    g(transform(args)...);     // expands to: g(transform(a1), transform(a2), ...)
    g(transform(args...));     // expands to: g(transform(a1, a2, a3, ...))
}
```

The first form passes all args as separate arguments. The second applies `transform` to each one individually. The third passes all args into a single call to `transform`. These are completely different - mix them up and you get a confusing error.

### Expansion Contexts

| Context | Pattern | Expands To |
| --- | --- | --- |
| Function arguments | `f(args...)` | `f(a1, a2, a3)` |
| Initializer list | `{args...}` | `{a1, a2, a3}` |
| Base classes | `struct D : Bases...` | `struct D : B1, B2, B3` |
| Template arguments | `tuple<Args...>` | `tuple<int, double, string>` |
| Fold expression (C++17) | `(args + ...)` | `(a1 + (a2 + a3))` |

### `sizeof...(pack)`

Returns the number of elements in the pack at **compile time**:

```cpp
template <typename... Args>
constexpr auto count() { return sizeof...(Args); }
// count<int, double, char>() == 3
```

---

## Self-Assessment

### Q1: Implement a type-safe printf using variadic templates and recursive expansion

The recursive expansion pattern works by peeling off one argument at a time. The base case handles the situation where all arguments have been consumed; the recursive case processes one and recurses on the rest. This was the primary technique before C++17 fold expressions.

```cpp
#include <iostream>
#include <string>
#include <stdexcept>

// === Base case: no more arguments ===
void safe_printf(const char* fmt) {
    while (*fmt) {
        if (*fmt == '{' && *(fmt + 1) == '}') {
            throw std::runtime_error("Missing argument for format placeholder");
        }
        std::cout << *fmt++;
    }
}

// === Recursive case: peel off one argument at a time ===
template <typename T, typename... Args>
void safe_printf(const char* fmt, const T& first, const Args&... rest) {
    while (*fmt) {
        if (*fmt == '{' && *(fmt + 1) == '}') {
            std::cout << first;          // print the current argument
            safe_printf(fmt + 2, rest...); // recurse with remaining args
            return;
        }
        std::cout << *fmt++;
    }
    // If we get here, there are extra arguments — silently ignored
    // (or you could throw)
}

int main() {
    safe_printf("Hello, {}! You are {} years old.\n", "Alice", 30);
    // Output: Hello, Alice! You are 30 years old.

    safe_printf("Values: {}, {}, {}\n", 1, 2.5, "three");
    // Output: Values: 1, 2.5, three

    safe_printf("No placeholders here.\n");
    // Output: No placeholders here.

    // Type-safe: works with any type that supports operator<<
    std::string name = "Bob";
    safe_printf("{} has {} items.\n", name, 42);
    // Output: Bob has 42 items.

    return 0;
}
```

**How recursive expansion works:**

```cpp
safe_printf("Hello, {}! Age: {}\n", "Alice", 30)
  -> prints "Hello, "
  -> prints "Alice" (first argument)
  -> safe_printf("! Age: {}\n", 30)
      -> prints "! Age: "
      -> prints 30 (first argument)
      -> safe_printf("\n")  (base case)
          -> prints "\n"
```

Each level of recursion instantiates a new template with one fewer type in `Args...`, so the compiler generates one function per argument count: `safe_printf(fmt, T1, T2, T3)`, then `safe_printf(fmt, T2, T3)`, then `safe_printf(fmt, T3)`, then the non-template base case.

### Q2: Explain the difference between `sizeof...(Args)` and expanding `Args...` in different positions

The distinction between `f(args)...` and `f(args...)` is one of the most common variadic template mistakes. Here is a clear side-by-side of what each form does.

```cpp
#include <iostream>
#include <tuple>
#include <vector>

// sizeof...(Args) — returns the COUNT of elements (compile-time constant)
template <typename... Args>
constexpr std::size_t pack_size() {
    return sizeof...(Args);  // number of types
}

template <typename... Args>
constexpr std::size_t value_pack_size(Args... args) {
    return sizeof...(args);  // number of values (same as sizeof...(Args))
}

// === Expansion position matters! ===

// 1) Args... as function arguments
template <typename... Args>
auto make_tuple_like(Args... args) {
    return std::make_tuple(args...);  // make_tuple(a1, a2, a3)
}

// 2) Pattern applied BEFORE ...
template <typename... Args>
auto make_pointer_tuple(Args... args) {
    return std::make_tuple(&args...);  // make_tuple(&a1, &a2, &a3)
}

// 3) Expansion in different contexts
template <typename... Args>
void show_positions(Args... args) {
    // Each expansion applies the pattern to EACH element:
    //   transform(args)...  ->  transform(a1), transform(a2), transform(a3)
    //   args + 1 ...        ->  a1+1, a2+1, a3+1

    // Using initializer list for side effects (C++11/14):
    using expand = int[];
    (void)expand{0, (std::cout << args << " ", 0)...};
    std::cout << "\n";
}

int main() {
    // sizeof... returns count
    std::cout << "pack_size<int,double,char>: " << pack_size<int, double, char>() << "\n";
    // Output: 3

    std::cout << "value_pack_size(1, 2.0, 'a'): " << value_pack_size(1, 2.0, 'a') << "\n";
    // Output: 3

    // Expansion positions:
    auto t1 = make_tuple_like(1, 2.0, "hello");
    std::cout << "tuple: " << std::get<0>(t1) << ", "
              << std::get<1>(t1) << ", " << std::get<2>(t1) << "\n";

    show_positions(10, 20, 30);  // prints: 10 20 30

    return 0;
}
```

**Key distinctions:**

| Expression | Meaning |
| --- | --- |
| `sizeof...(Args)` | Returns the **count** of pack elements (compile-time `size_t`) |
| `Args...` | Expands to a comma-separated list of the types: `int, double, char` |
| `args...` | Expands to a comma-separated list of the values: `a1, a2, a3` |
| `f(args)...` | Expands to: `f(a1), f(a2), f(a3)` |
| `f(args...)` | Expands to: `f(a1, a2, a3)` - one call with all args! |

The `f(args)...` vs `f(args...)` distinction is the one that trips people up most. The `...` goes after the pattern that gets repeated, not after the pack name.

### Q3: Write a function that applies a transformation to each element of a parameter pack using fold expressions

C++17 fold expressions eliminate the need for recursive instantiation in many cases. Instead of a base case and a recursive case, you write a single expression with `...` to fold the pack over a binary operator.

```cpp
#include <iostream>
#include <string>
#include <cmath>

// === Fold expression to apply a transform and print each ===
template <typename F, typename... Args>
void for_each_arg(F&& f, Args&&... args) {
    // Unary left fold over comma operator:
    // (f(a1), (f(a2), f(a3)))
    (f(std::forward<Args>(args)), ...);
}

// === Fold expression: transform and sum ===
template <typename F, typename... Args>
auto transform_and_sum(F&& f, Args&&... args) {
    // (f(a1) + f(a2) + f(a3))
    return (f(std::forward<Args>(args)) + ...);
}

// === Fold expression: transform and collect into initializer_list ===
template <typename F, typename... Args>
auto transform_all(F&& f, Args&&... args) {
    return std::tuple{f(std::forward<Args>(args))...};  // pack expansion (not fold)
}

// === Fold with binary operator ===
template <typename... Args>
auto multiply_all(Args... args) {
    return (args * ...);  // right fold: a1 * (a2 * a3)
}

template <typename... Args>
bool all_positive(Args... args) {
    return ((args > 0) && ...);  // fold over &&
}

int main() {
    std::cout << "=== for_each_arg: print doubled ===\n";
    for_each_arg([](auto x) {
        std::cout << x * 2 << " ";
    }, 1, 2, 3, 4, 5);
    std::cout << "\n";  // 2 4 6 8 10

    std::cout << "\n=== transform_and_sum: sum of squares ===\n";
    auto sum_sq = transform_and_sum([](auto x) { return x * x; }, 3, 4, 5);
    std::cout << "3^2 + 4^2 + 5^2 = " << sum_sq << "\n";  // 50

    std::cout << "\n=== multiply_all ===\n";
    std::cout << "1*2*3*4*5 = " << multiply_all(1, 2, 3, 4, 5) << "\n";  // 120

    std::cout << "\n=== all_positive ===\n";
    std::cout << std::boolalpha;
    std::cout << "all_positive(1,2,3): " << all_positive(1, 2, 3) << "\n";    // true
    std::cout << "all_positive(1,-2,3): " << all_positive(1, -2, 3) << "\n";  // false

    std::cout << "\n=== for_each_arg: string transform ===\n";
    for_each_arg([](const auto& s) {
        std::cout << "[" << s << "] ";
    }, "hello", 42, 3.14, std::string("world"));
    std::cout << "\n";

    return 0;
}
```

**Expected output:**

```text
=== for_each_arg: print doubled ===
2 4 6 8 10

=== transform_and_sum: sum of squares ===
3^2 + 4^2 + 5^2 = 50

=== multiply_all ===
1*2*3*4*5 = 120

=== all_positive ===
all_positive(1,2,3): true
all_positive(1,-2,3): false

=== for_each_arg: string transform ===
[hello] [42] [3.14] [world]
```

**Four types of fold expressions (C++17):**

| Syntax | Name | Expansion |
| --- | --- | --- |
| `(args op ...)` | Unary right fold | `a1 op (a2 op a3)` |
| `(... op args)` | Unary left fold | `(a1 op a2) op a3` |
| `(args op ... op init)` | Binary right fold | `a1 op (a2 op (a3 op init))` |
| `(init op ... op args)` | Binary left fold | `((init op a1) op a2) op a3` |

---

## Notes

- Variadic templates are the foundation for `std::tuple`, `std::variant`, `std::make_shared`, etc.
- **C++11/14:** Use recursive expansion (base case + peel-one-off pattern).
- **C++17:** Prefer **fold expressions** - cleaner, no recursion, fewer template instantiations.
- `sizeof...(pack)` is a compile-time constant - use it in `if constexpr`, `static_assert`, etc.
- The `...` always goes **after** the pattern being expanded.
- Common pitfall: `f(args)...` vs `f(args...)` - they mean completely different things!
