# Use std::optional monadic operations: transform, and_then, or_else (C++23)

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/optional/transform>  

---

## Topic Overview

C++23 adds three monadic operations to `std::optional` - `transform`, `and_then`, and `or_else` - so you can chain operations that might fail without writing nested `if (opt.has_value())` checks at every step. The idea comes from functional programming: each operation either threads the value forward through the next step, or short-circuits to empty. You get flat, readable pipelines instead of pyramids of null-checks.

### The Three Operations

| Operation | Signature (simplified) | Behavior |
| --- | --- | --- |
| `transform(f)` | `optional<T> -> (T->U) -> optional<U>` | If engaged, apply `f` to the value and wrap result in `optional<U>`. If empty, return empty `optional<U>`. |
| `and_then(f)` | `optional<T> -> (T->optional<U>) -> optional<U>` | If engaged, apply `f` (which itself returns `optional`). If empty, return empty. **Flat-maps** - avoids `optional<optional<U>>`. |
| `or_else(f)` | `optional<T> -> (()->optional<T>) -> optional<T>` | If engaged, return `*this`. If empty, call `f()` and return its result. |

### Visual Pipeline

The contrast between the monadic style and the manual if-check style makes the motivation clear. Both do the same thing - the monadic version just reads as a straight pipeline:

```cpp
optional<string> username = get_user(id);

username.and_then(find_profile)      // optional<Profile> — may fail
        .transform(get_avatar_url)    // optional<string> — transforms value
        .or_else([] { return optional<string>{"default.png"}; });
                                      // fallback if any step produced nullopt

// Without monadic ops (C++20):
//   auto user = get_user(id);
//   if (!user) return "default.png";
//   auto profile = find_profile(*user);
//   if (!profile) return "default.png";
//   return get_avatar_url(*profile);
```

### Core Examples

Here's a full pipeline that parses a string, validates the result, takes a square root, and converts back to string - with a fallback at the end. If any step produces `nullopt`, everything after it is skipped:

```cpp
#include <optional>
#include <string>
#include <iostream>
#include <charconv>

// transform: T -> U (wraps result in optional)
std::optional<int> parse_int(const std::string& s) {
    int result{};
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), result);
    if (ec == std::errc{} && ptr == s.data() + s.size())
        return result;
    return std::nullopt;
}

// and_then: T -> optional<U> (flat-maps)
std::optional<double> safe_sqrt(int n) {
    if (n < 0) return std::nullopt;
    return std::sqrt(static_cast<double>(n));
}

int main() {
    std::optional<std::string> input = "49";

    // Pipeline: parse string -> check non-negative -> sqrt -> double result
    auto result = input
        .and_then(parse_int)          // optional<string> -> optional<int>
        .and_then(safe_sqrt)          // optional<int> -> optional<double>
        .transform([](double d) {     // optional<double> -> optional<std::string>
            return std::to_string(d);
        })
        .or_else([]() -> std::optional<std::string> {
            return "computation failed";
        });

    std::cout << *result << "\n"; // "7.000000"

    // With empty input:
    std::optional<std::string> bad_input = std::nullopt;
    auto result2 = bad_input
        .and_then(parse_int)
        .and_then(safe_sqrt)
        .transform([](double d) { return std::to_string(d); })
        .or_else([]() -> std::optional<std::string> {
            return "computation failed";
        });

    std::cout << *result2 << "\n"; // "computation failed"
}
```

### transform vs and_then

The distinction is important: use `transform` when your function returns a plain value, and `and_then` when it returns an `optional`. Getting it backwards creates an `optional<optional<T>>` - double-wrapped and awkward:

```cpp
#include <optional>
#include <iostream>

int main() {
    std::optional<int> val = 42;

    // transform: f returns a plain value -> wrapped in optional
    auto doubled = val.transform([](int x) { return x * 2; });
    // doubled is optional<int>{84}

    // and_then: f returns optional -> NOT double-wrapped
    auto checked = val.and_then([](int x) -> std::optional<int> {
        if (x > 100) return std::nullopt;
        return x;
    });
    // checked is optional<int>{42}  (not optional<optional<int>>)

    // If you used transform with a function returning optional:
    auto bad = val.transform([](int x) -> std::optional<int> {
        return x;
    });
    // bad is optional<optional<int>>{optional<int>{42}} — nested! Use and_then instead.

    std::cout << *doubled << "\n";  // 84
    std::cout << *checked << "\n";  // 42
}
```

---

## Self-Assessment

### Q1: Chain optional lookups with and_then to avoid nested if-optional checks

**Answer:**

Each function in the chain returns `optional`. If any of them returns `nullopt`, the rest of the chain is skipped entirely and `nullopt` propagates to the end. No special wiring required - that's the whole point:

```cpp
#include <optional>
#include <string>
#include <unordered_map>
#include <iostream>

// Simulated database lookups — each may fail
std::unordered_map<int, std::string> users = {{1, "alice"}, {2, "bob"}};
std::unordered_map<std::string, std::string> emails = {{"alice", "alice@ex.com"}};
std::unordered_map<std::string, bool> verified = {{"alice@ex.com", true}};

std::optional<std::string> find_user(int id) {
    auto it = users.find(id);
    if (it != users.end()) return it->second;
    return std::nullopt;
}

std::optional<std::string> find_email(const std::string& name) {
    auto it = emails.find(name);
    if (it != emails.end()) return it->second;
    return std::nullopt;
}

std::optional<bool> is_verified(const std::string& email) {
    auto it = verified.find(email);
    if (it != verified.end()) return it->second;
    return std::nullopt;
}

int main() {
    // WITHOUT and_then (C++20 style):
    // auto user = find_user(1);
    // if (!user) { /* handle */ }
    // auto email = find_email(*user);
    // if (!email) { /* handle */ }
    // auto ver = is_verified(*email);
    // if (!ver) { /* handle */ }

    // WITH and_then (C++23) — flat, composable:
    auto result = std::optional<int>{1}
        .and_then(find_user)       // optional<int> -> optional<string>
        .and_then(find_email)      // optional<string> -> optional<string>
        .and_then(is_verified);    // optional<string> -> optional<bool>

    if (result)
        std::cout << "Verified: " << std::boolalpha << *result << "\n";
    else
        std::cout << "Lookup failed at some step\n";
    // Output: Verified: true

    // User 3 doesn't exist — chain short-circuits at find_user
    auto result2 = std::optional<int>{3}
        .and_then(find_user)
        .and_then(find_email)
        .and_then(is_verified);

    std::cout << result2.has_value() << "\n"; // 0 (false)
}
```

**Explanation:** `and_then` chains functions that each return `optional`. If any step returns `nullopt`, the rest of the chain is skipped and `nullopt` propagates. This is the "monadic bind" pattern - it eliminates the pyramid of if-checks while preserving short-circuit behavior.

### Q2: Use transform to apply a function to an optional value if present, without unwrapping

**Answer:**

`transform` is like applying a lambda to the inside of the box - if the box is empty, nothing happens and you get an empty box back. The function never has to deal with `nullopt` explicitly:

```cpp
#include <optional>
#include <string>
#include <iostream>
#include <cctype>
#include <algorithm>

std::string to_upper(std::string s) {
    std::transform(s.begin(), s.end(), s.begin(),
                   [](unsigned char c) { return std::toupper(c); });
    return s;
}

int main() {
    std::optional<std::string> name = "alice";
    std::optional<std::string> empty_name = std::nullopt;

    // transform: apply to_upper only if value is present
    auto upper = name.transform(to_upper);
    std::cout << upper.value_or("(none)") << "\n"; // "ALICE"

    auto upper2 = empty_name.transform(to_upper);
    std::cout << upper2.value_or("(none)") << "\n"; // "(none)"

    // Chaining transforms:
    auto result = std::optional<int>{42}
        .transform([](int x) { return x * 2; })        // optional<int>{84}
        .transform([](int x) { return std::to_string(x); })  // optional<string>{"84"}
        .transform([](const std::string& s) { return "Result: " + s; });
        // optional<string>{"Result: 84"}

    std::cout << *result << "\n"; // "Result: 84"

    // Transform with empty — nothing executes
    auto empty_result = std::optional<int>{}
        .transform([](int x) {
            std::cout << "This never prints\n";
            return x * 2;
        });
    std::cout << empty_result.has_value() << "\n"; // 0
}
```

**Explanation:** `transform` is like `std::ranges::transform` but for `optional`. It applies a function to the contained value and wraps the result in a new `optional`. If the source is empty, the function is never called and an empty optional is returned. Unlike `and_then`, the function must return a plain value (not `optional`).

### Q3: Use or_else to provide a fallback computation when an optional is empty

**Answer:**

`or_else` is the complement: it only fires when the optional is empty, giving you a chance to try an alternative source or supply a default. A chain of `or_else` calls lets you express a priority order of data sources cleanly:

```cpp
#include <optional>
#include <string>
#include <iostream>
#include <fstream>

std::optional<std::string> read_config(const std::string& path) {
    // Simulated: returns nullopt if file doesn't exist
    if (path == "/etc/app.conf") return "production";
    return std::nullopt;
}

std::optional<std::string> read_env_config() {
    // Simulated: check environment variable
    const char* val = std::getenv("APP_MODE");
    if (val) return std::string(val);
    return std::nullopt;
}

std::optional<std::string> default_config() {
    return "development";  // always succeeds
}

int main() {
    // or_else: try multiple config sources, fall back if each is empty
    auto config = read_config("/etc/app.conf")         // try system config
        .or_else(read_env_config)                       // try environment
        .or_else(default_config);                       // ultimate fallback

    std::cout << "Mode: " << *config << "\n";
    // Output: Mode: production   (if /etc/app.conf exists)
    // If not: tries env, then falls back to "development"

    // or_else with logging:
    auto result = std::optional<int>{}
        .or_else([]() -> std::optional<int> {
            std::cout << "Primary lookup failed, trying backup...\n";
            return std::nullopt;  // backup also fails
        })
        .or_else([]() -> std::optional<int> {
            std::cout << "Backup failed, using default.\n";
            return 0;
        });

    std::cout << "Value: " << *result << "\n";
    // Output:
    // Primary lookup failed, trying backup...
    // Backup failed, using default.
    // Value: 0

    // Combining all three in one pipeline:
    auto final_result = std::optional<std::string>{}
        .or_else([]() -> std::optional<std::string> { return std::nullopt; })
        .or_else([]() -> std::optional<std::string> { return "fallback"; })
        .transform([](const std::string& s) { return s + "!"; });

    std::cout << *final_result << "\n"; // "fallback!"
}
```

**Explanation:** `or_else` is the complement of `and_then` - it activates only when the optional is empty, attempting an alternative computation. If the optional has a value, `or_else` returns it unchanged. The fallback function must return `optional<T>` (same type), allowing cascading fallback chains.

---

## Notes

- **Availability:** `transform`, `and_then`, and `or_else` are C++23. GCC 13+, Clang 16+, MSVC 17.6+ support them.
- **`std::expected` (C++23)** has the same three operations, but carries an error value instead of being empty. The monadic API is identical in design.
- **No `optional::flat_map`:** `and_then` IS the flat_map. C++ chose the name from ranges/functional programming conventions.
- **`value_or()` vs `or_else()`:** `value_or(default)` returns the value or a static default. `or_else(f)` calls a function that returns another `optional<T>`, allowing it to perform computation or try alternative sources.
- **Lambda return types:** `or_else` lambdas often need explicit return type annotation: `[]() -> std::optional<T> { ... }`.
- **No short-circuit on transform:** `transform` always wraps the result. If your function can fail, use `and_then` instead.
- Compile with `-std=c++23 -Wall -Wextra`.
