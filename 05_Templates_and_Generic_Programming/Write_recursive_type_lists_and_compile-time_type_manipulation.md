# Write Recursive Type Lists and Compile-Time Type Manipulation

**Category:** Templates and Generic Programming  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/variadic_template>  

---

## Topic Overview

### Recursive Type Lists

A **recursive type list** uses partial specialization and recursion to implement operations on compile-time type sequences. Each operation peels off the first type, processes it, and recurses on the rest.

The reason this matters is that not every type list operation can be expressed as a single pack expansion. If what you compute for element N depends on what you already decided for elements 0 through N-1 (as in `unique` or `flatten`), you need recursion - there is no way to express that dependency in a one-shot expansion.

```cpp
type_list<int, double, char>
         │
    Head: int
    Tail: type_list<double, char>
                    │
               Head: double
               Tail: type_list<char>
                              │
                         Head: char
                         Tail: type_list<>  <- base case
```

### Pattern: Recursive Metafunction

Every recursive type list metafunction follows the same shape: a primary template, a base case for the empty list, and a recursive case that peels off the head and processes the tail.

```cpp
// General pattern for recursive type list operations:
template <typename List>
struct Operation;                           // primary (declaration)

template <>
struct Operation<type_list<>> { /* base case */ };

template <typename Head, typename... Tail>
struct Operation<type_list<Head, Tail...>> {
    // Process Head, recurse on type_list<Tail...>
};
```

### Key Operations

| Operation | Description | Recursion Pattern |
| --- | --- | --- |
| `contains` | Check if type exists | Compare head, recurse on tail |
| `transform` | Apply metafunction to each | Transform head, prepend to transformed tail |
| `flatten` | Unwrap nested lists | If head is a list, concat it with flattened tail |
| `unique` | Remove duplicates | Keep head if not in tail, recurse |
| `reverse` | Reverse the list | Reverse tail, append head |

---

## Self-Assessment

### Q1: Implement `type_list<Ts...>` and a `contains<T, List>` meta-function using partial specialization

`contains` is a good starter because it has a clean three-case structure: empty list (false), head matches (true), head doesn't match (recurse). The `unique` metafunction at the end of this snippet is harder - it needs to accumulate a "seen so far" set, which is why it carries a `Seen` parameter that starts as an empty list and grows as types are accepted.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === Type list ===
template <typename... Ts>
struct type_list {};

// === contains: is T in the type_list? ===
template <typename T, typename List>
struct contains;

// Base case: empty list -> false
template <typename T>
struct contains<T, type_list<>> : std::false_type {};

// Found: head matches T -> true
template <typename T, typename... Tail>
struct contains<T, type_list<T, Tail...>> : std::true_type {};

// Not found yet: head doesn't match -> recurse on tail
template <typename T, typename Head, typename... Tail>
struct contains<T, type_list<Head, Tail...>> : contains<T, type_list<Tail...>> {};

template <typename T, typename List>
inline constexpr bool contains_v = contains<T, List>::value;

// === size ===
template <typename List>
struct size;
template <typename... Ts>
struct size<type_list<Ts...>> : std::integral_constant<std::size_t, sizeof...(Ts)> {};
template <typename List>
inline constexpr std::size_t size_v = size<List>::value;

// === index_of: find position of T in list ===
template <typename T, typename List, std::size_t I = 0>
struct index_of;

template <typename T, std::size_t I>
struct index_of<T, type_list<>, I> : std::integral_constant<std::size_t, static_cast<std::size_t>(-1)> {};

template <typename T, typename... Tail, std::size_t I>
struct index_of<T, type_list<T, Tail...>, I> : std::integral_constant<std::size_t, I> {};

template <typename T, typename Head, typename... Tail, std::size_t I>
struct index_of<T, type_list<Head, Tail...>, I> : index_of<T, type_list<Tail...>, I + 1> {};

template <typename T, typename List>
inline constexpr std::size_t index_of_v = index_of<T, List>::value;

// === unique: remove duplicates ===
template <typename List, typename Seen = type_list<>>
struct unique;

template <typename Seen>
struct unique<type_list<>, Seen> {
    using type = Seen;
};

// Helper: append T to type_list
template <typename List, typename T>
struct append;
template <typename... Ts, typename T>
struct append<type_list<Ts...>, T> {
    using type = type_list<Ts..., T>;
};

template <typename Head, typename... Tail, typename Seen>
struct unique<type_list<Head, Tail...>, Seen> {
    using type = std::conditional_t<
        contains_v<Head, Seen>,
        typename unique<type_list<Tail...>, Seen>::type,
        typename unique<type_list<Tail...>, typename append<Seen, Head>::type>::type
    >;
};

template <typename List>
using unique_t = typename unique<List>::type;

int main() {
    using List = type_list<int, double, char, int, double, float>;

    // contains
    std::cout << std::boolalpha;
    std::cout << "contains int:    " << contains_v<int, List> << "\n";     // true
    std::cout << "contains float:  " << contains_v<float, List> << "\n";   // true
    std::cout << "contains string: " << contains_v<std::string, List> << "\n";  // false

    // size
    std::cout << "size: " << size_v<List> << "\n";  // 6

    // index_of
    std::cout << "index_of char: " << index_of_v<char, List> << "\n";  // 2

    // unique
    using Unique = unique_t<List>;
    std::cout << "unique size: " << size_v<Unique> << "\n";  // 4: int, double, char, float
    static_assert(std::is_same_v<Unique, type_list<int, double, char, float>>);
    std::cout << "unique verified\n";

    return 0;
}
```

**Expected output:**

```text
contains int:    true
contains float:  true
contains string: false
size: 6
index_of char: 2
unique size: 4
unique verified
```

### Q2: Write a `transform_list<List, F>` that applies a meta-function `F` to each type in the list

Here is a good illustration of when you don't need recursion. The simple `transform_simple` version uses a single pack expansion `typename F<Ts>::type...` to transform every type in one shot. The recursive version shown first is instructive but unnecessary - it's included so you can see both styles side by side.

```cpp
#include <iostream>
#include <type_traits>
#include <cstddef>
#include <utility>
#include <vector>

template <typename... Ts>
struct type_list {};

template <typename List>
struct size;
template <typename... Ts>
struct size<type_list<Ts...>> : std::integral_constant<std::size_t, sizeof...(Ts)> {};
template <typename List>
inline constexpr std::size_t size_v = size<List>::value;

// === transform_list: apply metafunction F to each type ===
// F is a template template parameter: F<T>::type is the transformed type
template <typename List, template <typename> class F>
struct transform_list;

template <template <typename> class F>
struct transform_list<type_list<>, F> {
    using type = type_list<>;
};

template <typename Head, typename... Tail, template <typename> class F>
struct transform_list<type_list<Head, Tail...>, F> {
    using transformed_head = typename F<Head>::type;
    using transformed_tail = typename transform_list<type_list<Tail...>, F>::type;

    // Prepend transformed head to transformed tail
    using type = decltype([]<typename... Ts>(type_list<Ts...>) {
        return type_list<transformed_head, Ts...>{};
    }(std::declval<transformed_tail>()));
};

// Simpler approach with pack expansion (no recursion needed!):
template <typename List, template <typename> class F>
struct transform_simple;

template <typename... Ts, template <typename> class F>
struct transform_simple<type_list<Ts...>, F> {
    using type = type_list<typename F<Ts>::type...>;  // pack expansion!
};

template <typename List, template <typename> class F>
using transform_t = typename transform_simple<List, F>::type;

// === Custom metafunctions ===
template <typename T>
struct make_unsigned_if_integral {
    using type = std::conditional_t<
        std::is_integral_v<T>,
        std::make_unsigned_t<T>,
        T
    >;
};

template <typename T>
struct wrap_in_vector {
    using type = std::vector<T>;
};

int main() {
    using Input = type_list<int, double, char, float, long>;

    // add_pointer: int -> int*, double -> double*, etc.
    using Pointers = transform_t<Input, std::add_pointer>;
    static_assert(std::is_same_v<Pointers,
        type_list<int*, double*, char*, float*, long*>>);
    std::cout << "add_pointer: verified (size " << size_v<Pointers> << ")\n";

    // add_const: int -> const int, etc.
    using Consts = transform_t<Input, std::add_const>;
    static_assert(std::is_same_v<Consts,
        type_list<const int, const double, const char, const float, const long>>);
    std::cout << "add_const: verified\n";

    // add_lvalue_reference
    using Refs = transform_t<Input, std::add_lvalue_reference>;
    static_assert(std::is_same_v<Refs,
        type_list<int&, double&, char&, float&, long&>>);
    std::cout << "add_lvalue_reference: verified\n";

    // Custom: make integral types unsigned
    using Input2 = type_list<int, double, short, float>;
    using Unsigned = transform_t<Input2, make_unsigned_if_integral>;
    static_assert(std::is_same_v<Unsigned,
        type_list<unsigned int, double, unsigned short, float>>);
    std::cout << "make_unsigned_if_integral: verified\n";

    return 0;
}
```

### Q3: Implement `flatten_list` that recursively unwraps nested `type_list`s into a single flat list

`flatten` is the most complex of the three exercises because the recursion has two directions: into the head (if it is itself a list) and through the tail. The key is the two specializations for the recursive case - one matches a head that is a `type_list`, and one matches a plain type. Both recurse on the tail, but the first also recurses into the nested list before concatenating.

```cpp
#include <iostream>
#include <type_traits>

template <typename... Ts>
struct type_list {};

template <typename List>
struct size;
template <typename... Ts>
struct size<type_list<Ts...>> : std::integral_constant<std::size_t, sizeof...(Ts)> {};
template <typename List>
inline constexpr std::size_t size_v = size<List>::value;

// === concat: merge two type_lists ===
template <typename L1, typename L2>
struct concat;
template <typename... As, typename... Bs>
struct concat<type_list<As...>, type_list<Bs...>> {
    using type = type_list<As..., Bs...>;
};
template <typename L1, typename L2>
using concat_t = typename concat<L1, L2>::type;

// === is_type_list: detect if T is a type_list ===
template <typename T>
struct is_type_list : std::false_type {};
template <typename... Ts>
struct is_type_list<type_list<Ts...>> : std::true_type {};
template <typename T>
inline constexpr bool is_type_list_v = is_type_list<T>::value;

// === flatten: recursively unwrap nested type_lists ===
template <typename List>
struct flatten;

// Base case: empty list
template <>
struct flatten<type_list<>> {
    using type = type_list<>;
};

// Head is a type_list -> flatten it, then concat with flattened tail
template <typename... Inner, typename... Tail>
struct flatten<type_list<type_list<Inner...>, Tail...>> {
    using flattened_head = typename flatten<type_list<Inner...>>::type;  // recurse into nested list
    using flattened_tail = typename flatten<type_list<Tail...>>::type;
    using type = concat_t<flattened_head, flattened_tail>;
};

// Head is NOT a type_list -> keep it, flatten tail
template <typename Head, typename... Tail>
struct flatten<type_list<Head, Tail...>> {
    using flattened_tail = typename flatten<type_list<Tail...>>::type;
    using type = concat_t<type_list<Head>, flattened_tail>;
};

template <typename List>
using flatten_t = typename flatten<List>::type;

int main() {
    // Simple case: no nesting
    using L1 = type_list<int, double, char>;
    using F1 = flatten_t<L1>;
    static_assert(std::is_same_v<F1, type_list<int, double, char>>);
    std::cout << "Flat list stays flat: size = " << size_v<F1> << "\n";

    // One level nesting
    using L2 = type_list<int, type_list<double, char>, float>;
    using F2 = flatten_t<L2>;
    static_assert(std::is_same_v<F2, type_list<int, double, char, float>>);
    std::cout << "One level nested: size = " << size_v<F2> << "\n";  // 4

    // Two levels nesting
    using L3 = type_list<
        int,
        type_list<double, type_list<char, short>>,
        float
    >;
    using F3 = flatten_t<L3>;
    static_assert(std::is_same_v<F3, type_list<int, double, char, short, float>>);
    std::cout << "Two levels nested: size = " << size_v<F3> << "\n";  // 5

    // Multiple nested lists
    using L4 = type_list<
        type_list<int, double>,
        type_list<char>,
        float,
        type_list<type_list<long, short>>
    >;
    using F4 = flatten_t<L4>;
    static_assert(std::is_same_v<F4, type_list<int, double, char, float, long, short>>);
    std::cout << "Complex nesting: size = " << size_v<F4> << "\n";  // 6

    // Empty nested lists
    using L5 = type_list<int, type_list<>, double>;
    using F5 = flatten_t<L5>;
    static_assert(std::is_same_v<F5, type_list<int, double>>);
    std::cout << "Empty nested list removed: size = " << size_v<F5> << "\n";  // 2

    std::cout << "\nAll flatten operations verified\n";

    return 0;
}
```

**Expected output:**

```text
Flat list stays flat: size = 3
One level nested: size = 4
Two levels nested: size = 5
Complex nesting: size = 6
Empty nested list removed: size = 2

All flatten operations verified
```

---

## Notes

- Recursive type list manipulation is the foundation of TMP (Template MetaProgramming).
- **Pack expansion** (`F<Ts>::type...`) is often simpler than explicit recursion - prefer it when possible.
- Recursive approaches are needed when the operation depends on previously processed elements (e.g., `unique`, `flatten`).
- Partial specialization is the mechanism that enables "pattern matching" on template arguments.
- **Template depth limits** can be hit with very deep nesting - GCC default is about 900.
- Libraries like Boost.Mp11 and Boost.Hana provide optimized type list operations.
- In C++26, static reflection may replace some TMP patterns with simpler code.
