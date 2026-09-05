# Use `std::forward_like` (C++23) for Forwarding with Ownership Propagation

**Category:** Move Semantics and Value Categories  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/forward_like>  

---

## Topic Overview

`std::forward_like` solves a problem that `std::forward` cannot: **forwarding a member of a forwarding reference while propagating the value category of the owning object**, not the member itself. When you have a forwarding reference `T&& obj` and you want to access `obj.member`, you need the member to be an lvalue reference when `obj` is an lvalue, and an rvalue reference when `obj` is an rvalue. Plain `std::forward<T>(obj).member` works in simple cases, but `std::forward_like` provides an explicit, composable, and correct solution for all cases.

The reason this trips people up is that `decltype(wrapper.value)` always gives you the member's own type, regardless of how `wrapper` was passed in. You lose the caller's value category entirely. `std::forward_like` restores it.

```cpp
// The problem forward_like solves:

template<typename T>
void process(T&& wrapper) {
    // We want to forward wrapper.value with the SAME
    // value category as wrapper:
    //   wrapper is lvalue  -> value should be lvalue
    //   wrapper is rvalue  -> value should be rvalue
    //
    // WRONG: std::forward<???>(wrapper.value)
    //   What type do we put? decltype(wrapper.value) is
    //   always the member type, losing wrapper's category.
    //
    // CORRECT (C++23):
    use(std::forward_like<T>(wrapper.value));
}
```

The signature is `template<class T, class U> constexpr auto&& forward_like(U&& x) noexcept;`. The type parameter `T` determines the value category and const-ness to apply, and `x` is the object to forward. The rules are:

```cpp
// T (Owner type)      |  U (Member)  |  Result type
// Owner&  (lvalue)    |  Member      |  Member&
// Owner&& (rvalue)    |  Member      |  Member&&
// const Owner& (clv)  |  Member      |  const Member&
// const Owner&& (crv) |  Member      |  const Member&&
```

Before C++23, achieving this required manual trait gymnastics or the `std::forward<T>(obj).member` pattern, which breaks down with `std::tuple`, `std::optional`, and other wrappers where member access goes through an accessor function rather than direct `.member` syntax. `std::forward_like` is the clean, generic solution.

---

## Self-Assessment

### Q1: What problem does `std::forward_like` solve that `std::forward` alone cannot

This example shows both the old approach and the C++23 approach side by side, with a helper that prints whether the value arrived as an lvalue or rvalue:

```cpp
// Requires C++23: compile with -std=c++23
#include <iostream>
#include <string>
#include <utility>
#include <type_traits>

struct Payload {
    std::string data;
};

struct Wrapper {
    Payload payload;
};

// Helper to show value category
template<typename T>
void consume(T&& val) {
    if constexpr (std::is_lvalue_reference_v<T&&>) {
        std::cout << "  received as LVALUE: " << val.data << "\n";
    } else {
        std::cout << "  received as RVALUE: " << val.data << "\n";
    }
}

// PRE-C++23 approach: forward the whole wrapper, then access member
template<typename T>
void process_old(T&& wrapper) {
    std::cout << "[old] forwarding via std::forward<T>(wrapper).payload:\n";
    consume(std::forward<T>(wrapper).payload);
}

// C++23 approach: forward_like propagates wrapper's category to payload
template<typename T>
void process_new(T&& wrapper) {
    std::cout << "[new] forwarding via std::forward_like<T>(wrapper.payload):\n";
    consume(std::forward_like<T>(wrapper.payload));
}

int main() {
    Wrapper w{{"Hello"}};

    // lvalue wrapper -> payload should be lvalue
    process_old(w);
    process_new(w);

    std::cout << "\n";

    // rvalue wrapper -> payload should be rvalue
    process_old(Wrapper{{"World"}});
    process_new(Wrapper{{"World"}});

    return 0;
}
```

Both approaches produce the same result here, but the `forward_like` version is clearer about intent and doesn't depend on having direct member access syntax. That's where the next example shows it pulling ahead.

### Q2: How does `std::forward_like` help with generic wrappers like `std::optional` or `std::tuple`

When you access a value through an accessor like `opt.value()` or `std::get<N>(tuple)`, you can't use `std::forward<T>(wrapper).value()` - there's no `.member` to chain after the forward. `std::forward_like` sidesteps the syntax problem entirely by working on the already-dereferenced value:

```cpp
// Requires C++23: compile with -std=c++23
#include <iostream>
#include <optional>
#include <string>
#include <utility>
#include <type_traits>

// A generic "apply if present" function for optional-like types.
// We want to call f with the contained value, forwarded with the
// same value category as the optional itself.
template<typename Opt, typename F>
decltype(auto) transform_if(Opt&& opt, F&& f) {
    if (opt.has_value()) {
        // forward_like<Opt> propagates the value category of the
        // optional to its contained value.
        // If Opt is an lvalue ref -> *opt is forwarded as lvalue
        // If Opt is an rvalue ref -> *opt is forwarded as rvalue
        return std::forward<F>(f)(
            std::forward_like<Opt>(*opt)
        );
    }
    using ReturnType = decltype(f(std::forward_like<Opt>(*opt)));
    if constexpr (std::is_void_v<ReturnType>) {
        return;
    } else {
        // In real code you'd return an empty optional - simplified here
        return ReturnType{};
    }
}

void show_lvalue(const std::string& s) {
    std::cout << "  lvalue handler: " << s << "\n";
}

void show_rvalue(std::string&& s) {
    std::cout << "  rvalue handler: " << s << " (moved)\n";
}

int main() {
    std::optional<std::string> opt{"C++23 is great"};

    // lvalue optional: contained value forwarded as lvalue
    std::cout << "Lvalue optional:\n";
    transform_if(opt, [](const std::string& s) {
        std::cout << "  got lvalue ref: " << s << "\n";
    });

    // rvalue optional: contained value forwarded as rvalue
    std::cout << "Rvalue optional:\n";
    transform_if(std::move(opt), [](std::string&& s) {
        std::cout << "  got rvalue ref: " << s << "\n";
    });

    return 0;
}
```

### Q3: How would you implement a simplified `forward_like` to understand its mechanics

Working through a hand-rolled implementation is the best way to internalize what `forward_like` actually computes. The logic is: copy const-ness from `T`, copy value category from `T`, apply both to `U`:

```cpp
// This shows the mechanism - use std::forward_like in real code.
#include <iostream>
#include <string>
#include <type_traits>
#include <utility>

// Simplified forward_like implementation
// T = the "owner" type that determines the resulting value category
// U = the actual object to forward
namespace detail {

    template<typename T, typename U>
    [[nodiscard]] constexpr auto&& forward_like_impl(U&& x) noexcept {
        // Determine const-ness from T
        constexpr bool is_adding_const =
            std::is_const_v<std::remove_reference_t<T>>;

        // Determine value category from T
        constexpr bool is_rvalue =
            !std::is_lvalue_reference_v<T>;  // T is non-ref or rvalue-ref

        using base_type = std::remove_reference_t<U>;
        using with_const = std::conditional_t<is_adding_const,
                              const base_type, base_type>;

        if constexpr (is_rvalue) {
            // T is an rvalue (T or T&&): produce rvalue reference
            static_assert(!std::is_lvalue_reference_v<T>);
            return static_cast<with_const&&>(x);
        } else {
            // T is an lvalue reference (T&): produce lvalue reference
            return static_cast<with_const&>(x);
        }
    }

} // namespace detail

// Demonstration
struct Inner {
    std::string name;
};

struct Outer {
    Inner inner;
};

template<typename T>
void inspect_forwarded(T&& wrapper) {
    // Use our implementation
    auto&& result = detail::forward_like_impl<T>(wrapper.inner);

    using ResultType = decltype(result);
    if constexpr (std::is_lvalue_reference_v<ResultType>) {
        if constexpr (std::is_const_v<std::remove_reference_t<ResultType>>) {
            std::cout << "  -> const Inner& (const lvalue)\n";
        } else {
            std::cout << "  -> Inner& (mutable lvalue)\n";
        }
    } else {
        if constexpr (std::is_const_v<std::remove_reference_t<ResultType>>) {
            std::cout << "  -> const Inner&& (const rvalue)\n";
        } else {
            std::cout << "  -> Inner&& (mutable rvalue)\n";
        }
    }
}

int main() {
    Outer o{{"test"}};
    const Outer co{{"const_test"}};

    std::cout << "Mutable lvalue:  ";
    inspect_forwarded(o);                 // T = Outer&, result: Inner&

    std::cout << "Const lvalue:    ";
    inspect_forwarded(co);                // T = const Outer&, result: const Inner&

    std::cout << "Mutable rvalue:  ";
    inspect_forwarded(std::move(o));      // T = Outer, result: Inner&&

    std::cout << "Const rvalue:    ";
    inspect_forwarded(std::move(co));     // T = const Outer, result: const Inner&&

    return 0;
}
```

The four cases map cleanly to the four rows in the table from the overview. Once you see that the implementation is just two `static_cast`s selected by `if constexpr`, the utility becomes less mysterious.

---

## Notes

- `std::forward_like<T>(x)` propagates the **value category and const-ness** of `T` onto `x`. Use `T` = the owner type, `x` = the member.
- It replaces the `std::forward<T>(wrapper).member` pattern, which fails when member access is through an accessor (like `opt.value()` or `std::get<N>(tuple)`).
- Think of `forward_like` as: "forward `x` *as if* it were a member of an object with type `T`."
- In generic library code (tuple-like, optional-like, variant-like types), `forward_like` is essential for correct forwarding of contained values.
- The "like" in the name means "give `x` a value category *like* that of `T`" - it doesn't actually require `x` to be a member of `T`.
- Before C++23, libraries like Boost and ranges-v3 had ad-hoc solutions; `forward_like` standardizes the pattern.
