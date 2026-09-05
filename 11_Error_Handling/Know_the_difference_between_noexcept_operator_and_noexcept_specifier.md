# Know the Difference Between `noexcept` Operator and `noexcept` Specifier

**Category:** Error Handling  
**Standard:** C++11, C++17 (conditional noexcept refinements)  
**Reference:** [cppreference - noexcept specifier](https://en.cppreference.com/w/cpp/language/noexcept_spec), [cppreference - noexcept operator](https://en.cppreference.com/w/cpp/language/noexcept)  

---

## Topic Overview

C++ reuses the keyword `noexcept` in two fundamentally different roles. As a **specifier**, `noexcept` is part of a function's type and tells the compiler (and callers) that the function will never emit an exception. As an **operator**, `noexcept(expr)` is a compile-time boolean expression that evaluates to `true` when `expr` is declared not to throw.

| Aspect | `noexcept` Specifier | `noexcept` Operator |
| --- | --- | --- |
| **Syntax position** | After parameter list: `void f() noexcept;` | Inside an expression: `noexcept(expr)` |
| **Evaluates to** | N/A - it is a declaration annotation | `bool` - compile-time constant |
| **Part of the type?** | Yes (since C++17) | No - yields a `constexpr bool` |
| **Operand evaluated?** | N/A | Never - unevaluated context |
| **Primary use** | Promise the function won't throw | Query whether an expression can throw |

The reason this trips people up is the visual similarity: both use the word `noexcept`, and when they're nested together it's easy to lose track of which role each plays.

The powerful idiom `noexcept(noexcept(expr))` combines both: the **outer** `noexcept` is the specifier, and the **inner** `noexcept(expr)` is the operator. The function becomes conditionally noexcept based on whether `expr` would throw.

```cpp
noexcept( noexcept( some_expression ) )
^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^
  |           |
  |           +-- operator: returns true/false
  +------------- specifier: applies that bool to the function
```

This pattern is essential for writing generic code that correctly propagates exception specifications. Without it, a template wrapper might silently weaken a `noexcept` guarantee or incorrectly promise one. In C++17, `noexcept` became part of the function type, meaning `void(*)() noexcept` and `void(*)()` are distinct types - making correct conditional `noexcept` even more important for templates and function pointers.

---

## Self-Assessment

### Q1: Implement a generic `swap_impl` that is conditionally `noexcept` based on the swapped type

The goal here is a `swap` that correctly advertises itself as `noexcept` for types whose move operations are `noexcept`, and as potentially-throwing for types whose move operations might throw. The `noexcept(noexcept(...))` idiom does exactly this - the inner operator queries the actual operations used, and the outer specifier applies the result to the function signature.

```cpp
// swap_conditional.cpp — C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o swap_conditional swap_conditional.cpp
#include <iostream>
#include <type_traits>
#include <utility>

// Conditionally noexcept swap using the noexcept(noexcept(...)) idiom.
template <typename T>
void swap_impl(T& a, T& b)
    noexcept(noexcept(
        // inner noexcept is the OPERATOR — queries move-construct + move-assign
        T(std::declval<T&&>()), std::declval<T&>() = std::declval<T&&>()
    ))
{
    T tmp = std::move(a);
    a     = std::move(b);
    b     = std::move(tmp);
}

// --- Demo types -------------------------------------------------------
struct Safe {
    int value{};
    Safe() = default;
    Safe(Safe&&) noexcept            = default;
    Safe& operator=(Safe&&) noexcept = default;
};

struct Risky {
    int value{};
    Risky() = default;
    Risky(Risky&&) noexcept(false)            { /* may throw */ }
    Risky& operator=(Risky&&) noexcept(false) { return *this; }
};

int main() {
    // operator noexcept queries
    static_assert( noexcept(swap_impl(std::declval<Safe&>(),  std::declval<Safe&>())),
                   "swap of Safe must be noexcept");
    static_assert(!noexcept(swap_impl(std::declval<Risky&>(), std::declval<Risky&>())),
                   "swap of Risky must NOT be noexcept");

    Safe  s1{1}, s2{2};
    swap_impl(s1, s2);
    std::cout << "Safe:  s1=" << s1.value << " s2=" << s2.value << "\n";

    Risky r1{3}, r2{4};
    swap_impl(r1, r2);
    std::cout << "Risky: r1=" << r1.value << " r2=" << r2.value << "\n";
}
// Output:
// Safe:  s1=2 s2=1
// Risky: r1=4 r2=3
```

The `static_assert` lines use the operator form to verify at compile time that the specifier is set correctly. That's a useful pattern in templates: not just computing the right `noexcept` value, but asserting it as documentation.

### Q2: Show how `noexcept` being part of the function type (C++17) affects function pointers

Before C++17, `noexcept` was just an informal attribute on a function - it had no effect on the function's type. In C++17 it became part of the type itself, so `void(*)()` and `void(*)() noexcept` are genuinely different types. This matters for generic code, callbacks, and `std::function` wrappers.

```cpp
// noexcept_type.cpp — C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o noexcept_type noexcept_type.cpp
#include <iostream>
#include <type_traits>

void throwing_fn()           { throw 42; }
void safe_fn()      noexcept {}

int main() {
    // In C++17, noexcept is part of the type.
    using FnThrow   = void(*)();
    using FnNoThrow = void(*)() noexcept;

    static_assert(!std::is_same_v<FnThrow, FnNoThrow>,
                  "Different types since C++17");

    // A noexcept function pointer can implicitly convert to a non-noexcept one.
    FnThrow   p1 = safe_fn;        // OK: noexcept -> may-throw is safe
    // FnNoThrow p2 = throwing_fn;  // ERROR: may-throw -> noexcept is not allowed

    FnNoThrow p3 = safe_fn;         // OK

    // The noexcept operator queries the pointer's declared exception spec
    std::cout << "p1 noexcept? " << noexcept(p1()) << "\n"; // 0 — FnThrow
    std::cout << "p3 noexcept? " << noexcept(p3()) << "\n"; // 1 — FnNoThrow

    p1(); // call safe_fn through a non-noexcept pointer — fine at runtime
    p3(); // call safe_fn through a noexcept pointer

    std::cout << "Done.\n";
}
// Output:
// p1 noexcept? 0
// p3 noexcept? 1
// Done.
```

Notice that `p1` holds `safe_fn` but `noexcept(p1())` returns 0 - because the operator looks at the declared type of the pointer (`FnThrow`), not at what function is currently stored in it. The noexcept-ness of a call through a pointer is determined statically by the pointer's type, not dynamically by the callee.

### Q3: Write a compile-time trait that detects whether a type's constructor is `noexcept` and use it to select an algorithm

This is the practical application: using the operator to select between a fast move-based path and a safe copy-based path at compile time. `std::vector::push_back` does essentially the same thing internally - it moves elements when it can guarantee no exception, and copies otherwise to preserve the strong exception guarantee.

```cpp
// select_algorithm.cpp — C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o select_algorithm select_algorithm.cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <type_traits>
#include <utility>

// Trait: is T nothrow move-constructible AND nothrow move-assignable?
template <typename T>
inline constexpr bool is_nothrow_relocatable_v =
    std::is_nothrow_move_constructible_v<T> &&
    std::is_nothrow_move_assignable_v<T>;

// Fast path — moves elements (only safe if move ops are noexcept)
template <typename T>
void optimized_transfer(std::vector<T>& dst, std::vector<T>& src)
    noexcept(is_nothrow_relocatable_v<T>)
{
    if constexpr (is_nothrow_relocatable_v<T>) {
        std::cout << "[fast path: move]\n";
        dst.reserve(dst.size() + src.size());
        for (auto& e : src)
            dst.push_back(std::move(e));
    } else {
        std::cout << "[slow path: copy for strong guarantee]\n";
        dst.reserve(dst.size() + src.size());
        for (const auto& e : src)
            dst.push_back(e);  // copy — if it throws, dst is unchanged
    }
    src.clear();
}

struct Widget {
    int id{};
    Widget() = default;
    Widget(const Widget&) = default;
    Widget(Widget&&) noexcept = default;
    Widget& operator=(Widget&&) noexcept = default;
    Widget& operator=(const Widget&) = default;
};

struct FragileWidget {
    int id{};
    FragileWidget() = default;
    FragileWidget(const FragileWidget&) = default;
    FragileWidget(FragileWidget&&) noexcept(false) {}
    FragileWidget& operator=(FragileWidget&&) noexcept(false) { return *this; }
    FragileWidget& operator=(const FragileWidget&) = default;
};

int main() {
    // noexcept operator used as a static assertion
    static_assert( noexcept(optimized_transfer(
        std::declval<std::vector<Widget>&>(),
        std::declval<std::vector<Widget>&>())));
    static_assert(!noexcept(optimized_transfer(
        std::declval<std::vector<FragileWidget>&>(),
        std::declval<std::vector<FragileWidget>&>())));

    std::vector<Widget> a{{1},{2}}, b;
    optimized_transfer(b, a);
    std::cout << "b.size()=" << b.size() << "\n";

    std::vector<FragileWidget> c{{3},{4}}, d;
    optimized_transfer(d, c);
    std::cout << "d.size()=" << d.size() << "\n";
}
// Output:
// [fast path: move]
// b.size()=2
// [slow path: copy for strong guarantee]
// d.size()=2
```

The `if constexpr` branch is selected at compile time based on the trait - no runtime overhead. This is the same reasoning that makes marking your move constructors `noexcept` so important: it's not just documentation, it directly affects which code path the standard library chooses.

---

## Notes

- `noexcept` without parentheses on a function is equivalent to `noexcept(true)`.
- The `noexcept` operator never evaluates its operand - it operates in an unevaluated context like `sizeof` and `decltype`.
- Since C++17, `noexcept` is part of the function **type**, not just a decoration. This affects overload resolution, function pointers, and `std::function` template arguments.
- `std::move_if_noexcept` and `std::vector::push_back` internally rely on the `noexcept` operator to decide between move and copy, directly impacting performance.
- Throwing from a `noexcept` function calls `std::terminate` - there is no stack unwinding. Use `noexcept` only when you can genuinely guarantee no exceptions.
- Conditional `noexcept` on destructors is rarely needed: destructors are implicitly `noexcept(true)` unless a base or member has a potentially-throwing destructor.
- Prefer `noexcept(noexcept(expr))` over manually writing `noexcept(std::is_nothrow_...)` traits - the operator-based form automatically tracks the actual expression.
