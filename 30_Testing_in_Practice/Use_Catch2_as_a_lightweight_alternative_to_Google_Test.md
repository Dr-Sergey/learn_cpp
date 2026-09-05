# Use Catch2 as a lightweight alternative to Google Test

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Catch2** is a header-only (v2) or single-library (v3) C++ test framework known for its expressive syntax, minimal boilerplate, and BDD-style sections. Unlike Google Test, Catch2 requires no test registration macros - tests are self-registering. If you have ever been frustrated by the ceremony around `TEST_F` and `INSTANTIATE_TEST_SUITE_P`, Catch2's approach will feel refreshingly direct.

### Google Test vs Catch2

The table below summarizes the main differences. The columns that matter most in practice are the fixture model and the parameterization story - those are where the two frameworks feel most different.

| Feature | Google Test | Catch2 |
| --- | --- | --- |
| **Setup** | FetchContent or library link | Header-only (v2) or CMake (v3) |
| **Test syntax** | `TEST()`, `TEST_F()` | `TEST_CASE()`, `SECTION()` |
| **Assertions** | `EXPECT_EQ(a, b)` | `REQUIRE(a == b)`, `CHECK(a == b)` |
| **Fixture** | Class-based `TEST_F` | `SECTION()` nesting (no class needed) |
| **Mocking** | Built-in Google Mock | Use trompeloeil or manual mocks |
| **BDD** | Not built-in | `SCENARIO`/`GIVEN`/`WHEN`/`THEN` |
| **Matchers** | `EXPECT_THAT(x, Gt(5))` | `REQUIRE_THAT(x, Catch::Matchers::WithinAbs(5, 0.1))` |
| **Parameterized** | `TEST_P()` + `INSTANTIATE_` | `GENERATE()` inline |
| **Output** | XML, JSON | JUnit XML, TAP, compact, console |

---

## Self-Assessment

### Q1: Write tests with Catch2 using TEST_CASE and SECTION

**Answer:**

The key thing to understand before reading this code is the `SECTION` model. Each `SECTION` block causes Catch2 to re-run all the code above it - so the stack `s` is freshly constructed before each section executes. You get fixture-level isolation without a fixture class.

```cpp
// === CMakeLists.txt for Catch2 v3 ===
/*
include(FetchContent)
FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG v3.5.2
)
FetchContent_MakeAvailable(Catch2)

add_executable(tests test_main.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2WithMain)

include(Catch)
catch_discover_tests(tests)
*/

#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_string.hpp>
#include <string>
#include <vector>
#include <stdexcept>

// === Production code ===
class Stack {
public:
    void push(int value) { data_.push_back(value); }

    int pop() {
        if (data_.empty()) throw std::underflow_error("Stack empty");
        int val = data_.back();
        data_.pop_back();
        return val;
    }

    int top() const {
        if (data_.empty()) throw std::underflow_error("Stack empty");
        return data_.back();
    }

    bool empty() const { return data_.empty(); }
    size_t size() const { return data_.size(); }

private:
    std::vector<int> data_;
};

// === Catch2 Tests ===

// SECTIONs share the setup but run independently.
// Each SECTION re-runs the code before it (like a fresh fixture).
TEST_CASE("Stack operations", "[stack]") {
    Stack s;  // Fresh stack for EACH section

    SECTION("newly created stack is empty") {
        REQUIRE(s.empty());
        REQUIRE(s.size() == 0);
    }

    SECTION("pushing an element") {
        s.push(42);

        REQUIRE_FALSE(s.empty());
        REQUIRE(s.size() == 1);
        REQUIRE(s.top() == 42);

        SECTION("and then popping") {
            int val = s.pop();
            REQUIRE(val == 42);
            REQUIRE(s.empty());
        }

        SECTION("and pushing another") {
            s.push(99);
            REQUIRE(s.size() == 2);
            REQUIRE(s.top() == 99);  // LIFO
        }
    }

    SECTION("popping from empty stack throws") {
        REQUIRE_THROWS_AS(s.pop(), std::underflow_error);
        REQUIRE_THROWS_WITH(s.pop(), "Stack empty");
    }
}

// CHECK vs REQUIRE:
// - REQUIRE: fatal, stops test on failure (like ASSERT_*)
// - CHECK: non-fatal, continues test (like EXPECT_*)
TEST_CASE("CHECK continues on failure", "[stack]") {
    Stack s;
    s.push(1);
    s.push(2);

    CHECK(s.size() == 2);   // Non-fatal
    CHECK(s.top() == 2);    // Continues even if above fails
    REQUIRE(s.pop() == 2);  // Fatal
}
```

The `REQUIRE` vs `CHECK` split maps directly onto Google Test's `ASSERT_*` vs `EXPECT_*`. Use `REQUIRE` when the remaining assertions are meaningless if this one fails; use `CHECK` when you want to accumulate all failures in one run.

### Q2: Use Catch2's BDD syntax and GENERATE for parameterized tests

**Answer:**

The `SCENARIO`/`GIVEN`/`WHEN`/`THEN` macros are just renamed `TEST_CASE` and `SECTION` under the hood - they add no runtime overhead, they only improve readability for requirement-style tests. `GENERATE` is Catch2's killer feature for data-driven tests: you declare the set of inputs inline, right where they are used, without any separate `INSTANTIATE_TEST_SUITE_P` boilerplate.

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/generators/catch_generators.hpp>
#include <catch2/generators/catch_generators_range.hpp>
#include <cmath>

// === BDD Style ===
SCENARIO("Withdrawing from an account", "[account]") {
    GIVEN("An account with $100") {
        double balance = 100.0;

        WHEN("$50 is withdrawn") {
            balance -= 50.0;

            THEN("balance is $50") {
                REQUIRE(balance == 50.0);
            }
        }

        WHEN("$200 is withdrawn") {
            // Overdraft scenario
            THEN("it should fail or go negative") {
                // Business rule: don't allow overdraft
                REQUIRE(balance < 200.0);
            }
        }
    }
}

// === GENERATE: Inline parameterized tests ===
TEST_CASE("Square root of perfect squares", "[math]") {
    auto value = GENERATE(1, 4, 9, 16, 25, 36, 49, 64, 81, 100);

    int root = static_cast<int>(std::sqrt(value));
    REQUIRE(root * root == value);
}

// GENERATE with ranges
TEST_CASE("Fibonacci is always positive", "[math]") {
    auto n = GENERATE(range(1, 20));

    // Simple fib
    auto fib = [](int n) {
        int a = 0, b = 1;
        for (int i = 0; i < n; ++i) { int t = b; b = a + b; a = t; }
        return a;
    };

    CAPTURE(n);  // Print n on failure
    REQUIRE(fib(n) > 0);
}

// GENERATE with table-like data
TEST_CASE("String operations", "[string]") {
    auto [input, expected_upper] = GENERATE(table<std::string, std::string>({
        {"hello", "HELLO"},
        {"World", "WORLD"},
        {"", ""},
        {"123", "123"},
    }));

    std::string result = input;
    std::transform(result.begin(), result.end(), result.begin(), ::toupper);
    REQUIRE(result == expected_upper);
}
```

Notice `CAPTURE(n)` - that is Catch2's way of printing the current value of a variable when a test fails. This is invaluable when a parameterized test fails and you need to know which input triggered it.

### Q3: Show Catch2 matchers and custom matchers for domain assertions

**Answer:**

Matchers give you much richer failure messages than plain `REQUIRE(a == b)`. When a plain assertion fails, you see the raw values. When a matcher fails, you see a sentence like "expected 3.14 to be within 0.001 of 3.14159" - which makes the nature of the failure immediately obvious. Writing a custom matcher for your own domain types is straightforward, as the example below shows.

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_all.hpp>
#include <vector>
#include <string>

using Catch::Matchers::StartsWith;
using Catch::Matchers::EndsWith;
using Catch::Matchers::ContainsSubstring;
using Catch::Matchers::WithinAbs;
using Catch::Matchers::WithinRel;
using Catch::Matchers::SizeIs;
using Catch::Matchers::Contains;
using Catch::Matchers::UnorderedEquals;

TEST_CASE("Built-in matchers", "[matchers]") {
    // String matchers
    std::string msg = "Error: file not found";
    REQUIRE_THAT(msg, StartsWith("Error") && ContainsSubstring("not found"));

    // Floating point matchers
    double pi = 3.14159265;
    REQUIRE_THAT(pi, WithinAbs(3.14159, 0.001));
    REQUIRE_THAT(pi, WithinRel(3.14159, 0.001));  // 0.1% relative

    // Container matchers
    std::vector<int> v = {1, 2, 3, 4, 5};
    REQUIRE_THAT(v, SizeIs(5));
    REQUIRE_THAT(v, Contains(3));
    REQUIRE_THAT(v, UnorderedEquals(std::vector{5, 3, 1, 2, 4}));
}

// === Custom Matcher ===
struct IsEvenMatcher : Catch::Matchers::MatcherBase<int> {
    bool match(int const& val) const override {
        return val % 2 == 0;
    }

    std::string describe() const override {
        return "is even";
    }
};

inline IsEvenMatcher IsEven() { return {}; }

// Domain-specific matcher
struct IsValidEmailMatcher : Catch::Matchers::MatcherBase<std::string> {
    bool match(const std::string& val) const override {
        auto at = val.find('@');
        if (at == std::string::npos || at == 0) return false;
        auto dot = val.find('.', at);
        return dot != std::string::npos && dot > at + 1 && dot < val.size() - 1;
    }

    std::string describe() const override {
        return "is a valid email address";
    }
};

inline IsValidEmailMatcher IsValidEmail() { return {}; }

TEST_CASE("Custom matchers", "[matchers]") {
    REQUIRE_THAT(42, IsEven());
    REQUIRE_THAT("user@example.com", IsValidEmail());
    REQUIRE_THAT("invalid", !IsValidEmail());
}
```

The `describe()` method is what appears in the failure message, so make it a proper English sentence. A failure from `IsValidEmail` will print something like "expected 'invalid' to match: is a valid email address", which is far more useful than a raw boolean comparison.

---

## Notes

- Catch2 v3 is not header-only - it's a compiled library. Use CMake FetchContent.
- `SECTION()` is Catch2's killer feature: replaces fixtures with nested, composable setup.
- Each `SECTION` re-executes all code above it - like a separate test with shared arrange.
- `GENERATE()` replaces parameterized tests - cleaner, inline, no separate `INSTANTIATE_`.
- Use `[tags]` to organize and filter tests: `--tag [math]` runs only math tests.
- For mocking with Catch2, use **trompeloeil** (C++ mock framework compatible with Catch2).
- `catch_discover_tests()` in CMake is analogous to `gtest_discover_tests()`.
- `REQUIRE_THAT(value, matcher)` gives much better error messages than plain `REQUIRE(value == x)`.
