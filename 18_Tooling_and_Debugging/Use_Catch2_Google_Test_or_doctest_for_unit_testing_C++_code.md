# Use Catch2, Google Test, or doctest for unit testing C++ code

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://github.com/catchorg/Catch2>  

---

## Topic Overview

All three major C++ unit testing frameworks are solid choices. The main differences come down to setup overhead, compilation speed, and which features matter to you. Here is a quick side-by-side to help you pick:

| Framework | Header-only? | Macros | Speed | Notable Feature |
| --- | --- | --- | --- | --- |
| Catch2 v3 | No (lib) | `TEST_CASE`, `REQUIRE` | Fast | Sections, BDD style |
| Google Test | No (lib) | `TEST`, `EXPECT_EQ` | Fast | Mocking (gmock) |
| doctest | Yes | `TEST_CASE`, `CHECK` | Fastest compile | Fastest compilation |

---

## Self-Assessment

### Q1: Parameterized test with a table of inputs

Each framework has its own syntax for running the same test logic against multiple inputs. Here is how to do it in all three.

**Catch2 v3:**

Catch2 gives you two useful approaches: `SECTION` for grouping related checks, and `GENERATE` for driving the same test body with a table of values.

```cpp
// test_math.cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/generators/catch_generators.hpp>
#include <cmath>

int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

TEST_CASE("Factorial is computed correctly", "[math]") {
    // Table-driven test using SECTION:
    SECTION("Base cases") {
        CHECK(factorial(0) == 1);
        CHECK(factorial(1) == 1);
    }
    SECTION("General cases") {
        CHECK(factorial(5) == 120);
        CHECK(factorial(10) == 3628800);
    }
}

// Using GENERATE for parameterized tests:
TEST_CASE("Square root of perfect squares", "[math]") {
    auto [input, expected] = GENERATE(table<int, int>({
        {0, 0}, {1, 1}, {4, 2}, {9, 3}, {16, 4}, {25, 5}
    }));

    REQUIRE(static_cast<int>(std::sqrt(input)) == expected);
}
```

**Google Test:**

Google Test's parameterized approach is more verbose but very explicit. You define a test fixture, write the parameterized test body, then separately instantiate it with your value table.

```cpp
#include <gtest/gtest.h>
#include <cmath>

struct SqrtParam {
    int input;
    int expected;
};

class SqrtTest : public testing::TestWithParam<SqrtParam> {};

TEST_P(SqrtTest, PerfectSquares) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(static_cast<int>(std::sqrt(input)), expected);
}

INSTANTIATE_TEST_SUITE_P(Math, SqrtTest, testing::Values(
    SqrtParam{0, 0}, SqrtParam{1, 1}, SqrtParam{4, 2},
    SqrtParam{9, 3}, SqrtParam{16, 4}, SqrtParam{25, 5}
));
```

**doctest:**

doctest is the simplest to integrate - it is header-only and compiles faster than the other two. The parameterization is manual but perfectly readable:

```cpp
#define DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN
#include <doctest/doctest.h>
#include <cmath>

TEST_CASE("sqrt of perfect squares") {
    int inputs[]   = {0, 1, 4, 9, 16, 25};
    int expected[] = {0, 1, 2, 3,  4,  5};

    for (int i = 0; i < 6; ++i) {
        CAPTURE(inputs[i]);
        CHECK(static_cast<int>(std::sqrt(inputs[i])) == expected[i]);
    }
}
```

### Q2: REQUIRE vs CHECK in Catch2

The distinction between `REQUIRE` and `CHECK` is one of the most practical things to know about Catch2. The rule of thumb is simple: use `REQUIRE` when a failure would make subsequent assertions meaningless or dangerous, and use `CHECK` when assertions are independent and you want to see all the failures at once.

```cpp
#include <catch2/catch_test_macros.hpp>
#include <vector>

std::vector<int> get_data() { return {1, 2, 3}; }

TEST_CASE("REQUIRE vs CHECK demonstration") {
    auto data = get_data();

    // REQUIRE = hard failure: stops test immediately if false
    // Use when subsequent assertions depend on this being true
    REQUIRE(!data.empty());      // If empty, data[0] would crash!
    REQUIRE(data.size() >= 3);   // Need at least 3 elements below

    // CHECK = soft failure: continues test even if false
    // Use when assertions are independent of each other
    CHECK(data[0] == 1);  // If this fails, still check the others
    CHECK(data[1] == 2);  // Reports ALL failures, not just first
    CHECK(data[2] == 3);
}

// Rule of thumb:
//   REQUIRE for preconditions (would crash/UB if false)
//   CHECK for independent assertions (want to see all failures)
```

The reason this distinction matters: if you use `REQUIRE` everywhere, the first failing assertion stops the entire test and you might miss several other failures that could have helped you diagnose the root cause. If you use `CHECK` everywhere, you risk a null pointer dereference or out-of-bounds access on a line that only makes sense if an earlier check passed.

### Q3: Code coverage with CMake + gcov/llvm-cov

Knowing which lines your tests actually exercise is important for finding gaps. Coverage is enabled at the compiler level with special instrumentation flags, and CMake makes it easy to turn on via a build option.

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(CoverageDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)

# Coverage option
option(ENABLE_COVERAGE "Enable coverage reporting" OFF)

if(ENABLE_COVERAGE)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        add_compile_options(--coverage -fprofile-arcs -ftest-coverage)
        add_link_options(--coverage)
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        add_compile_options(-fprofile-instr-generate -fcoverage-mapping)
        add_link_options(-fprofile-instr-generate)
    endif()
endif()

add_library(mylib src/math.cpp)
add_executable(tests tests/test_math.cpp)
target_link_libraries(tests mylib)

enable_testing()
add_test(NAME unit_tests COMMAND tests)
```

With GCC you run `gcovr` after the tests execute; with Clang you merge profile data first:

```bash
# GCC + gcov workflow:
cmake -B build -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
cmake --build build
cd build && ctest
gcovr --root .. --html --html-details -o coverage.html
# Open coverage.html in browser

# Clang + llvm-cov workflow:
cmake -B build -DENABLE_COVERAGE=ON -DCMAKE_CXX_COMPILER=clang++
cmake --build build
cd build && LLVM_PROFILE_FILE="coverage.profraw" ctest
llvm-profdata merge -sparse coverage.profraw -o coverage.profdata
llvm-cov report ./tests -instr-profile=coverage.profdata
llvm-cov show ./tests -instr-profile=coverage.profdata --format=html > coverage.html
```

---

## Notes

- Catch2 v3 requires CMake `FetchContent` or package manager; v2 was header-only.
- doctest compiles ~10x faster than Catch2/gtest - great for TDD workflows.
- Google Test has the best mocking support (gmock).
- All three integrate with CTest: `add_test(NAME ... COMMAND ...)`.
- Target 80%+ line coverage; 100% is usually not cost-effective.
