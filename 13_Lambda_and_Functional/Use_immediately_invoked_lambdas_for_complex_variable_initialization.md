# Use immediately invoked lambdas for complex variable initialization

**Category:** Lambda and Functional  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

An **Immediately Invoked Function Expression (IIFE)** - or immediately invoked lambda - is a lambda that is called at the point of definition. In C++, this lets you initialize `const` variables with complex logic (loops, switches, conditionals) that would otherwise require a mutable variable or a separate helper function.

The pattern looks like this - notice the `()` at the very end, which is what triggers the call:

```cpp
const auto result = [&]() {
    // complex initialization logic
    // ...
    return computed_value;
}();  // <- note the () at the end - invokes immediately!
```

### Why Use This

Without the IIFE pattern, any variable that needs complex initialization has to start as a mutable, non-`const` variable so you can assign to it in pieces. That means it stays mutable for the rest of its scope, which is both a correctness risk and a communication problem (a `const` variable clearly signals "this never changes"). IIFEs let you do all the work up front and still end up with a `const` result:

| Without IIFE | With IIFE |
| --- | --- |
| Variable must be mutable | Variable can be `const` |
| Logic is at same scope | Logic is contained in lambda |
| Need separate helper function | Inline, self-contained |
| `result` might be used before fully set | `result` is initialized atomically |

---

## Self-Assessment

### Q1: Use an IIFE (`[]{...}()`) to initialize a `const` variable that requires a switch statement

Switch statements can't be used as expressions in C++, which is exactly the kind of thing that forces you into the "start mutable, assign in branches" pattern. The IIFE wraps the switch so the whole thing becomes a single expression with a value:

```cpp
#include <iostream>
#include <string>

enum class LogLevel { Debug, Info, Warning, Error };

int main() {
    LogLevel level = LogLevel::Warning;

    // WITHOUT IIFE: need mutable variable
    // std::string label;  // NOT const!
    // switch (level) {
    //     case LogLevel::Debug:   label = "DEBUG";   break;
    //     case LogLevel::Info:    label = "INFO";    break;
    //     case LogLevel::Warning: label = "WARNING"; break;
    //     case LogLevel::Error:   label = "ERROR";   break;
    // }

    // WITH IIFE: const initialization!
    const std::string label = [&]() -> std::string {
        switch (level) {
            case LogLevel::Debug:   return "DEBUG";
            case LogLevel::Info:    return "INFO";
            case LogLevel::Warning: return "WARNING";
            case LogLevel::Error:   return "ERROR";
        }
        return "UNKNOWN";  // unreachable but satisfies compiler
    }();  // <- immediately invoked!

    std::cout << "Log level: " << label << "\n";

    // More complex: multi-step initialization
    const auto config = [&]() {
        struct Config { int threads; bool verbose; std::string mode; };
        if (level == LogLevel::Debug)
            return Config{1, true, "debug"};
        else
            return Config{8, false, "release"};
    }();

    std::cout << "Threads: " << config.threads
              << ", Mode: " << config.mode << "\n";
}
// Expected output:
//   Log level: WARNING
//   Threads: 8, Mode: release
```

The second example shows an IIFE initializing a struct. Temporary helper types defined inside lambdas are scoped to the lambda, so they don't leak into the surrounding code at all.

---

### Q2: Explain why this idiom is preferable to a separate helper function in some contexts

A named helper function is often the right answer for logic that's reused, but for one-shot initialization it has drawbacks. It has to live somewhere else in the file, it needs every piece of context passed as a parameter, and it gives a name to something that only happens once. The IIFE captures local context directly and keeps the logic right where it's used:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// Approach 1: Separate helper function
// Drawbacks:
// - Pollutes enclosing scope with a function name
// - Must pass ALL context as parameters
// - Function defined far from use site
// - Hard to reason about scope/lifetime
std::vector<int> make_filtered_sorted(const std::vector<int>& input, int threshold) {
    std::vector<int> result;
    for (int x : input)
        if (x > threshold) result.push_back(x);
    std::sort(result.begin(), result.end());
    return result;
}

int main() {
    std::vector<int> data = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    int threshold = 4;

    // Approach 1: helper function
    const auto result1 = make_filtered_sorted(data, threshold);

    // Approach 2: IIFE - inline, captures context naturally
    const auto result2 = [&]() {
        std::vector<int> v;
        for (int x : data)          // 'data' captured by reference
            if (x > threshold)       // 'threshold' captured by reference
                v.push_back(x);
        std::sort(v.begin(), v.end());
        return v;
    }();  // immediately invoked - result2 is const!

    // Benefits of IIFE:
    // No separate function (logic stays at use site)
    // Captures local context naturally (no parameter passing)
    // Result is const - can't be accidentally modified
    // lambda scope limits temp variables (v is gone after)
    // Used once - not reusable, and that's the point

    std::cout << "Result: ";
    for (int x : result2) std::cout << x << " ";
    std::cout << "\n";

    // When to prefer a named function instead:
    // - Logic is reused in multiple places
    // - Logic is complex enough to deserve a name
    // - Unit testing the logic separately
}
// Expected output:
//   Result: 5 6 7 8 9
```

The temporary variable `v` inside the IIFE is worth noting: it exists, does its job, and disappears. In the named-function version it would have the same behavior, but in the surrounding scope you might have been tempted to reuse or expose it somehow. The lambda scope is a natural container for "working variables."

---

### Q3: Show a `std::optional` initialized via an immediately invoked lambda with complex logic

`std::optional` is a great fit for IIFE initialization because the logic for deciding whether to return a value or `nullopt` often involves several steps. Without IIFE, you'd need to construct the optional separately and conditionally assign to it, which makes `const` impossible:

```cpp
#include <iostream>
#include <optional>
#include <string>
#include <map>

struct User {
    std::string name;
    int age;
};

int main() {
    // Simulated database
    std::map<int, User> db = {
        {1, {"Alice", 30}},
        {2, {"Bob", 25}},
        {3, {"Charlie", 35}},
    };

    int requested_id = 2;
    int min_age = 18;

    // Initialize const std::optional via IIFE
    const std::optional<User> user = [&]() -> std::optional<User> {
        auto it = db.find(requested_id);
        if (it == db.end())
            return std::nullopt;  // not found

        if (it->second.age < min_age)
            return std::nullopt;  // too young

        return it->second;  // valid user
    }();

    if (user)
        std::cout << "Found: " << user->name << " (age " << user->age << ")\n";
    else
        std::cout << "No valid user found\n";

    // Another example: optional with complex parsing
    const std::optional<int> parsed = [&]() -> std::optional<int> {
        std::string input = "  42  ";

        // Trim whitespace
        size_t start = input.find_first_not_of(" \t");
        size_t end = input.find_last_not_of(" \t");
        if (start == std::string::npos) return std::nullopt;

        std::string trimmed = input.substr(start, end - start + 1);

        try {
            return std::stoi(trimmed);
        } catch (...) {
            return std::nullopt;
        }
    }();

    if (parsed)
        std::cout << "Parsed: " << *parsed << "\n";
}
// Expected output:
//   Found: Bob (age 25)
//   Parsed: 42
```

Both optionals are `const`. The multi-condition check reads naturally as a sequence of early returns, which is cleaner than nested `if` statements around a mutable optional.

---

## Notes

- **Performance:** IIFE has zero overhead - the compiler inlines the lambda and eliminates the call entirely.
- **`constexpr` IIFE:** Works since C++17: `constexpr auto x = []{ return 42; }();`
- **`constinit` + IIFE (C++20):** Initialize `static` variables with complex logic at compile time.
- **Capture `[&]` is safe here** because the lambda is called immediately - references never dangle. The captured references go out of scope before any danger can arise.
- **Return type:** Usually deduced, but use `-> Type` when the lambda has multiple return paths with different types to make the intent explicit.
- **Alternative:** C++17 `if` with initializer: `if (auto x = compute(); x > 0) { ... }` - but only for conditions, not `const` initialization.
