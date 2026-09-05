# Understand `decltype` and `decltype(auto)`

**Category:** Type System and Deduction  
**Standard:** C++11, C++14 (`decltype`)  
**Reference:** <https://en.cppreference.com/w/cpp/language/decltype>  

---

## Topic Overview

### What Is `decltype`

`decltype` inspects the **declared type** of an entity or the **type and value category** of an expression - without evaluating it. Think of it as asking the compiler "what type does this thing have?" at zero runtime cost.

### The Two Rules of `decltype`

This is the **single most important** thing to understand about `decltype`. There are exactly two modes, and which one fires depends on the form of the operand:

| Form | Input | Result |
| --- | --- | --- |
| `decltype(entity)` | Unparenthesized name of variable, function, member, enumerator | The **declared type** of that entity |
| `decltype(expr)` | Any other expression (including parenthesized names) | Type modified by value category: `T` if prvalue, `T&` if lvalue, `T&&` if xvalue |

The reason this trips people up is that adding parentheses looks harmless, but it switches you from Rule 1 to Rule 2.

### `decltype(x)` vs `decltype((x))` - The Critical Difference

Here's what that looks like concretely. A single pair of parentheses changes everything:

```cpp
int x = 42;

// decltype(x)   -> int        (x is an unparenthesized variable name -> declared type)
// decltype((x)) -> int&       ((x) is an lvalue expression -> T& )

const int& r = x;
// decltype(r)   -> const int& (declared type of r)
// decltype((r)) -> const int& ((r) is lvalue expression of type const int -> const int&)
```

The parentheses turn a **name lookup** (Rule 1) into an **expression analysis** (Rule 2).

### `decltype` with Different Expression Categories

To internalize the rules, run through the full set. `decltype` on a function call gives you the function's return type as declared - without calling it:

```cpp
int  x = 10;
int& r = x;
int  f();       // returns prvalue
int& g();       // returns lvalue
int&& h();      // returns xvalue

// Names (Rule 1)
decltype(x)     // int
decltype(r)     // int&

// Expressions (Rule 2)
decltype(f())   // int     (prvalue -> T)
decltype(g())   // int&    (lvalue  -> T&)
decltype(h())   // int&&   (xvalue  -> T&&)
decltype(x+0)   // int     (prvalue -> T)
decltype((x))   // int&    (lvalue  -> T&)
decltype(std::move(x))  // int&&  (xvalue -> T&&)
```

### What Is `decltype(auto)`

`decltype(auto)` (C++14) is a placeholder that applies **decltype rules** to the initializer expression. Where plain `auto` always decays and strips references, `decltype(auto)` preserves the exact type including references and value category:

```cpp
int x = 42;

auto        a = x;    // int        (auto decays, copies)
auto&       b = x;    // int&       (explicit reference)
decltype(auto) c = x;    // int     (decltype(x) -> int, name rule)
decltype(auto) d = (x);  // int&   (decltype((x)) -> int&, expression rule!)
```

**Warning:** `decltype(auto) d = (x)` creates a reference! This is a common source of bugs and the example to memorize.

### `decltype(auto)` for Return Types

The primary use case is preserving the exact type and value category from a `return` expression. You need this whenever you're writing a forwarding wrapper that must not silently strip references from whatever it wraps:

```cpp
// auto return: always decays, strips references
auto bad_forward() {
    int x = 42;
    int& r = x;
    return r;       // Returns int (auto strips the reference!)
}

// decltype(auto): preserves the exact type
template<typename F, typename... Args>
decltype(auto) perfect_invoke(F&& f, Args&&... args) {
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
    // If f returns int&,  this returns int&
    // If f returns int,   this returns int
    // If f returns int&&, this returns int&&
}
```

With `auto`, your wrapper always returns a value copy and callers can never modify the underlying object through the result. With `decltype(auto)`, the wrapper is transparent.

### `decltype` in Trailing Return Types

You'll also see `decltype` in trailing return type position. This is how you express "the return type depends on what `t + u` produces" - something `auto` alone can't do in C++11:

```cpp
// When return type depends on parameters:
template<typename T, typename U>
auto add(T t, U u) -> decltype(t + u) {
    return t + u;
}

// Useful for SFINAE:
template<typename T>
auto size(const T& c) -> decltype(c.size()) {
    return c.size();
}
// If T has no .size(), the function is removed from overload set (SFINAE)
```

The SFINAE example is particularly useful: the function disappears from the overload set for any type that doesn't have `.size()`, giving you clean constraint checking before C++20 concepts.

### `decltype` vs `auto` Deduction Comparison

| Feature | `auto` | `decltype(auto)` | `decltype(expr)` |
| --- | --- | --- | --- |
| Strips references | Yes | No | No |
| Strips top-level cv | Yes | No | No |
| Array decay | Yes | No | No |
| Function decay | Yes | No | No |
| Preserves value category | No | Yes | Yes |
| Evaluates expression | - | No | No |

### Practical Guidelines

When to reach for each tool:

```cpp
// USE decltype(auto) when:
//   Writing generic forwarding wrappers
//   Return type must exactly match the inner expression
//   Implementing proxy objects / expression templates

// USE decltype(expr) when:
//   Trailing return types with dependent types
//   SFINAE on expression validity
//   Template metaprogramming (type computation)

// AVOID decltype(auto) when:
//   Returning local variables (dangling reference risk!)
//   Simple functions where auto suffices
//   When parentheses might accidentally create references
```

---

## Self-Assessment

### Q1: Explain why `decltype(x)` and `decltype((x))` differ when x is an lvalue variable

`decltype` has two distinct rules depending on the form of its operand:

**Rule 1 - Named entity (unparenthesized):**  
When the operand is an unparenthesized id-expression (a simple variable name, member access, etc.), `decltype` yields the **declared type** of that entity.

**Rule 2 - Expression (anything else):**  
When the operand is any other expression (including a parenthesized name), `decltype` yields a type modified by the expression's value category:

- **prvalue** -> `T`
- **lvalue** -> `T&`
- **xvalue** -> `T&&`

The example below makes this concrete, and also shows the dangerous `decltype(auto) b = (x)` pattern in action so you recognize it when you see it:

```cpp
#include <iostream>
#include <type_traits>

int main() {
    int x = 42;
    const int cx = 10;

    // Rule 1: decltype(name) -> declared type
    static_assert(std::is_same_v<decltype(x), int>);
    static_assert(std::is_same_v<decltype(cx), const int>);

    // Rule 2: decltype((name)) -> expression value category
    // (x) is an lvalue expression of type int -> int&
    static_assert(std::is_same_v<decltype((x)), int&>);
    // (cx) is an lvalue expression of type const int -> const int&
    static_assert(std::is_same_v<decltype((cx)), const int&>);

    // Demonstration with decltype(auto)
    decltype(auto) a = x;    // decltype(x)   = int       -> copies
    decltype(auto) b = (x);  // decltype((x)) = int&      -> reference!

    b = 100;  // Modifies x through the reference!
    std::cout << "x after b=100: " << x << "\n";  // 100

    // More examples of Rule 2
    static_assert(std::is_same_v<decltype(x + 0), int>);       // prvalue -> T
    static_assert(std::is_same_v<decltype(std::move(x)), int&&>); // xvalue -> T&&

    std::cout << "All assertions passed.\n";
    return 0;
}
```

**Output:**

```text
x after b=100: 100
All assertions passed.
```

**Key insight:** Parentheses change the operand from a **name** (Rule 1) to an **expression** (Rule 2). Since a variable name used as an expression is an lvalue, Rule 2 adds `&`. This is why `decltype(auto) d = (x)` silently creates a reference - a notorious pitfall.

### Q2: Use `decltype(auto)` to write a forwarding wrapper that preserves reference semantics

Notice how `decltype(auto)` as the return type makes the wrapper fully transparent - it passes through `int&`, `const int&`, and `int` all correctly without you doing anything special for each case:

```cpp
#include <iostream>
#include <type_traits>
#include <utility>
#include <cstddef>
#include <functional>

// A container with element access returning references
struct Container {
    int data[3] = {10, 20, 30};

    int&       at(std::size_t i)       { return data[i]; }
    const int& at(std::size_t i) const { return data[i]; }

    int value_copy(std::size_t i) const { return data[i]; }
};

// Forwarding wrapper using decltype(auto)
// Preserves EXACT return type: int&, const int&, or int
template<typename Obj, typename Method, typename... Args>
decltype(auto) forward_call(Obj&& obj, Method method, Args&&... args) {
    return (std::forward<Obj>(obj).*method)(std::forward<Args>(args)...);
}

// Logging wrapper: logs then forwards perfectly
template<typename F, typename... Args>
decltype(auto) logged_call(const char* name, F&& f, Args&&... args) {
    std::cout << "[LOG] Calling " << name << "\n";
    decltype(auto) result = std::invoke(std::forward<F>(f),
                                         std::forward<Args>(args)...);
    // If result is a reference, we return the reference
    // If result is a value, we return the value (no extra copy due to NRVO)
    if constexpr (std::is_reference_v<decltype(result)>) {
        std::cout << "[LOG] Returned reference\n";
    } else {
        std::cout << "[LOG] Returned value\n";
    }
    return static_cast<decltype(result)>(result);
}

int main() {
    Container c;

    // Forward mutable reference
    decltype(auto) ref = forward_call(c, &Container::at, 1);
    static_assert(std::is_same_v<decltype(ref), int&>);
    ref = 99;
    std::cout << "c.data[1] after modification: " << c.data[1] << "\n";  // 99

    // Forward const reference
    const Container& cc = c;
    decltype(auto) cref = forward_call(cc, &Container::at, 0);
    static_assert(std::is_same_v<decltype(cref), const int&>);
    std::cout << "cref = " << cref << "\n";

    // Forward value
    decltype(auto) val = forward_call(cc, &Container::value_copy, 2);
    static_assert(std::is_same_v<decltype(val), int>);
    std::cout << "val = " << val << "\n";

    // Logged call
    auto get_ref = [](Container& c, int i) -> int& { return c.data[i]; };
    decltype(auto) logged_ref = logged_call("get_ref", get_ref, c, 0);
    static_assert(std::is_same_v<decltype(logged_ref), int&>);

    return 0;
}
```

**Output:**

```text
c.data[1] after modification: 99
cref = 10
val = 30
[LOG] Calling get_ref
[LOG] Returned reference
```

**How this works:**

- `decltype(auto)` as return type applies `decltype` to the `return` expression
- If the inner function returns `int&`, the wrapper returns `int&` - the reference is preserved
- If the inner function returns `int`, the wrapper returns `int` - no unnecessary reference
- Using plain `auto` would always strip references, breaking the forwarding semantics
- The `static_cast<decltype(result)>(result)` in `logged_call` ensures we don't accidentally turn an xvalue into an lvalue

### Q3: Show how `decltype` is used in trailing return types to express complex dependent types

Each example here targets a different scenario where you actually need `decltype` in a trailing position. Pay attention to Example 2 - that's the SFINAE trick that predates C++20 concepts:

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <type_traits>

// Example 1: Dependent return type from binary operation
template<typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) {
    return a * b;
}

// Example 2: SFINAE with decltype — only valid if T has .size()
template<typename Container>
auto get_size(const Container& c) -> decltype(c.size()) {
    return c.size();
}

// Example 3: Complex dependent type — iterator value type
template<typename Container>
auto front_element(Container& c) -> decltype(*c.begin()) {
    return *c.begin();
}

// Example 4: Chained member access
template<typename T>
auto inner_value(T& obj) -> decltype(obj.get().value) {
    return obj.get().value;
}

// Example 5: decltype in non-type template parameter context
template<typename T, typename U>
struct ResultOf {
    using type = decltype(std::declval<T>() + std::declval<U>());
};

// Helper using std::declval to compute types without constructing objects
struct NonConstructible {
    NonConstructible() = delete;
    int operator+(const NonConstructible&) const;
};

int main() {
    // Example 1: mixed arithmetic
    auto r1 = multiply(3, 2.5);
    static_assert(std::is_same_v<decltype(r1), double>);
    std::cout << "3 * 2.5 = " << r1 << "\n";

    // Example 2: SFINAE — works for containers
    std::vector<int> v = {1, 2, 3};
    auto sz = get_size(v);
    std::cout << "vector size: " << sz << "\n";

    // get_size(42);  // ERROR: int has no .size() — SFINAE removes this overload

    // Example 3: returns reference into container
    std::vector<std::string> words = {"hello", "world"};
    decltype(auto) front = front_element(words);
    static_assert(std::is_same_v<decltype(front), std::string&>);
    front = "MODIFIED";
    std::cout << "words[0] = " << words[0] << "\n";  // MODIFIED

    // Example 5: type computation without construction
    static_assert(std::is_same_v<ResultOf<int, double>::type, double>);
    // Even works for deleted-constructor types:
    static_assert(std::is_same_v<
        decltype(std::declval<NonConstructible>() + std::declval<NonConstructible>()),
        int
    >);

    std::cout << "All type checks passed.\n";
    return 0;
}
```

**Output:**

```text
3 * 2.5 = 7.5
vector size: 3
words[0] = MODIFIED
All type checks passed.
```

**How this works:**

- **Trailing return type** `-> decltype(expr)` lets us express return types that depend on parameter types
- The expression in `decltype` is **not evaluated** - it's only used for type deduction
- `std::declval<T>()` creates an unevaluated reference to `T`, letting us query operations on types that can't be constructed
- SFINAE: if the expression in `decltype` is invalid, the function is silently removed from the overload set
- In C++14+, `auto` return type deduction often replaces trailing `decltype`, but `decltype` trailing returns are still needed for **SFINAE** and when you need the **exact expression type** (not decayed)

---

## Notes

- **`decltype(auto)` with structured bindings:** Be careful - `decltype(auto) [a, b] = pair;` is not valid. Structured bindings have their own deduction rules.
- **`decltype` does not evaluate:** `decltype(f())` does not call `f`. This is guaranteed by the standard.
- **Common bug:** `decltype(auto)` returning a local variable in parentheses creates a dangling reference:

  ```cpp
  decltype(auto) bad() {
      int x = 42;
      return (x);  // decltype((x)) = int& -> DANGLING REFERENCE!
  }
  ```

- **`decltype` with lambdas:** Each lambda has a unique unnamed type, so `decltype` of a lambda is the only way to name its type (before C++20 template lambdas).
- **Prefer `auto` over `decltype(auto)`** unless you specifically need reference/value category preservation. `auto` is safer because it always makes a copy.
