# Understand Class Template Specialization for Traits (Detection Idiom)

**Category:** Templates and Generic Programming  
**Standard:** C++17, C++20 (`void_t`)  
**Reference:** <https://en.cppreference.com/w/cpp/types/void_t>  

---

## Topic Overview

### What Is the Detection Idiom

The **detection idiom** lets you ask at compile time: "does this type have a particular member, method, or nested type?" The answer comes back as a `true_type` or `false_type` that you can branch on with `if constexpr` or use in `enable_if` conditions.

The mechanism underneath is SFINAE with `std::void_t` and partial specialization. The two-piece structure is what makes it tick:

```cpp
Primary template:          -> false (default - trait not detected)
Partial specialization:    -> true  (only valid if expression compiles)
```

If you have never seen this pattern before, the reason it works is subtle: the partial specialization is only a valid candidate when the expression inside `void_t<...>` is well-formed. If the expression fails (say, because `T` has no `.size()`), SFINAE eliminates that specialization and the compiler falls back to the primary template, which inherits from `false_type`.

### Building Blocks

| Component | Purpose | Example |
| --- | --- | --- |
| `std::void_t<Expr...>` | Maps any valid type expression to `void`; triggers SFINAE on failure | `void_t<decltype(T::size())>` |
| Primary template | Default case (false) | `template<class,class=void> struct has_X : false_type {};` |
| Partial specialization | Detected case (true) | `template<class T> struct has_X<T, void_t<...>> : true_type {};` |

### How `void_t` Detection Works

Here is the minimal pattern for detecting whether `T` has a `.size()` method. The reason the second template parameter is `void` by default is so that the partial specialization (which also produces `void` via `void_t`) can match and take over:

```cpp
// Step 1: Primary template - default is false
template <typename T, typename = void>
struct has_size : std::false_type {};

// Step 2: Specialization - true if T.size() is valid
template <typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>>
    : std::true_type {};

// has_size<std::vector<int>>::value == true   (vector has .size())
// has_size<int>::value            == false  (int has no .size())
```

The `std::declval<T>()` is necessary because `T` may not be default-constructible - `declval` gives you a fake reference to `T` purely for use in unevaluated contexts like `decltype`.

### Evolution: void_t -> is_detected -> Concepts

The idiom has gotten progressively cleaner over the years:

| Approach | Era | Verbosity | Readability |
| --- | :---: | :---: | :---: |
| Raw `void_t` + specialization | C++17 | High | Low |
| `is_detected` (TS / custom) | C++17 | Medium | Medium |
| `requires` expression | **C++20** | **Low** | **High** |

---

## Self-Assessment

### Q1: Implement a `has_size<T>` trait using `std::void_t` and partial specialization

This shows the full pattern applied to four different capabilities. Each trait is independent, but they all follow the exact same two-piece structure:

```cpp
#include <iostream>
#include <type_traits>
#include <vector>
#include <string>

// Primary template: default = false
template <typename T, typename = void>
struct has_size : std::false_type {};

// Partial specialization: true if T has a .size() member function
template <typename T>
struct has_size<T, std::void_t<decltype(std::declval<const T&>().size())>>
    : std::true_type {};

// Helper variable template
template <typename T>
constexpr bool has_size_v = has_size<T>::value;

// More traits using the same pattern:

// Detect .begin() and .end()
template <typename T, typename = void>
struct is_iterable : std::false_type {};

template <typename T>
struct is_iterable<T, std::void_t<
    decltype(std::declval<T&>().begin()),
    decltype(std::declval<T&>().end())
>> : std::true_type {};

// Detect nested type ::value_type
template <typename T, typename = void>
struct has_value_type : std::false_type {};

template <typename T>
struct has_value_type<T, std::void_t<typename T::value_type>>
    : std::true_type {};

// Detect operator<<
template <typename T, typename = void>
struct is_printable : std::false_type {};

template <typename T>
struct is_printable<T, std::void_t<
    decltype(std::declval<std::ostream&>() << std::declval<const T&>())
>> : std::true_type {};

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== has_size ===\n";
    std::cout << "vector<int>: " << has_size_v<std::vector<int>> << "\n";  // true
    std::cout << "string:      " << has_size_v<std::string> << "\n";       // true
    std::cout << "int:         " << has_size_v<int> << "\n";               // false
    std::cout << "double:      " << has_size_v<double> << "\n";            // false

    std::cout << "\n=== is_iterable ===\n";
    std::cout << "vector<int>: " << is_iterable<std::vector<int>>::value << "\n";  // true
    std::cout << "int:         " << is_iterable<int>::value << "\n";               // false

    std::cout << "\n=== has_value_type ===\n";
    std::cout << "vector<int>: " << has_value_type<std::vector<int>>::value << "\n";  // true
    std::cout << "int:         " << has_value_type<int>::value << "\n";               // false

    std::cout << "\n=== is_printable ===\n";
    std::cout << "int:         " << is_printable<int>::value << "\n";               // true
    std::cout << "vector<int>: " << is_printable<std::vector<int>>::value << "\n";  // false

    return 0;
}
```

Notice that `is_iterable` checks two expressions in a single `void_t<...>` - you can pack multiple validity checks together and the whole thing only succeeds if every expression is valid.

**Expected output:**

```text
=== has_size ===
vector<int>: true
string:      true
int:         false
double:      false

=== is_iterable ===
vector<int>: true
int:         false

=== has_value_type ===
vector<int>: true
int:         false

=== is_printable ===
int:         true
vector<int>: false
```

### Q2: Show why `is_detected` (or equivalent) is cleaner than raw `void_t`

The `is_detected` utility from Library Fundamentals TS v2 wraps the two-piece `void_t` boilerplate into a single reusable machine. Instead of writing six lines per trait, you write one alias template and query it. The machinery is a bit involved to implement, but you only write it once:

```cpp
#include <iostream>
#include <type_traits>
#include <vector>
#include <string>

// === Custom is_detected implementation ===
// (Mirrors std::experimental::is_detected from Library Fundamentals TS v2)

namespace detail {
    template <class Default, class AlwaysVoid, template<class...> class Op, class... Args>
    struct detector {
        using value_t = std::false_type;
        using type = Default;
    };

    template <class Default, template<class...> class Op, class... Args>
    struct detector<Default, std::void_t<Op<Args...>>, Op, Args...> {
        using value_t = std::true_type;
        using type = Op<Args...>;
    };

    struct nonesuch {
        nonesuch() = delete;
        ~nonesuch() = delete;
        nonesuch(const nonesuch&) = delete;
        void operator=(const nonesuch&) = delete;
    };
}

template <template<class...> class Op, class... Args>
using is_detected = typename detail::detector<detail::nonesuch, void, Op, Args...>::value_t;

template <template<class...> class Op, class... Args>
constexpr bool is_detected_v = is_detected<Op, Args...>::value;

template <template<class...> class Op, class... Args>
using detected_t = typename detail::detector<detail::nonesuch, void, Op, Args...>::type;

// === Using is_detected: define detection aliases ===

// Alias: T.size() -> returns its type
template <class T>
using size_expr = decltype(std::declval<const T&>().size());

// Alias: T.push_back(val) -> returns its type
template <class T>
using push_back_expr = decltype(std::declval<T&>().push_back(std::declval<typename T::value_type>()));

// Alias: T::iterator
template <class T>
using iterator_type = typename T::iterator;

int main() {
    std::cout << std::boolalpha;

    // Compare verbosity:
    // RAW void_t: 6 lines per trait (primary + specialization + helper)
    // is_detected: 1 line alias + 1 line usage

    std::cout << "=== is_detected - much cleaner ===\n";
    std::cout << "has .size():       vector=" << is_detected_v<size_expr, std::vector<int>>
              << ", int=" << is_detected_v<size_expr, int> << "\n";

    std::cout << "has .push_back():  vector=" << is_detected_v<push_back_expr, std::vector<int>>
              << ", string=" << is_detected_v<push_back_expr, std::string> << "\n";

    std::cout << "has ::iterator:    vector=" << is_detected_v<iterator_type, std::vector<int>>
              << ", int=" << is_detected_v<iterator_type, int> << "\n";

    // detected_t gives you the actual type
    // detected_t<size_expr, std::vector<int>> -> std::vector<int>::size_type

    return 0;
}
```

The bonus is `detected_t<Op, T>`, which gives you the actual return type of the expression rather than just a yes/no answer. That is useful when you need to branch on what type a method returns.

### Q3: Rewrite the detection idiom using a C++20 `requires` expression for the same check

In C++20, all of this collapses into a single `concept` definition. What took six lines and a helper becomes one line that reads almost like English:

```cpp
#include <iostream>
#include <concepts>
#include <vector>
#include <string>

// === C++20 requires-based detection - dramatically simpler ===

// One-liner concepts replace 6+ lines of void_t boilerplate
template <typename T>
concept HasSize = requires(const T& t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

template <typename T>
concept IsIterable = requires(T& t) {
    t.begin();
    t.end();
};

template <typename T>
concept HasValueType = requires { typename T::value_type; };

template <typename T>
concept IsPrintable = requires(std::ostream& os, const T& t) {
    { os << t } -> std::same_as<std::ostream&>;
};

// Concepts enable direct constrained overloads
template <HasSize T>
void print_info(const T& c) {
    std::cout << "  Container with size = " << c.size() << "\n";
}

template <typename T> requires (!HasSize<T>)
void print_info(const T& val) {
    std::cout << "  Scalar value (no size)\n";
}

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== requires-based detection (C++20) ===\n\n";

    std::cout << "HasSize:\n";
    std::cout << "  vector<int>: " << HasSize<std::vector<int>> << "\n";  // true
    std::cout << "  string:      " << HasSize<std::string> << "\n";       // true
    std::cout << "  int:         " << HasSize<int> << "\n";               // false

    std::cout << "\nIsIterable:\n";
    std::cout << "  vector<int>: " << IsIterable<std::vector<int>> << "\n";  // true
    std::cout << "  int:         " << IsIterable<int> << "\n";               // false

    std::cout << "\n=== Comparison: void_t vs is_detected vs requires ===\n\n";

    std::cout << "void_t approach (C++17):\n";
    std::cout << "  template<class T, class=void> struct has_X : false_type {};\n";
    std::cout << "  template<class T> struct has_X<T, void_t<expr>> : true_type {};\n";
    std::cout << "  -> 6 lines, hard to read, easy to get wrong\n\n";

    std::cout << "is_detected approach (C++17 TS):\n";
    std::cout << "  template<class T> using X_expr = decltype(expr);\n";
    std::cout << "  is_detected_v<X_expr, T>\n";
    std::cout << "  -> 2 lines, but requires is_detected machinery\n\n";

    std::cout << "requires approach (C++20):\n";
    std::cout << "  concept HasX = requires(T t) { t.x(); };\n";
    std::cout << "  -> 1 line, clear, composable, first-class language feature\n\n";

    std::cout << "=== Constrained overloads ===\n";
    std::vector<int> v{1,2,3};
    int x = 42;
    print_info(v);  // Container with size = 3
    print_info(x);  // Scalar value (no size)

    return 0;
}
```

The `requires` expression inside a concept definition is also part of the "immediate context" for SFINAE purposes, which means you get cleaner error messages and can compose concepts together with `&&` and `||`.

---

## Notes

- The detection idiom checks at compile time whether a type supports a given expression.
- `std::void_t<Expr>` maps valid expressions to `void`; substitution failure triggers SFINAE -> falls back to primary template.
- `is_detected` (Library Fundamentals TS) wraps the `void_t` pattern into a reusable utility.
- C++20 `requires` expressions are the modern replacement - shorter, clearer, composable via `&&`/`||`.
- Use `void_t` only in pre-C++20 codebases; prefer concepts in new code.
