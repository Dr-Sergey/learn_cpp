# Use CTest and CMake testing infrastructure effectively

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**CTest** is CMake's built-in test driver, and it's more capable than most people realize. It discovers tests, runs them (in parallel if you want), filters them by name or label, handles timeouts, and produces output in JUnit XML format for CI systems. The key insight is that CTest doesn't run your tests directly - it runs test executables that you've registered, and those executables report pass/fail. `gtest_discover_tests()` makes the registration automatic for Google Test binaries.

The reason you want to invest in this infrastructure is that it standardizes how tests are run. Every developer runs `ctest --test-dir build` and gets the same results. CI uses the same command. You add labels to separate fast unit tests from slow integration tests, and CI can run just the unit tests on every commit and the integration tests on merge.

### CTest Key Features

| Feature | Command / CMake | Purpose |
| --- | --- | --- |
| Run all tests | `ctest --test-dir build` | Execute registered tests |
| Parallel | `ctest -j8` | Run 8 tests simultaneously |
| Filter by name | `ctest -R "Unit.*"` | Regex match on test name |
| Filter by label | `ctest -L integration` | Run only labeled tests |
| Exclude | `ctest -E "Slow.*"` | Skip matching tests |
| Verbose | `ctest -V` or `--output-on-failure` | Show output on failure |
| Timeout | `set_tests_properties(t PROPERTIES TIMEOUT 30)` | Per-test timeout |
| Repeat | `ctest --repeat until-fail:10` | Run 10 times, stop on failure |
| Output | `ctest --output-junit results.xml` | JUnit XML for CI |

---

## Self-Assessment

### Q1: Set up CTest with multiple test categories and properties

Start with the top-level `CMakeLists.txt`. The `enable_testing()` call must go here - not in a subdirectory - or CTest won't work at all. This is the single most common setup mistake.

```cmake
# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.14)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
enable_testing()  # Must be in top-level CMakeLists.txt

add_subdirectory(src)
add_subdirectory(tests)
```

In the test subdirectory, you use `gtest_discover_tests()` instead of `add_test()` when you're using Google Test. The difference matters: `gtest_discover_tests()` runs the test binary at build time to enumerate individual test names, so CTest knows about each test case individually. This means you can filter by test name, get per-test results, and run a single failing test in isolation.

```cmake
# === tests/CMakeLists.txt ===
include(FetchContent)
FetchContent_Declare(googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.14.0)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
include(GoogleTest)

# --- Unit tests ---
add_executable(unit_tests
    math_test.cpp
    string_test.cpp
)
target_link_libraries(unit_tests PRIVATE my_lib GTest::gtest_main)

# Auto-discover with labels
gtest_discover_tests(unit_tests
    PROPERTIES
        LABELS "unit"
        TIMEOUT 10
)

# --- Integration tests ---
add_executable(integration_tests
    db_integration_test.cpp
)
target_link_libraries(integration_tests PRIVATE my_lib GTest::gtest_main)

gtest_discover_tests(integration_tests
    PROPERTIES
        LABELS "integration"
        TIMEOUT 60
)

# --- Manual test registration (for non-gtest executables) ---
add_executable(benchmark_test benchmark_test.cpp)
target_link_libraries(benchmark_test PRIVATE my_lib)

add_test(
    NAME benchmark_test
    COMMAND benchmark_test --iterations=1000
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)
set_tests_properties(benchmark_test PROPERTIES
    LABELS "benchmark;slow"
    TIMEOUT 120
    ENVIRONMENT "MY_ENV_VAR=test_value"
)

# --- Test that is expected to fail ---
add_test(NAME expected_failure COMMAND unit_tests --gtest_filter="*Broken*")
set_tests_properties(expected_failure PROPERTIES
    WILL_FAIL TRUE  # Test passes if the command returns non-zero
)
```

Once the build is configured, here's how you drive the different test categories from the command line:

```bash
# Run all unit tests
ctest --test-dir build -L unit --output-on-failure -j$(nproc)

# Run integration tests only
ctest --test-dir build -L integration -V

# Exclude slow tests
ctest --test-dir build -E "benchmark" -j8

# Run specific tests by regex
ctest --test-dir build -R "Math.*Add" --output-on-failure
```

### Q2: Use CTest fixtures for resource setup/teardown between test groups

CTest fixtures are different from Google Test fixtures. A CTest fixture is a setup or cleanup test that CTest runs automatically before or after a group of tests that require it. This is how you manage external resources - databases, servers, file system state - that need to be started before your tests run and torn down cleanly afterward.

```cmake
# CTest FIXTURES: manage test dependencies and resource lifecycle
# Not to be confused with Google Test fixtures

# Setup test: starts a database
add_test(NAME start_database COMMAND ${CMAKE_SOURCE_DIR}/scripts/start_test_db.sh)
set_tests_properties(start_database PROPERTIES
    FIXTURES_SETUP Database   # This test sets up the "Database" fixture
    TIMEOUT 30
)

# Cleanup test: stops the database
add_test(NAME stop_database COMMAND ${CMAKE_SOURCE_DIR}/scripts/stop_test_db.sh)
set_tests_properties(stop_database PROPERTIES
    FIXTURES_CLEANUP Database  # This test cleans up "Database"
    TIMEOUT 10
)

# Tests that need the database
gtest_discover_tests(integration_tests
    PROPERTIES
        FIXTURES_REQUIRED Database  # Auto-runs start/stop around these
        LABELS "integration"
)

# Execution order (automatic):
# 1. start_database     (FIXTURES_SETUP)
# 2. integration_tests  (FIXTURES_REQUIRED)
# 3. stop_database      (FIXTURES_CLEANUP)
```

You can compose multiple fixtures for tests that need several external resources. CTest takes care of the ordering automatically - you just declare the dependencies.

```cmake
# === Multiple fixtures can be composed ===

# Redis setup
add_test(NAME start_redis COMMAND redis-server --port 6380 --daemonize yes)
set_tests_properties(start_redis PROPERTIES FIXTURES_SETUP Redis)

add_test(NAME stop_redis COMMAND redis-cli -p 6380 shutdown)
set_tests_properties(stop_redis PROPERTIES FIXTURES_CLEANUP Redis)

# Tests requiring both Database AND Redis
set_tests_properties(cache_integration_test PROPERTIES
    FIXTURES_REQUIRED "Database;Redis"  # Both fixtures required
)
```

### Q3: Show CI-optimized CTest configuration with JUnit output and failure handling

For CI, you want CTest to produce JUnit XML output (most CI platforms can parse this and display per-test results), to run tests in parallel, and to handle flaky tests gracefully with a rerun mechanism. CMake Presets standardize all of this so every developer and every CI job uses the same configuration.

```cmake
# === CTestConfig.cmake (optional: CDash integration) ===
set(CTEST_PROJECT_NAME "MyProject")
set(CTEST_NIGHTLY_START_TIME "01:00:00 UTC")

# === CMake presets for testing (CMakePresets.json) ===
```

```json
{
  "version": 6,
  "testPresets": [
    {
      "name": "unit",
      "configurePreset": "debug",
      "output": {
        "verbosity": "default",
        "outputOnFailure": true,
        "outputJUnitFile": "test-results/unit.xml"
      },
      "filter": {
        "include": { "label": "unit" }
      },
      "execution": {
        "jobs": 4,
        "timeout": 300,
        "noTestsAction": "error"
      }
    },
    {
      "name": "integration",
      "configurePreset": "debug",
      "output": {
        "outputOnFailure": true,
        "outputJUnitFile": "test-results/integration.xml"
      },
      "filter": {
        "include": { "label": "integration" }
      },
      "execution": {
        "jobs": 2,
        "timeout": 600
      }
    }
  ]
}
```

The GitHub Actions workflow below puts it all together. Notice the `--rerun-failed --repeat until-pass:3` step that only runs on failure - this is how you handle tests that are occasionally flaky without letting them block every merge. You still want to investigate flaky tests, but this prevents them from completely stopping your CI pipeline.

```yaml
# === GitHub Actions CI ===
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Configure

        run: cmake -B build -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=ON

      - name: Build

        run: cmake --build build -j$(nproc)

      - name: Unit Tests

        run: ctest --test-dir build -L unit
                   --output-on-failure
                   --output-junit unit-results.xml
                   -j$(nproc)
                   --timeout 60

      - name: Integration Tests

        run: ctest --test-dir build -L integration
                   --output-on-failure
                   --output-junit integration-results.xml
                   --timeout 120

      - name: Upload Test Results

        if: always()  # Upload even on failure
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/*-results.xml

      # Flaky test rerun

      - name: Rerun Failed Tests

        if: failure()
        run: ctest --test-dir build --rerun-failed
                   --output-on-failure
                   --repeat until-pass:3
```

A few useful commands for day-to-day work with CTest that don't appear in the workflow above:

```bash
# === Useful CTest commands ===

# List all tests without running
ctest --test-dir build -N

# Show test properties
ctest --test-dir build --show-only=json-v1 | python -m json.tool

# Repeat until failure (find flaky tests)
ctest --test-dir build --repeat until-fail:100 -j4

# Run with memory checking (Valgrind)
ctest --test-dir build -T memcheck

# Cost-based ordering (run slow tests first for better parallelism)
ctest --test-dir build --test-load 4  # Limit to 4 parallel test processes
```

---

## Notes

- `enable_testing()` must be in the top-level `CMakeLists.txt` for CTest to work. Putting it in a subdirectory is a common mistake that silently causes no tests to be registered.
- `gtest_discover_tests()` runs at build time to enumerate test names; `gtest_add_tests()` runs at configure time. The discovery approach is more reliable for large test suites.
- Use labels (`LABELS` property) to separate unit, integration, and benchmark tests. This lets CI run fast tests on every commit and slow tests only on merge.
- CTest fixtures (`FIXTURES_SETUP`/`FIXTURES_REQUIRED`/`FIXTURES_CLEANUP`) manage resource lifecycle across test groups - use them for databases, servers, and any other external state.
- `--output-on-failure` should be your default in CI - you don't want every test's output flooding your logs, but you do want to see output when something fails.
- `noTestsAction: error` in your test preset fails the build if no tests are found. This catches the case where a build misconfiguration silently registers zero tests and everything appears to pass.
- CMake presets (`testPresets`) standardize test invocation across developers and CI so everyone runs tests the same way.
- CTest writes a cost file (`CTestCostData.txt`) that records historical test durations. CTest uses this for optimal scheduling when you run in parallel, so slow tests start first.
