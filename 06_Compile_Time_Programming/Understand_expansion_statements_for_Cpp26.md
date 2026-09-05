# Understand expansion statements for C++26

**Category:** Compile Time Programming  
**Standard:** C++26  
**Reference:** <https://wg21.link/P1306>  

---

## Topic Overview

Expansion statements (`for...` over packs and tuples) allow iterating over parameter packs and compile-time sequences directly in regular code, replacing recursive template metaprogramming. This is one of those features that makes you wonder why it wasn't in C++ from the beginning - once you see it, the old workarounds look unnecessarily painful.

### Current Workarounds (Pre-C++26)

Before C++26, iterating over a tuple or a parameter pack requires an indirection through an index sequence. The idea is to "unpack" the compile-time indices as a parameter pack and then use a fold expression over them. It works, but it's noisy and requires a helper function:

```cpp
#include <tuple>
#include <iostream>

// To iterate a tuple, you need:
// 1. index_sequence trick
template<typename Tuple, size_t... Is>
void print_impl(const Tuple& t, std::index_sequence<Is...>) {
    ((std::cout << std::get<Is>(t) << " "), ...);
}

template<typename... Ts>
void print_tuple(const std::tuple<Ts...>& t) {
    print_impl(t, std::index_sequence_for<Ts...>{});
}
```

Every time you need to do something element-by-element with a tuple you have to write this same two-function pattern. The real logic is buried in the fold expression inside `print_impl`.

### C++26 Expansion Statements

C++26 lets you write the same thing as a direct loop. `template for` unrolls at compile time - each iteration may have a different type, so it's really the compiler generating N copies of the loop body with different types substituted in:

```cpp
#include <tuple>
#include <iostream>

// C++26: direct iteration over tuples and packs
template<typename... Ts>
void print_all(const std::tuple<Ts...>& t) {
    template for (const auto& elem : t) {
        std::cout << elem << " ";
    }
}

// Also works with parameter packs:
template<typename... Args>
void log(Args&&... args) {
    template for (auto&& arg : {args...}) {
        std::cout << arg << "\n";
    }
}
```

### Compile-Time Conditional in Expansion

The real power shows up when you combine `template for` with `if constexpr`. Because each loop iteration has a potentially different type, you can branch on the type inside the loop body - something a regular for loop can never do:

```cpp
#include <tuple>
#include <type_traits>
#include <iostream>

template<typename... Ts>
void print_ints_only(const std::tuple<Ts...>& t) {
    template for (const auto& elem : t) {
        if constexpr (std::is_integral_v<std::remove_cvref_t<decltype(elem)>>) {
            std::cout << elem << " ";
        }
    }
}

auto t = std::tuple{42, "hello", 3.14, 99};
print_ints_only(t);  // Prints: 42 99
```

---

## Self-Assessment

### Q1: How does `template for` differ from a regular for loop

`template for` unrolls at compile time - each iteration may have a different type. A regular for loop requires a single type for the loop variable. `template for` is essentially a compile-time fold over heterogeneous collections. Think of it as the compiler writing N separate code blocks (one per element) rather than generating a single loop in the output.

### Q2: What can you iterate over with expansion statements

Tuples, parameter packs, and `std::integer_sequence`. Anything that has a compile-time-known size and heterogeneous element types.

### Q3: What does this replace

Index sequence tricks, fold expressions (for complex operations), recursive template instantiation, `std::apply`, and `std::tuple_cat` workarounds. Expansion statements make tuple/pack iteration as simple as iterating a range.

---

## Notes

- `template for` is the keyword combination (subject to change before C++26 ships).
- Each "iteration" instantiates the loop body with a different type.
- Massively simplifies reflection-based code generation (combined with P2996).
- Reduces compile times vs recursive template instantiation.
