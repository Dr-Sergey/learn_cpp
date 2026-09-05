# Write a Type List and Implement Compile-Time Operations on It

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/variadic_template>  

---

## Topic Overview

### What Is a Type List

A **type list** is a compile-time data structure that holds a sequence of types. It has no runtime representation - it exists purely in the type system. Think of it as a `std::vector` whose elements are types, not values, and whose entire lifetime is at compile time.

```cpp
template <typename... Ts>
struct TypeList {};

using MyTypes = TypeList<int, double, char, std::string>;
// MyTypes is a type, not a value - it holds 4 types at compile time
```

You never create a `TypeList` object at runtime. The entire point is to manipulate type sequences using template specializations, so that the results appear as new types in your program.

### Common Compile-Time Operations

Here is a quick overview of what you can build on top of a basic type list:

| Operation | Description | Example |
| --- | --- | --- |
| `Size` | Count of types | `Size<TypeList<int,char>>` -> 2 |
| `Head` | First type | `Head<TypeList<int,char>>` -> `int` |
| `Tail` | All except first | `Tail<TypeList<int,char,double>>` -> `TypeList<char,double>` |
| `Append` | Add type at end | `Append<TypeList<int>, char>` -> `TypeList<int,char>` |
| `Prepend` | Add type at front | `Prepend<TypeList<int>, char>` -> `TypeList<char,int>` |
| `Contains` | Check membership | `Contains<TypeList<int,char>, int>` -> `true` |
| `IndexOf` | Position of type | `IndexOf<TypeList<int,char>, char>` -> 1 |
| `Filter` | Keep types matching predicate | `Filter<TypeList<int,double>, is_integral>` -> `TypeList<int>` |
| `Transform` | Apply metafunction to each | `Transform<TypeList<int>, add_pointer>` -> `TypeList<int*>` |

---

## Self-Assessment

### Q1: Define `template<typename...> struct TypeList{};` and implement a `Size` metafunction

The pattern for every type list metafunction is the same: write a primary template (often just declared, not defined), then write one or more partial specializations that match specific shapes of the template argument. For `Size`, the specialization matches `TypeList<Ts...>` and uses `sizeof...(Ts)` to count the pack.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === TypeList: a compile-time list of types ===
template <typename... Ts>
struct TypeList {};

// === Size: count types in a TypeList ===
template <typename List>
struct Size;

template <typename... Ts>
struct Size<TypeList<Ts...>> {
    static constexpr std::size_t value = sizeof...(Ts);
};

template <typename List>
inline constexpr std::size_t Size_v = Size<List>::value;

// === IsEmpty ===
template <typename List>
struct IsEmpty : std::bool_constant<Size_v<List> == 0> {};

template <typename List>
inline constexpr bool IsEmpty_v = IsEmpty<List>::value;

// === Head: get first type ===
template <typename List>
struct Head;

template <typename T, typename... Ts>
struct Head<TypeList<T, Ts...>> {
    using type = T;
};

template <typename List>
using Head_t = typename Head<List>::type;

// === Tail: all except first ===
template <typename List>
struct Tail;

template <typename T, typename... Ts>
struct Tail<TypeList<T, Ts...>> {
    using type = TypeList<Ts...>;
};

template <typename List>
using Tail_t = typename Tail<List>::type;

// === Contains: check if type is in list ===
template <typename List, typename T>
struct Contains;

template <typename T>
struct Contains<TypeList<>, T> : std::false_type {};

template <typename T, typename... Ts>
struct Contains<TypeList<T, Ts...>, T> : std::true_type {};

template <typename U, typename... Ts, typename T>
struct Contains<TypeList<U, Ts...>, T> : Contains<TypeList<Ts...>, T> {};

template <typename List, typename T>
inline constexpr bool Contains_v = Contains<List, T>::value;

int main() {
    using List = TypeList<int, double, char, std::string>;

    std::cout << "Size: " << Size_v<List> << "\n";                    // 4
    std::cout << "Empty: " << IsEmpty_v<List> << "\n";                // 0
    std::cout << "Empty<>: " << IsEmpty_v<TypeList<>> << "\n";        // 1

    // Head
    static_assert(std::is_same_v<Head_t<List>, int>);
    std::cout << "Head is int: true\n";

    // Tail
    static_assert(std::is_same_v<Tail_t<List>, TypeList<double, char, std::string>>);
    std::cout << "Tail is <double,char,string>: true\n";

    // Contains
    std::cout << "Contains int:   " << Contains_v<List, int> << "\n";     // 1
    std::cout << "Contains float: " << Contains_v<List, float> << "\n";   // 0

    return 0;
}
```

Notice that `Contains` uses three specializations to implement what is essentially a linear search over the type list at compile time: match nothing (false), match the head (true), or peel the head and recurse.

**Expected output:**

```text
Size: 4
Empty: 0
Empty<>: 1
Head is int: true
Tail is <double,char,string>: true
Contains int:   1
Contains float: 0
```

### Q2: Implement an `Append<TypeList<A,B>, C>` that returns `TypeList<A,B,C>`

`Append` and `Prepend` are both trivial when you realize that pack expansion lets you just unpack the existing types alongside the new one. The interesting one here is `Concat`, which merges two lists by expanding both packs into a single `TypeList<As..., Bs...>` in one shot - no recursion needed.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

template <typename... Ts>
struct TypeList {};

template <typename List>
struct Size;
template <typename... Ts>
struct Size<TypeList<Ts...>> {
    static constexpr std::size_t value = sizeof...(Ts);
};
template <typename List>
inline constexpr std::size_t Size_v = Size<List>::value;

// === Append: add a type at the end ===
template <typename List, typename T>
struct Append;

template <typename... Ts, typename T>
struct Append<TypeList<Ts...>, T> {
    using type = TypeList<Ts..., T>;
};

template <typename List, typename T>
using Append_t = typename Append<List, T>::type;

// === Prepend: add a type at the front ===
template <typename List, typename T>
struct Prepend;

template <typename... Ts, typename T>
struct Prepend<TypeList<Ts...>, T> {
    using type = TypeList<T, Ts...>;
};

template <typename List, typename T>
using Prepend_t = typename Prepend<List, T>::type;

// === Concat: merge two TypeLists ===
template <typename List1, typename List2>
struct Concat;

template <typename... As, typename... Bs>
struct Concat<TypeList<As...>, TypeList<Bs...>> {
    using type = TypeList<As..., Bs...>;
};

template <typename L1, typename L2>
using Concat_t = typename Concat<L1, L2>::type;

// === IndexOf: find position of type ===
template <typename List, typename T, std::size_t I = 0>
struct IndexOf;

template <typename T, std::size_t I>
struct IndexOf<TypeList<>, T, I> {
    static constexpr std::size_t value = static_cast<std::size_t>(-1); // not found
};

template <typename T, typename... Ts, std::size_t I>
struct IndexOf<TypeList<T, Ts...>, T, I> {
    static constexpr std::size_t value = I;
};

template <typename U, typename... Ts, typename T, std::size_t I>
struct IndexOf<TypeList<U, Ts...>, T, I> : IndexOf<TypeList<Ts...>, T, I + 1> {};

template <typename List, typename T>
inline constexpr std::size_t IndexOf_v = IndexOf<List, T>::value;

int main() {
    using Base = TypeList<int, double>;

    // Append
    using Appended = Append_t<Base, char>;
    static_assert(std::is_same_v<Appended, TypeList<int, double, char>>);
    std::cout << "Append<{int,double}, char> -> size " << Size_v<Appended> << "\n";  // 3

    // Prepend
    using Prepended = Prepend_t<Base, char>;
    static_assert(std::is_same_v<Prepended, TypeList<char, int, double>>);
    std::cout << "Prepend<{int,double}, char> -> size " << Size_v<Prepended> << "\n";  // 3

    // Concat
    using List1 = TypeList<int, double>;
    using List2 = TypeList<char, float>;
    using Merged = Concat_t<List1, List2>;
    static_assert(std::is_same_v<Merged, TypeList<int, double, char, float>>);
    std::cout << "Concat -> size " << Size_v<Merged> << "\n";  // 4

    // IndexOf
    std::cout << "IndexOf double: " << IndexOf_v<Merged, double> << "\n";  // 1
    std::cout << "IndexOf float:  " << IndexOf_v<Merged, float> << "\n";   // 3

    return 0;
}
```

### Q3: Implement `Filter<TypeList<int, double, char>, std::is_integral>` returning `TypeList<int, char>`

`Filter` is where things get a little more involved because you need to decide, for each element, whether to include it in the output. The cleanest approach uses a `PrependHelper` and `std::conditional_t`: if the predicate fires for the head, prepend it to the filtered tail; otherwise just return the filtered tail. `Transform` is simpler - pack expansion handles it in one line.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

template <typename... Ts>
struct TypeList {};

template <typename List>
struct Size;
template <typename... Ts>
struct Size<TypeList<Ts...>> {
    static constexpr std::size_t value = sizeof...(Ts);
};
template <typename List>
inline constexpr std::size_t Size_v = Size<List>::value;

// === Filter: keep only types satisfying Predicate ===
template <typename List, template <typename> class Pred>
struct Filter;

// Base case: empty list
template <template <typename> class Pred>
struct Filter<TypeList<>, Pred> {
    using type = TypeList<>;
};

// Recursive case: T satisfies predicate -> keep it
template <typename T, typename... Ts, template <typename> class Pred>
struct Filter<TypeList<T, Ts...>, Pred> {
    using rest = typename Filter<TypeList<Ts...>, Pred>::type;

    using type = std::conditional_t<
        Pred<T>::value,
        // Prepend T to the filtered rest
        typename decltype([]<typename... Rs>(TypeList<Rs...>) {
            return TypeList<T, Rs...>{};
        }(std::declval<rest>()))::type_identity_helper,   // <- complex, let's use a helper
        rest
    >;
};

// Simpler approach using a Prepend helper:
template <typename List, typename T>
struct PrependHelper;
template <typename... Ts, typename T>
struct PrependHelper<TypeList<Ts...>, T> {
    using type = TypeList<T, Ts...>;
};

// Cleaner Filter using PrependHelper:
template <typename List, template <typename> class Pred>
struct FilterClean;

template <template <typename> class Pred>
struct FilterClean<TypeList<>, Pred> {
    using type = TypeList<>;
};

template <typename T, typename... Ts, template <typename> class Pred>
struct FilterClean<TypeList<T, Ts...>, Pred> {
    using rest = typename FilterClean<TypeList<Ts...>, Pred>::type;
    using type = std::conditional_t<
        Pred<T>::value,
        typename PrependHelper<rest, T>::type,
        rest
    >;
};

template <typename List, template <typename> class Pred>
using Filter_t = typename FilterClean<List, Pred>::type;

// === Transform: apply metafunction to each type ===
template <typename List, template <typename> class F>
struct Transform;

template <template <typename> class F>
struct Transform<TypeList<>, F> {
    using type = TypeList<>;
};

template <typename... Ts, template <typename> class F>
struct Transform<TypeList<Ts...>, F> {
    using type = TypeList<typename F<Ts>::type...>;
};

template <typename List, template <typename> class F>
using Transform_t = typename Transform<List, F>::type;

int main() {
    using Input = TypeList<int, double, char, float, long, std::string>;

    // Filter: keep only integral types
    using Integrals = Filter_t<Input, std::is_integral>;
    static_assert(std::is_same_v<Integrals, TypeList<int, char, long>>);
    std::cout << "Filter<is_integral>: size = " << Size_v<Integrals> << "\n";  // 3

    // Filter: keep only floating-point types
    using Floats = Filter_t<Input, std::is_floating_point>;
    static_assert(std::is_same_v<Floats, TypeList<double, float>>);
    std::cout << "Filter<is_floating_point>: size = " << Size_v<Floats> << "\n";  // 2

    // Transform: add pointer to each type
    using Pointers = Transform_t<TypeList<int, double, char>, std::add_pointer>;
    static_assert(std::is_same_v<Pointers, TypeList<int*, double*, char*>>);
    std::cout << "Transform<add_pointer>: size = " << Size_v<Pointers> << "\n";  // 3

    // Transform: add const to each type
    using Consts = Transform_t<TypeList<int, double>, std::add_const>;
    static_assert(std::is_same_v<Consts, TypeList<const int, const double>>);
    std::cout << "Transform<add_const>: verified\n";

    std::cout << "\nAll compile-time type list operations verified!\n";

    return 0;
}
```

**Expected output:**

```text
Filter<is_integral>: size = 3
Filter<is_floating_point>: size = 2
Transform<add_pointer>: size = 3
Transform<add_const>: verified

All compile-time type list operations verified!
```

---

## Notes

- Type lists are the foundation of template metaprogramming libraries (Boost.MPL, Boost.Hana, mp11).
- All operations happen at compile time - zero runtime cost.
- Type lists + `Filter` + `Transform` give you a functional programming model at the type level.
- In C++17/20, `if constexpr` and concepts can simplify some metaprogramming that previously required type lists.
- `std::tuple` can serve as a type list in practice, since `std::tuple_size` and `std::tuple_element` provide similar operations.
- Watch template instantiation depth - deep recursion can hit compiler limits (typically 256-1024).
