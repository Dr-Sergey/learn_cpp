# Use `std::make_integer_sequence` and `std::index_sequence` for Compile-Time Iteration

**Category:** Compile Time Programming  
**Standard:** C++14 (integer_sequence in C++14, concept available since C++11 via manual impl)  
**Reference:** <https://en.cppreference.com/w/cpp/utility/integer_sequence>  

---

## Topic Overview

### What Is `std::index_sequence`

`std::index_sequence<Is...>` is a compile-time sequence of `std::size_t` values. It is an alias for `std::integer_sequence<std::size_t, Is...>`.

The reason it exists is that pack expansion - the `...` mechanism you use to expand variadic templates - only works over parameter packs, not over integer ranges. If you want to do "something for each index 0, 1, 2, ... N-1", you need those integers baked into a pack so you can expand them. `std::index_sequence` is how you manufacture that pack. Without it, you would have to write recursive template metaprogramming structs to achieve anything index-based - which is exactly what people did in C++11, and why it was so painful.

```cpp
// std::index_sequence<0, 1, 2> represents the compile-time sequence 0, 1, 2
// std::make_index_sequence<3> generates std::index_sequence<0, 1, 2>
```

### Key Types

| Type | Definition |
| --- | --- |
| `std::integer_sequence<T, Is...>` | Compile-time sequence of values of type `T` |
| `std::index_sequence<Is...>` | Alias for `integer_sequence<size_t, Is...>` |
| `std::make_index_sequence<N>` | Generates `index_sequence<0, 1, ..., N-1>` |
| `std::make_integer_sequence<T, N>` | Generates `integer_sequence<T, 0, 1, ..., N-1>` |
| `std::index_sequence_for<Ts...>` | `make_index_sequence<sizeof...(Ts)>` |

### The Pattern

The fundamental pattern is always the same. You write a helper function that takes an `index_sequence<Is...>` parameter, and inside it you expand an expression that uses `Is` as a pack. The outer function generates the sequence and passes it to the helper. The compiler sees the pack `Is...` and expands the expression once for each index.

Here is a minimal example of the pattern in action - printing all elements of a tuple:

```cpp
template<typename Tuple, std::size_t... Is>
void print_impl(const Tuple& t, std::index_sequence<Is...>) {
    ((std::cout << std::get<Is>(t) << " "), ...);  // fold expression over Is
}

template<typename... Args>
void print_tuple(const std::tuple<Args...>& t) {
    print_impl(t, std::index_sequence_for<Args...>{});
}
```

The outer `print_tuple` does nothing except manufacture the right sequence type and pass it to the helper. All the real work happens in `print_impl` where `Is...` is a proper pack and can be expanded. The helper function only ever exists to receive the pack - you would never call it directly.

---

## Self-Assessment

### Q1: Use `std::index_sequence` to expand a tuple into a function call at compile time

Expanding a tuple into a function call is one of the canonical uses of `index_sequence`. The trick is that `std::get<N>(tuple)` requires `N` to be a compile-time constant, so you cannot loop over a tuple at runtime. You have to expand it with pack expansion over an index sequence instead. C++17's `std::apply` does exactly this internally, but writing it yourself is the best way to understand what is happening.

```cpp
#include <iostream>
#include <tuple>
#include <utility>
#include <string>

// === apply_impl: expand tuple elements as function arguments ===
template<typename F, typename Tuple, std::size_t... Is>
decltype(auto) apply_impl(F&& f, Tuple&& t, std::index_sequence<Is...>) {
    return std::forward<F>(f)(std::get<Is>(std::forward<Tuple>(t))...);
}

// === apply: call f with tuple elements as arguments ===
template<typename F, typename Tuple>
decltype(auto) my_apply(F&& f, Tuple&& t) {
    constexpr auto size = std::tuple_size_v<std::remove_reference_t<Tuple>>;
    return apply_impl(std::forward<F>(f), std::forward<Tuple>(t),
                      std::make_index_sequence<size>{});
}

// === Print all tuple elements ===
template<typename Tuple, std::size_t... Is>
void print_impl(const Tuple& t, std::index_sequence<Is...>) {
    std::size_t idx = 0;
    ((std::cout << (idx++ ? ", " : "") << std::get<Is>(t)), ...);
}

template<typename... Args>
void print_tuple(const std::tuple<Args...>& t) {
    std::cout << "(";
    print_impl(t, std::index_sequence_for<Args...>{});
    std::cout << ")";
}

// Test function
int sum3(int a, int b, int c) { return a + b + c; }

void greet(const std::string& name, int age) {
    std::cout << "Hello " << name << ", age " << age << "\n";
}

int main() {
    // === Expand tuple into function call ===
    auto args1 = std::make_tuple(10, 20, 30);
    int result = my_apply(sum3, args1);
    std::cout << "sum3(10, 20, 30) = " << result << "\n";

    auto args2 = std::make_tuple(std::string("Alice"), 30);
    my_apply(greet, args2);

    // Lambda with tuple arguments
    auto args3 = std::make_tuple(3.14, 2.0);
    auto product = my_apply([](double a, double b) { return a * b; }, args3);
    std::cout << "3.14 * 2.0 = " << product << "\n";

    // === Print tuple ===
    std::cout << "\nTuple: ";
    print_tuple(std::make_tuple(1, "hello", 3.14, 'x'));
    std::cout << "\n";

    // Note: std::apply (C++17) does the same as my_apply
    int r2 = std::apply(sum3, args1);
    std::cout << "\nstd::apply result: " << r2 << "\n";

    return 0;
}
```

The key line in `apply_impl` is `std::get<Is>(std::forward<Tuple>(t))...` - this expands to `std::get<0>(t), std::get<1>(t), std::get<2>(t)` and so on, effectively unpacking the tuple as a function argument list. The `...` at the end of the expression is what drives the expansion over the `Is` pack.

**Expected output:**

```text
sum3(10, 20, 30) = 60
Hello Alice, age 30
3.14 * 2.0 = 6.28

Tuple: (1, hello, 3.14, x)

std::apply result: 60
```

### Q2: Implement a compile-time loop over a `constexpr` array using `index_sequence`

The same index sequence technique works for transforming `constexpr` arrays. Because the result needs to be a new `std::array` with the same size, and all the element values are computed from compile-time expressions, you can do the entire thing at compile time. Watch the `static_assert` lines - they prove these are genuine compile-time constants, not just values that happen to be computed early.

```cpp
#include <iostream>
#include <array>
#include <utility>

// === Compile-time loop: transform each element of a constexpr array ===

template<typename T, std::size_t N, typename F, std::size_t... Is>
constexpr auto transform_impl(const std::array<T, N>& arr, F f,
                                std::index_sequence<Is...>) {
    return std::array<decltype(f(arr[0])), N>{ f(arr[Is])... };
}

template<typename T, std::size_t N, typename F>
constexpr auto transform(const std::array<T, N>& arr, F f) {
    return transform_impl(arr, f, std::make_index_sequence<N>{});
}

// === Compile-time sum ===
template<typename T, std::size_t N, std::size_t... Is>
constexpr T sum_impl(const std::array<T, N>& arr, std::index_sequence<Is...>) {
    return (arr[Is] + ... + T{0});
}

template<typename T, std::size_t N>
constexpr T sum(const std::array<T, N>& arr) {
    return sum_impl(arr, std::make_index_sequence<N>{});
}

// === Compile-time reverse ===
template<typename T, std::size_t N, std::size_t... Is>
constexpr auto reverse_impl(const std::array<T, N>& arr, std::index_sequence<Is...>) {
    return std::array<T, N>{ arr[N - 1 - Is]... };
}

template<typename T, std::size_t N>
constexpr auto reverse(const std::array<T, N>& arr) {
    return reverse_impl(arr, std::make_index_sequence<N>{});
}

// === Tests ===
constexpr std::array<int, 5> data = {1, 2, 3, 4, 5};

constexpr auto squared = transform(data, [](int x) { return x * x; });
static_assert(squared[0] == 1);
static_assert(squared[1] == 4);
static_assert(squared[2] == 9);
static_assert(squared[4] == 25);

constexpr auto total = sum(data);
static_assert(total == 15);

constexpr auto reversed = reverse(data);
static_assert(reversed[0] == 5);
static_assert(reversed[4] == 1);

int main() {
    std::cout << "=== transform (square each element) ===\n";
    std::cout << "Original: ";
    for (auto v : data) std::cout << v << " ";
    std::cout << "\nSquared:  ";
    for (auto v : squared) std::cout << v << " ";

    std::cout << "\n\n=== sum ===\n";
    std::cout << "Sum = " << total << "\n";

    std::cout << "\n=== reverse ===\n";
    std::cout << "Reversed: ";
    for (auto v : reversed) std::cout << v << " ";
    std::cout << "\n";

    // All computed at compile time - zero runtime cost
    std::cout << "\nAll results are constexpr (compile-time computed).\n";

    return 0;
}
```

At runtime the program just reads pre-computed values out of read-only memory. The `transform`, `sum`, and `reverse` function bodies are never emitted as machine code for these calls.

**Expected output:**

```text
=== transform (square each element) ===
Original: 1 2 3 4 5
Squared:  1 4 9 16 25

=== sum ===
Sum = 15

=== reverse ===
Reversed: 5 4 3 2 1

All results are constexpr (compile-time computed).
```

### Q3: Show how `index_sequence` replaces recursive template metaprogramming for tuple manipulation

Before C++14, doing anything indexed with tuples required writing a recursive struct template - a base case and a general case, with each instantiation printing one element and recursing for the rest. It works, but it is verbose and hard to read. `index_sequence` with a fold expression replaces the entire recursive machinery with a single function. This is not just a style improvement - it also compiles faster because the compiler only needs to instantiate one function template instead of a chain of N recursive struct specializations.

```cpp
#include <iostream>
#include <tuple>
#include <utility>
#include <string>

// ===================================================================
// OLD WAY: Recursive template metaprogramming (C++11)
// ===================================================================

// Print tuple recursively
template<std::size_t I, typename Tuple>
struct TuplePrinterOld {
    static void print(const Tuple& t) {
        TuplePrinterOld<I - 1, Tuple>::print(t);
        std::cout << ", " << std::get<I>(t);
    }
};

template<typename Tuple>
struct TuplePrinterOld<0, Tuple> {
    static void print(const Tuple& t) {
        std::cout << std::get<0>(t);
    }
};

template<typename... Args>
void print_tuple_old(const std::tuple<Args...>& t) {
    std::cout << "(";
    TuplePrinterOld<sizeof...(Args) - 1, std::tuple<Args...>>::print(t);
    std::cout << ")";
}

// ===================================================================
// NEW WAY: index_sequence (C++14) - no recursion
// ===================================================================

template<typename Tuple, std::size_t... Is>
void print_tuple_impl(const Tuple& t, std::index_sequence<Is...>) {
    std::size_t n = 0;
    ((std::cout << (n++ ? ", " : "") << std::get<Is>(t)), ...);
}

template<typename... Args>
void print_tuple_new(const std::tuple<Args...>& t) {
    std::cout << "(";
    print_tuple_impl(t, std::index_sequence_for<Args...>{});
    std::cout << ")";
}

// === Tuple head/tail with index_sequence - no recursion ===
template<typename Tuple, std::size_t... Is>
auto tail_impl(const Tuple& t, std::index_sequence<Is...>) {
    return std::make_tuple(std::get<Is + 1>(t)...);
}

template<typename Head, typename... Tail>
auto tail(const std::tuple<Head, Tail...>& t) {
    return tail_impl(t, std::make_index_sequence<sizeof...(Tail)>{});
}

// === Tuple zip with index_sequence ===
template<typename T1, typename T2, std::size_t... Is>
auto zip_impl(const T1& a, const T2& b, std::index_sequence<Is...>) {
    return std::make_tuple(std::make_pair(std::get<Is>(a), std::get<Is>(b))...);
}

template<typename... A, typename... B>
auto zip(const std::tuple<A...>& a, const std::tuple<B...>& b) {
    static_assert(sizeof...(A) == sizeof...(B), "Tuples must have same size");
    return zip_impl(a, b, std::make_index_sequence<sizeof...(A)>{});
}

int main() {
    auto t = std::make_tuple(1, "hello", 3.14, 'x');

    std::cout << "=== OLD: Recursive TMP ===\n";
    print_tuple_old(t);
    std::cout << "\n";

    std::cout << "\n=== NEW: index_sequence ===\n";
    print_tuple_new(t);
    std::cout << "\n";

    // === tail ===
    auto t2 = std::make_tuple(10, 20, 30, 40);
    auto t2_tail = tail(t2);
    std::cout << "\n=== tail({10,20,30,40}) ===\n";
    print_tuple_new(t2_tail);
    std::cout << "\n";

    // === zip ===
    auto names = std::make_tuple("Alice", "Bob", "Carol");
    auto ages = std::make_tuple(30, 25, 35);
    auto zipped = zip(names, ages);
    std::cout << "\n=== zip ===\n";
    std::cout << "Pair 0: " << std::get<0>(zipped).first << " = "
              << std::get<0>(zipped).second << "\n";
    std::cout << "Pair 1: " << std::get<1>(zipped).first << " = "
              << std::get<1>(zipped).second << "\n";
    std::cout << "Pair 2: " << std::get<2>(zipped).first << " = "
              << std::get<2>(zipped).second << "\n";

    std::cout << "\n=== Comparison ===\n";
    std::cout << "Recursive TMP: multiple struct specializations, complex\n";
    std::cout << "index_sequence: one function, fold expression, simple\n";

    return 0;
}
```

Both approaches produce identical results. The new way is not just shorter: it also compiles faster because the compiler only needs to instantiate one function template instead of a chain of N recursive struct specializations.

**Expected output:**

```text
=== OLD: Recursive TMP ===
(1, hello, 3.14, x)

=== NEW: index_sequence ===
(1, hello, 3.14, x)

=== tail({10,20,30,40}) ===
(20, 30, 40)

=== zip ===
Pair 0: Alice = 30
Pair 1: Bob = 25
Pair 2: Carol = 35

=== Comparison ===
Recursive TMP: multiple struct specializations, complex
index_sequence: one function, fold expression, simple
```

---

## Notes

- `std::make_index_sequence<N>` generates `index_sequence<0, 1, ..., N-1>` - no manual listing.
- `std::index_sequence_for<Ts...>` generates an index sequence matching a parameter pack's size.
- The pattern is always: helper function takes `index_sequence<Is...>`, expands pack over `Is...`.
- Fold expressions (C++17) work perfectly with `index_sequence` for one-line expansions.
- This completely replaces recursive template metaprogramming for indexed access patterns.
- `std::apply` (C++17) uses `index_sequence` internally - it's the standard "tuple to function call" utility.
