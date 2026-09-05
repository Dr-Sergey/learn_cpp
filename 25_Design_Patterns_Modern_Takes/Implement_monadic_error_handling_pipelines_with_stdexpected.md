# Implement monadic error handling pipelines with std::expected

**Category:** Design Patterns Modern Takes  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

`std::expected<T, E>` (C++23) holds either a value of type `T` or an error of type `E`. The really useful part is its **monadic interface** - the methods `and_then`, `transform`, `or_else`, and `transform_error` - which lets you chain fallible operations into a clean pipeline without manually checking for errors at each step. If you've used Rust's `Result::and_then` or Haskell's `>>=`, this is the same idea landing in C++.

The core insight is that once an error occurs in the pipeline, every subsequent step is automatically skipped. You write the happy path from top to bottom, and error propagation takes care of itself.

### Expected vs Exceptions - At a Glance

If you're used to exceptions, the table below captures the tradeoffs. The biggest practical differences are that `std::expected` keeps error information in the type signature (so callers can't ignore it) and avoids stack unwinding on failure (which can be significant when errors are frequent).

| Aspect | `std::expected` pipeline | `try/catch` exceptions |
| --- | --- | --- |
| Control flow | Linear chain, explicit | Non-local jump |
| Performance (error path) | Predictable, no unwinding | Stack unwinding overhead |
| Composability | Chainable with `.and_then()` | Nested try/catch blocks |
| Error visibility | In the type signature | Hidden; checked at runtime |
| Readability | Functional pipeline style | Imperative style |

---

## Self-Assessment

### Q1: Chain five fallible operations using and_then without a single try/catch or if-error check

**Answer:**

Here's the key thing to observe: each of the five steps returns `std::expected<something, Error>`. When you chain them with `.and_then()`, the chain short-circuits the moment any step returns an error - later steps never run. At the call site, you write a clean linear pipeline and check once at the end.

```cpp
#include <iostream>
#include <string>
#include <expected>
#include <charconv>
#include <cmath>

using Error = std::string;

// Five fallible steps
auto step1_read(const std::string& input)
    -> std::expected<std::string, Error>
{
    if (input.empty()) return std::unexpected<Error>("empty input");
    return input;
}

auto step2_parse(const std::string& s)
    -> std::expected<double, Error>
{
    double val{};
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), val);
    if (ec != std::errc{}) return std::unexpected<Error>("parse failed: " + s);
    return val;
}

auto step3_validate(double v)
    -> std::expected<double, Error>
{
    if (v < 0) return std::unexpected<Error>("negative value");
    return v;
}

auto step4_compute(double v)
    -> std::expected<double, Error>
{
    double result = std::sqrt(v);
    if (std::isnan(result))
        return std::unexpected<Error>("computation produced NaN");
    return result;
}

auto step5_format(double v)
    -> std::expected<std::string, Error>
{
    return "Result = " + std::to_string(v);
}

int main() {
    // Chain five steps - ZERO if-checks in between
    auto result = step1_read("144.0")
        .and_then(step2_parse)
        .and_then(step3_validate)
        .and_then(step4_compute)
        .and_then(step5_format);

    if (result)
        std::cout << *result << '\n';   // Result = 12.000000

    // Error propagation - step3 fails, steps 4-5 skipped
    auto bad = step1_read("-9")
        .and_then(step2_parse)
        .and_then(step3_validate)   // fails here
        .and_then(step4_compute)    // skipped
        .and_then(step5_format);    // skipped

    if (!bad)
        std::cout << "Error: " << bad.error() << '\n';
    // Error: negative value
}
```

Notice that the second pipeline fails at `step3_validate` because of the negative value, and the remaining two steps are never called. The error just flows through automatically.

### Q2: Use transform_error to convert between error types at pipeline boundaries

**Answer:**

Real systems often have multiple layers - a database layer, a service layer, an API layer - each with its own error type. `transform_error` lets you map from one error type to another as the result crosses a boundary, without losing any context. Think of it as a lens that only activates when something went wrong.

```cpp
#include <iostream>
#include <string>
#include <expected>

// Layer-specific error types
struct DbError  { int code; std::string detail; };
struct AppError { std::string source; std::string message; };

// Database layer returns DbError
auto db_lookup(int id)
    -> std::expected<std::string, DbError>
{
    if (id == 0) return std::unexpected(DbError{404, "record not found"});
    return "user_" + std::to_string(id);
}

// Application layer expects AppError
auto process_user(const std::string& name)
    -> std::expected<std::string, AppError>
{
    if (name.size() < 3)
        return std::unexpected(AppError{"validation", "name too short"});
    return "Processed: " + name;
}

int main() {
    // Convert DbError -> AppError at the boundary
    auto result = db_lookup(0)
        .transform_error([](DbError e) -> AppError {
            return AppError{
                "database",
                "DB error " + std::to_string(e.code) + ": " + e.detail
            };
        });
    // result is now expected<string, AppError>

    if (!result)
        std::cout << "[" << result.error().source << "] "
                  << result.error().message << '\n';
    // [database] DB error 404: record not found

    // Success path: convert error type but never invoked
    auto good = db_lookup(42)
        .transform_error([](DbError e) -> AppError {
            return AppError{"database", e.detail};  // Never called
        })
        .and_then(process_user);

    if (good) std::cout << *good << '\n';
    // Processed: user_42
}
```

When the lookup succeeds, the `transform_error` lambda is never called - it only runs if there's actually an error to transform. This makes it safe to use in both success and error paths without any extra branching.

### Q3: Compare the readability of an expected pipeline with a traditional exception hierarchy

**Answer:**

The reason this comparison matters is that exception-based code forces you to think about two control flows simultaneously - the happy path and the error path interleave through nested `try/catch` blocks. The `expected` approach collapses them into a single linear chain that reads top-to-bottom.

```cpp
#include <iostream>
#include <string>
#include <expected>
#include <stdexcept>

// APPROACH 1: Traditional exceptions
namespace exc_style {
    std::string fetch(int id) {
        if (id <= 0) throw std::runtime_error("invalid id");
        return "item_" + std::to_string(id);
    }
    int parse_quantity(const std::string& s) {
        int v = std::stoi(s);  // may throw
        if (v <= 0) throw std::domain_error("non-positive qty");
        return v;
    }
    double compute_price(const std::string& item, int qty) {
        return qty * 9.99;
    }

    void process() {
        try {
            auto item = fetch(1);
            try {
                auto qty = parse_quantity("5");
                try {
                    auto price = compute_price(item, qty);
                    std::cout << "Price: " << price << '\n';
                } catch (const std::exception& e) {
                    std::cerr << "Compute error: " << e.what() << '\n';
                }
            } catch (const std::domain_error& e) {
                std::cerr << "Validation: " << e.what() << '\n';
            }
        } catch (const std::runtime_error& e) {
            std::cerr << "Fetch error: " << e.what() << '\n';
        }
    }
}

// APPROACH 2: Expected pipeline
namespace exp_style {
    using Error = std::string;

    auto fetch(int id) -> std::expected<std::string, Error> {
        if (id <= 0) return std::unexpected<Error>("invalid id");
        return "item_" + std::to_string(id);
    }
    auto parse_quantity(const std::string& s) -> std::expected<int, Error> {
        int v = std::stoi(s);
        if (v <= 0) return std::unexpected<Error>("non-positive qty");
        return v;
    }
    auto compute_price(int qty) -> std::expected<double, Error> {
        return qty * 9.99;
    }

    void process() {
        auto result = fetch(1)
            .and_then([](auto&&) { return parse_quantity("5"); })
            .and_then(compute_price);

        if (result)
            std::cout << "Price: " << *result << '\n';
        else
            std::cout << "Error: " << result.error() << '\n';
    }
}

int main() {
    exc_style::process();  // Nested try/catch - hard to follow
    exp_style::process();  // Linear pipeline - clear data flow
}
// Both print: Price: 49.95
```

The exception version has the same logic but it's scattered across three nesting levels. The `expected` version reads almost like a description of what should happen, left-to-right.

---

## Notes

- `and_then(f)` requires `f` to return `expected<U, E>` (same error type) - use this for steps that can themselves fail.
- `transform(f)` wraps `f`'s return in `expected` automatically - use this for pure mapping steps that can't fail.
- Error propagation is automatic: once an error occurs, subsequent `and_then`/`transform` calls are skipped and the error passes through unchanged.
- The monadic interface requires C++23; for C++17 use `tl::expected` from <https://github.com/TartanLlama/expected>.
