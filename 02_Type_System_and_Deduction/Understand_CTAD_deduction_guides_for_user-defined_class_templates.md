# Understand CTAD Deduction Guides for User-Defined Class Templates

**Category:** Type System and Deduction  
**Standard:** C++17, C++20 (deduction guides)  
**Reference:** <https://en.cppreference.com/w/cpp/language/class_template_argument_deduction>  

---

## Topic Overview

### What Are Deduction Guides

A **deduction guide** is a pattern that tells the compiler how to deduce class template arguments from constructor arguments. While the compiler can implicitly generate guides from constructors, **user-defined deduction guides** give you explicit control over the deduction process.

The syntax looks like a function declaration but with a trailing `->` that says "this argument pattern maps to this template specialization":

```cpp
template-name(parameter-list) -> template-id;
```

### Why Do We Need User-Defined Deduction Guides

| Situation | Without Guide | With Guide |
| --- | --- | --- |
| Constructor takes a different type than stored | Deduces wrong type | Correct deduction |
| Type decay desired (array -> pointer, etc.) | No automatic decay | Explicit decay |
| Multiple constructors create ambiguity | Compile error | Clear resolution |
| Aggregate types (pre-C++20) | No CTAD support | Enables CTAD |
| Want to deduce from non-constructor patterns | Not possible | Custom mapping |

### Implicit vs Explicit Deduction Guides

The compiler generates **implicit deduction guides** from each constructor. For simple cases this works perfectly:

```cpp
template<typename T>
struct Wrapper {
    T value;
    Wrapper(T v) : value(v) {}  // Generates: Wrapper(T) -> Wrapper<T>
};

Wrapper w{42};  // Wrapper<int> - implicit guide works fine
```

**User-defined (explicit) deduction guides** override or supplement these. Here's the classic case: you want `const char*` to be stored as `std::string`, not as a raw pointer:

```cpp
template<typename T>
struct Wrapper {
    T value;
    Wrapper(T v) : value(v) {}
};

// User-defined deduction guide: const char* - std::string
Wrapper(const char*) -> Wrapper<std::string>;

Wrapper w{"hello"};  // Wrapper<std::string>, NOT Wrapper<const char*>
```

Without the guide, you'd silently get a `Wrapper<const char*>` storing a dangling pointer risk. The guide redirects that.

### Deduction Guide Syntax

```cpp
// Basic form
TemplateName(param-types) -> TemplateName<deduced-types>;

// Templated guide
template<typename T>
TemplateName(T, T) -> TemplateName<T>;

// Guide with constraints (C++20)
template<typename T>
    requires std::integral<T>
TemplateName(T) -> TemplateName<int>;
```

### Priority Rules

When both implicit and explicit guides exist, the compiler treats them as an **overload set** and applies overload resolution:

```cpp
1. Both implicit and explicit guides are candidates
2. If equally ranked, explicit guides are preferred
3. More specialized guides win over less specialized ones
4. If still ambiguous, compilation fails
```

### Guide for Iterators (Real-World Pattern)

The standard library uses deduction guides extensively. The vector iterator-pair guide is a good example - without it, CTAD would have no way to figure out the element type from two iterators:

```cpp
// std::vector has this guide:
// template<class InputIt>
// vector(InputIt, InputIt) -> vector<typename iterator_traits<InputIt>::value_type>;

std::vector v(list.begin(), list.end());  // Deduces element type from iterators
```

### Deduction Guides and Aggregate Initialization (C++20)

Starting in C++20, aggregates get **implicit deduction guides** from their members. Before C++20 you had to write explicit guides even for simple structs:

```cpp
template<typename T, typename U>
struct Pair {
    T first;
    U second;
};

// C++20: works with NO explicit guide!
Pair p{42, 3.14};  // Pair<int, double>
```

Before C++20, you needed explicit guides for aggregates:

```cpp
// Pre-C++20 explicit guide for aggregates
template<typename T, typename U>
Pair(T, U) -> Pair<T, U>;
```

### Copy vs Non-Copy Deduction

One subtlety worth knowing: when you initialize one `Box` from another, CTAD produces a copy, not a nested wrapper. That's usually what you want, but it can surprise you:

```cpp
template<typename T>
struct Box {
    T value;
    Box(T v) : value(v) {}
};

Box b1{42};       // Box<int> - normal deduction
Box b2{b1};       // Box<int> - copy deduction (does NOT wrap: Box<Box<int>>)
Box b3 = b1;      // Box<int> - copy deduction

// To force wrapping, use an explicit guide or be explicit:
Box<Box<int>> b4{b1};  // Explicit: Box<Box<int>>
```

### Common Pitfalls

Three traps to be aware of before writing your first deduction guide:

```cpp
// PITFALL 1: Guide doesn't match any constructor
template<typename T>
struct Bad {
    std::vector<T> data;
    Bad(std::initializer_list<T> il) : data(il) {}
};
Bad(const char*) -> Bad<std::string>;  // Guide exists, but no matching ctor!
// Bad b{"hello"};  // ERROR: no matching constructor

// PITFALL 2: Forgetting that guides are NOT constructors
// They only guide deduction, they don't create new ways to construct

// PITFALL 3: Guide creates dangling reference
template<typename T>
struct Ref {
    const T& ref;
    Ref(const T& r) : ref(r) {}
};
// Ref r{42};  // Ref<int>, but ref binds to temporary - DANGLING!
```

---

## Self-Assessment

### Q1: Write a user-defined deduction guide for a custom wrapper that deduces from a single constructor argument

Three different guides, three different behaviors. The `static_assert` lines confirm at compile time that the guides are actually doing what you intended:

```cpp
#include <iostream>
#include <string>
#include <type_traits>

template<typename T>
struct Holder {
    T value;

    // Constructor takes const T&
    explicit Holder(const T& v) : value(v) {
        std::cout << "Constructed Holder<" << typeid(T).name() << ">\n";
    }
};

// Deduction guide 1: string literals - std::string
Holder(const char*) -> Holder<std::string>;

// Deduction guide 2: arrays decay to pointers
template<typename T, std::size_t N>
Holder(T(&)[N]) -> Holder<T*>;

// Deduction guide 3: force int for all integral types
template<typename T>
    requires std::integral<T>
Holder(T) -> Holder<int>;

int main() {
    // Guide 1: const char* - std::string
    Holder h1{"hello"};
    static_assert(std::is_same_v<decltype(h1), Holder<std::string>>);
    std::cout << "h1.value = " << h1.value << "\n";

    // Guide 3: short - int
    short s = 5;
    Holder h2{s};
    static_assert(std::is_same_v<decltype(h2), Holder<int>>);
    std::cout << "h2.value = " << h2.value << "\n";

    // No guide needed: double deduces normally
    Holder h3{3.14};
    static_assert(std::is_same_v<decltype(h3), Holder<double>>);
    std::cout << "h3.value = " << h3.value << "\n";

    return 0;
}
```

**Output:**

```text
Constructed Holder<...string...>
h1.value = hello
Constructed Holder<int>
h2.value = 5
Constructed Holder<double>
h3.value = 3.14
```

`Holder(const char*) -> Holder<std::string>` maps C-string arguments to `Holder<std::string>`. The integral-constrained guide catches `short`, `char`, `long` etc. and maps them all to `Holder<int>`. When no explicit guide matches better, the implicit guide from the constructor applies (e.g., `double`). The `static_assert` lines confirm compile-time type correctness.

### Q2: Show a case where CTAD picks the wrong type and a deduction guide is required to fix it

Without a guide, `Name{"Alice"}` deduces `Name<const char*>` - storing a raw pointer to a string literal. That's a subtle ownership hazard. The guides redirect both `const char*` and `string_view` to `Name<std::string>` so the string is always owned:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <type_traits>
#include <cstddef>

// A "Name" wrapper: we always want to store std::string internally
template<typename T>
struct Name {
    T data;
    Name(const T& d) : data(d) {}
};

// WITHOUT a deduction guide:
// Name n1{"Alice"};           // Deduces Name<const char*> - NOT what we want!
// Name n2{std::string_view{"Bob"}};  // Deduces Name<string_view> - also wrong

// FIX: deduction guides that coerce to std::string
Name(const char*) -> Name<std::string>;
Name(std::string_view) -> Name<std::string>;

// Another real-world case: span-like view
template<typename T>
struct Span {
    const T* ptr;
    std::size_t len;

    template<std::size_t N>
    Span(T (&arr)[N]) : ptr(arr), len(N) {}
};

// Without guide, CTAD fails because T appears in non-deduced context (array ref)
// With guide, we explicitly map array references:
template<typename T, std::size_t N>
Span(T (&)[N]) -> Span<T>;

int main() {
    // Case 1: string coercion
    Name n1{"Alice"};
    static_assert(std::is_same_v<decltype(n1), Name<std::string>>);
    std::cout << "n1.data = " << n1.data << "\n";

    Name n2{std::string_view{"Bob"}};
    static_assert(std::is_same_v<decltype(n2), Name<std::string>>);
    std::cout << "n2.data = " << n2.data << "\n";

    // Case 2: array-to-span deduction
    int arr[] = {1, 2, 3, 4, 5};
    Span s{arr};
    static_assert(std::is_same_v<decltype(s), Span<int>>);
    std::cout << "Span: len=" << s.len << ", first=" << s.ptr[0] << "\n";

    return 0;
}
```

**Output:**

```text
n1.data = Alice
n2.data = Bob
Span: len=5, first=1
```

For `Span`, the template constructor uses `T(&)[N]` which is a non-deduced context for CTAD; the explicit guide resolves this. Explicit guides are **preferred over implicit guides** when equally ranked, ensuring the fix always applies.

### Q3: Explain how deduction guides interact with aggregate initialization in C++20

**Before C++20:** Aggregates had no implicit deduction guides. CTAD simply didn't work for aggregate class templates:

```cpp
template<typename T, typename U>
struct Point {
    T x;
    U y;
};
// Point p{1, 2.0};  // ERROR in C++17: no deduction guide
```

You needed explicit guides:

```cpp
template<typename T, typename U>
Point(T, U) -> Point<T, U>;  // Required in C++17
```

**C++20 change:** The compiler automatically generates deduction guides from aggregate members, treating each member as if it were a constructor parameter:

```cpp
// C++20 - works with NO explicit guide:
template<typename T, typename U>
struct Point {
    T x;
    U y;
};

Point p1{1, 2.0};    // Point<int, double>
Point p2{3.14f, 0};  // Point<float, int>
```

**How the implicit aggregate guide is synthesized:**

For an aggregate `Agg<T1, T2, ...>` with members of types `M1, M2, ...`:

```cpp
template<typename T1, typename T2, ...>
Agg(M1, M2, ...) -> Agg<T1, T2, ...>;
```

**Example with nested aggregates:**

```cpp
#include <iostream>
#include <string>
#include <type_traits>

template<typename T>
struct Inner {
    T value;
};

template<typename T, typename U>
struct Outer {
    Inner<T> first;
    U second;
};

// C++20: aggregate CTAD with nested aggregates
// Note: nested aggregate members are deduced from their sub-aggregates

int main() {
    // Direct aggregate CTAD
    Inner i{42};                       // Inner<int>
    static_assert(std::is_same_v<decltype(i), Inner<int>>);

    // Outer with braced init for Inner
    Outer o{Inner{1}, 2.0};           // Outer<int, double>
    static_assert(std::is_same_v<decltype(o), Outer<int, double>>);

    std::cout << "Inner: " << i.value << "\n";
    std::cout << "Outer: " << o.first.value << ", " << o.second << "\n";

    // Designated initializers + CTAD (C++20)
    // Note: designated initializers with CTAD require explicit guide for most cases
    // because designated init doesn't participate in guide synthesis

    return 0;
}
```

**Output:**

```text
Inner: 42
Outer: 1, 2.0
```

**Key rules for aggregate CTAD:**

| Rule | Detail |
| --- | --- |
| Applies to | Class templates that are aggregates (no user-declared constructors) |
| Guide source | One guide per braced-init sequence matching members |
| Nested aggregates | Each nested aggregate member deduces from its own brace-init |
| Base classes | Aggregate bases (C++17) also participate in guide synthesis |
| Designated init | Limited interaction - may require explicit guides |
| Explicit guides | Always override implicit aggregate guides when matched |

---

## Notes

- **Deduction guides are not constructors.** They only participate in template argument deduction, not actual construction. A guide can map to a type that no constructor directly accepts - this causes a compile error at the construction step.
- **Explicit keyword on guides:** You can mark a guide `explicit` to prevent it from being used in copy-initialization (`=` syntax):

  ```cpp
  explicit Holder(const char*) -> Holder<std::string>;
  Holder h1 = "hello";   // ERROR: explicit guide
  Holder h2{"hello"};    // OK
  ```

- **Standard library guides to study:** `std::pair`, `std::tuple`, `std::array` (C++20 aggregate), `std::vector` (iterator-pair guide), `std::basic_string`.
- **Interaction with `std::initializer_list`:** When a constructor takes `initializer_list<T>`, the implicit guide deduces from the list. Explicit guides can override this.
- In production code, prefer writing deduction guides that produce **owning types** (e.g., `std::string` over `const char*`) to avoid dangling pointers.
