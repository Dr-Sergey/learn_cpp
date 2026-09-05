# Provide customization points via ADL, tag_invoke, or CPO patterns

**Category:** API and Library Design  
**Standard:** C++20, C++23  
**Reference:** <https://wg21.link/P1895R0>  

---

## Topic Overview

A **customization point** is a function that library code calls but that users can override for their own types. The classic example is `swap` - the standard library calls `swap` internally, and you can teach it to use a faster version for your type. Three patterns exist for building these hooks, each with a different trade-off between simplicity and robustness.

### Pattern 1: ADL (the classic "swap" pattern)

The oldest and simplest approach. Your library provides a default implementation in its own namespace, the user provides an override in their namespace, and ADL makes the right one get called. Here's how it fits together:

```cpp
#include <utility>
#include <iostream>

namespace lib {
    // Default implementation
    template<typename T>
    void serialize(const T& val, std::ostream& os) {
        os << val;  // Default: use operator<<
    }
}

namespace user {
    struct Point { double x, y; };

    // Customization: found via ADL
    void serialize(const Point& p, std::ostream& os) {
        os << "(" << p.x << "," << p.y << ")";
    }
}

template<typename T>
void log_value(const T& val) {
    using lib::serialize;  // Bring default into scope
    serialize(val, std::cout);  // ADL finds user::serialize for user types
    std::cout << "\n";
}
```

The `using lib::serialize;` line is the crucial part. It ensures there's a fallback if ADL finds nothing, while still giving ADL a chance to find a type-specific overload in the argument's namespace.

### Pattern 2: tag_invoke (proposed for std::execution)

ADL has a weakness: any function with the right name in the argument's namespace can accidentally be picked up. `tag_invoke` sidesteps this by using a single, globally unique dispatch function. The "tag" - a function object type - identifies which operation you're customizing, making it impossible for an unrelated function to hijack the call.

```cpp
#include <iostream>
#include <type_traits>

// A tag type identifies the operation
struct serialize_fn {
    template<typename T>
    void operator()(const T& val, std::ostream& os) const {
        // Dispatch via tag_invoke
        tag_invoke(*this, val, os);
    }
};
inline constexpr serialize_fn serialize{};

// Default via tag_invoke
template<typename T>
void tag_invoke(serialize_fn, const T& val, std::ostream& os) {
    os << val;
}

// User customization
namespace user {
    struct Widget { int id; };
    void tag_invoke(serialize_fn, const Widget& w, std::ostream& os) {
        os << "Widget#" << w.id;
    }
}
```

When you call `serialize(w, os)`, it calls `tag_invoke(serialize, w, os)`. ADL finds the user's `tag_invoke` overload because the tag type `serialize_fn` is from the library namespace and the `Widget` is from the user's namespace - but crucially, the user's function must accept `serialize_fn` as its first argument, which prevents any accidental match.

### Pattern 3: CPO (Customization Point Object) - Used by Ranges

A Customization Point Object is an `inline constexpr` function object that wraps the ADL dispatch in a controlled way. This is what `std::ranges` uses for `ranges::begin`, `ranges::end`, and `ranges::swap`. The user still provides ADL-visible functions, but the CPO is what callers actually invoke - and it's always accessed via qualified name, so there's no hijacking risk on the call site.

```cpp
#include <concepts>
#include <iostream>

namespace lib {
    namespace _detail {
        // Fallback
        void serialize(auto const&, auto&) = delete;

        struct _serialize_fn {
            template<typename T, typename OS>
            requires requires(const T& t, OS& os) {
                serialize(t, os);  // ADL-found serialize
            }
            void operator()(const T& val, OS& os) const {
                serialize(val, os);  // Call ADL-found version
            }
        };
    }
    inline constexpr _detail::_serialize_fn serialize{};
}

// User just provides an ADL-visible serialize and the CPO finds it
```

The deleted fallback in `_detail` means that if no ADL overload is found, the concept check in `requires` fails cleanly - no confusing cascade of errors.

---

## Self-Assessment

### Q1: Compare the three customization point patterns

Here's a summary of the trade-offs. None of the three is universally best; it depends on how much control you need:

| Pattern | Pros | Cons |
| --- | --- | --- |
| ADL | Simple, well-understood | Hijacking risk, `using` dance |
| tag_invoke | No hijacking, single dispatch point | Verbose, non-standard |
| CPO | Type-safe, no hijacking | Complex to implement |

### Q2: What is "ADL hijacking" and how do CPOs prevent it

ADL hijacking is when an unrelated namespace accidentally provides a function that matches your customization point name. Imagine your library calls `serialize(val, os)` and the user's type lives in a namespace that also has a completely unrelated `serialize` - ADL will find it, and now you have a confusing mismatch.

CPOs prevent this because callers always use a qualified name like `lib::serialize(x, os)` to invoke the CPO. The CPO object itself is not a free function, so the call is not subject to ADL at the call site. Inside the CPO, ADL is used in a controlled, explicit way - and you can add concept requirements to guard against unintended matches.

### Q3: Show why `using std::swap; swap(a, b);` is necessary

Without the `using` declaration, the unqualified `swap(a, b)` only finds ADL-visible overloads. If `a` and `b` are built-in types like `int` or raw pointers, ADL has no associated namespace to search and finds nothing - the call would fail. The `using std::swap;` brings the standard fallback into scope so there's always a valid candidate, while still letting ADL find a type-specific overload for user-defined types. It's the classic two-line idiom for generic swap.

---

## Notes

- `std::ranges` uses CPOs everywhere: `ranges::begin`, `ranges::end`, `ranges::swap` - if you've used ranges, you've already relied on this pattern.
- `std::execution` (P2300) uses `tag_invoke` for sender/receiver customization in its async machinery.
- For new libraries, CPOs are the recommended approach - no hijacking risk and the interface is composable.
- ADL is still perfectly fine for simple, well-scoped customization points like `swap` and `operator<<` where hijacking is unlikely in practice.
