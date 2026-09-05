# Implement parameterized tests for data-driven testing

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Parameterized tests** run the same test logic with different data inputs. The idea is to avoid the copy-paste problem where you write essentially the same test five times with slightly different inputs and expected outputs. With parameterized tests, you write the logic once and supply a table of cases. Adding a new case is a one-liner, and every case gets the same rigorous test code automatically.

Google Test provides `TEST_P` with `INSTANTIATE_TEST_SUITE_P` for value-parameterized tests and `TYPED_TEST` for type-parameterized tests. Catch2 offers `GENERATE()` as a simpler but less flexible alternative.

### When to Use Parameterized Tests

| Scenario | Example |
| --- | --- |
| Same validation, different inputs | Parsing various date formats |
| Equivalence class testing | Positive, negative, zero, boundary values |
| Cross-platform behavior | Same logic on int, long, double |
| Truth tables | Input combinations with expected output |
| Regression | Known bug inputs with expected outputs |

---

## Self-Assessment

### Q1: Implement value-parameterized tests in Google Test

**Answer:**

The pattern has three parts: a fixture class that inherits from `TestWithParam<T>`, a `TEST_P` body that calls `GetParam()` to retrieve the current case, and an `INSTANTIATE_TEST_SUITE_P` macro that supplies the actual data. Google Test generates one named test case for each entry.

```cpp
#include <gtest/gtest.h>
#include <string>
#include <tuple>
#include <cmath>
#include <optional>

// === Production code ===
std::optional<int> parse_int(const std::string& s) {
    try {
        size_t pos;
        int val = std::stoi(s, &pos);
        if (pos != s.size()) return std::nullopt;  // Trailing garbage
        return val;
    } catch (...) {
        return std::nullopt;
    }
}

// === Simple value parameterization ===

class ParseIntValidTest : public ::testing::TestWithParam<std::pair<std::string, int>> {};

TEST_P(ParseIntValidTest, ParsesCorrectly) {
    auto [input, expected] = GetParam();
    auto result = parse_int(input);
    ASSERT_TRUE(result.has_value()) << "Failed to parse: " << input;
    EXPECT_EQ(*result, expected);
}

INSTANTIATE_TEST_SUITE_P(
    ValidIntegers, ParseIntValidTest,
    ::testing::Values(
        std::make_pair("0", 0),
        std::make_pair("42", 42),
        std::make_pair("-1", -1),
        std::make_pair("2147483647", 2147483647),   // INT_MAX
        std::make_pair("-2147483648", -2147483648),  // INT_MIN
        std::make_pair("+42", 42),
        std::make_pair("00123", 123)  // Leading zeros
    )
);

// === Invalid inputs should return nullopt ===

class ParseIntInvalidTest : public ::testing::TestWithParam<std::string> {};

TEST_P(ParseIntInvalidTest, ReturnsNullopt) {
    EXPECT_FALSE(parse_int(GetParam()).has_value())
        << "Should not parse: " << GetParam();
}

INSTANTIATE_TEST_SUITE_P(
    InvalidInputs, ParseIntInvalidTest,
    ::testing::Values(
        "",
        "abc",
        "12.5",
        "42abc",       // Trailing non-digits
        "  42",        // Leading spaces (depends on your contract)
        "99999999999999999999"  // Overflow
    )
);

// === Struct-based parameterization (clearer than tuples) ===

struct MathTestCase {
    double input;
    double expected;
    std::string name;

    // Custom name generator
    friend std::ostream& operator<<(std::ostream& os, const MathTestCase& tc) {
        return os << tc.name;
    }
};

class SqrtTest : public ::testing::TestWithParam<MathTestCase> {};

TEST_P(SqrtTest, ComputesCorrectly) {
    auto [input, expected, name] = GetParam();
    EXPECT_NEAR(std::sqrt(input), expected, 1e-10)
        << "Failed for: " << name;
}

INSTANTIATE_TEST_SUITE_P(
    SqrtCases, SqrtTest,
    ::testing::Values(
        MathTestCase{0.0, 0.0, "zero"},
        MathTestCase{1.0, 1.0, "one"},
        MathTestCase{4.0, 2.0, "four"},
        MathTestCase{9.0, 3.0, "nine"},
        MathTestCase{2.0, 1.41421356237, "two"}
    ),
    // Custom name generator for readable test names
    [](const ::testing::TestParamInfo<MathTestCase>& info) {
        return info.param.name;
    }
);
```

Notice the custom name generator at the end of `INSTANTIATE_TEST_SUITE_P`. Without it, Google Test names the cases `SqrtCases/SqrtTest.ComputesCorrectly/0`, `...1`, and so on, which is useless in a failure report. With it, you get `SqrtCases/SqrtTest.ComputesCorrectly/zero` - immediately obvious what failed.

### Q2: Use type-parameterized tests to test across container types

**Answer:**

Type-parameterized tests are different from value-parameterized tests: instead of varying the *data*, you vary the *type* the test operates on. This is the right tool for testing generic algorithms or verifying that multiple container types all meet the same behavioral contract.

```cpp
#include <gtest/gtest.h>
#include <vector>
#include <deque>
#include <list>
#include <algorithm>

// === Type-parameterized: same tests for different types ===

template<typename T>
class ContainerTest : public ::testing::Test {
protected:
    T container;

    void fill(std::initializer_list<int> values) {
        for (int v : values)
            container.push_back(v);
    }
};

using ContainerTypes = ::testing::Types<
    std::vector<int>,
    std::deque<int>,
    std::list<int>
>;

TYPED_TEST_SUITE(ContainerTest, ContainerTypes);

TYPED_TEST(ContainerTest, EmptyByDefault) {
    EXPECT_TRUE(this->container.empty());
    EXPECT_EQ(this->container.size(), 0);
}

TYPED_TEST(ContainerTest, PushBackIncreasesSize) {
    this->container.push_back(42);
    EXPECT_EQ(this->container.size(), 1);
    EXPECT_EQ(this->container.front(), 42);
}

TYPED_TEST(ContainerTest, ClearMakesEmpty) {
    this->fill({1, 2, 3});
    this->container.clear();
    EXPECT_TRUE(this->container.empty());
}

TYPED_TEST(ContainerTest, SortProducesSameResult) {
    this->fill({3, 1, 4, 1, 5});

    // std::list has its own sort
    if constexpr (std::is_same_v<TypeParam, std::list<int>>) {
        this->container.sort();
    } else {
        std::sort(this->container.begin(), this->container.end());
    }

    auto it = this->container.begin();
    EXPECT_EQ(*it++, 1);
    EXPECT_EQ(*it++, 1);
    EXPECT_EQ(*it++, 3);
}


// === Combine value AND type parameterization ===
// Use when testing N types x M values

template<typename T>
class NumericBoundaryTest : public ::testing::Test {};

using NumericTypes = ::testing::Types<int, long, int64_t>;
TYPED_TEST_SUITE(NumericBoundaryTest, NumericTypes);

TYPED_TEST(NumericBoundaryTest, MaxValueDoesNotOverflowOnIncrement) {
    TypeParam max_val = std::numeric_limits<TypeParam>::max();
    // This would overflow - verify the raw max value is correct
    EXPECT_GT(max_val, 0);
}
```

Inside a `TYPED_TEST`, `TypeParam` is the concrete type for the current instantiation. The `if constexpr` handles the case where different types need slightly different treatment. That is a feature, not a limitation - it lets you keep one test suite for a family of types even when they diverge slightly in their API.

### Q3: Show advanced patterns: Combine, ValuesIn, and custom generators

**Answer:**

`Combine` generates the cross-product of multiple parameter sets. `ValuesIn` loads test data from a container computed at runtime (a file, a database, a generation function). Both are useful when you need systematic coverage without manually listing every combination.

```cpp
#include <gtest/gtest.h>
#include <string>
#include <vector>
#include <tuple>

// === Combine: cross-product of parameters ===

enum class Protocol { HTTP, HTTPS, WS };
enum class AuthMethod { None, Basic, Token };

class ConnectionTest : public ::testing::TestWithParam<
    std::tuple<Protocol, AuthMethod, int /*port*/>> {};

TEST_P(ConnectionTest, CanEstablishConnection) {
    auto [protocol, auth, port] = GetParam();
    // Test all combinations of protocol x auth x port
    EXPECT_TRUE(port > 0 && port < 65536);
    // ... actual connection test
}

INSTANTIATE_TEST_SUITE_P(
    AllCombinations, ConnectionTest,
    ::testing::Combine(
        ::testing::Values(Protocol::HTTP, Protocol::HTTPS, Protocol::WS),
        ::testing::Values(AuthMethod::None, AuthMethod::Basic, AuthMethod::Token),
        ::testing::Values(80, 443, 8080)
    )
    // This generates 3 x 3 x 3 = 27 test cases!
);

// === ValuesIn: parameters from runtime data ===

std::vector<std::string> load_test_cases_from_file() {
    // In practice, read from a JSON/CSV file
    return {"case1", "case2", "case3", "regression_bug_123"};
}

class FileBasedTest : public ::testing::TestWithParam<std::string> {};

TEST_P(FileBasedTest, ProcessesInput) {
    auto input = GetParam();
    EXPECT_FALSE(input.empty());
}

INSTANTIATE_TEST_SUITE_P(
    FromFile, FileBasedTest,
    ::testing::ValuesIn(load_test_cases_from_file())
);

// === Custom name generator for readable output ===

struct HttpTestCase {
    int status_code;
    std::string body;
    bool should_retry;
};

class HttpRetryTest : public ::testing::TestWithParam<HttpTestCase> {};

TEST_P(HttpRetryTest, RetryDecision) {
    auto tc = GetParam();
    // bool actual = should_retry_request(tc.status_code);
    // EXPECT_EQ(actual, tc.should_retry);
}

INSTANTIATE_TEST_SUITE_P(
    HttpCodes, HttpRetryTest,
    ::testing::Values(
        HttpTestCase{200, "OK", false},
        HttpTestCase{500, "Internal Server Error", true},
        HttpTestCase{503, "Service Unavailable", true},
        HttpTestCase{404, "Not Found", false},
        HttpTestCase{429, "Too Many Requests", true}
    ),
    [](const ::testing::TestParamInfo<HttpTestCase>& info) {
        return "status_" + std::to_string(info.param.status_code);
    }
);
```

The `Combine` pattern is powerful but watch the combinatorial explosion. Three values times three values times three values already gives 27 test cases. Four dimensions with four values each would be 256. Make sure you actually need all those combinations before reaching for `Combine`.

---

## Notes

- Prefer structs over `std::tuple` for test parameters - named fields are self-documenting and failures are much easier to read.
- Always provide a custom name generator - default numeric names like `/0`, `/1`, `/2` are useless when a test fails.
- `Combine()` generates the full cross-product of its arguments - watch for combinatorial explosion with more than two or three dimensions.
- `ValuesIn()` enables loading test data from files or other runtime sources, which is great for regression test suites built from real-world bug reports.
- Type-parameterized tests with `TYPED_TEST` are the right tool for testing generic algorithms or container-like classes against multiple concrete types.
- For very large parameter sets, consider generating them in a separate helper file and using `ValuesIn()` to keep the test file readable.
- Do not parameterize if each case genuinely needs different assertions - use separate `TEST` functions for those. Parameterization only pays off when the test logic is truly identical across cases.
- The Catch2 alternative `GENERATE(values...)` is simpler to write but less flexible than gtest's suite paradigm for large test matrices.
