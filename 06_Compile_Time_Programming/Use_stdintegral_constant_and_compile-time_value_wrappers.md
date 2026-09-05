# Use `std::integral_constant` and Compile-Time Value Wrappers

**Category:** Compile Time Programming  
**Standard:** C++11 (enhanced in C++14/C++17)  
**Reference:** <https://en.cppreference.com/w/cpp/types/integral_constant>  

---

## Topic Overview

### What Is `std::integral_constant`

Here is a question that feels odd at first: why would you want to turn a value into a type? In C++, template metaprogramming works entirely with types, not values. If you want to pass a number through a chain of templates, you have to wrap it in a type first. That is exactly what `std::integral_constant` does - it is the simplest possible wrapper that lifts a compile-time value into the type system.

```cpp
template<class T, T v>
struct integral_constant {
    static constexpr T value = v;
    using value_type = T;
    using type = integral_constant;
    constexpr operator value_type() const noexcept { return value; }
    constexpr value_type operator()() const noexcept { return value; } // C++14
};
```

Once a value lives inside a type, the compiler can use it for template specialization, overload resolution, and SFINAE - none of which work with plain values. The reason this matters in practice is that every type trait in `<type_traits>` is built on this exact foundation. When you ask `std::is_integral<int>::value`, you are reading a `static constexpr bool` that lives inside a class that inherits from `integral_constant<bool, true>`. The whole system rests on this one building block.

### Key Type Aliases

Two aliases are so common they get their own names in the standard library:

| Alias | Definition |
| --- | --- |
| `std::true_type` | `std::integral_constant<bool, true>` |
| `std::false_type` | `std::integral_constant<bool, false>` |
| `std::bool_constant<B>` | `std::integral_constant<bool, B>` (C++17) |

Every type trait you have ever used - `std::is_integral<T>`, `std::is_pointer<T>`, and so on - inherits from one of these two aliases, which is why you can check `::value` on all of them.

### Why Types Instead of Values

Using types to represent values enables capabilities that would otherwise require special language support. If the table feels abstract, think of it this way: you cannot overload a function on the value of an argument at compile time, but you can overload it on the type of an argument. `integral_constant` is the bridge.

| Capability | Example |
| --- | --- |
| Tag dispatch | `foo(std::true_type{})` vs `foo(std::false_type{})` |
| Trait composition | `std::is_integral<T>` inherits from `true_type` or `false_type` |
| Type-level computation | Recursive template metaprogramming with types |
| SFINAE / concepts | `enable_if_t<is_trivial_v<T>>` uses inherited `::value` |

### How Standard Traits Use It

Every predicate in `<type_traits>` follows the same pattern: specialize on the types you care about, inherit from `true_type` or `false_type` accordingly, and the `::value` member comes along for free. Here is the simplified definition of `is_pointer`:

```cpp
// Standard library definition (simplified)
template<class T> struct is_pointer     : std::false_type {};
template<class T> struct is_pointer<T*> : std::true_type  {};

static_assert(std::is_pointer<int*>::value);   // true
static_assert(!std::is_pointer<int>::value);   // false
```

The primary template says "no, this is not a pointer" by inheriting from `false_type`. The partial specialization for `T*` says "yes, this one is" by inheriting from `true_type`. Simple, but it scales to arbitrarily complex traits.

---

## Self-Assessment

### Q1: Explain how `std::true_type` and `std::false_type` are aliases for `integral_constant<bool, ...>`

The most practical thing you can do with `true_type` and `false_type` is tag dispatch - passing them as dummy arguments to select between overloaded implementations at compile time, without any `if` statements or virtual calls. The idea is that the overload resolution itself does the branching for you, and the compiler eliminates the unchosen branch entirely.

```cpp
#include <iostream>
#include <type_traits>

// === true_type and false_type are just integral_constant<bool, ...> ===

// Verify at compile time
static_assert(std::is_same_v<std::true_type, std::integral_constant<bool, true>>);
static_assert(std::is_same_v<std::false_type, std::integral_constant<bool, false>>);

// They provide ::value, implicit conversion, and operator()
static_assert(std::true_type::value == true);
static_assert(std::false_type::value == false);

// === Tag Dispatch Example ===
// Different algorithms for trivially copyable vs non-trivially copyable types

template<typename T>
void copy_impl(const T* src, T* dst, std::size_t n, std::true_type /*trivially_copyable*/) {
    std::cout << "  -> Using memcpy (trivially copyable)\n";
    std::memcpy(dst, src, n * sizeof(T));
}

template<typename T>
void copy_impl(const T* src, T* dst, std::size_t n, std::false_type /*not trivially_copyable*/) {
    std::cout << "  -> Using element-wise copy (non-trivial)\n";
    for (std::size_t i = 0; i < n; ++i)
        dst[i] = src[i];
}

template<typename T>
void smart_copy(const T* src, T* dst, std::size_t n) {
    // Tag dispatch: the trait inherits from true_type or false_type
    copy_impl(src, dst, n, std::is_trivially_copyable<T>{});
}

// === Custom Trait Using integral_constant ===

template<typename T>
struct is_string_like : std::false_type {};

template<>
struct is_string_like<std::string> : std::true_type {};

template<>
struct is_string_like<const char*> : std::true_type {};

template<std::size_t N>
struct is_string_like<char[N]> : std::true_type {};

static_assert(is_string_like<std::string>::value);
static_assert(is_string_like<const char*>::value);
static_assert(!is_string_like<int>::value);

int main() {
    // === Implicit conversion operator ===
    std::cout << "true_type converts to: " << std::true_type{} << "\n";   // 1
    std::cout << "false_type converts to: " << std::false_type{} << "\n"; // 0

    // === operator() (C++14) ===
    constexpr auto val = std::true_type{}();
    std::cout << "true_type() returns: " << val << "\n";  // 1

    // === Tag dispatch in action ===
    int src[] = {1, 2, 3};
    int dst[3];
    std::cout << "\nCopying int[3]:\n";
    smart_copy(src, dst, 3);

    std::string s_src[] = {"hello", "world"};
    std::string s_dst[2];
    std::cout << "Copying string[2]:\n";
    smart_copy(s_src, s_dst, 2);

    // === bool_constant (C++17) ===
    using is_64bit = std::bool_constant<sizeof(void*) == 8>;
    std::cout << "\n64-bit platform: " << is_64bit::value << "\n";

    return 0;
}
```

Notice how `smart_copy` passes `std::is_trivially_copyable<T>{}` as the last argument. That is a temporary object whose type is either `true_type` or `false_type`, and the compiler picks the right overload of `copy_impl` at compile time. No runtime branch, no virtual dispatch - just overload resolution doing what it always does, except the "argument" is a zero-size tag type.

**Expected output (64-bit, trivially-copyable string impl may vary):**

```text
true_type converts to: 1
false_type converts to: 0
true_type() returns: 1

Copying int[3]:
  -> Using memcpy (trivially copyable)
Copying string[2]:
  -> Using element-wise copy (non-trivial)

64-bit platform: 1
```

### Q2: Write a type list with variadic templates and extract the Nth type

Type lists are the building blocks of more advanced metaprogramming. The idea is to store a sequence of types in a template parameter pack, then write operations like `type_at<N>` that peel the list recursively. `std::integral_constant` shows up here as the base class for index computations - inheriting from it gives your struct a `::value` member automatically, which keeps your API consistent with the rest of the standard library.

The reason `type_at` has to be recursive is that C++ does not let you index into a parameter pack with a runtime value. You have to peel off the head one step at a time until you reach index zero. Each recursive instantiation reduces `N` by one and drops the front type from the list.

```cpp
#include <iostream>
#include <type_traits>
#include <cstddef>
#include <string>

// === Type List ===
template<typename... Ts>
struct type_list {
    static constexpr std::size_t size = sizeof...(Ts);
};

// === Extract Nth type (recursive) ===
template<std::size_t N, typename List>
struct type_at;

template<std::size_t N, typename Head, typename... Tail>
struct type_at<N, type_list<Head, Tail...>>
    : type_at<N - 1, type_list<Tail...>> {};

template<typename Head, typename... Tail>
struct type_at<0, type_list<Head, Tail...>> {
    using type = Head;
};

template<std::size_t N, typename List>
using type_at_t = typename type_at<N, List>::type;

// === Contains check ===
template<typename T, typename List>
struct contains;

template<typename T>
struct contains<T, type_list<>> : std::false_type {};

template<typename T, typename Head, typename... Tail>
struct contains<T, type_list<Head, Tail...>>
    : std::bool_constant<std::is_same_v<T, Head> || contains<T, type_list<Tail...>>::value> {};

// === Index of type ===
template<typename T, typename List>
struct index_of;

template<typename T, typename Head, typename... Tail>
struct index_of<T, type_list<Head, Tail...>>
    : std::integral_constant<std::size_t,
          std::is_same_v<T, Head> ? 0 : 1 + index_of<T, type_list<Tail...>>::value> {};

// Compile-time tests
using my_types = type_list<int, double, char, std::string>;

static_assert(my_types::size == 4);
static_assert(std::is_same_v<type_at_t<0, my_types>, int>);
static_assert(std::is_same_v<type_at_t<1, my_types>, double>);
static_assert(std::is_same_v<type_at_t<2, my_types>, char>);
static_assert(std::is_same_v<type_at_t<3, my_types>, std::string>);

static_assert(contains<int, my_types>::value);
static_assert(contains<double, my_types>::value);
static_assert(!contains<float, my_types>::value);

static_assert(index_of<int, my_types>::value == 0);
static_assert(index_of<char, my_types>::value == 2);

int main() {
    std::cout << "=== Type List Operations ===\n";
    std::cout << "List size: " << my_types::size << "\n";

    std::cout << "\ntype_at<0> = " << typeid(type_at_t<0, my_types>).name() << "\n";
    std::cout << "type_at<1> = " << typeid(type_at_t<1, my_types>).name() << "\n";
    std::cout << "type_at<2> = " << typeid(type_at_t<2, my_types>).name() << "\n";
    std::cout << "type_at<3> = " << typeid(type_at_t<3, my_types>).name() << "\n";

    std::cout << "\ncontains<int>:    " << contains<int, my_types>::value << "\n";
    std::cout << "contains<float>:  " << contains<float, my_types>::value << "\n";

    std::cout << "\nindex_of<int>:    " << index_of<int, my_types>::value << "\n";
    std::cout << "index_of<char>:   " << index_of<char, my_types>::value << "\n";

    // === Using integral_constant for recursive computation ===
    // Count types matching a predicate
    std::cout << "\n=== How integral_constant enables recursion ===\n";
    std::cout << "Each recursive step returns a TYPE (integral_constant<size_t, N>)\n";
    std::cout << "The compiler resolves the chain at compile time.\n";

    return 0;
}
```

The reason `index_of` uses `std::integral_constant<std::size_t, ...>` as its base class is that it gives the struct a `::value` member automatically, following the same convention as every other type trait. You could have stored the value in a plain `static constexpr` member instead, but inheriting from `integral_constant` keeps your trait consistent with the rest of the standard library, which means it will work anywhere a standard trait is expected.

**Expected output (type names are compiler-dependent):**

```text
=== Type List Operations ===
List size: 4

type_at<0> = i
type_at<1> = d
type_at<2> = c
type_at<3> = NSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE

contains<int>:    1
contains<float>:  0

index_of<int>:    0
index_of<char>:   2

=== How integral_constant enables recursion ===
Each recursive step returns a TYPE (integral_constant<size_t, N>)
The compiler resolves the chain at compile time.
```

### Q3: Build a compile-time map from types to integer values using template specializations

You can use `integral_constant` specializations as a zero-overhead compile-time lookup table that maps types to integer IDs. This pattern comes up in serialization, type erasure, and any system that needs to assign unique tags to types at compile time. The idea is simple: define the primary template without a body (so using an unregistered type produces a clean error), then add a full specialization for each type you want to support.

The `has_type_id` SFINAE check at the end is worth paying attention to - it uses `std::void_t` to detect whether `type_id<T>::value` exists. If the primary `type_id<T>` template is undefined, that expression is ill-formed and SFINAE kicks in, making `has_type_id<T>` inherit from `false_type`. This lets you write a `static_assert` inside `serialize` that catches unregistered types at compile time rather than producing a confusing linker error.

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// === Compile-Time Type -> Value Map ===
// Uses integral_constant to associate each type with a unique integer ID

// Primary template: no mapping (will cause compile error if used for unmapped type)
template<typename T>
struct type_id; // intentionally undefined - SFINAE-friendly

// Specializations: map types to integer IDs
template<> struct type_id<int>         : std::integral_constant<int, 1> {};
template<> struct type_id<double>      : std::integral_constant<int, 2> {};
template<> struct type_id<char>        : std::integral_constant<int, 3> {};
template<> struct type_id<std::string> : std::integral_constant<int, 4> {};
template<> struct type_id<float>       : std::integral_constant<int, 5> {};
template<> struct type_id<bool>        : std::integral_constant<int, 6> {};

// Convenient alias
template<typename T>
inline constexpr int type_id_v = type_id<T>::value;

// Compile-time verification
static_assert(type_id_v<int> == 1);
static_assert(type_id_v<double> == 2);
static_assert(type_id_v<std::string> == 4);

// === Type -> String Name Map (using pointer constant) ===
template<typename T>
struct type_name;

template<> struct type_name<int>    { static constexpr const char* value = "int"; };
template<> struct type_name<double> { static constexpr const char* value = "double"; };
template<> struct type_name<char>   { static constexpr const char* value = "char"; };
template<> struct type_name<float>  { static constexpr const char* value = "float"; };
template<> struct type_name<bool>   { static constexpr const char* value = "bool"; };

// === SFINAE check: does a type have a mapping? ===
template<typename T, typename = void>
struct has_type_id : std::false_type {};

template<typename T>
struct has_type_id<T, std::void_t<decltype(type_id<T>::value)>> : std::true_type {};

static_assert(has_type_id<int>::value);
static_assert(has_type_id<double>::value);
static_assert(!has_type_id<long long>::value);  // not mapped

// === Practical: Serialization Tag ===
template<typename T>
void serialize(const T& val) {
    static_assert(has_type_id<T>::value, "Type must be registered in type_id map");
    // Write type tag then value
    std::cout << "[tag=" << type_id_v<T> << "] ";
    if constexpr (std::is_same_v<T, std::string>)
        std::cout << "\"" << val << "\"\n";
    else
        std::cout << val << "\n";
}

int main() {
    std::cout << "=== Compile-Time Type -> ID Map ===\n";
    std::cout << "int    -> " << type_id_v<int> << "\n";
    std::cout << "double -> " << type_id_v<double> << "\n";
    std::cout << "char   -> " << type_id_v<char> << "\n";
    std::cout << "string -> " << type_id_v<std::string> << "\n";
    std::cout << "float  -> " << type_id_v<float> << "\n";
    std::cout << "bool   -> " << type_id_v<bool> << "\n";

    std::cout << "\n=== SFINAE: has_type_id ===\n";
    std::cout << "int:       " << has_type_id<int>::value << "\n";
    std::cout << "long long: " << has_type_id<long long>::value << "\n";

    std::cout << "\n=== Practical: Tagged Serialization ===\n";
    serialize(42);
    serialize(3.14);
    serialize(std::string("hello"));

    // === integral_constant implicit conversion ===
    std::cout << "\n=== Implicit Conversion ===\n";
    // type_id<int>{} converts to int via operator value_type()
    int id = type_id<int>{};
    std::cout << "type_id<int>{} as int: " << id << "\n";

    // Can use in switch
    switch (type_id_v<double>) {
        case 1: std::cout << "int\n"; break;
        case 2: std::cout << "double\n"; break;
        default: std::cout << "other\n"; break;
    }

    return 0;
}
```

The whole system is zero-overhead: `type_id_v<int>` is a `constexpr int` with value 1, resolved entirely at compile time. The `serialize` function is also a template, so the `static_assert` fires during instantiation if you try to serialize a type that has not been registered.

**Expected output:**

```text
=== Compile-Time Type -> ID Map ===
int    -> 1
double -> 2
char   -> 3
string -> 4
float  -> 5
bool   -> 6

=== SFINAE: has_type_id ===
int:       1
long long: 0

=== Practical: Tagged Serialization ===
[tag=1] 42
[tag=2] 3.14
[tag=4] "hello"

=== Implicit Conversion ===
type_id<int>{} as int: 1
double
```

---

## Notes

- `std::integral_constant<T, v>` turns a compile-time value `v` into a type - the basis of all type traits.
- `std::true_type` / `std::false_type` are the boolean specializations used by every `<type_traits>` predicate.
- Tag dispatch uses `true_type{}` / `false_type{}` as function arguments to select overloads without `if constexpr`.
- `std::bool_constant<B>` (C++17) simplifies `integral_constant<bool, B>`.
- The implicit `operator value_type()` lets you use `integral_constant` instances directly in boolean/numeric contexts.
- Custom type-to-value maps via specialization give zero-overhead compile-time lookup tables.
