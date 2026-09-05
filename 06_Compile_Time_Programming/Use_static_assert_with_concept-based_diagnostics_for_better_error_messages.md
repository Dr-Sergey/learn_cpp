# Use static_assert with concept-based diagnostics for better error messages

**Category:** Compile Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/static_assert>  

---

## Topic Overview

`static_assert` has been available since C++11, but C++20 concepts dramatically improve the quality of error messages by testing each requirement individually. Instead of one large, opaque failure, you get pinpointed diagnostics that tell you exactly which requirement wasn't satisfied and why.

### Basic static_assert

Even before concepts, `static_assert` in a class body is a good way to give clear error messages for unsupported instantiations. The message string is shown directly in the compiler output, which makes the intention explicit:

```cpp
#include <type_traits>

template<typename T>
class Container {
    static_assert(!std::is_reference_v<T>,
                  "Container does not support reference types. Use std::reference_wrapper.");
    static_assert(std::is_destructible_v<T>,
                  "T must be destructible.");
    static_assert(!std::is_void_v<T>,
                  "Container<void> is not supported.");
};

// Container<int&> ->
// error: static assertion failed: Container does not support reference types.
```

You get one assertion failure at a time, each with a specific message. That's already much better than a SFINAE failure.

### Concept-Based Diagnostics

Concepts take this further. When a concept constraint fails, the compiler can tell you which specific sub-expression or requirement wasn't satisfied - not just that the overall constraint was false. This is the core ergonomic win of C++20 concepts over SFINAE.

```cpp
#include <concepts>
#include <type_traits>
#include <string>

template<typename T>
concept Serializable =
    requires(const T& obj, std::ostream& os) {
        { obj.serialize(os) } -> std::same_as<void>;
    } &&
    requires {
        { T::type_name() } -> std::convertible_to<std::string>;
    };

// When the concept fails:
// "constraint not satisfied: 'obj.serialize(os)' is not a valid expression"
// Much better than SFINAE's "no matching function"

template<Serializable T>
void save(const T& obj);

// Layered concepts + static_assert:
template<typename T>
void detailed_check() {
    static_assert(requires(const T& obj, std::ostream& os) {
        obj.serialize(os);
    }, "T must have a serialize(ostream&) method");

    static_assert(requires { T::type_name(); },
                  "T must have a static type_name() method");

    static_assert(std::is_nothrow_move_constructible_v<T>,
                  "T must be noexcept movable for container compatibility");
}
```

Using multiple `static_assert`s - one per requirement - means the first failure tells you exactly what's missing. A single combined concept check would stop at the first failure inside the concept, and you might not know which part failed without reading the concept definition carefully.

### Compile-Time Diagnostics with consteval

You can push error message quality even further by using `consteval` functions to validate string-like input at compile time. If the validation fails inside a `consteval` function, the error appears at the call site - exactly where the bad input was provided:

```cpp
#include <string_view>
#include <algorithm>

consteval bool validate_identifier(std::string_view name) {
    if (name.empty()) return false;
    if (!std::isalpha(name[0]) && name[0] != '_') return false;
    return std::all_of(name.begin(), name.end(),
                       [](char c) { return std::isalnum(c) || c == '_'; });
}

template<size_t N>
struct FieldName {
    char data[N];
    consteval FieldName(const char (&str)[N]) {
        std::copy(str, str + N, data);
        static_assert(validate_identifier({str, N - 1}),
                      "Field name must be a valid identifier");
    }
};
```

The `static_assert` inside the `consteval` constructor fires immediately if you construct a `FieldName` with an invalid string like `"123bad"` or `"has space"`. The error is caught at the definition site, not somewhere deep in generated code.

---

## Self-Assessment

### Q1: Why are concept error messages better than SFINAE

Concepts report which specific requirement failed: "the expression `obj.serialize(os)` is not valid." SFINAE reports "no matching function" and dumps the entire overload set. Concepts diagnose the WHY; SFINAE only says WHAT failed.

The reason the difference matters in practice is that SFINAE errors are often pages long - the compiler lists every candidate and why each one didn't match. When you're the user of a library, you typically don't care about the library's internal overload set. You just want to know what your type is missing. Concepts give you that directly.

### Q2: Show how to test multiple requirements with individual error messages

Use multiple `static_assert`s, each testing one requirement. The first failure gives a specific message. With a single concept, you get one combined error. With individual assertions, each failure is isolated and named.

The pattern is especially useful inside a template function body as a "type contract" - you're saying "before any real code runs, verify that T meets all of these requirements, and tell me which one failed if it doesn't."

### Q3: When to use static_assert vs concepts vs SFINAE

- **static_assert**: hard errors inside a template body - "this type is NOT supported." Use when you want a clear error at the point of instantiation with a custom message.
- **Concepts**: soft errors in overload resolution - "try another overload." Use when you want the compiler to pick a different overload if this one doesn't match, and when you want good diagnostic messages when no overload matches.
- **SFINAE**: legacy code, fine-grained overload control before C++20. Prefer concepts for new code.

The rule of thumb is: if you're writing new C++20 code and you want to constrain a template, use concepts. If you're inside a template body and want to fail loudly with a custom message, use `static_assert`. SFINAE is mainly for maintaining compatibility with pre-C++20 code or for unusual cases where concepts don't quite fit.

---

## Notes

- `static_assert(false)` is ill-formed even in uninstantiated templates (C++23 relaxes this).
- Use `static_assert` in the class body to fail early with clear messages.
- Concept subsumption provides better overload resolution than SFINAE.
- Combine concepts and static_assert: concepts for overload selection, static_assert for hard requirements.
