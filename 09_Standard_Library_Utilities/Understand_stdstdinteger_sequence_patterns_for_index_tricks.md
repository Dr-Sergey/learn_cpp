# Understand std::integer_sequence patterns for index tricks

**Category:** Standard Library Utilities  
**Standard:** C++14, C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/integer_sequence>  

---

## Topic Overview

`std::integer_sequence` and `std::index_sequence` provide compile-time integer lists used for tuple decomposition, array initialization, and pack expansion patterns.

The core idea is simple: sometimes you want to expand a parameter pack with indices attached - "call `std::get<0>`, then `std::get<1>`, then `std::get<2>`..." - and the only way to do that is to have those indices available as a template parameter pack you can expand with `...`. That is exactly what `index_sequence` gives you.

### Tuple Iteration

Here is the classic two-step pattern. You call a public function that creates an index sequence matching the tuple's length, then pass it to an implementation function that can expand both the indices and the tuple elements together.

```cpp
#include <tuple>
#include <iostream>
#include <utility>

template<typename Tuple, size_t... Is>
void print_impl(const Tuple& t, std::index_sequence<Is...>) {
    ((std::cout << (Is == 0 ? "" : ", ") << std::get<Is>(t)), ...);
}

template<typename... Ts>
void print_tuple(const std::tuple<Ts...>& t) {
    print_impl(t, std::index_sequence_for<Ts...>{});
}

int main() {
    auto t = std::make_tuple(42, "hello", 3.14);
    print_tuple(t);  // 42, hello, 3.14
}
```

The reason you need the helper function is that you cannot get the indices into scope any other way - you have to let template deduction unpack `index_sequence<Is...>` into the `Is...` pack.

### Compile-Time Array Generation

Once you have an index sequence, you can use it to generate an array at compile time, where each element is a function of its index. The lambda template trick in the second example uses a C++20 explicit template parameter on the lambda.

```cpp
#include <array>
#include <utility>
#include <cstddef>

template<size_t... Is>
constexpr auto make_squares(std::index_sequence<Is...>) {
    return std::array{(Is * Is)...};
}

constexpr auto squares = make_squares(std::make_index_sequence<10>{});
// squares = {0, 1, 4, 9, 16, 25, 36, 49, 64, 81}

// Generate a lookup table at compile time:
template<size_t N, typename F>
constexpr auto generate_table(F f) {
    return [&]<size_t... Is>(std::index_sequence<Is...>) {
        return std::array{f(Is)...};
    }(std::make_index_sequence<N>{});
}

constexpr auto sin_table = generate_table<360>([](size_t deg) {
    // Simplified - real code would use constexpr math
    return static_cast<double>(deg);
});
```

### Apply Function to Tuple Arguments

This is the pattern behind `std::apply`. It unpacks a tuple into a function call's argument list by expanding both the tuple and an index sequence together.

```cpp
#include <tuple>
#include <utility>

template<typename F, typename Tuple, size_t... Is>
auto apply_impl(F&& f, Tuple&& t, std::index_sequence<Is...>) {
    return f(std::get<Is>(std::forward<Tuple>(t))...);
}

// std::apply does exactly this:
// std::apply(f, tuple{1, 2, 3}) -> f(1, 2, 3)
```

---

## Self-Assessment

### Q1: What is `make_index_sequence<N>`

It generates `index_sequence<0, 1, 2, ..., N-1>`. This is the compile-time equivalent of `range(N)` in Python. The compiler generates it in O(log N) template instantiations.

### Q2: How to reverse an index sequence

Given an existing index sequence, you can produce a reversed one by transforming each index `i` into `N - 1 - i` during the pack expansion.

```cpp
template<size_t N, size_t... Is>
auto reverse_seq(std::index_sequence<Is...>) {
    return std::index_sequence<(N - 1 - Is)...>{};
}
// reverse_seq<5>(index_sequence<0,1,2,3,4>) -> index_sequence<4,3,2,1,0>
```

### Q3: What replaces index_sequence in C++26

Expansion statements (`template for`) will eliminate most uses of index_sequence by allowing direct iteration over tuples and packs. You will be able to write a simple loop over a tuple without any index machinery.

---

## Notes

- `index_sequence_for<Ts...>` is shorthand for `make_index_sequence<sizeof...(Ts)>`.
- `std::apply` (C++17) wraps the index_sequence pattern for tuple-to-arguments conversion.
- Used heavily in tuple utilities, structured bindings implementation, and variadic template code.
- C++26 expansion statements will make many index_sequence patterns obsolete.
