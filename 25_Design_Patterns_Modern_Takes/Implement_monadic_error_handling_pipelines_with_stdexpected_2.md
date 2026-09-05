# Design a multi-stage std::expected pipeline architecture

**Category:** Design Patterns Modern Takes  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

This file focuses on a **practical parse -> validate -> transform pipeline** where each step returns `expected<T, E>`, demonstrating real-world data processing with monadic chaining and `transform` for pure value mapping.

The distinction between `and_then` and `transform` is worth paying attention to here. Use `and_then` when the next step is itself fallible and returns an `expected`. Use `transform` when the next step is a pure function that can't fail - `transform` automatically wraps the return value in `expected` for you.

### Pipeline Architecture

Here's the shape of the pipeline you'll be building. Each arrow is one `.and_then()` or `.transform()` call, and a failure at any stage stops the chain immediately.

```cpp
Input string
    │
    ▼
  parse()        -> expected<RawData, Error>
    │ .and_then()
    ▼
  validate()     -> expected<ValidData, Error>
    │ .and_then()
    ▼
  transform()    -> expected<Output, Error>
    │ .transform()
    ▼
  format()       -> expected<string, Error>   (pure mapping, auto-wrapped)
```

---

## Self-Assessment

### Q1: Chain a parse() -> validate() -> transform() pipeline where each step returns expected<T, E>

**Answer:**

This example processes CSV lines through three typed stages. Notice how the types change across each step - `RawRecord` becomes `ValidRecord` becomes `Report` - and how the final `.transform()` call at the end is a pure formatting step that can't fail on its own.

```cpp
#include <iostream>
#include <string>
#include <expected>
#include <charconv>

struct Error { std::string stage; std::string detail; };

// Domain types
struct RawRecord {
    std::string name;
    std::string age_str;
    std::string score_str;
};

struct ValidRecord {
    std::string name;
    int age;
    double score;
};

struct Report {
    std::string name;
    std::string grade;
    bool eligible;
};

// Step 1: Parse CSV line
auto parse(const std::string& line)
    -> std::expected<RawRecord, Error>
{
    auto p1 = line.find(',');
    auto p2 = line.find(',', p1 + 1);
    if (p1 == std::string::npos || p2 == std::string::npos)
        return std::unexpected(Error{"parse", "expected 3 comma-separated fields"});

    return RawRecord{
        line.substr(0, p1),
        line.substr(p1 + 1, p2 - p1 - 1),
        line.substr(p2 + 1)
    };
}

// Step 2: Validate & convert types
auto validate(RawRecord raw)
    -> std::expected<ValidRecord, Error>
{
    int age{};
    auto [p1, e1] = std::from_chars(
        raw.age_str.data(), raw.age_str.data() + raw.age_str.size(), age);
    if (e1 != std::errc{})
        return std::unexpected(Error{"validate", "invalid age: " + raw.age_str});

    if (age < 0 || age > 150)
        return std::unexpected(Error{"validate", "age out of range"});

    double score{};
    auto [p2, e2] = std::from_chars(
        raw.score_str.data(), raw.score_str.data() + raw.score_str.size(), score);
    if (e2 != std::errc{})
        return std::unexpected(Error{"validate", "invalid score: " + raw.score_str});

    return ValidRecord{raw.name, age, score};
}

// Step 3: Transform to domain output
auto evaluate(ValidRecord rec)
    -> std::expected<Report, Error>
{
    std::string grade;
    if (rec.score >= 90) grade = "A";
    else if (rec.score >= 80) grade = "B";
    else if (rec.score >= 70) grade = "C";
    else grade = "F";

    bool eligible = (rec.age >= 18 && rec.score >= 70);
    return Report{rec.name, grade, eligible};
}

int main() {
    // Full pipeline
    std::string lines[] = {
        "Alice,25,92.5",
        "Bob,17,85.0",
        "Charlie,abc,70",    // Bad age
        "Diana,30"           // Missing field
    };

    for (const auto& line : lines) {
        auto result = parse(line)
            .and_then(validate)
            .and_then(evaluate)
            .transform([](Report r) {
                return r.name + ": grade=" + r.grade +
                       " eligible=" + (r.eligible ? "yes" : "no");
            });

        if (result)
            std::cout << *result << '\n';
        else
            std::cout << "[" << result.error().stage << "] "
                      << result.error().detail << '\n';
    }
    // Output:
    // Alice: grade=A eligible=yes
    // Bob: grade=B eligible=no
    // [validate] invalid age: abc
    // [parse] expected 3 comma-separated fields
}
```

The error output tells you exactly which stage caught the problem - that's the `stage` field in the `Error` struct, set by each step individually. That kind of precise attribution is hard to achieve cleanly with exceptions.

### Q2: Use and_then to propagate success and transform_error to enrich error context

**Answer:**

`transform_error` is handy whenever you want to add context to an error as it bubbles up through pipeline stages. Think of it like wrapping an error message with "this happened while doing X" - each layer can add its own context without disturbing the normal flow.

```cpp
#include <iostream>
#include <string>
#include <expected>

struct Error {
    std::string msg;
    std::string context;
};

auto load_config(const std::string& path)
    -> std::expected<std::string, Error>
{
    if (path.empty())
        return std::unexpected(Error{"file not found", ""});
    return "timeout=30;retries=3";
}

auto extract_timeout(const std::string& config)
    -> std::expected<int, Error>
{
    auto pos = config.find("timeout=");
    if (pos == std::string::npos)
        return std::unexpected(Error{"missing 'timeout' key", ""});
    return std::stoi(config.substr(pos + 8));
}

int main() {
    // and_then propagates; transform_error enriches
    auto result = load_config("")
        .transform_error([](Error e) {
            e.context = "while loading app config";
            return e;
        })
        .and_then(extract_timeout)
        .transform_error([](Error e) {
            if (e.context.empty()) e.context = "while parsing timeout";
            return e;
        });

    if (!result)
        std::cout << result.error().msg << " (" << result.error().context << ")\n";
    // file not found (while loading app config)
}
```

Because `load_config("")` fails immediately, the second `transform_error` is never reached - the context added by the first one is already in the error. If `load_config` had succeeded but `extract_timeout` failed instead, it would be the second `transform_error` that adds context.

### Q3: Compare the expected pipeline with a try/catch chain for readability and performance

**Answer:**

Beyond readability, there is a real performance story here. Throwing an exception causes the runtime to walk the call stack and unwind it - that's expensive and non-trivial even for modern implementations. With `std::expected`, returning an error is just returning a value, with no stack unwinding at all. The benchmark below makes this concrete.

```cpp
#include <iostream>
#include <string>
#include <expected>
#include <chrono>

// Performance comparison
// Exception path: stack unwinding is expensive when errors are frequent
// Expected path:  returning a value - same cost as a normal return

using Error = std::string;

auto maybe_fail_expected(int i) -> std::expected<int, Error> {
    if (i % 2 == 0) return std::unexpected<Error>("even");
    return i * 2;
}

int maybe_fail_exception(int i) {
    if (i % 2 == 0) throw std::runtime_error("even");
    return i * 2;
}

int main() {
    constexpr int N = 100'000;

    // Benchmark expected
    auto t1 = std::chrono::high_resolution_clock::now();
    int sum1 = 0;
    for (int i = 0; i < N; ++i) {
        auto r = maybe_fail_expected(i);
        if (r) sum1 += *r;
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark exceptions
    auto t3 = std::chrono::high_resolution_clock::now();
    int sum2 = 0;
    for (int i = 0; i < N; ++i) {
        try {
            sum2 += maybe_fail_exception(i);
        } catch (...) { /* swallow */ }
    }
    auto t4 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "expected:   " << ms(t2 - t1).count() << " ms\n";
    std::cout << "exceptions: " << ms(t4 - t3).count() << " ms\n";
    // Typical result: expected ~0.5ms, exceptions ~500ms (1000x slower on error path!)

    /*
    Readability comparison:

    EXPECTED PIPELINE (linear, composable):
      auto r = step1(input)
                .and_then(step2)
                .and_then(step3)
                .transform(format);

    EXCEPTION CHAIN (nested, non-local control flow):
      try {
          auto a = step1(input);
          try {
              auto b = step2(a);
              auto c = step3(b);
              format(c);
          } catch (const Step2Error&) { ... }
        } catch (const Step1Error&) { ... }
    */
}
```

The 1000x difference in the comment is a realistic worst case when every other call fails. In practice the gap depends on your compiler, platform, and how often the error path is actually hit - but the trend is always the same direction.

---

## Notes

- `transform(f)` vs `and_then(f)`: `transform` wraps `f`'s return in `expected` automatically; `and_then` expects `f` to return `expected` directly. Pick the one that matches your step's signature.
- When errors are **rare** (for example, I/O failures), exceptions may be perfectly acceptable; when errors are **frequent** or **expected** (for example, parsing user input), `std::expected` avoids stack unwinding overhead.
- The pipeline style makes it trivial to add, remove, or reorder steps - just insert another `.and_then()` where you need it.
