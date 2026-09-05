# Use `std::optional` Correctly as a Nullable Value Type

**Category:** Type System and Deduction  
**Standard:** C++17 (monadic operations: C++23)  
**Reference:** <https://en.cppreference.com/w/cpp/utility/optional>  

---

## Topic Overview

### What Is `std::optional`

`std::optional<T>` is a vocabulary type that either contains a value of type `T` or is **empty** (disengaged). It replaces error-prone patterns like returning `-1`, `nullptr`, or sentinel values to indicate "no result." The key benefit is that the absence of a value is expressed in the type itself, so the compiler can help you handle it correctly.

### Core API

Here is the essential interface you'll use most often:

```cpp
#include <optional>
#include <string>

std::optional<int> maybe;           // disengaged (empty)
std::optional<int> has_val = 42;    // engaged, holds 42

// Check engagement
if (maybe)              { /* engaged */ }
if (maybe.has_value())  { /* engaged */ }

// Access value
int v1 = has_val.value();          // throws bad_optional_access if empty
int v2 = *has_val;                 // UB if empty! (no check)
int v3 = has_val.value_or(-1);     // returns -1 if empty (safe)

// Construct in-place
std::optional<std::string> s(std::in_place, 5, 'x');  // "xxxxx"
s.emplace("hello");               // destroys old, constructs new

// Reset
s.reset();                         // disengages
s = std::nullopt;                  // same effect
```

### Why Not Pointers or Sentinels

You've probably used the old patterns before. Here is why `optional` beats them:

| Pattern | Problem |
| --- | --- |
| Return `-1` for "not found" | What if `-1` is a valid value? |
| Return `nullptr` | Only works for pointer types |
| Output parameter `bool find(T& out)` | Awkward API, requires default-constructible T |
| `std::optional<T>` | Clear semantics, works for any type, no allocation |

### Monadic Operations (C++23)

C++23 adds functional-style chaining to avoid nested `if` blocks:

| Method | Signature | Behavior |
| --- | --- | --- |
| `transform(f)` | `optional<U> transform(F f)` | If engaged, applies `f(*this)` and wraps result |
| `and_then(f)` | `optional<U> and_then(F f)` | If engaged, calls `f(*this)` which must return `optional<U>` |
| `or_else(f)` | `optional<T> or_else(F f)` | If empty, calls `f()` which must return `optional<T>` |

---

## Self-Assessment

### Q1: Replace a find function that returns -1 or a sentinel pointer with one returning `std::optional`

This shows three classic "bad" patterns replaced by `optional`. Notice how the API intent becomes self-documenting:

```cpp
#include <iostream>
#include <optional>
#include <vector>
#include <string>
#include <map>

// OLD: returns -1 as sentinel for "not found"
int old_find_index(const std::vector<int>& v, int target) {
    for (size_t i = 0; i < v.size(); ++i) {
        if (v[i] == target) return static_cast<int>(i);
    }
    return -1;  // Problem: -1 could be confused with valid data
}

// NEW: returns optional - impossible to confuse with valid values
std::optional<size_t> find_index(const std::vector<int>& v, int target) {
    for (size_t i = 0; i < v.size(); ++i) {
        if (v[i] == target) return i;    // implicit conversion to optional
    }
    return std::nullopt;                  // explicitly "no result"
}

// OLD: returns nullptr as sentinel
const std::string* old_lookup(const std::map<int, std::string>& m, int key) {
    auto it = m.find(key);
    if (it != m.end()) return &it->second;
    return nullptr;  // caller must check for null
}

// NEW: returns optional - value semantics, no dangling pointer risk
std::optional<std::string> lookup(const std::map<int, std::string>& m, int key) {
    auto it = m.find(key);
    if (it != m.end()) return it->second;
    return std::nullopt;
}

// Parsing example: converting string to int safely
std::optional<int> parse_int(const std::string& s) {
    try {
        size_t pos;
        int result = std::stoi(s, &pos);
        if (pos == s.size()) return result;  // entire string consumed
        return std::nullopt;                  // trailing characters
    } catch (...) {
        return std::nullopt;                  // not a valid integer
    }
}

int main() {
    std::vector<int> data = {10, 20, 30, 40, 50};

    // find_index
    if (auto idx = find_index(data, 30)) {
        std::cout << "Found 30 at index " << *idx << "\n";
    }

    if (auto idx = find_index(data, 99)) {
        std::cout << "Found 99\n";  // not reached
    } else {
        std::cout << "99 not found\n";
    }

    // lookup
    std::map<int, std::string> users = {{1, "Alice"}, {2, "Bob"}};
    auto name = lookup(users, 1);
    std::cout << "User 1: " << name.value_or("(unknown)") << "\n";
    std::cout << "User 3: " << lookup(users, 3).value_or("(unknown)") << "\n";

    // parse_int
    auto a = parse_int("42");
    auto b = parse_int("hello");
    auto c = parse_int("12abc");

    std::cout << "parse '42':    " << a.value_or(-1) << "\n";
    std::cout << "parse 'hello': " << b.value_or(-1) << "\n";
    std::cout << "parse '12abc': " << c.value_or(-1) << "\n";

    return 0;
}
```

**Output:**

```text
Found 30 at index 2
99 not found
User 1: Alice
User 3: (unknown)
parse '42':    42
parse 'hello': -1
parse '12abc': -1
```

### Q2: Explain the danger of dereferencing a disengaged optional and how `value_or` avoids it

The three access methods form a spectrum from unsafe-but-fast to safe-and-convenient. Know which one to reach for in each situation:

```cpp
#include <iostream>
#include <optional>
#include <string>
#include <stdexcept>

std::optional<int> maybe_get_value(bool should_have) {
    if (should_have) return 42;
    return std::nullopt;
}

int main() {
    auto empty = maybe_get_value(false);  // disengaged optional

    // DANGER 1: operator* on disengaged optional -> UNDEFINED BEHAVIOR
    // The compiler does NOT check - it just accesses garbage memory
    // int bad = *empty;  // UB! May crash, return garbage, or seem to work

    // DANGER 2: .value() on disengaged optional -> throws exception
    try {
        int val = empty.value();  // throws std::bad_optional_access
        std::cout << val;         // never reached
    } catch (const std::bad_optional_access& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    // SAFE: .value_or() - returns the default if disengaged
    int safe_val = empty.value_or(-1);
    std::cout << "value_or(-1): " << safe_val << "\n";

    // SAFE: Check before access
    if (empty) {
        std::cout << "Value: " << *empty << "\n";
    } else {
        std::cout << "No value present\n";
    }

    // SAFE: Structured binding style with if-init
    if (auto val = maybe_get_value(true); val.has_value()) {
        std::cout << "Got: " << *val << "\n";  // safe - guarded by check
    }

    // LESSON: operator* is for performance-critical paths where you've
    // ALREADY verified engagement. For all other cases use value_or()
    // or check has_value()/operator bool() first.

    // Summary:
    //   *opt           -> fast, UB if empty (like raw pointer dereference)
    //   opt.value()    -> safe, throws if empty (like vector::at())
    //   opt.value_or() -> safe, returns default if empty (most convenient)

    // Practical pattern: value_or with meaningful defaults
    std::optional<std::string> username;
    std::cout << "Hello, " << username.value_or("Guest") << "!\n";

    std::optional<int> timeout_ms;
    int actual_timeout = timeout_ms.value_or(5000);  // 5s default
    std::cout << "Timeout: " << actual_timeout << "ms\n";

    return 0;
}
```

**Output:**

```text
Caught: bad optional access
value_or(-1): -1
No value present
Got: 42
Hello, Guest!
Timeout: 5000ms
```

### Q3: Show how to monadic chain optionals with `and_then` / `transform` (C++23)

The monadic operations let you chain transformations without writing `if (!opt) return std::nullopt;` at every step. If any step produces `nullopt`, the rest of the chain is skipped automatically:

```cpp
#include <iostream>
#include <optional>
#include <string>
#include <charconv>
#include <cmath>

// Helper: parse string to int (returns optional)
std::optional<int> parse_int(const std::string& s) {
    int result;
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), result);
    if (ec == std::errc{} && ptr == s.data() + s.size())
        return result;
    return std::nullopt;
}

// Helper: safe square root (only for non-negative)
std::optional<double> safe_sqrt(int x) {
    if (x < 0) return std::nullopt;
    return std::sqrt(static_cast<double>(x));
}

// Helper: format to string
std::string format_result(double d) {
    return "Result: " + std::to_string(d);
}

int main() {
    // === C++23 Monadic Operations ===

    // WITHOUT monadic operations (C++17 - nested ifs):
    auto compute_old = [](const std::string& input) -> std::optional<std::string> {
        auto parsed = parse_int(input);
        if (!parsed) return std::nullopt;

        auto root = safe_sqrt(*parsed);
        if (!root) return std::nullopt;

        return format_result(*root);
    };

    // WITH monadic operations (C++23 - flat chain):
    auto compute_new = [](const std::string& input) -> std::optional<std::string> {
        return parse_int(input)               // optional<int>
            .and_then(safe_sqrt)              // optional<double> (f returns optional)
            .transform(format_result);        // optional<string> (f returns value)
    };

    // Test both approaches produce the same results:
    std::cout << "=== Monadic chaining (C++23) ===\n";

    for (const auto& input : {"25", "-4", "hello", "0", "144"}) {
        auto old_result = compute_old(input);
        auto new_result = compute_new(input);

        std::cout << "Input '" << input << "': "
                  << new_result.value_or("(no result)") << "\n";
    }

    // === transform: maps the value inside optional ===
    std::cout << "\n=== transform examples ===\n";

    std::optional<int> five = 5;
    std::optional<int> none;

    // transform applies function only if engaged
    auto doubled = five.transform([](int x) { return x * 2; });
    auto nothing = none.transform([](int x) { return x * 2; });

    std::cout << "5 * 2 = " << doubled.value_or(0) << "\n";
    std::cout << "none * 2 = " << nothing.value_or(0) << "\n";

    // transform can change the type
    auto as_string = five.transform([](int x) { return std::to_string(x); });
    static_assert(std::is_same_v<decltype(as_string), std::optional<std::string>>);
    std::cout << "5 as string: " << as_string.value_or("") << "\n";

    // === and_then: for functions that return optional (flatmap) ===
    std::cout << "\n=== and_then examples ===\n";

    auto safe_divide = [](int x) -> std::optional<double> {
        if (x == 0) return std::nullopt;
        return 100.0 / x;
    };

    auto r1 = std::optional<int>(4).and_then(safe_divide);
    auto r2 = std::optional<int>(0).and_then(safe_divide);
    auto r3 = std::optional<int>().and_then(safe_divide);

    std::cout << "100/4 = " << r1.value_or(0.0) << "\n";
    std::cout << "100/0 = " << r2.value_or(0.0) << "\n";
    std::cout << "none  = " << r3.value_or(0.0) << "\n";

    // === or_else: provides fallback when empty ===
    std::cout << "\n=== or_else examples ===\n";

    auto with_default = none.or_else([]() -> std::optional<int> { return 99; });
    auto already_has = five.or_else([]() -> std::optional<int> { return 99; });

    std::cout << "none or_else 99: " << with_default.value_or(0) << "\n";
    std::cout << "5 or_else 99: " << already_has.value_or(0) << "\n";

    // === Full pipeline ===
    std::cout << "\n=== Full pipeline ===\n";
    auto pipeline = parse_int("16")
        .and_then(safe_sqrt)                               // 4.0
        .transform([](double d) { return d + 1; })        // 5.0
        .transform([](double d) { return d * d; })        // 25.0
        .transform([](double d) { return "Answer: " + std::to_string(static_cast<int>(d)); });

    std::cout << pipeline.value_or("failed") << "\n";

    return 0;
}
```

**Output:**

```text
=== Monadic chaining (C++23) ===
Input '25': Result: 5.000000
Input '-4': (no result)
Input 'hello': (no result)
Input '0': Result: 0.000000
Input '144': Result: 12.000000

=== transform examples ===
5 * 2 = 10
none * 2 = 0
5 as string: 5

=== and_then examples ===
100/4 = 25
100/0 = 0
none  = 0

=== or_else examples ===
none or_else 99: 99
5 or_else 99: 5

=== Full pipeline ===
Answer: 25
```

---

## Notes

- **`optional<T&>` is not supported until C++26.** Use `optional<std::reference_wrapper<T>>` or pointers for optional references in C++17/20/23.
- **Don't use `optional<bool>` or `optional<optional<T>>`** - three-state logic becomes confusing. Consider an enum or `std::variant` instead.
- **`optional<T>` stores T inline** - no heap allocation. Size is `sizeof(T) + alignment padding + 1 byte` for the engagement flag.
- **Move semantics:** optional propagates moves. Moving from an engaged optional leaves it engaged but with a moved-from value (not disengaged).
- **Comparison:** `optional` supports comparison with `T`, `nullopt`, and other `optional<T>`. A disengaged optional compares less than any engaged one.
