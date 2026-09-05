# Measure and enforce code coverage with gcov, lcov, and llvm-cov

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Code coverage answers a simple question: which parts of your source code actually ran during your test suite? It's a **necessary but not sufficient** quality metric - 100% coverage doesn't mean your code is bug-free, but low coverage tells you with certainty that you have untested blind spots. The major tools for C++ are **gcov** (GCC's built-in coverage tool), **llvm-cov** (Clang's equivalent), and **lcov** (a front-end that turns raw coverage data into readable HTML reports).

The important mental model here is that coverage is about building confidence, not about hitting a magic number. A function that gets called but whose edge cases are never exercised can show as "covered" even though important paths were skipped. That's why branch coverage is more valuable than line coverage - it forces you to think about every `if`/`else` arm, not just whether a line of code was touched.

### Coverage Metrics

The table below summarizes the main metrics you'll encounter. If it feels like a lot, the short version is: start with line coverage to find dead code, then move to branch coverage for real confidence.

| Metric | What It Measures | Usefulness |
| --- | --- | --- |
| **Line coverage** | Lines executed / total lines | Basic - misses branch logic |
| **Branch coverage** | Branches taken / total branches | Better - catches untested if/else |
| **Function coverage** | Functions called / total functions | Finds dead code |
| **Region coverage** (llvm-cov) | Code regions executed | Most precise |
| **MC/DC** | Modified condition/decision | Required for safety-critical (DO-178C) |

### Tool Chain

Here's how the pieces fit together. Your source and test code get compiled with special instrumentation flags that tell the compiler to count executions. When you run the tests, the runtime writes out that count data, and then the coverage tools read both the compile-time notes and the runtime data to produce a report.

```cpp
Source + Tests -> Compile with coverage flags -> Run tests
    │                                              │
    │           .gcno (compile-time notes)         │ .gcda (runtime data)
    │                                              │
    └────────► gcov / llvm-cov ─────────────────┘
                    │
              lcov / genhtml -> HTML report
```

---

## Self-Assessment

### Q1: Set up coverage measurement with GCC/gcov and lcov

**Answer:**

The first thing you need is a CMake option that adds the right compiler flags when coverage is requested. You don't want these flags in your normal build - coverage instrumentation slows execution by 2-5x and bloats the binary.

```cmake
# === CMakeLists.txt: coverage build type ===
option(ENABLE_COVERAGE "Build with coverage instrumentation" OFF)

if(ENABLE_COVERAGE)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        add_compile_options(--coverage -fprofile-arcs -ftest-coverage)
        add_link_options(--coverage)
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        add_compile_options(-fprofile-instr-generate -fcoverage-mapping)
        add_link_options(-fprofile-instr-generate)
    endif()
endif()
```

Note that `--coverage` is just shorthand for `-fprofile-arcs -ftest-coverage` on GCC. Both spellings work; the long form makes what's happening more explicit.

Once you've built with coverage enabled and run your tests, here's the full lcov workflow that takes you from raw `.gcda` files all the way to an HTML report:

```bash
# === GCC + gcov + lcov workflow ===

# 1. Build with coverage
cmake -B build -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# 2. Run tests (generates .gcda files)
cd build && ctest --output-on-failure

# 3. Capture coverage data
lcov --capture --directory . --output-file coverage.info

# 4. Remove unwanted paths (system headers, test code)
lcov --remove coverage.info \
    '/usr/*' \
    '*/test/*' \
    '*/googletest/*' \
    --output-file coverage_filtered.info

# 5. Generate HTML report
genhtml coverage_filtered.info \
    --output-directory coverage_report \
    --title "MyProject Coverage" \
    --legend --highlight --demangle-cpp

# 6. Open report
# open coverage_report/index.html

# 7. Check coverage threshold (fails if below 80%)
lcov --summary coverage_filtered.info | grep -E 'lines|functions'
# Output: lines......: 85.3% (256 of 300 lines)
```

The filtering step (step 4) is important - without it your metrics will be polluted by system headers and the test framework itself, which would make your numbers look either better or worse than they really are.

If you want to look at a single file in detail rather than the full report, `gcov` can annotate the source directly:

```bash
# === Per-file coverage with gcov ===
gcov -b -c src/calculator.cpp
# Produces calculator.cpp.gcov with line-by-line annotations:
#     1:   12: double Calculator::add(double a, double b) {
#     1:   13:     return a + b;
#     -:   14: }
#####:   15: double Calculator::unused_method() {
# ^^^^^ means line was never executed
```

The `#####` marker is your signal that a line was never hit. Any function showing that is a candidate for either a new test or deletion if the code is truly dead.

### Q2: Set up coverage with Clang/llvm-cov

**Answer:**

Clang uses a different instrumentation format that's generally more precise than GCC's. The extra step of merging `.profraw` files into a `.profdata` file might seem annoying, but it allows you to merge data from multiple test runs cleanly - useful when you have different test executables covering different parts of the code.

```bash
# === Clang + llvm-cov workflow ===

# 1. Build with coverage
clang++ -fprofile-instr-generate -fcoverage-mapping \
    -o test_runner src/*.cpp tests/*.cpp -lgtest -lgtest_main

# 2. Run tests (generates default.profraw)
LLVM_PROFILE_FILE="test.profraw" ./test_runner

# 3. Merge profile data
llvm-profdata merge -sparse test.profraw -o test.profdata

# 4. Generate report
# Text summary:
llvm-cov report ./test_runner \
    -instr-profile=test.profdata \
    -ignore-filename-regex='test|googletest'

# Detailed source view:
llvm-cov show ./test_runner \
    -instr-profile=test.profdata \
    -format=html \
    -output-dir=coverage_report \
    -ignore-filename-regex='test|googletest' \
    -show-line-counts-or-regions \
    -show-branches=count

# 5. Export for CI tools:
llvm-cov export ./test_runner \
    -instr-profile=test.profdata \
    -format=lcov > coverage.lcov
```

The export to lcov format at the end is worth doing even if you primarily use llvm-cov for local analysis - it makes the output compatible with Codecov, Coveralls, and similar CI reporting tools.

When you want to drill into a specific file, the report command accepts a filename filter:

```bash
# === Check specific file coverage ===
llvm-cov report ./test_runner \
    -instr-profile=test.profdata \
    src/calculator.cpp

# Output:
# Filename         Regions  Miss  Cover  Lines  Miss  Cover  Branches  Miss  Cover
# calculator.cpp        12     1  91.7%     45     3  93.3%        8     2  75.0%
```

That three-column breakdown (Regions / Lines / Branches) gives you a much richer picture than line coverage alone. The branch column in particular tends to reveal the gaps that pure line coverage hides.

### Q3: Enforce coverage thresholds in CI and integrate with CMake

**Answer:**

The most common pattern is a CMake custom target that automates the whole workflow - reset counters, run tests, capture, filter, and generate the report in one `cmake --build build --target coverage` command.

```cmake
# === CMake custom target for coverage ===
if(ENABLE_COVERAGE)
    find_program(LCOV lcov)
    find_program(GENHTML genhtml)

    add_custom_target(coverage
        # Reset counters
        COMMAND ${LCOV} --zerocounters --directory ${CMAKE_BINARY_DIR}
        # Run tests
        COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
        # Capture
        COMMAND ${LCOV} --capture
            --directory ${CMAKE_BINARY_DIR}
            --output-file ${CMAKE_BINARY_DIR}/coverage.info
        # Filter
        COMMAND ${LCOV} --remove ${CMAKE_BINARY_DIR}/coverage.info
            '/usr/*' '*/test/*' '*/build/*'
            --output-file ${CMAKE_BINARY_DIR}/coverage_filtered.info
        # Report
        COMMAND ${GENHTML} ${CMAKE_BINARY_DIR}/coverage_filtered.info
            --output-directory ${CMAKE_BINARY_DIR}/coverage_report
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        COMMENT "Generating code coverage report..."
    )
endif()
```

In CI you want to not only generate the report but also fail the build if coverage drops below your threshold. The GitHub Actions workflow below handles that, and also uploads to Codecov so you get trend tracking across PRs:

```yaml
# === GitHub Actions: enforce coverage threshold ===
name: Coverage
on: [push, pull_request]
jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Build with coverage

        run: |
          cmake -B build -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
          cmake --build build -j$(nproc)

      - name: Run tests

        run: ctest --test-dir build --output-on-failure -j$(nproc)

      - name: Generate coverage

        run: |
          lcov --capture --directory build --output-file coverage.info
          lcov --remove coverage.info '/usr/*' '*/test/*' '*/googletest/*' \
               --output-file coverage_filtered.info

      - name: Enforce minimum coverage

        run: |
          COVERAGE=$(lcov --summary coverage_filtered.info 2>&1 | \
                     grep 'lines' | grep -oP '[0-9.]+(?=%)')
          echo "Line coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below threshold 80%"
            exit 1
          fi

      - name: Upload to Codecov

        uses: codecov/codecov-action@v4
        with:
          files: coverage_filtered.info
          fail_ci_if_error: true
```

The threshold check in the "Enforce minimum coverage" step is the key part - it extracts the line coverage percentage from `lcov --summary` and exits with an error if you're below 80%. That makes coverage enforcement automatic rather than something a reviewer has to notice manually.

---

## Notes

- Coverage is a necessary but not sufficient metric - it shows what code runs, not whether it's correct.
- Aim for 80%+ line coverage as a baseline; 90%+ for critical code.
- Branch coverage is more valuable than line coverage - it catches missed if/else paths.
- Always filter out test code, third-party code, and generated code from metrics.
- `--coverage` is shorthand for `-fprofile-arcs -ftest-coverage` (GCC).
- Use `LLVM_PROFILE_FILE` environment variable to control output location.
- Coverage slows execution 2-5x - use dedicated coverage builds, not release builds.
- `gcovr` is a Python alternative to lcov that produces Cobertura XML (for Jenkins/GitLab).
- Mutation testing (see separate topic) is a stronger quality metric than coverage.
