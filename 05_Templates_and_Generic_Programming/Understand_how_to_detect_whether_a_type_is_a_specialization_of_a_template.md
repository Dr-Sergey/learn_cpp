# Understand How to Detect Whether a Type Is a Specialization of a Template

**Category:** Templates and Generic Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/partial_specialization>  

---

## Topic Overview

### The Problem

Given a type `T`, you want to check at compile time: "Is `T` some specialization of `std::vector`?" - i.e., `std::vector<int>`, `std::vector<string>`, etc. You cannot just compare types with `std::is_same` because you do not know the element type ahead of time - you want to match the template itself, not a specific instantiation.

### The `is_specialization_of` Trait

The solution uses partial specialization to match the shape `Template<Args...>`. The primary template says "no" by default, and the partial specialization fires only when `T` actually decomposes into that form:

```cpp
// Primary: false by default
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

// Partial specialization: true when T matches Template<Args...>
template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};
```

The key is the template template parameter `template <typename...> class Template`. That lets you pass `std::vector` itself (not `std::vector<int>`) as an argument, so the partial specialization can match any instantiation of it.

| Expression | Result |
| --- | :---: |
| `is_specialization_of<std::vector<int>, std::vector>` | `true` |
| `is_specialization_of<std::optional<int>, std::optional>` | `true` |
| `is_specialization_of<int, std::vector>` | `false` |
| `is_specialization_of<std::vector<int>, std::optional>` | `false` |

### Limitation: Non-Type Template Parameters

This technique only works with templates that take **type** parameters. It breaks for `std::array<T,N>` (has a non-type `N`) and `std::integer_sequence<T, Ns...>`. The reason is that `template <typename...> class Template` can only match parameters that are types - it cannot match a `size_t` parameter.

---

## Self-Assessment

### Q1: Write `is_specialization_of<T, std::vector>` that is true only when `T` is a `std::vector`

Notice that the same trait works for multi-parameter templates like `std::map` without any changes - the variadic `Args...` captures all the type arguments at once:

```cpp
#include <iostream>
#include <type_traits>
#include <vector>
#include <optional>
#include <map>
#include <string>
#include <variant>

// Primary template: false
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

// Partial specialization: true when T = Template<Args...>
template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};

// Helper variable template
template <typename T, template <typename...> class Template>
constexpr bool is_specialization_of_v = is_specialization_of<T, Template>::value;

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== is_specialization_of ===\n\n";

    // vector tests
    std::cout << "vector<int>       is vector:   "
              << is_specialization_of_v<std::vector<int>, std::vector> << "\n";        // true
    std::cout << "vector<string>    is vector:   "
              << is_specialization_of_v<std::vector<std::string>, std::vector> << "\n"; // true
    std::cout << "vector<int>       is optional: "
              << is_specialization_of_v<std::vector<int>, std::optional> << "\n";       // false

    // optional tests
    std::cout << "optional<int>     is optional: "
              << is_specialization_of_v<std::optional<int>, std::optional> << "\n";     // true

    // map test (has multiple type params - still works!)
    std::cout << "map<string,int>   is map:      "
              << is_specialization_of_v<std::map<std::string,int>, std::map> << "\n";   // true

    // non-template types
    std::cout << "int               is vector:   "
              << is_specialization_of_v<int, std::vector> << "\n";                      // false
    std::cout << "double            is optional:  "
              << is_specialization_of_v<double, std::optional> << "\n";                 // false

    // variant
    std::cout << "variant<int,str>  is variant:  "
              << is_specialization_of_v<std::variant<int,std::string>, std::variant> << "\n"; // true

    return 0;
}
```

The trait correctly handles every type-only template, regardless of how many type parameters it takes.

**Expected output:**

```text
=== is_specialization_of ===

vector<int>       is vector:   true
vector<string>    is vector:   true
vector<int>       is optional: false
optional<int>     is optional: true
map<string,int>   is map:      true
int               is vector:   false
double            is optional:  false
variant<int,str>  is variant:  true
```

### Q2: Show the limitation: this approach breaks for non-type template parameter specializations

`std::array<T, N>` is the canonical example of a template that cannot be handled by the generic trait. The compiler rejects the attempt because `template <typename...>` expects only type parameters, but `std::array` has a `size_t N`. The workaround is to write a dedicated trait for each such template:

```cpp
#include <iostream>
#include <type_traits>
#include <array>

// Our is_specialization_of (type-only version)
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};

template <typename T, template <typename...> class Template>
constexpr bool is_specialization_of_v = is_specialization_of<T, Template>::value;

// std::array<T, N> takes a TYPE (T) and a NON-TYPE (N = size_t)
// template<typename...> class Template can only match TYPE parameters!

// This will NOT compile:
// is_specialization_of_v<std::array<int, 5>, std::array>
// Error: std::array expects <typename, size_t>, not <typename...>

// Workaround for specific templates with non-type params:
template <typename T>
struct is_std_array : std::false_type {};

template <typename T, std::size_t N>
struct is_std_array<std::array<T, N>> : std::true_type {};

template <typename T>
constexpr bool is_std_array_v = is_std_array<T>::value;

// Similarly for std::integer_sequence:
template <typename T>
struct is_integer_sequence : std::false_type {};

template <typename T, T... Ints>
struct is_integer_sequence<std::integer_sequence<T, Ints...>> : std::true_type {};

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== Limitation: non-type template parameters ===\n\n";

    // This WORKS (type-only parameters):
    // std::cout << is_specialization_of_v<std::vector<int>, std::vector>;  // true

    // This FAILS to compile:
    // std::cout << is_specialization_of_v<std::array<int,5>, std::array>;
    // Error: template template argument has different template params

    std::cout << "Broken: is_specialization_of<array<int,5>, array>  -> compile error\n";
    std::cout << "Reason: std::array<T,N> has non-type param N (size_t)\n";
    std::cout << "        template<typename...> can only match type params\n\n";

    // Workaround: dedicated trait
    std::cout << "Workaround: is_std_array<T>\n";
    std::cout << "array<int,5>:   " << is_std_array_v<std::array<int,5>> << "\n";    // true
    std::cout << "array<float,3>: " << is_std_array_v<std::array<float,3>> << "\n";  // true
    std::cout << "int:            " << is_std_array_v<int> << "\n";                   // false

    std::cout << "\nOther templates with non-type params:\n";
    std::cout << "  std::array<T, N>              (size_t N)\n";
    std::cout << "  std::integer_sequence<T, Is>  (T... Is)\n";
    std::cout << "  std::bitset<N>                (size_t N)\n";
    std::cout << "  std::span<T, Extent>          (size_t Extent)\n";
    std::cout << "Each needs a dedicated trait.\n";

    return 0;
}
```

There is no single generic workaround for all non-type parameter templates - you write a dedicated `is_std_array`, `is_bitset`, and so on for each one you need.

### Q3: Use a concept to constrain a function to accept only `std::optional` specializations

Once you have the trait, wrapping it in a concept gives you a clean, composable constraint that produces readable error messages when someone passes the wrong type:

```cpp
#include <iostream>
#include <concepts>
#include <optional>
#include <string>
#include <vector>

// is_specialization_of trait
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};

// Concept: T must be a std::optional<X> for some X
template <typename T>
concept IsOptional = is_specialization_of<std::remove_cvref_t<T>, std::optional>::value;

// Similarly for vector
template <typename T>
concept IsVector = is_specialization_of<std::remove_cvref_t<T>, std::vector>::value;

// Function constrained to only accept std::optional<T>
template <IsOptional Opt>
void unwrap_or_print(const Opt& opt, const typename Opt::value_type& fallback) {
    if (opt.has_value()) {
        std::cout << "Has value: " << *opt << "\n";
    } else {
        std::cout << "No value, using fallback: " << fallback << "\n";
    }
}

// Function constrained to only accept std::vector<T>
template <IsVector V>
void print_first(const V& v) {
    if (!v.empty()) {
        std::cout << "First element: " << v.front() << "\n";
    }
}

int main() {
    std::cout << "=== Concept-constrained to std::optional ===\n\n";

    std::optional<int> a = 42;
    std::optional<int> b = std::nullopt;
    std::optional<std::string> c = "hello";

    unwrap_or_print(a, 0);           // Has value: 42
    unwrap_or_print(b, -1);          // No value, using fallback: -1
    unwrap_or_print(c, std::string("default")); // Has value: hello

    // These would NOT compile - int is not an optional:
    // unwrap_or_print(42, 0);       // ERROR: constraint not satisfied
    // unwrap_or_print("hello", ""); // ERROR: constraint not satisfied

    std::cout << "\n=== Concept-constrained to std::vector ===\n\n";

    std::vector<int> v{10, 20, 30};
    print_first(v);  // First element: 10

    // print_first(42);  // ERROR: constraint not satisfied

    return 0;
}
```

The `std::remove_cvref_t<T>` in the concept definition is important - without it, a `const std::optional<int>&` would not match because the partial specialization would try to match `const std::optional<int>&`, not `std::optional<int>`.

**Expected output:**

```text
=== Concept-constrained to std::optional ===

Has value: 42
No value, using fallback: -1
Has value: hello

=== Concept-constrained to std::vector ===

First element: 10
```

---

## Notes

- `is_specialization_of<T, Template>` uses partial specialization to match `Template<Args...>`.
- Works for templates with **only type parameters** (`vector`, `optional`, `map`, `variant`).
- **Breaks** for templates with non-type parameters (`array<T,N>`, `bitset<N>`, `span<T,E>`) - use dedicated traits.
- Wrap in a concept for clean constrained overloads: `concept IsOptional = is_specialization_of<T, std::optional>::value;`
- Always use `std::remove_cvref_t<T>` in the concept to handle const/ref-qualified types.
