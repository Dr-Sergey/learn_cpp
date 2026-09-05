# Use std::expected monadic operations: transform, and_then, or_else

**Category:** Error Handling  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

C++23 adds **monadic operations** to `std::expected`, enabling functional-style error handling pipelines. Instead of checking errors at every step with `if` statements, you chain operations that automatically propagate errors.

### The Three Monadic Operations

| Operation | Signature (on `expected<T,E>`) | Purpose |
| --- | --- | --- |
| `transform(f)` | `f: T -> U` -> `expected<U,E>` | Map the value (like `std::optional::transform`) |
| `and_then(f)` | `f: T -> expected<U,E>` -> `expected<U,E>` | Chain fallible operations (like `flatMap`/`bind`) |
| `or_else(f)` | `f: E -> expected<T,E2>` -> `expected<T,E2>` | Handle/recover from errors |

### Visual Pipeline

The key thing to internalize is what happens when a step produces an error: every subsequent `transform` and `and_then` in the chain is simply skipped, and the error travels through untouched until it hits an `or_else` or reaches the final result:

```cpp
                    transform(f)         and_then(g)         or_else(h)
                   Maps success         Chains fallible      Handles error
 +---------+      +----------+        +----------+        +----------+
 | expected |--OK->| f(value) |--OK--> | g(value) |--OK--> | pass     |--> result
 | <T, E>   |      | -> U     |        | -> exp<U>|        | through  |
 |          |-ERR->| skip f   |-ERR--> | skip g   |-ERR--> | h(error) |--> recovery
 +---------+      +----------+        +----------+        +----------+

Key insight:
  transform:  f takes T, returns U             -> wraps in expected<U,E>
  and_then:   f takes T, returns expected<U,E> -> NO double wrapping
  or_else:    f takes E, returns expected<T,E2> -> error recovery
```

### Side-by-Side: Imperative vs Monadic

Here is the contrast that motivates the whole approach. The imperative version is correct, but the `if (!result) return` noise on every step makes the happy path hard to see. The monadic version reads as a plain pipeline of what you actually want to happen:

```cpp
#include <expected>
#include <string>

using Error = std::string;

std::expected<int, Error>    parse(const std::string& s);
std::expected<int, Error>    validate(int n);
std::expected<double, Error> compute(int n);

// BAD: Imperative - verbose, repetitive
double process_imperative(const std::string& input) {
    auto parsed = parse(input);
    if (!parsed) return /* handle error */;
    auto validated = validate(*parsed);
    if (!validated) return /* handle error */;
    auto result = compute(*validated);
    if (!result) return /* handle error */;
    return *result;
}

// GOOD: Monadic - clean pipeline
std::expected<double, Error> process_monadic(const std::string& input) {
    return parse(input)
        .and_then(validate)
        .and_then(compute);
}
```

### Additional Operations (C++23)

| Operation | Purpose |
| --- | --- |
| `transform_error(f)` | Map the error value: `f: E -> E2` -> `expected<T,E2>` |
| `value_or(default)` | Return value if present, else default |

---

## Self-Assessment

### Q1: Chain three fallible operations using `and_then` without any if-checks

**Solution - User Registration Pipeline:**

Each validation step returns an `expected<User, RegError>`. If any step fails, `and_then` skips all remaining steps and propagates the first error - just like short-circuit evaluation, but for error-returning functions:

```cpp
#include <expected>
#include <string>
#include <iostream>
#include <cctype>
#include <algorithm>

enum class RegError {
    empty_name,
    invalid_chars,
    name_too_short,
    age_out_of_range,
    email_missing_at
};

std::string to_string(RegError e) {
    switch (e) {
        case RegError::empty_name:       return "Name cannot be empty";
        case RegError::invalid_chars:    return "Name contains invalid characters";
        case RegError::name_too_short:   return "Name must be at least 2 characters";
        case RegError::age_out_of_range: return "Age must be 1-150";
        case RegError::email_missing_at: return "Email must contain @";
    }
    return "Unknown";
}

struct User {
    std::string name;
    int age;
    std::string email;
};

// Three fallible steps - each returns expected<User, RegError>

std::expected<User, RegError> validate_name(User user) {
    if (user.name.empty())
        return std::unexpected(RegError::empty_name);
    if (user.name.size() < 2)
        return std::unexpected(RegError::name_too_short);
    if (!std::all_of(user.name.begin(), user.name.end(),
                     [](char c) { return std::isalpha(c) || c == ' '; }))
        return std::unexpected(RegError::invalid_chars);
    return user;
}

std::expected<User, RegError> validate_age(User user) {
    if (user.age < 1 || user.age > 150)
        return std::unexpected(RegError::age_out_of_range);
    return user;
}

std::expected<User, RegError> validate_email(User user) {
    if (user.email.find('@') == std::string::npos)
        return std::unexpected(RegError::email_missing_at);
    return user;
}

// Chain with and_then - NO if-checks!
std::expected<User, RegError> register_user(User user) {
    return validate_name(std::move(user))
        .and_then(validate_age)
        .and_then(validate_email);
}

int main() {
    // Success case
    auto result1 = register_user({"Alice", 30, "alice@example.com"});
    if (result1)
        std::cout << "Registered: " << result1->name << "\n";

    // Failure at name validation
    auto result2 = register_user({"", 25, "bob@example.com"});
    if (!result2)
        std::cout << "Error: " << to_string(result2.error()) << "\n";

    // Failure at age validation
    auto result3 = register_user({"Charlie", -5, "charlie@example.com"});
    if (!result3)
        std::cout << "Error: " << to_string(result3.error()) << "\n";

    // Failure at email validation
    auto result4 = register_user({"Diana", 28, "no-at-sign"});
    if (!result4)
        std::cout << "Error: " << to_string(result4.error()) << "\n";
}
// Expected output:
//   Registered: Alice
//   Error: Name cannot be empty
//   Error: Age must be 1-150
//   Error: Email must contain @
```

**How `and_then` short-circuits:**

```cpp
register_user({"", 25, "bob@example.com"})
  -> validate_name("") -> unexpected{empty_name}
  -> and_then(validate_age) -> SKIPPED (already error)
  -> and_then(validate_email) -> SKIPPED (already error)
  -> unexpected{empty_name}   <- first error propagated
```

---

### Q2: Use `transform` to map the success value without unwrapping

**Solution - `transform` vs `and_then`:**

The distinction between `transform` and `and_then` trips people up at first. The rule is: if your mapping function can fail and returns an `expected`, use `and_then`. If it is a plain transformation that always succeeds, use `transform` - it handles the wrapping for you:

```cpp
#include <expected>
#include <string>
#include <iostream>
#include <vector>
#include <numeric>
#include <cmath>

using Error = std::string;

std::expected<std::string, Error> read_input(bool success) {
    if (!success)
        return std::unexpected<Error>("read failed");
    return "42";
}

std::expected<int, Error> parse_int(const std::string& s) {
    try {
        return std::stoi(s);
    } catch (...) {
        return std::unexpected<Error>("parse error: " + s);
    }
}

int main() {
    // transform: takes T, returns U -> wrapped in expected<U,E> automatically
    // Use when the mapping function CANNOT fail

    auto result = read_input(true)
        .and_then(parse_int)           // string -> expected<int,Error>  (can fail)
        .transform([](int n) {         // int -> double  (cannot fail)
            return n * 2.5;
        })
        .transform([](double d) {      // double -> string (cannot fail)
            return "Result: " + std::to_string(d);
        });

    if (result)
        std::cout << *result << "\n";  // "Result: 105.000000"

    // Error case - transform is skipped
    auto err_result = read_input(false)
        .and_then(parse_int)
        .transform([](int n) {
            std::cout << "This never runs!\n";
            return n * 2.5;
        });

    if (!err_result)
        std::cout << "Error: " << err_result.error() << "\n";

    // === transform vs and_then comparison ===
    // transform: f returns U      -> result is expected<U,E>
    // and_then:  f returns exp<U> -> result is expected<U,E> (flattened)

    // ERROR: using and_then for a non-fallible function
    //    Would need to wrap: .and_then([](int n) -> expected<double,Error> { return n*2.5; })

    // CORRECT: use transform for non-fallible mapping
    //    .transform([](int n) { return n * 2.5; })

    // === transform_error: map the error side ===
    auto mapped = read_input(false)
        .transform_error([](const std::string& msg) {
            return "IO Error: " + msg;  // enrich the error message
        });

    if (!mapped)
        std::cout << mapped.error() << "\n";
}
// Expected output:
//   Result: 105.000000
//   Error: read failed
//   IO Error: read failed
```

**Quick Reference:**

```cpp
transform(f):        value ->  f(value)  -> expected<U,E>     (auto-wrapped)
and_then(f):         value ->  f(value)  -> expected<U,E>     (f returns expected)
transform_error(f):  error ->  f(error)  -> expected<T,E2>    (maps error)
or_else(f):          error ->  f(error)  -> expected<T,E2>    (f returns expected)
```

---

### Q3: Use `or_else` to provide a fallback value when an expected holds an error

**Solution - Error Recovery with `or_else`:**

`or_else` is the mirror image of `and_then`: it runs only when there is an error, and it can either recover (return a success value) or propagate a different error. This makes it ideal for cascading fallback strategies - try one source, then another, then a default:

```cpp
#include <expected>
#include <string>
#include <iostream>
#include <fstream>

enum class ConfigError { file_not_found, parse_error, permission_denied };

std::string to_string(ConfigError e) {
    switch (e) {
        case ConfigError::file_not_found:    return "file not found";
        case ConfigError::parse_error:       return "parse error";
        case ConfigError::permission_denied: return "permission denied";
    }
    return "unknown";
}

struct Config {
    std::string host = "localhost";
    int port = 8080;
};

std::expected<Config, ConfigError> load_from_file(const std::string& path) {
    // Simulate: file doesn't exist
    return std::unexpected(ConfigError::file_not_found);
}

std::expected<Config, ConfigError> load_from_env() {
    // Simulate: env vars not set -> parse error
    return std::unexpected(ConfigError::parse_error);
}

std::expected<Config, ConfigError> default_config() {
    return Config{"localhost", 8080};
}

int main() {
    // or_else: try fallback sources when the previous one fails
    // f takes E, returns expected<T,E>

    auto config = load_from_file("/etc/myapp.conf")
        .or_else([](ConfigError e) -> std::expected<Config, ConfigError> {
            std::cout << "File failed (" << to_string(e) << "), trying env...\n";
            return load_from_env();
        })
        .or_else([](ConfigError e) -> std::expected<Config, ConfigError> {
            std::cout << "Env failed (" << to_string(e) << "), using defaults...\n";
            return default_config();
        });

    if (config) {
        std::cout << "Config: " << config->host << ":" << config->port << "\n";
    }

    // === or_else with selective recovery ===
    auto selective = load_from_file("missing.conf")
        .or_else([](ConfigError e) -> std::expected<Config, ConfigError> {
            if (e == ConfigError::permission_denied)
                return std::unexpected(e);  // propagate - can't recover
            // For other errors, provide default
            return Config{"fallback-host", 9090};
        });

    if (selective)
        std::cout << "Selective: " << selective->host << ":" << selective->port << "\n";

    // === Complete pipeline: and_then + transform + or_else ===
    auto result = load_from_file("app.conf")
        .or_else([](ConfigError) -> std::expected<Config, ConfigError> {
            return Config{"default-host", 3000};  // fallback
        })
        .and_then([](Config c) -> std::expected<Config, ConfigError> {
            if (c.port < 1 || c.port > 65535)
                return std::unexpected(ConfigError::parse_error);
            return c;  // validation passed
        })
        .transform([](Config c) {
            return c.host + ":" + std::to_string(c.port);  // format
        });

    if (result)
        std::cout << "Endpoint: " << *result << "\n";
}
// Expected output:
//   File failed (file not found), trying env...
//   Env failed (parse error), using defaults...
//   Config: localhost:8080
//   Selective: fallback-host:9090
//   Endpoint: default-host:3000
```

Notice the selective recovery pattern: the `or_else` lambda inspects the error and can choose to propagate it unchanged by returning `std::unexpected(e)`. You are not forced to recover - you can let certain errors through while catching others.

`or_else` vs direct error check:

```cpp
// BAD: Imperative fallback chain
auto config = load_from_file("app.conf");
if (!config) {
    config = load_from_env();
    if (!config) {
        config = default_config();
    }
}

// GOOD: Monadic fallback chain
auto config = load_from_file("app.conf")
    .or_else([](auto) { return load_from_env(); })
    .or_else([](auto) { return default_config(); });
```

---

## Notes

- **`and_then`** is for **chaining fallible operations** - the function returns `expected<U,E>`. Like Haskell's `>>=` (bind) or Rust's `and_then`.
- **`transform`** is for **mapping values** - the function returns `U`, which is automatically wrapped. Like `fmap` / Rust's `map`.
- **`or_else`** is for **error recovery** - the function takes the error and returns a new `expected<T,E>`.
- **`transform_error`** is for **mapping errors** without touching the value - useful for error enrichment or conversion.
- **Error propagation is automatic** - if the expected holds an error, `transform` and `and_then` skip their function and propagate the error.
- All four operations are **const-correct** and have **rvalue overloads** for move optimization.
- **`std::optional`** also has `transform`, `and_then`, `or_else` in C++23 - same pattern.
- **Compiler requirement:** GCC 13+, Clang 16+, MSVC 19.34+ for full C++23 monadic support.
