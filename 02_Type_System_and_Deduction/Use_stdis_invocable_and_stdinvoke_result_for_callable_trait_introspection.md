# Use `std::is_invocable` and `std::invoke_result` for Callable Trait Introspection

**Category:** Type System and Deduction  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/types/is_invocable>  

---

## Topic Overview

### What Are Callable Traits

C++17 introduced a family of type traits that let you **query callable objects** (functions, lambdas, functors, member pointers) at compile time. If you've ever needed to write a higher-order function template and struggled with "what does this callable actually return?", these are your tools:

| Trait | Checks |
| --- | --- |
| `std::is_invocable<F, Args...>` | Can `F` be called with `Args...`? |
| `std::is_invocable_r<R, F, Args...>` | Can `F(Args...)` be converted to `R`? |
| `std::is_nothrow_invocable<F, Args...>` | Is `F(Args...)` noexcept? |
| `std::invoke_result_t<F, Args...>` | What type does `F(Args...)` return? |

### Why Not Just Use `decltype(f(args...))`

The reason these traits exist instead of raw `decltype` is that `std::invoke` handles **all callable forms** uniformly - not just regular function calls. Member function pointers and member data pointers are part of that:

```cpp
#include <functional>

struct Foo {
    int value = 42;
    int method(double d) { return static_cast<int>(d + value); }
};

Foo foo;

// std::invoke handles:
std::invoke(std::plus{}, 1, 2);           // free function object
std::invoke(&Foo::method, foo, 3.14);     // member function pointer + object
std::invoke(&Foo::value, foo);            // member data pointer + object
std::invoke([](int x){ return x*2; }, 5); // lambda
```

The traits mirror this - `is_invocable<&Foo::method, Foo&, double>` is `true` even though `&Foo::method(foo, 3.14)` won't compile directly. That's the key advantage over a raw `decltype`.

### `std::is_invocable` - Check Callability

Before calling a generic callable, you often want to verify at compile time that the call is well-formed. Here is how to do that:

```cpp
#include <type_traits>

auto lambda = [](int x, double y) -> bool { return x > y; };

// Can lambda be called with (int, double)?
static_assert(std::is_invocable_v<decltype(lambda), int, double>);

// Can it be called with (int)?  No - wrong arity
static_assert(!std::is_invocable_v<decltype(lambda), int>);

// Can it be called with (string, string)?  No - wrong types
static_assert(!std::is_invocable_v<decltype(lambda), std::string, std::string>);
```

### `std::invoke_result_t` - Get Return Type

Once you know a call is valid, you can ask what it returns:

```cpp
// What does lambda(int, double) return?
using R = std::invoke_result_t<decltype(lambda), int, double>;
static_assert(std::is_same_v<R, bool>);

// What does std::plus<int>()(int, int) return?
using R2 = std::invoke_result_t<std::plus<int>, int, int>;
static_assert(std::is_same_v<R2, int>);
```

### The Old Way vs The New Way

Before C++17, getting the return type of a callable required verbose manual `decltype` expressions - and they couldn't handle member pointers at all:

```cpp
// OLD (pre-C++17): manual decltype with declval - ugly and incomplete
template<typename F, typename... Args>
using old_result = decltype(std::declval<F>()(std::declval<Args>()...));
// Problem: doesn't work for member function pointers or member data pointers!

// NEW (C++17): clean and handles ALL callable forms
template<typename F, typename... Args>
using new_result = std::invoke_result_t<F, Args...>;
```

### Constraining Templates with `is_invocable`

You can use `is_invocable` both as a `requires` constraint and inside `if constexpr` to adapt behavior at compile time:

```cpp
// Higher-order function: only accepts callable with matching signature
template<typename F, typename... Args>
    requires std::is_invocable_v<F, Args...>
auto apply(F&& f, Args&&... args) {
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}

// Or with concepts (C++20):
template<typename F, typename... Args>
    requires std::invocable<F, Args...>  // concept version
auto apply2(F&& f, Args&&... args) {
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}
```

---

## Self-Assessment

### Q1: Use `std::is_invocable_v<F, Args...>` to constrain a function template to callable `F`

This builds a `safe_call` wrapper and an `EventHandler` class that both use the trait to enforce correctness at the call site:

```cpp
#include <iostream>
#include <type_traits>
#include <functional>
#include <string>
#include <concepts>

// A higher-order function constrained to accept only valid callables
template<typename F, typename... Args>
    requires std::is_invocable_v<F, Args...>
auto safe_call(F&& f, Args&&... args)
    -> std::invoke_result_t<F, Args...>
{
    std::cout << "  [safe_call] invoking callable\n";
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}

// A callback registration system with compile-time validation
template<typename Callback>
    requires std::is_invocable_v<Callback, int, const std::string&>
class EventHandler {
    Callback cb_;
public:
    explicit EventHandler(Callback cb) : cb_(std::move(cb)) {}

    void fire(int code, const std::string& msg) {
        std::invoke(cb_, code, msg);
    }
};

// Deduction guide
template<typename Callback>
EventHandler(Callback) -> EventHandler<Callback>;

struct Functor {
    int operator()(double d) { return static_cast<int>(d * 2); }
};

int main() {
    // Lambda
    auto result1 = safe_call([](int a, int b) { return a + b; }, 3, 4);
    std::cout << "Lambda result: " << result1 << "\n";

    // Functor
    auto result2 = safe_call(Functor{}, 3.14);
    std::cout << "Functor result: " << result2 << "\n";

    // std::function
    std::function<std::string(int)> to_str = [](int x) { return std::to_string(x); };
    auto result3 = safe_call(to_str, 42);
    std::cout << "std::function result: " << result3 << "\n";

    // Event handler with constrained callback
    EventHandler handler([](int code, const std::string& msg) {
        std::cout << "Event [" << code << "]: " << msg << "\n";
    });
    handler.fire(200, "OK");

    // These would fail at compile time:
    // safe_call([](int x) {}, "wrong type");           // string is not int
    // safe_call(42, 1, 2);                              // 42 is not callable
    // EventHandler bad_handler([](double d) {});        // wrong callback signature

    return 0;
}
```

**Output:**

```text
  [safe_call] invoking callable
Lambda result: 7
  [safe_call] invoking callable
Functor result: 6
  [safe_call] invoking callable
std::function result: 42
Event [200]: OK
```

### Q2: Use `invoke_result_t<F, Args...>` to deduce the return type of a callable in a higher-order function

Here is where `invoke_result_t` really earns its keep - it lets you write a `map` function whose return container type is automatically deduced from the transformation:

```cpp
#include <iostream>
#include <type_traits>
#include <functional>
#include <vector>
#include <string>
#include <optional>

// map: applies F to each element, deduces result container type
template<typename F, typename T>
auto map(const std::vector<T>& input, F&& f)
    -> std::vector<std::invoke_result_t<F, const T&>>
{
    using Result = std::invoke_result_t<F, const T&>;
    std::vector<Result> output;
    output.reserve(input.size());
    for (const auto& elem : input) {
        output.push_back(std::invoke(f, elem));
    }
    return output;
}

// retry: calls F, returns optional<R> where R is the return type of F
template<typename F>
auto retry(F&& f, int max_attempts)
    -> std::optional<std::invoke_result_t<F>>
{
    using R = std::invoke_result_t<F>;  // Deduce return type
    for (int i = 0; i < max_attempts; ++i) {
        try {
            return std::invoke(std::forward<F>(f));
        } catch (...) {
            std::cout << "  Attempt " << (i+1) << " failed\n";
        }
    }
    return std::nullopt;
}

// compose: creates a new callable that applies g then f
template<typename F, typename G>
auto compose(F f, G g) {
    return [f = std::move(f), g = std::move(g)](auto&&... args)
        -> std::invoke_result_t<F, std::invoke_result_t<G, decltype(args)...>>
    {
        return std::invoke(f, std::invoke(g, std::forward<decltype(args)>(args)...));
    };
}

int main() {
    // map: int -> string (return type deduced via invoke_result_t)
    std::vector<int> nums = {1, 2, 3, 4, 5};
    auto strings = map(nums, [](int x) { return std::to_string(x) + "!"; });

    static_assert(std::is_same_v<decltype(strings), std::vector<std::string>>);
    std::cout << "map result: ";
    for (const auto& s : strings) std::cout << s << " ";
    std::cout << "\n";

    // map: int -> double
    auto doubled = map(nums, [](int x) { return x * 2.5; });
    static_assert(std::is_same_v<decltype(doubled), std::vector<double>>);
    std::cout << "doubled: ";
    for (double d : doubled) std::cout << d << " ";
    std::cout << "\n";

    // retry: return type deduced from callable
    int counter = 0;
    auto result = retry([&]() -> int {
        if (++counter < 3) throw std::runtime_error("not yet");
        return 42;
    }, 5);

    std::cout << "\nretry result: " << result.value_or(-1) << "\n";

    // compose: f(g(x))
    auto double_then_string = compose(
        [](double d) { return std::to_string(d); },
        [](int x) { return x * 2.0; }
    );
    std::cout << "compose(2): " << double_then_string(21) << "\n";

    return 0;
}
```

**Output:**

```text
map result: 1! 2! 3! 4! 5!
doubled: 2.5 5 7.5 10 12.5

  Attempt 1 failed
  Attempt 2 failed
retry result: 42
compose(2): 42.000000
```

### Q3: Show how these traits replace manual `decltype(std::declval<F>()(std::declval<Args>()...))` patterns

The old `decltype + declval` idiom breaks silently on member pointers. Here is a direct comparison that shows exactly where the old pattern fails and why `invoke_result_t` is the correct replacement:

```cpp
#include <iostream>
#include <type_traits>
#include <functional>
#include <string>

// === OLD PATTERN (pre-C++17): manual decltype + declval ===

// Problem 1: doesn't handle member function pointers
template<typename F, typename... Args>
using old_result_t = decltype(std::declval<F>()(std::declval<Args>()...));

// Problem 2: SFINAE with decltype is verbose
template<typename F, typename... Args,
         typename = decltype(std::declval<F>()(std::declval<Args>()...))>
auto old_apply(F&& f, Args&&... args) {
    return f(std::forward<Args>(args)...);
}

// === NEW PATTERN (C++17): invoke_result_t + is_invocable ===

// Clean, handles ALL callable forms
template<typename F, typename... Args>
    requires std::is_invocable_v<F, Args...>
std::invoke_result_t<F, Args...> new_apply(F&& f, Args&&... args) {
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}

// === Demonstration of what old pattern CAN'T do ===
struct MyClass {
    int value = 42;
    std::string describe(int detail_level) const {
        return "value=" + std::to_string(value) + " (level " + std::to_string(detail_level) + ")";
    }
};

int main() {
    std::cout << std::boolalpha;

    // Regular callable: both work
    auto lambda = [](int x) { return x * 2; };

    using OldR = old_result_t<decltype(lambda), int>;
    using NewR = std::invoke_result_t<decltype(lambda), int>;
    static_assert(std::is_same_v<OldR, int>);
    static_assert(std::is_same_v<NewR, int>);

    // Member function pointer: OLD fails, NEW works
    // using BadR = old_result_t<decltype(&MyClass::describe), MyClass&, int>;
    // ERROR: can't call &MyClass::describe like a function

    using GoodR = std::invoke_result_t<decltype(&MyClass::describe), MyClass&, int>;
    static_assert(std::is_same_v<GoodR, std::string>);

    // Member data pointer: OLD fails, NEW works
    // using BadR2 = old_result_t<decltype(&MyClass::value), MyClass&>;
    // ERROR

    using GoodR2 = std::invoke_result_t<decltype(&MyClass::value), MyClass&>;
    static_assert(std::is_same_v<GoodR2, int&>);  // returns reference to member!

    // is_invocable: clean SFINAE without manual decltype tricks
    std::cout << "=== is_invocable checks ===\n";
    std::cout << "lambda(int): "
              << std::is_invocable_v<decltype(lambda), int> << "\n";
    std::cout << "&MyClass::describe(MyClass&, int): "
              << std::is_invocable_v<decltype(&MyClass::describe), MyClass&, int> << "\n";
    std::cout << "&MyClass::value(MyClass&): "
              << std::is_invocable_v<decltype(&MyClass::value), MyClass&> << "\n";
    std::cout << "lambda(string): "
              << std::is_invocable_v<decltype(lambda), std::string> << "\n";

    // Using new_apply with member function pointer
    MyClass obj;
    auto desc = new_apply(&MyClass::describe, obj, 3);
    std::cout << "\nnew_apply with member function: " << desc << "\n";

    auto val = new_apply(&MyClass::value, obj);
    std::cout << "new_apply with member data: " << val << "\n";

    std::cout << "\nSummary: invoke_result_t replaces manual decltype+declval\n";
    std::cout << "  Handles functions, lambdas, functors, member ptrs\n";
    std::cout << "  Cleaner syntax, no declval boilerplate\n";
    std::cout << "  Works with std::invoke (the INVOKE concept)\n";

    return 0;
}
```

**Output:**

```text
=== is_invocable checks ===
lambda(int): true
&MyClass::describe(MyClass&, int): true
&MyClass::value(MyClass&): true
lambda(string): false

new_apply with member function: value=42 (level 3)
new_apply with member data: 42

Summary: invoke_result_t replaces manual decltype+declval
  Handles functions, lambdas, functors, member ptrs
  Cleaner syntax, no declval boilerplate
  Works with std::invoke (the INVOKE concept)
```

---

## Notes

- **C++20 concept versions:** `std::invocable<F, Args...>` and `std::regular_invocable<F, Args...>` are concept-based equivalents. Prefer these in concept constraints.
- **`is_invocable_r<R, F, Args...>`** additionally checks that the return type is convertible to `R`. Useful when your higher-order function requires a specific return type.
- **`result_of` is deprecated.** `std::result_of<F(Args...)>` was the C++11 version but had confusing syntax and was deprecated in C++17, removed in C++20. Always use `invoke_result_t`.
- `std::invoke` adds a small overhead for member pointers (it checks the callable type). For function objects and lambdas, compilers optimize it to a direct call.
