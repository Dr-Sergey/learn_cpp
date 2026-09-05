# Use `std::tuple` as a Type-Level Cons Cell for Compile-Time Type Lists

**Category:** Templates and Generic Programming  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/utility/tuple>  

---

## Topic Overview

### What Does This Mean

In functional programming, a **cons cell** builds a list: `cons(head, tail)`. The idea is that any list is either empty or a pair of "the first element" plus "the rest of the list." You can apply the same mental model to types at compile time, and `std::tuple<Ts...>` is the standard tool that makes it practical:

```cpp
using TypeList = std::tuple<int, double, std::string>;
// "Head" = int
// "Tail" = std::tuple<double, std::string>
```

Unlike a runtime list, this list has no storage at all when you are only using it for type computations - the tuple exists as a type, not a value.

### Why Use Tuple as a Type List

| Approach | Pros | Cons |
| --- | --- | --- |
| Hand-rolled `TypeList<Ts...>` | Customizable | Must write everything from scratch |
| `std::tuple<Ts...>` | `tuple_size`, `tuple_element`, `tuple_cat` built-in | Creates actual types (heavier) |
| Boost.Mp11 / Boost.Hana | Powerful, well-tested | External dependency |

### Available Operations on Tuples

| Operation | Standard Tool |
| --- | --- |
| Size (count types) | `std::tuple_size_v<Tuple>` |
| Access Nth type | `std::tuple_element_t<N, Tuple>` |
| Concatenate | `std::tuple_cat(t1, t2)` - but for types, use template metaprogramming |
| Head | `std::tuple_element_t<0, Tuple>` |
| Tail | Custom metafunction needed |

---

## Self-Assessment

### Q1: Implement `Head<Tuple>` and `Tail<Tuple>` metafunctions using `std::tuple_element` and `tuple_cat`

The trick for all of these is partial specialization on the pack form of `std::tuple`. Once you have `std::tuple<H, Ts...>`, the pack `Ts...` is your tail. The compiler does all the work.

```cpp
#include <iostream>
#include <tuple>
#include <type_traits>
#include <utility>
#include <string>

// === Head: get the first type ===
template <typename Tuple>
struct Head;

template <typename H, typename... Ts>
struct Head<std::tuple<H, Ts...>> {
    using type = H;
};

template <typename Tuple>
using Head_t = typename Head<Tuple>::type;

// === Tail: remove the first type, return rest as tuple ===
template <typename Tuple>
struct Tail;

template <typename H, typename... Ts>
struct Tail<std::tuple<H, Ts...>> {
    using type = std::tuple<Ts...>;
};

template <typename Tuple>
using Tail_t = typename Tail<Tuple>::type;

// === Size: number of types ===
template <typename Tuple>
constexpr std::size_t Size_v = std::tuple_size_v<Tuple>;

// === Cons: prepend a type ===
template <typename T, typename Tuple>
struct Cons;

template <typename T, typename... Ts>
struct Cons<T, std::tuple<Ts...>> {
    using type = std::tuple<T, Ts...>;
};

template <typename T, typename Tuple>
using Cons_t = typename Cons<T, Tuple>::type;

// === Append: add a type at the end ===
template <typename Tuple, typename T>
struct Append;

template <typename... Ts, typename T>
struct Append<std::tuple<Ts...>, T> {
    using type = std::tuple<Ts..., T>;
};

template <typename Tuple, typename T>
using Append_t = typename Append<Tuple, T>::type;

// === Concat: concatenate two tuple-lists ===
template <typename T1, typename T2>
struct Concat;

template <typename... T1s, typename... T2s>
struct Concat<std::tuple<T1s...>, std::tuple<T2s...>> {
    using type = std::tuple<T1s..., T2s...>;
};

template <typename T1, typename T2>
using Concat_t = typename Concat<T1, T2>::type;

int main() {
    using List = std::tuple<int, double, std::string, char>;

    std::cout << "=== Head, Tail, Size ===\n";
    static_assert(std::is_same_v<Head_t<List>, int>);
    static_assert(std::is_same_v<Tail_t<List>, std::tuple<double, std::string, char>>);
    static_assert(Size_v<List> == 4);
    std::cout << "Head = int, Tail = tuple<double,string,char>, Size = 4\n";

    std::cout << "\n=== Cons (prepend) ===\n";
    using WithBool = Cons_t<bool, List>;
    static_assert(std::is_same_v<WithBool, std::tuple<bool, int, double, std::string, char>>);
    static_assert(Size_v<WithBool> == 5);
    std::cout << "Cons<bool, List> -> size " << Size_v<WithBool> << "\n";

    std::cout << "\n=== Append ===\n";
    using WithFloat = Append_t<List, float>;
    static_assert(std::is_same_v<WithFloat, std::tuple<int, double, std::string, char, float>>);
    std::cout << "Append<List, float> -> size " << Size_v<WithFloat> << "\n";

    std::cout << "\n=== Concat ===\n";
    using A = std::tuple<int, char>;
    using B = std::tuple<double, float>;
    using AB = Concat_t<A, B>;
    static_assert(std::is_same_v<AB, std::tuple<int, char, double, float>>);
    std::cout << "Concat<A,B> -> size " << Size_v<AB> << "\n";

    return 0;
}
```

The `static_assert` calls here are not just tests - they are documentation. They tell any future reader exactly what each operation is supposed to produce, verified by the compiler.

### Q2: Build a compile-time transform that applies a metafunction to each type in a tuple-list

This is where the type-list approach gets genuinely useful. If you have a list of types and you want to produce a new list where each type is modified in the same way - for example, wrapping each type in a pointer or adding const - you write a `Transform` metafunction.

```cpp
#include <iostream>
#include <tuple>
#include <type_traits>
#include <string>
#include <vector>

// === Transform: apply a metafunction F to each type ===
template <typename Tuple, template <typename> class F>
struct Transform;

template <typename... Ts, template <typename> class F>
struct Transform<std::tuple<Ts...>, F> {
    using type = std::tuple<typename F<Ts>::type...>;
};

template <typename Tuple, template <typename> class F>
using Transform_t = typename Transform<Tuple, F>::type;

// === Filter: keep types satisfying a predicate ===
template <typename Tuple, template <typename> class Pred>
struct Filter;

template <template <typename> class Pred>
struct Filter<std::tuple<>, Pred> {
    using type = std::tuple<>;
};

template <typename H, typename... Ts, template <typename> class Pred>
struct Filter<std::tuple<H, Ts...>, Pred> {
    using rest = typename Filter<std::tuple<Ts...>, Pred>::type;
    using type = std::conditional_t<
        Pred<H>::value,
        decltype(std::tuple_cat(std::declval<std::tuple<H>>(), std::declval<rest>())),
        rest
    >;
};

template <typename Tuple, template <typename> class Pred>
using Filter_t = typename Filter<Tuple, Pred>::type;

// === Reverse ===
template <typename Tuple>
struct Reverse;

template <>
struct Reverse<std::tuple<>> {
    using type = std::tuple<>;
};

template <typename H, typename... Ts>
struct Reverse<std::tuple<H, Ts...>> {
    using reversed_tail = typename Reverse<std::tuple<Ts...>>::type;
    using type = decltype(std::tuple_cat(
        std::declval<reversed_tail>(), std::declval<std::tuple<H>>()));
};

template <typename Tuple>
using Reverse_t = typename Reverse<Tuple>::type;

int main() {
    using List = std::tuple<int, char, double, float>;

    std::cout << "=== Transform: add_pointer ===\n";
    using Pointers = Transform_t<List, std::add_pointer>;
    static_assert(std::is_same_v<Pointers,
        std::tuple<int*, char*, double*, float*>>);
    std::cout << "tuple<int,char,double,float> -> tuple<int*,char*,double*,float*>\n";

    std::cout << "\n=== Transform: add_const ===\n";
    using Consts = Transform_t<List, std::add_const>;
    static_assert(std::is_same_v<Consts,
        std::tuple<const int, const char, const double, const float>>);
    std::cout << "-> tuple<const int, const char, const double, const float>\n";

    std::cout << "\n=== Filter: keep integral types ===\n";
    using Integrals = Filter_t<List, std::is_integral>;
    static_assert(std::is_same_v<Integrals, std::tuple<int, char>>);
    std::cout << "Filter<is_integral> -> tuple<int, char>\n";

    std::cout << "\n=== Filter: keep floating point ===\n";
    using Floats = Filter_t<List, std::is_floating_point>;
    static_assert(std::is_same_v<Floats, std::tuple<double, float>>);
    std::cout << "Filter<is_floating_point> -> tuple<double, float>\n";

    std::cout << "\n=== Reverse ===\n";
    using Rev = Reverse_t<List>;
    static_assert(std::is_same_v<Rev, std::tuple<float, double, char, int>>);
    std::cout << "Reverse -> tuple<float, double, char, int>\n";

    std::cout << "\nAll compile-time operations verified with static_assert!\n";

    return 0;
}
```

### Q3: Show how this replaces hand-rolled recursive `TypeList` structs with standard library primitives

Before variadic templates and `std::tuple`, people built type lists using a recursive struct pattern that looked like a LISP linked list. The modern approach is much flatter.

```cpp
#include <iostream>
#include <tuple>
#include <type_traits>
#include <string>

// === OLD: Hand-rolled TypeList (pre-C++11 style) ===
// struct Nil {};

// template <typename H, typename T = Nil>
// struct TypeList {
//     using head = H;
//     using tail = T;
// };

// using MyList = TypeList<int, TypeList<double, TypeList<char, Nil>>>;
// // Accessing types requires recursive template specializations:
// template <typename TL> struct Length;
// template <> struct Length<Nil> { static constexpr int value = 0; };
// template <typename H, typename T>
// struct Length<TypeList<H,T>> { static constexpr int value = 1 + Length<T>::value; };

// === NEW: std::tuple as TypeList (modern C++) ===
using MyList = std::tuple<int, double, char>;

// Size: built-in!
constexpr auto list_size = std::tuple_size_v<MyList>;  // 3

// Access Nth type: built-in!
using Second = std::tuple_element_t<1, MyList>;  // double

// Head:
template <typename Tuple>
using Head_t = std::tuple_element_t<0, Tuple>;

// Tail:
template <typename Tuple>
struct Tail;
template <typename H, typename... Ts>
struct Tail<std::tuple<H, Ts...>> { using type = std::tuple<Ts...>; };
template <typename Tuple>
using Tail_t = typename Tail<Tuple>::type;

// Contains: check if type is in list
template <typename Tuple, typename T>
struct Contains;
template <typename T>
struct Contains<std::tuple<>, T> : std::false_type {};
template <typename T, typename... Ts>
struct Contains<std::tuple<T, Ts...>, T> : std::true_type {};
template <typename H, typename... Ts, typename T>
struct Contains<std::tuple<H, Ts...>, T> : Contains<std::tuple<Ts...>, T> {};

// IndexOf: find position of type
template <typename Tuple, typename T, std::size_t I = 0>
struct IndexOf;
template <typename T, std::size_t I>
struct IndexOf<std::tuple<>, T, I> { static constexpr std::size_t value = static_cast<std::size_t>(-1); };
template <typename T, typename... Ts, std::size_t I>
struct IndexOf<std::tuple<T, Ts...>, T, I> { static constexpr std::size_t value = I; };
template <typename H, typename... Ts, typename T, std::size_t I>
struct IndexOf<std::tuple<H, Ts...>, T, I> : IndexOf<std::tuple<Ts...>, T, I + 1> {};

// Unique: remove duplicate types
template <typename Tuple, typename Seen = std::tuple<>>
struct Unique;
template <typename Seen>
struct Unique<std::tuple<>, Seen> { using type = Seen; };
template <typename H, typename... Ts, typename... Seen>
struct Unique<std::tuple<H, Ts...>, std::tuple<Seen...>> {
    using type = std::conditional_t<
        Contains<std::tuple<Seen...>, H>::value,
        typename Unique<std::tuple<Ts...>, std::tuple<Seen...>>::type,
        typename Unique<std::tuple<Ts...>, std::tuple<Seen..., H>>::type
    >;
};
template <typename Tuple>
using Unique_t = typename Unique<Tuple>::type;

int main() {
    std::cout << "=== std::tuple as TypeList ===\n";

    std::cout << "Size: " << list_size << "\n";  // 3
    std::cout << "Second type is double: "
              << std::is_same_v<Second, double> << "\n";  // 1

    std::cout << "\n=== Head and Tail ===\n";
    static_assert(std::is_same_v<Head_t<MyList>, int>);
    static_assert(std::is_same_v<Tail_t<MyList>, std::tuple<double, char>>);
    std::cout << "Head = int, Tail = tuple<double, char>\n";

    std::cout << "\n=== Contains ===\n";
    std::cout << "Contains<double>: " << Contains<MyList, double>::value << "\n";  // 1
    std::cout << "Contains<float>: " << Contains<MyList, float>::value << "\n";    // 0

    std::cout << "\n=== IndexOf ===\n";
    std::cout << "IndexOf<int>: " << IndexOf<MyList, int>::value << "\n";       // 0
    std::cout << "IndexOf<char>: " << IndexOf<MyList, char>::value << "\n";     // 2

    std::cout << "\n=== Unique ===\n";
    using Dupes = std::tuple<int, double, int, char, double, char>;
    using NoDupes = Unique_t<Dupes>;
    static_assert(std::is_same_v<NoDupes, std::tuple<int, double, char>>);
    std::cout << "Unique<int,double,int,char,double,char> -> tuple<int,double,char>\n";

    std::cout << "\n=== Advantage over hand-rolled TypeList ===\n";
    std::cout << "  No need for Nil sentinel\n";
    std::cout << "  tuple_size_v and tuple_element_t are built-in\n";
    std::cout << "  tuple_cat for concatenation\n";
    std::cout << "  Familiar to anyone who knows std::tuple\n";

    return 0;
}
```

The advantage is not just less code - it is interoperability. Any library that already knows how to iterate over `std::tuple` members can work with your type list without any adaptation layer.

---

## Notes

- `std::tuple<Ts...>` serves as a compile-time type list with built-in `tuple_size` and `tuple_element`.
- **Head** = `tuple_element_t<0, Tuple>`, **Tail** requires a simple partial specialization.
- **Transform**, **Filter**, **Reverse**, **Contains**, **Unique** - all implementable as template metafunctions.
- Replaces old-style recursive `TypeList<H, TypeList<...>>` with variadic pack-based flat lists.
- Downside: `tuple` creates an actual type with storage layout - for pure type-level work, a lightweight `TypeList<Ts...>` struct is cheaper to compile.
- For production metaprogramming, consider Boost.Mp11 or Boost.Hana.
