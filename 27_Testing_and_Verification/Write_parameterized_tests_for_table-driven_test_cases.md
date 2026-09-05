# Write parameterized tests for table-driven test cases

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://google.github.io/googletest/advanced.html#value-parameterized-tests>  

---

## Topic Overview

**Parameterized tests** (also called table-driven tests) run the same test logic against multiple input/output pairs. Instead of writing N separate `TEST()` functions that all say the same thing, you define one test body and a table of data. This is one of those techniques that feels like a minor convenience until you have 30 test cases - then it becomes indispensable.

### Comparison: Manual vs Parameterized

Here is the same test written both ways. The manual version repeats the `EXPECT_EQ(fizz(...), ...)` pattern six times. The parameterized version writes it once and feeds it data:

```cpp
Manual (6 tests, lots of duplication):          Parameterized (1 test, 6 rows):
TEST(Fizz, 3)  { EXPECT_EQ(fizz(3),  "Fizz"); }   TEST_P(FizzTest, Output) {
TEST(Fizz, 5)  { EXPECT_EQ(fizz(5),  "Buzz"); }       auto [in, out] = GetParam();
TEST(Fizz, 15) { EXPECT_EQ(fizz(15), "FizzBuzz"); }    EXPECT_EQ(fizz(in), out);
TEST(Fizz, 1)  { EXPECT_EQ(fizz(1),  "1"); }       }
TEST(Fizz, 0)  { EXPECT_EQ(fizz(0),  "0"); }       INSTANTIATE with 6 rows...
TEST(Fizz, -3) { EXPECT_EQ(fizz(-3), "Fizz"); }
```

### Framework Support

Most C++ testing frameworks have first-class support for parameterized tests. The syntax varies but the idea is the same:

| Framework | Mechanism | Syntax |
| --- | --- | --- |
| Google Test | `TEST_P` + `INSTANTIATE_TEST_SUITE_P` | Class-based |
| Catch2 | `GENERATE()` | Inline generator |
| doctest | `TEST_CASE_TEMPLATE` / SUBCASE loop | Template or loop |
| Boost.Test | `BOOST_DATA_TEST_CASE` | Data-driven |

---

## Self-Assessment

### Q1: Use INSTANTIATE_TEST_SUITE_P in Google Test to run a test over a vector of input/output pairs

**Answer:**

The Google Test approach has three moving parts: a fixture class that names the parameter type, a `TEST_P` body that unpacks it, and an `INSTANTIATE_TEST_SUITE_P` call that provides the data. Each data row becomes a separate test with its own pass/fail result:

```cpp
#include <gtest/gtest.h>
#include <string>
#include <tuple>

// Function under test
std::string classify_bmi(double bmi) {
    if (bmi < 18.5) return "Underweight";
    if (bmi < 25.0) return "Normal";
    if (bmi < 30.0) return "Overweight";
    return "Obese";
}

// Step 1: Define the test fixture
class BmiTest : public ::testing::TestWithParam<std::tuple<double, std::string>> {};

// Step 2: Write the parameterized test
TEST_P(BmiTest, ClassifiesCorrectly) {
    auto [bmi, expected] = GetParam();
    EXPECT_EQ(classify_bmi(bmi), expected);
}

// Step 3: Instantiate with the data table
INSTANTIATE_TEST_SUITE_P(
    BmiClassification,         // Prefix for test names
    BmiTest,                   // Test fixture class
    ::testing::Values(
        std::make_tuple(15.0,  "Underweight"),
        std::make_tuple(18.4,  "Underweight"),
        std::make_tuple(18.5,  "Normal"),       // Boundary
        std::make_tuple(22.0,  "Normal"),
        std::make_tuple(24.9,  "Normal"),
        std::make_tuple(25.0,  "Overweight"),   // Boundary
        std::make_tuple(29.9,  "Overweight"),
        std::make_tuple(30.0,  "Obese"),        // Boundary
        std::make_tuple(40.0,  "Obese")
    ),
    // Optional: custom name generator for readable output
    [](const auto& info) {
        auto bmi = std::get<0>(info.param);
        return "BMI_" + std::to_string(static_cast<int>(bmi * 10));
    }
);

// Output:
// BmiClassification/BmiTest.ClassifiesCorrectly/BMI_150
// BmiClassification/BmiTest.ClassifiesCorrectly/BMI_184
// BmiClassification/BmiTest.ClassifiesCorrectly/BMI_185
// ...

// Multiple parameters
struct ConversionCase {
    double celsius;
    double expected_fahrenheit;
    std::string name;
};

class TempTest : public ::testing::TestWithParam<ConversionCase> {};

TEST_P(TempTest, CelsiusToFahrenheit) {
    auto c = GetParam();
    double result = c.celsius * 9.0 / 5.0 + 32.0;
    EXPECT_NEAR(result, c.expected_fahrenheit, 0.01);
}

INSTANTIATE_TEST_SUITE_P(
    TemperatureConversions,
    TempTest,
    ::testing::Values(
        ConversionCase{0.0,   32.0,  "Freezing"},
        ConversionCase{100.0, 212.0, "Boiling"},
        ConversionCase{-40.0, -40.0, "Equal"},
        ConversionCase{37.0,  98.6,  "Body"}
    ),
    [](const auto& info) { return info.param.name; }
);
```

The custom name generator at the end of `INSTANTIATE_TEST_SUITE_P` is worth the extra line. Without it, test names come out as `/0`, `/1`, `/2` - unhelpful when one of them fails. With it, you get `BMI_185` or `Freezing`, which tells you immediately what went wrong.

### Q2: Use GENERATE in Catch2 to parameterize a TEST_CASE over multiple values

**Answer:**

Catch2's `GENERATE` is more concise than Google Test's approach - there is no fixture class to define. You just call `GENERATE` inside the test body and the framework re-runs the test for each value. The `table<>` variant lets you pair inputs with expected outputs directly:

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/generators/catch_generators.hpp>
#include <catch2/generators/catch_generators_adapters.hpp>
#include <string>
#include <algorithm>

// Function under test
bool is_palindrome(const std::string& s) {
    std::string lower = s;
    std::transform(lower.begin(), lower.end(), lower.begin(), ::tolower);
    std::string rev = lower;
    std::reverse(rev.begin(), rev.end());
    return lower == rev;
}

// GENERATE: inline parameterization
TEST_CASE("Palindrome detection") {
    // GENERATE creates a parameter - test runs once per value
    auto [input, expected] = GENERATE(table<std::string, bool>({
        {"racecar",     true},
        {"Madam",       true},
        {"hello",       false},
        {"a",           true},
        {"",            true},
        {"ab",          false},
        {"AaBbAa",      false},  // case-sensitive? our impl is case-insensitive
    }));

    CAPTURE(input);  // Show input in failure message
    REQUIRE(is_palindrome(input) == expected);
}

// GENERATE with ranges and combinations
TEST_CASE("Multiplication table boundaries") {
    auto a = GENERATE(range(1, 11));   // 1 through 10
    auto b = GENERATE(range(1, 11));   // 1 through 10 (cartesian product!)

    REQUIRE(a * b == b * a);           // Commutativity
    REQUIRE(a * b > 0);                // Both positive -> product positive
}
// Runs 100 times (10 x 10)

// GENERATE with filter
TEST_CASE("Division by non-zero") {
    auto divisor = GENERATE(filter(
        [](int x) { return x != 0; },  // Skip zero
        range(-10, 11)
    ));

    REQUIRE(100 / divisor != 0);  // No division by zero
}

// GENERATE with map
TEST_CASE("String length categories") {
    auto input = GENERATE(as<std::string>{},
        "", "a", "ab", "abc", "abcde", "abcdefghij"
    );

    auto len = input.length();
    if (len == 0) {
        REQUIRE(input.empty());
    } else {
        REQUIRE(input[0] != '\0');
    }
}
```

Notice that two `GENERATE` calls in the same test create a cartesian product - every combination of `a` and `b` is tested. This is particularly handy for testing commutativity, symmetry, or any property that should hold for all pairs from two sets.

### Q3: Show how parameterized tests reduce boilerplate vs writing one test per case

**Answer:**

Here is a before-and-after for HTTP status text lookup. The "before" version works but has a serious maintenance problem: every new status code requires copy-pasting a line, and if the `EXPECT_EQ` pattern ever needs to change, you have to update it everywhere:

```cpp
// BEFORE: One test per case (75 lines)
TEST(HttpStatus, Ok)           { EXPECT_EQ(status_text(200), "OK"); }
TEST(HttpStatus, Created)      { EXPECT_EQ(status_text(201), "Created"); }
TEST(HttpStatus, BadRequest)   { EXPECT_EQ(status_text(400), "Bad Request"); }
TEST(HttpStatus, Unauthorized) { EXPECT_EQ(status_text(401), "Unauthorized"); }
TEST(HttpStatus, Forbidden)    { EXPECT_EQ(status_text(403), "Forbidden"); }
TEST(HttpStatus, NotFound)     { EXPECT_EQ(status_text(404), "Not Found"); }
TEST(HttpStatus, ServerError)  { EXPECT_EQ(status_text(500), "Internal Server Error"); }
TEST(HttpStatus, BadGateway)   { EXPECT_EQ(status_text(502), "Bad Gateway"); }
TEST(HttpStatus, Unavailable)  { EXPECT_EQ(status_text(503), "Service Unavailable"); }
// Problems:
//   - Adding a new status = copy-paste a line (easy to introduce typos)
//   - Logic is duplicated 9 times
//   - Can't easily loop over all cases for exhaustive check

// AFTER: Parameterized (25 lines)
struct StatusCase {
    int code;
    std::string text;
};

class HttpStatusTest : public ::testing::TestWithParam<StatusCase> {};

TEST_P(HttpStatusTest, ReturnsCorrectText) {
    auto c = GetParam();
    EXPECT_EQ(status_text(c.code), c.text);
}

INSTANTIATE_TEST_SUITE_P(AllStatuses, HttpStatusTest, ::testing::Values(
    StatusCase{200, "OK"},
    StatusCase{201, "Created"},
    StatusCase{400, "Bad Request"},
    StatusCase{401, "Unauthorized"},
    StatusCase{403, "Forbidden"},
    StatusCase{404, "Not Found"},
    StatusCase{500, "Internal Server Error"},
    StatusCase{502, "Bad Gateway"},
    StatusCase{503, "Service Unavailable"}
));

// Benefits:
// Add new case = add one line to the table
// Test logic written ONCE
// Each row runs as a separate test (individual pass/fail)
// Easy to generate from external data

// Even better: combine with typed tests
// Google Test TYPED_TEST for testing same interface across implementations:

template <typename T>
class ContainerTest : public ::testing::Test {};

using ContainerTypes = ::testing::Types<
    std::vector<int>, std::deque<int>, std::list<int>
>;
TYPED_TEST_SUITE(ContainerTest, ContainerTypes);

TYPED_TEST(ContainerTest, PushBackIncreasesSize) {
    TypeParam container;
    container.push_back(42);
    EXPECT_EQ(container.size(), 1u);
    container.push_back(99);
    EXPECT_EQ(container.size(), 2u);
}
// Runs for vector, deque, AND list - zero duplication
```

The `TYPED_TEST` at the end is a different but related tool: instead of varying the data, you vary the *type*. This is how you test that multiple container implementations all satisfy the same contract. Combined with value-parameterized tests, you can build a thorough test matrix with very little code.

---

## Notes

- Google Test: `TEST_P` for value params, `TYPED_TEST` for type params - you can combine both for a matrix of type x value combinations.
- Catch2 `GENERATE()` is more concise - no fixture class needed, which makes it easier to read for simple cases.
- Always use custom name generators in Google Test to get readable names like `BMI_185/Normal` instead of `/2`.
- Table-driven tests excel for pure functions; for stateful tests that need setup/teardown, use fixtures.
- Adding edge cases to a parameterized test is trivial - just add a row to the table. This lowers the cost of being thorough.
- `::testing::Combine(Values(...), Values(...))` creates cartesian products in Google Test, just like nested `GENERATE` calls do in Catch2.
