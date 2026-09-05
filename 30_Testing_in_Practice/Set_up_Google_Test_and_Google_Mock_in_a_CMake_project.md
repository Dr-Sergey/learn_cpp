# Set up Google Test and Google Mock in a CMake project

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Google Test (gtest) and Google Mock (gmock) are the de-facto standard C++ testing framework, and they integrate tightly with CMake. Getting the integration right from the start matters because it determines whether your tests are discoverable by `ctest`, parallelizable across CI matrix builds, and reproducible across developer machines.

The most important modern practice here is using `FetchContent` to pull a pinned version of googletest directly into your build. The reason this beats using a system-installed gtest is version stability - two developers with different distros can have very different gtest versions installed, and test behavior (including mock expectations) can vary between releases. With `FetchContent` and a pinned `GIT_TAG`, everyone builds against exactly the same version.

### Integration Methods

| Method | Pros | Cons |
| --- | --- | --- |
| **FetchContent** (recommended) | Pinned version, no manual downloads | Slower first configure |
| Git submodule | Works offline after clone | Must manage submodule updates |
| `find_package` (system) | Fast configure | Version mismatches, not portable |
| Conan / vcpkg | Package manager benefits | Extra tooling dependency |

### Project Layout

Here's the recommended project structure. The key insight is that `src/` builds a library that the test executable links against - this means your test binary never needs access to `main.cpp` or any platform-specific entry point.

```cpp
project/
├── CMakeLists.txt          # Top-level: project, FetchContent
├── src/
│   ├── CMakeLists.txt      # Library target
│   ├── calculator.h
│   └── calculator.cpp
└── tests/
    ├── CMakeLists.txt      # Test targets
    ├── calculator_test.cpp
    └── integration_test.cpp
```

---

## Self-Assessment

### Q1: Set up a complete CMake project with Google Test using FetchContent

**Answer:**

Let's walk through this layer by layer. The top-level `CMakeLists.txt` is minimal - it sets up the project, includes the source, and conditionally adds tests behind a `BUILD_TESTING` option. The `enable_testing()` call must be in the top-level file (not in the tests subdirectory) for `ctest` to work correctly.

```cmake
# === CMakeLists.txt (top-level) ===
cmake_minimum_required(VERSION 3.14)
project(MyProject VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Production code
add_subdirectory(src)

# Tests - controlled by option so users can skip
option(BUILD_TESTING "Build the test suite" ON)
if(BUILD_TESTING)
    enable_testing()  # Must be in top-level CMakeLists.txt for ctest
    add_subdirectory(tests)
endif()
```

The source library is straightforward - it exposes its include directory publicly so anything that links against it automatically gets the headers:

```cmake
# === src/CMakeLists.txt ===
add_library(calculator
    calculator.cpp
)
target_include_directories(calculator PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
```

Here's the class under test. It's a simple calculator, but notice it has a stateful method (`accumulate`) and a throwing method (`divide`) - both are important to test because they exercise patterns that appear constantly in real code:

```cpp
// === src/calculator.h ===
#pragma once

class Calculator {
public:
    double add(double a, double b) const;
    double divide(double a, double b) const;  // throws on div by zero
    double accumulate(double value);           // stateful
    double total() const;
private:
    double total_ = 0.0;
};
```

```cpp
// === src/calculator.cpp ===
#include "calculator.h"
#include <stdexcept>

double Calculator::add(double a, double b) const { return a + b; }

double Calculator::divide(double a, double b) const {
    if (b == 0.0) throw std::invalid_argument("Division by zero");
    return a / b;
}

double Calculator::accumulate(double value) {
    total_ += value;
    return total_;
}

double Calculator::total() const { return total_; }
```

The tests `CMakeLists.txt` is where `FetchContent` lives. Pay attention to the `gtest_force_shared_crt` option - on Windows, skipping this causes linker errors because gtest and your code end up compiled against different C runtime versions:

```cmake
# === tests/CMakeLists.txt ===
include(FetchContent)

FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0  # Pin to specific release
)

# Prevent gtest from overriding our compiler settings on Windows
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)

FetchContent_MakeAvailable(googletest)

# Include GoogleTest module for gtest_discover_tests
include(GoogleTest)

# Unit test executable
add_executable(calculator_test
    calculator_test.cpp
)
target_link_libraries(calculator_test
    PRIVATE
        calculator       # Library under test
        GTest::gtest_main  # Provides main()
        GTest::gmock       # Google Mock
)

# Auto-discover tests for ctest
gtest_discover_tests(calculator_test)
```

And here are the tests themselves. Notice `EXPECT_THROW` for the exception case and `EXPECT_DOUBLE_EQ` for floating-point comparisons (never use `==` with doubles in tests):

```cpp
// === tests/calculator_test.cpp ===
#include <gtest/gtest.h>
#include "calculator.h"

TEST(CalculatorTest, AddPositiveNumbers) {
    Calculator calc;
    EXPECT_DOUBLE_EQ(calc.add(2.0, 3.0), 5.0);
}

TEST(CalculatorTest, DivideByZeroThrows) {
    Calculator calc;
    EXPECT_THROW(calc.divide(1.0, 0.0), std::invalid_argument);
}

TEST(CalculatorTest, AccumulateState) {
    Calculator calc;
    calc.accumulate(10.0);
    calc.accumulate(20.0);
    EXPECT_DOUBLE_EQ(calc.total(), 30.0);
}
```

To build and run:

```bash
# Build and run:
mkdir build && cd build
cmake .. -DBUILD_TESTING=ON
cmake --build .
ctest --output-on-failure -j$(nproc)
```

### Q2: What are the key configuration options and best practices

**Answer:**

Here are the decisions that matter most for a maintainable test setup. The distinction between `EXPECT_*` and `ASSERT_*` is worth understanding well - `ASSERT_*` stops the current test function on failure, which can hide subsequent failures. Prefer `EXPECT_*` unless the test genuinely cannot continue after a particular check fails.

| Practice | Why |
| --- | --- |
| Use `gtest_discover_tests` over `add_test` | Auto-discovers TEST macros; no manual registration |
| Pin `GIT_TAG` to a release | Reproducible builds |
| `gtest_force_shared_crt ON` on Windows | Avoids CRT mismatch linker errors |
| Link `GTest::gtest_main` | Don't write your own `main()` unless customizing |
| Use `EXPECT_*` not `ASSERT_*` by default | `EXPECT` continues test; `ASSERT` aborts the test case |
| Separate test executables by module | Parallel ctest execution; faster feedback |
| `BUILD_TESTING` option | Let consumers disable tests |

The most common mistake with `add_test` versus `gtest_discover_tests` is worth spelling out. With `add_test`, CTest sees your test executable as a single test - so if any `TEST()` case inside it fails, the entire thing is marked as failed and you lose all the individual pass/fail granularity. With `gtest_discover_tests`, CTest registers each `TEST()` macro as its own CTest test, giving you fine-grained results and the ability to run individual test cases with `-R`.

```cmake
# BAD: Don't use add_test manually with gtest
add_test(NAME my_test COMMAND calculator_test)  # Discovers only 1 "test"

# GOOD: Auto-discover individual TEST() entries
gtest_discover_tests(calculator_test
    PROPERTIES TIMEOUT 30          # Per-test timeout
    DISCOVERY_TIMEOUT 10           # Discovery phase timeout
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)
```

### Q3: Show advanced setup with multiple test suites, custom main, and CI integration

**Answer:**

As a project grows you'll want to separate fast unit tests from slower integration tests, share test utility code, and occasionally take control of `main()` for global setup. Here's how to do all three:

```cmake
# === Advanced tests/CMakeLists.txt ===

# Shared test utilities library
add_library(test_utils STATIC test_helpers.cpp)
target_link_libraries(test_utils PUBLIC GTest::gtest GTest::gmock calculator)

# Unit tests
add_executable(unit_tests
    calculator_test.cpp
    parser_test.cpp
)
target_link_libraries(unit_tests PRIVATE test_utils GTest::gtest_main GTest::gmock)
gtest_discover_tests(unit_tests PROPERTIES LABELS "unit")

# Integration tests (may be slower)
add_executable(integration_tests
    database_integration_test.cpp
)
target_link_libraries(integration_tests PRIVATE test_utils GTest::gtest_main)
gtest_discover_tests(integration_tests PROPERTIES LABELS "integration")

# Custom main for tests that need global setup
add_executable(custom_main_tests
    custom_main_test.cpp
)
target_link_libraries(custom_main_tests PRIVATE test_utils GTest::gtest GTest::gmock)
gtest_discover_tests(custom_main_tests)
```

When you need one-time setup before any test runs (like starting an embedded database or creating a test fixture directory), use a `::testing::Environment`. This runs its `SetUp` before the first test and `TearDown` after the last, regardless of how many test cases there are:

<!-- compile: needs third-party library header `gtest/gtest.h` -->
```cpp
// === custom_main_test.cpp ===
#include <gtest/gtest.h>
#include <gmock/gmock.h>

// Global test environment for one-time setup/teardown
class DatabaseEnvironment : public ::testing::Environment {
public:
    void SetUp() override {
        // Start embedded database, create schema, etc.
        std::cout << "[GLOBAL] Database initialized\n";
    }
    void TearDown() override {
        std::cout << "[GLOBAL] Database cleaned up\n";
    }
};

int main(int argc, char** argv) {
    ::testing::InitGoogleMock(&argc, argv);
    ::testing::AddGlobalTestEnvironment(new DatabaseEnvironment);
    return RUN_ALL_TESTS();
}
```

In CI you can use the `LABELS` you set above to run only the fast tests on every PR and the full suite on main branch pushes:

```bash
# CI: Run only unit tests, with JUnit XML output
ctest --test-dir build -L unit --output-junit results.xml --output-on-failure -j4

# Run with verbose output
ctest --test-dir build --verbose

# Run specific test by regex
ctest --test-dir build -R "Calculator.*Divide"
```

---

## Notes

- `GTest::gtest_main` provides a default `main()` - only write custom main for global setup.
- `gtest_discover_tests` runs at build time (not configure time) - more robust than `gtest_add_tests`.
- On Windows: always set `gtest_force_shared_crt ON` before `FetchContent_MakeAvailable`.
- Use `PROPERTIES LABELS` with `ctest -L` for running subsets (unit vs integration).
- Consider `DISCOVERY_MODE PRE_TEST` for header-only tests where the binary isn't available at build time.
- Google Mock is included in the googletest repo since 2019 - just link `GTest::gmock`.
