# Understand Pack Indexing (C++26) for Accessing Parameter Pack Elements by Index

**Category:** Compile Time Programming  
**Standard:** C++26 (P2662R3)  
**Reference:** <https://en.cppreference.com/w/cpp/language/pack_indexing>  

---

## Topic Overview

### What Is Pack Indexing

Pack indexing (C++26) introduces **direct subscript syntax** for accessing elements of a parameter pack by index, both for **types** and **values**. It's about as simple as it sounds - you just write `Args...[I]` and you get the I-th element:

```cpp
// Type pack indexing: Args...[I] gives the I-th type
template <typename... Args>
using First = Args...[0];       // First type in pack

// Value pack indexing: args...[I] gives the I-th value
template <auto... vals>
constexpr auto second = vals...[1];  // Second value in pack
```

The index must be a **constant expression** - this is resolved entirely at compile time.

### Before C++26: The Workaround Problem

Accessing a specific pack element before C++26 required verbose tricks, and every approach had a cost. The reason this was so painful is that a parameter pack is not a type you can index - it's a language construct that can only be "unpacked" all at once, not picked apart element by element, unless you use one of these workarounds:

| Approach | Code | Drawback |
| --- | --- | --- |
| `std::tuple_element` | `std::tuple_element_t<I, std::tuple<Args...>>` | Creates intermediate tuple type |
| Recursive template | `template<size_t, typename...> struct At;` + specializations | Verbose, slow compile |
| `std::get<I>(std::make_tuple(args...))` | Creates runtime tuple | Not zero-cost at compile time |
| Fold + index counter | Complex fold expression | Hard to read |

### C++26 Pack Indexing - Clean and Direct

The C++26 syntax eliminates all of those workarounds. Notice how the return type annotation `-> Args...[0]&&` also works - you can use pack indexing wherever a type is expected:

```cpp
// Type indexing
template <typename... Ts>
struct FirstType {
    using type = Ts...[0];
};

// Value indexing
template <typename... Args>
constexpr auto get_nth(std::size_t n, Args... args) {
    return args...[n];  // Only if n is a constant expression
}

// In function parameter context
template <typename... Args>
auto first_arg(Args&&... args) -> Args...[0]&& {
    return std::forward<Args...[0]>(args...[0]);
}
```

**Important:** The index must be a **constant expression** - it's resolved at compile time.

---

## Self-Assessment

### Q1: Use `Args...[0]` to extract the first type of a parameter pack in C++26

Here's a full demonstration of type and value pack indexing, including aliases for "first", "last", and "Nth", plus a `PackInfo` struct that bundles all of them together.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === C++26: Direct pack indexing for types ===

// Get the first type from a parameter pack
template <typename... Args>
using First = Args...[0];

// Get the last type from a parameter pack
template <typename... Args>
using Last = Args...[sizeof...(Args) - 1];

// Get the N-th type from a parameter pack
template <std::size_t N, typename... Args>
using NthType = Args...[N];

// === Function that returns the first argument ===
template <typename... Args>
auto get_first(Args&&... args) -> Args...[0] {
    return args...[0];  // Value pack indexing
}

// === Function that returns the last argument ===
template <typename... Args>
auto get_last(Args&&... args) -> Args...[sizeof...(Args) - 1] {
    return args...[sizeof...(Args) - 1];
}

// === Type traits using pack indexing ===
template <typename... Args>
struct PackInfo {
    using first_type = Args...[0];
    using last_type  = Args...[sizeof...(Args) - 1];
    static constexpr std::size_t size = sizeof...(Args);
};

int main() {
    // Type indexing
    static_assert(std::is_same_v<First<int, double, char>, int>);
    static_assert(std::is_same_v<Last<int, double, char>, char>);
    static_assert(std::is_same_v<NthType<1, int, double, char>, double>);

    // Value indexing
    auto f = get_first(42, 3.14, std::string("hello"));
    std::cout << "First: " << f << "\n";  // 42

    auto l = get_last(42, 3.14, std::string("hello"));
    std::cout << "Last: " << l << "\n";   // hello

    // PackInfo
    using Info = PackInfo<int, double, std::string>;
    static_assert(std::is_same_v<Info::first_type, int>);
    static_assert(std::is_same_v<Info::last_type, std::string>);
    static_assert(Info::size == 3);
    std::cout << "Pack size: " << Info::size << "\n";

    return 0;
}
```

**Expected output:**

```text
First: 42
Last: hello
Pack size: 3
```

### Q2: Show the C++20 workaround using `std::tuple_element` for the same operation

Seeing the old way right next to the new way makes the improvement concrete. The `NthType_recursive` version is especially instructive - it shows the O(N) recursive instantiation chain that pack indexing replaces with a single lookup.

```cpp
#include <iostream>
#include <tuple>
#include <type_traits>
#include <string>

// === C++20 Workaround: std::tuple_element ===

// Get N-th type via tuple_element (C++11)
template <std::size_t N, typename... Args>
using NthType_v20 = std::tuple_element_t<N, std::tuple<Args...>>;

// Aliases for first and last
template <typename... Args>
using First_v20 = NthType_v20<0, Args...>;

template <typename... Args>
using Last_v20 = NthType_v20<sizeof...(Args) - 1, Args...>;

// Get N-th value via std::get on a tuple
template <std::size_t N, typename... Args>
auto get_nth(Args&&... args) {
    return std::get<N>(std::forward_as_tuple(std::forward<Args>(args)...));
}

// === Alternative C++20 workaround: recursive template ===
template <std::size_t N, typename First, typename... Rest>
struct NthTypeImpl {
    using type = typename NthTypeImpl<N - 1, Rest...>::type;
};

template <typename First, typename... Rest>
struct NthTypeImpl<0, First, Rest...> {
    using type = First;
};

template <std::size_t N, typename... Args>
using NthType_recursive = typename NthTypeImpl<N, Args...>::type;

// === Comparison table ===
// C++26:  Args...[N]                         - direct, zero overhead
// C++20:  tuple_element_t<N, tuple<Args...>> - creates tuple type
// C++11:  NthTypeImpl<N, Args...>::type      - recursive instantiation

int main() {
    // Type checks
    static_assert(std::is_same_v<First_v20<int, double, char>, int>);
    static_assert(std::is_same_v<Last_v20<int, double, char>, char>);
    static_assert(std::is_same_v<NthType_v20<1, int, double, char>, double>);
    static_assert(std::is_same_v<NthType_recursive<2, int, double, char>, char>);

    // Value access
    auto val = get_nth<1>(42, 3.14, std::string("hello"));
    std::cout << "get_nth<1>(42, 3.14, \"hello\") = " << val << "\n";  // 3.14

    auto first = get_nth<0>(42, 3.14, std::string("hello"));
    std::cout << "get_nth<0>(...) = " << first << "\n";  // 42

    auto last = get_nth<2>(42, 3.14, std::string("hello"));
    std::cout << "get_nth<2>(...) = " << last << "\n";   // hello

    std::cout << "\n=== Comparison ===\n";
    std::cout << "C++26: Args...[N]                          (direct syntax)\n";
    std::cout << "C++20: tuple_element_t<N, tuple<Args...>>  (creates tuple type)\n";
    std::cout << "C++11: recursive template NthTypeImpl       (O(N) instantiations)\n";

    return 0;
}
```

**Expected output:**

```text
get_nth<1>(42, 3.14, "hello") = 3.14
get_nth<0>(...) = 42
get_nth<2>(...) = hello

=== Comparison ===
C++26: Args...[N]                          (direct syntax)
C++20: tuple_element_t<N, tuple<Args...>>  (creates tuple type)
C++11: recursive template NthTypeImpl       (O(N) instantiations)
```

### Q3: Apply pack indexing to implement a compile-time index-based tuple accessor

This builds a `SimpleTuple` from scratch and uses pack indexing for both the `get<N>` function return type and the `TupleElement` type alias. It's a good exercise to trace through how the recursive `get` implementation eventually bottoms out at `N == 0`.

```cpp
#include <iostream>
#include <string>
#include <type_traits>
#include <cstddef>
#include <utility>

// === C++26: Compile-time tuple using pack indexing ===

template <typename... Ts>
struct SimpleTuple;

// Base case: empty tuple
template <>
struct SimpleTuple<> {};

// Recursive case
template <typename Head, typename... Tail>
struct SimpleTuple<Head, Tail...> {
    Head value;
    SimpleTuple<Tail...> rest;

    constexpr SimpleTuple(Head h, Tail... t) : value(h), rest(t...) {}
};

// === C++26 get<N> using pack indexing ===
template <std::size_t N, typename... Ts>
constexpr auto& get(SimpleTuple<Ts...>& t) {
    // Return type is Ts...[N]
    if constexpr (N == 0) {
        return t.value;
    } else {
        return get<N - 1>(t.rest);
    }
}

template <std::size_t N, typename... Ts>
constexpr const auto& get(const SimpleTuple<Ts...>& t) {
    if constexpr (N == 0) {
        return t.value;
    } else {
        return get<N - 1>(t.rest);
    }
}

// === Type access using pack indexing (C++26) ===
template <std::size_t N, typename... Ts>
using TupleElement = Ts...[N];

// === Size trait ===
template <typename T>
struct TupleSize;

template <typename... Ts>
struct TupleSize<SimpleTuple<Ts...>> {
    static constexpr std::size_t value = sizeof...(Ts);
};

// === Print all elements using fold + index_sequence ===
template <typename Tuple, std::size_t... Is>
void print_impl(const Tuple& t, std::index_sequence<Is...>) {
    ((std::cout << (Is == 0 ? "" : ", ") << get<Is>(t)), ...);
}

template <typename... Ts>
void print_tuple(const SimpleTuple<Ts...>& t) {
    std::cout << "(";
    print_impl(t, std::make_index_sequence<sizeof...(Ts)>{});
    std::cout << ")";
}

int main() {
    // Create a tuple
    constexpr SimpleTuple t(42, 3.14, 'X');

    // Access by index
    static_assert(get<0>(t) == 42);
    static_assert(get<1>(t) == 3.14);
    static_assert(get<2>(t) == 'X');

    std::cout << "Element 0: " << get<0>(t) << "\n";  // 42
    std::cout << "Element 1: " << get<1>(t) << "\n";  // 3.14
    std::cout << "Element 2: " << get<2>(t) << "\n";  // X

    // Type access (C++26)
    static_assert(std::is_same_v<TupleElement<0, int, double, char>, int>);
    static_assert(std::is_same_v<TupleElement<1, int, double, char>, double>);
    static_assert(std::is_same_v<TupleElement<2, int, double, char>, char>);

    // Size
    static_assert(TupleSize<SimpleTuple<int, double, char>>::value == 3);

    // Print
    std::cout << "Tuple: ";
    print_tuple(t);
    std::cout << "\n";

    // Mutable access
    SimpleTuple t2(10, 20.0, 'A');
    get<0>(t2) = 99;
    get<2>(t2) = 'Z';
    std::cout << "Modified: ";
    print_tuple(t2);
    std::cout << "\n";

    std::cout << "\n=== Pack Indexing Summary ===\n";
    std::cout << "Type:  Ts...[N]  - direct type access in pack\n";
    std::cout << "Value: ts...[N]  - direct value access in pack\n";
    std::cout << "Index must be a constant expression\n";
    std::cout << "Replaces tuple_element_t<N, tuple<Ts...>> pattern\n";

    return 0;
}
```

**Expected output:**

```text
Element 0: 42
Element 1: 3.14
Element 2: X
Tuple: (42, 3.14, X)
Modified: (99, 20, Z)

=== Pack Indexing Summary ===
Type:  Ts...[N]  - direct type access in pack
Value: ts...[N]  - direct value access in pack
Index must be a constant expression
Replaces tuple_element_t<N, tuple<Ts...>> pattern
```

---

## Notes

- Pack indexing (`Args...[N]`) is a C++26 feature - check compiler support before using.
- The index must be a constant expression (known at compile time).
- Works for both **type packs** (`typename... Ts`) and **value packs** (`auto... vals`).
- Eliminates the need for `std::tuple_element_t<N, std::tuple<Args...>>` workaround.
- Compared to recursive templates, pack indexing has O(1) compile-time cost (no recursive instantiation).
- Before C++26, `std::tuple_element` + `std::get` is the standard-conforming approach.
