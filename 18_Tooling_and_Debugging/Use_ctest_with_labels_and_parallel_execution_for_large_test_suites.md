# Use ctest with labels and parallel execution for large test suites

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://cmake.org/cmake/help/latest/manual/ctest.1.html>  

---

## Topic Overview

CTest is CMake's built-in test runner. It is not a testing framework - it does not know about assertions or test cases - but it is the tool that discovers, schedules, and runs the executables that your testing framework produces. For large projects, two CTest features are especially valuable: **labels** let you run a targeted subset of tests, and **parallel execution** lets you run many tests simultaneously to keep CI times reasonable.

Here is a quick look at the most common invocations:

```bash
ctest --test-dir build -j8                 # run 8 tests in parallel
ctest --test-dir build -L fast             # run only "fast" labeled tests
ctest --test-dir build -L slow --timeout 60 # "slow" tests with timeout
ctest --test-dir build --output-on-failure  # show stdout/stderr on failure
```

---

## Self-Assessment

### Q1: Label tests and run subsets

Labels give you a way to categorize tests in `CMakeLists.txt` and then run only the category you care about at a given moment. The classic split is between fast unit tests you run on every save and slow integration tests you run on a nightly schedule.

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(TestDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
enable_testing()

# Fast unit tests
add_executable(test_math tests/test_math.cpp)
add_test(NAME MathBasic COMMAND test_math)
set_tests_properties(MathBasic PROPERTIES
    LABELS "fast;unit"
    TIMEOUT 5
)

add_executable(test_string tests/test_string.cpp)
add_test(NAME StringOps COMMAND test_string)
set_tests_properties(StringOps PROPERTIES
    LABELS "fast;unit"
    TIMEOUT 5
)

# Slow integration tests
add_executable(test_db tests/test_db.cpp)
add_test(NAME DatabaseIntegration COMMAND test_db)
set_tests_properties(DatabaseIntegration PROPERTIES
    LABELS "slow;integration"
    TIMEOUT 60
)

add_executable(test_network tests/test_network.cpp)
add_test(NAME NetworkE2E COMMAND test_network)
set_tests_properties(NetworkE2E PROPERTIES
    LABELS "slow;e2e"
    TIMEOUT 120
)
```

With those labels in place, you can choose exactly what to run depending on the context:

```bash
# Run only fast tests (for local development):
ctest --test-dir build -L fast
# Output:
#   Test #1: MathBasic ......... Passed  0.01 sec
#   Test #2: StringOps ......... Passed  0.02 sec
#   100% tests passed, 0 tests failed

# Run only slow tests (in CI nightly):
ctest --test-dir build -L slow

# Exclude slow tests:
ctest --test-dir build -LE slow

# Run tests matching a regex:
ctest --test-dir build -R "Math|String"

# List available tests and labels:
ctest --test-dir build -N
```

The `-N` flag is handy when you are first setting things up and want to confirm that your labels and test names were registered correctly without actually running anything.

### Q2: Parallel execution and test dependencies

Running tests in parallel is straightforward - pass `-j` with a number. The harder part is making sure your tests are safe to run in parallel, and CTest gives you the tools to handle the cases where they are not.

```cmake
# Parallel-safe tests (no shared state):
set_tests_properties(MathBasic StringOps PROPERTIES
    LABELS "fast;unit"
    PROCESSORS 1        # each test uses 1 CPU core
)

# Test that needs exclusive access to a resource:
set_tests_properties(DatabaseIntegration PROPERTIES
    LABELS "slow;integration"
    RESOURCE_LOCK "database"   # prevent parallel DB tests
)

# Test that depends on another test completing first:
set_tests_properties(NetworkE2E PROPERTIES
    DEPENDS DatabaseIntegration  # runs AFTER DatabaseIntegration
    LABELS "slow;e2e"
)

# Multi-core test:
set_tests_properties(HeavyComputation PROPERTIES
    PROCESSORS 4          # tells CTest this test uses 4 cores
    # CTest won't schedule more than (total_cores - 4) other tests simultaneously
)
```

The `RESOURCE_LOCK` property is particularly useful for integration tests that share a database, a port, or a file. Tests with the same lock name will never run at the same time, even in a parallel run. The `PROCESSORS` property lets CTest make smart scheduling decisions for tests that are themselves multi-threaded.

```bash
# Run with 8 parallel jobs:
ctest --test-dir build -j8

# CTest parallel scheduling:
#   t=0:  MathBasic, StringOps, test3, test4, test5, test6, test7, test8
#   t=0.02: (MathBasic done) -> start test9
#   ...until all done
#
# Tests with DEPENDS wait for predecessors
# Tests with RESOURCE_LOCK serialize on that resource
# Tests with PROCESSORS > 1 reduce parallelism accordingly

# Verbose output:
ctest --test-dir build -j8 -V --output-on-failure
```

### Q3: Generate JUnit XML reports for CI

Most CI systems can display test results in a rich format if you give them a JUnit XML report. CTest can produce that report directly, which means you get pass/fail tracking, timing information, and failure details integrated into your CI dashboard without any extra tooling.

```bash
# Generate JUnit XML report:
ctest --test-dir build --output-junit test-results.xml --output-on-failure -j4

# The XML file is compatible with:
#   - GitHub Actions (upload-artifact)
#   - Jenkins (JUnit plugin)
#   - GitLab CI (artifacts:reports:junit)
#   - Azure DevOps (PublishTestResults)
```

Here is a complete GitHub Actions workflow that builds, tests, and uploads the results:

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Configure

        run: cmake -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build

        run: cmake --build build -j$(nproc)

      - name: Test

        run: |
          cd build
          ctest --output-junit test-results.xml \
                --output-on-failure \
                -j$(nproc) \
                --timeout 120

      - name: Upload Test Results

        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/test-results.xml
```

The `if: always()` on the upload step ensures that test results are uploaded even when tests fail - which is exactly when you most want to see them.

---

## Notes

- `-j0` uses all available cores automatically, which is a convenient default for CI.
- `RESOURCE_LOCK` prevents tests that share a resource (a database, a network port, a temp file) from running in parallel and interfering with each other.
- Use `FIXTURES_SETUP` and `FIXTURES_CLEANUP` when you have complex setup or teardown sequences that need to wrap around a group of tests.
- `--rerun-failed` re-runs only the tests that failed in the previous run, which dramatically speeds up the fix-and-retest cycle.
- `--repeat until-pass:3` retries flaky tests up to 3 times, useful for tests with intermittent network or timing dependencies.
- Add `--no-tests=error` to make CTest fail if no tests are discovered - this catches configuration mistakes where `enable_testing()` was forgotten or tests were not registered.
