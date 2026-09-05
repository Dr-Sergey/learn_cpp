# Use source-based code coverage (LLVM) to find untested code paths

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://clang.llvm.org/docs/SourceBasedCodeCoverage.html>  

---

## Topic Overview

Code coverage is the answer to "did my tests actually exercise this code?" Without it, you can have a large test suite and still be leaving critical paths completely untested. LLVM's source-based coverage instruments your binary so that when you run your tests, every executed line, branch, and function is recorded. You then generate a report that highlights exactly which parts of your code your tests never touched.

The reason LLVM's approach is called "source-based" is that it instruments at the level of abstract syntax tree regions rather than at the assembly instruction level. This means the coverage data maps precisely to your source code, which gives you accurate branch-level information - not just "was this line reached" but "was the `else` arm of this `if` ever taken?"

```cpp
Coverage workflow:

  1. Compile with coverage flags
  2. Run tests (generates .profraw)
  3. Merge profiles with llvm-profdata
  4. Generate report with llvm-cov

Coverage types:
  Line coverage:     Was this line executed?     (basic)
  Branch coverage:   Was each if/else taken?     (reveals missed paths)
  Function coverage:  Was this function called?  (finds dead code)
```

---

## Self-Assessment

### Q1: Generate an HTML coverage report

This calculator has division and error-handling paths that the initial test suite deliberately skips. The coverage report will make those gaps unmistakable.

```cpp
// calculator.cpp
#include <stdexcept>
#include <string>

double calculate(double a, double b, const std::string& op) {
    if (op == "+") return a + b;
    if (op == "-") return a - b;
    if (op == "*") return a * b;
    if (op == "/") {
        if (b == 0.0)
            throw std::domain_error("division by zero");
        return a / b;
    }
    throw std::invalid_argument("unknown operator: " + op);
}

// test_calculator.cpp
#include <cassert>
#include <iostream>

double calculate(double, double, const std::string&);

int main() {
    assert(calculate(2, 3, "+") == 5.0);
    assert(calculate(5, 3, "-") == 2.0);
    assert(calculate(4, 3, "*") == 12.0);
    // NOTE: division not tested!
    // NOTE: error paths not tested!
    std::cout << "Tests passed\n";
}
```

The five-step process below compiles with special flags, runs the tests to produce a raw profile, merges it, and generates an HTML report you can open in a browser and explore interactively.

```bash
# Step 1: Compile with coverage
$ clang++ -std=c++20 -fprofile-instr-generate -fcoverage-mapping \
    calculator.cpp test_calculator.cpp -o test_calc

# Step 2: Run tests
$ ./test_calc
# Creates: default.profraw

# Step 3: Merge profile data
$ llvm-profdata merge default.profraw -o test_calc.profdata

# Step 4: Generate HTML report
$ llvm-cov show ./test_calc \
    -instr-profile=test_calc.profdata \
    --format=html \
    -output-dir=coverage_report \
    calculator.cpp

# Step 5: View report
$ open coverage_report/index.html

# Quick summary:
$ llvm-cov report ./test_calc -instr-profile=test_calc.profdata
# Filename         Regions    Miss   Cover   Lines   Miss   Cover
# calculator.cpp        8       3   62.5%      12      4   66.7%
# test_calculator       2       0  100.0%       8      0  100.0%
```

The report shows 62.5% region coverage for `calculator.cpp`. The HTML view will highlight the uncovered lines in red, making it immediately obvious which operators and error paths are untested.

### Q2: Find and cover untested exception handling

The annotated output from `llvm-cov` uses execution counts on the left margin. A count of zero means the line was never reached during the test run.

```cpp
calculator.cpp:
   1|      |double calculate(double a, double b, const std::string& op) {
   2|     3|    if (op == "+") return a + b;           // COVERED
   3|     2|    if (op == "-") return a - b;           // COVERED
   4|     1|    if (op == "*") return a * b;           // COVERED
   5|     0|    if (op == "/") {                       // NOT COVERED!
   6|     0|        if (b == 0.0)                      // NOT COVERED!
   7|     0|            throw std::domain_error(...);   // NOT COVERED!
   8|     0|        return a / b;                      // NOT COVERED!
   9|      |    }
  10|     0|    throw std::invalid_argument(...);       // NOT COVERED!
```

Every line from the `/` branch onward shows zero. The fix is to add tests for division, division by zero, and an unknown operator. Once you add those, the coverage jumps to 100%.

```cpp
// Add missing tests:
int main() {
    // Existing tests...
    assert(calculate(2, 3, "+") == 5.0);
    assert(calculate(5, 3, "-") == 2.0);
    assert(calculate(4, 3, "*") == 12.0);

    // NEW: Test division
    assert(calculate(10, 2, "/") == 5.0);

    // NEW: Test division by zero
    try {
        calculate(1, 0, "/");
        assert(false && "should have thrown");
    } catch (const std::domain_error&) {
        // expected
    }

    // NEW: Test unknown operator
    try {
        calculate(1, 2, "%");
        assert(false && "should have thrown");
    } catch (const std::invalid_argument&) {
        // expected
    }

    std::cout << "All tests passed\n";
}
// Now coverage = 100% lines, 100% branches, 100% functions
```

### Q3: Line vs branch vs function coverage

These three coverage metrics tell you different things, and it's easy to be fooled by a high number in one while being dangerously low in another. The table below clarifies what each one actually measures.

| Metric | Measures | Example |
| --- | --- | --- |
| **Line** | Was this source line executed at least once? | `if (x)` line hit = covered |
| **Branch** | Was each branch direction taken? | `if(x)` needs both true AND false |
| **Function** | Was this function called at least once? | `foo()` called = covered |

The reason this trips people up is that 100% line coverage does not imply 100% branch coverage. A function can be "fully covered" in lines but only ever take the `true` path of every conditional. Branch coverage is the stricter metric and reveals far more bugs.

```cpp
void example(int x) {
    if (x > 0) {          // Line: covered if reached
        do_positive(x);   // Branch: only "true" branch covered
    } else {
        do_negative(x);   // Branch: only "false" branch covered
    }                     // 100% line != 100% branch!
}
// Test with x=5 only:
//   Line coverage: 4/5 = 80% (else branch uncovered)
//   Branch coverage: 1/2 = 50% (only true branch taken)
//   Function coverage: 1/1 = 100%
```

If you only looked at function coverage (100%) or even line coverage (80%), you might think you're in good shape. Branch coverage shows you the reality: you've never tested the negative case.

```bash
# Show branch coverage specifically:
$ llvm-cov show ./test_calc \
    -instr-profile=test_calc.profdata \
    -show-branches=count

# Export as JSON/LCOV for CI tools:
$ llvm-cov export ./test_calc \
    -instr-profile=test_calc.profdata \
    -format=lcov > coverage.lcov

# Upload to Codecov/Coveralls:
$ bash <(curl -s https://codecov.io/bash) -f coverage.lcov

# Set coverage thresholds in CI:
$ COVERAGE=$(llvm-cov report ... | grep TOTAL | awk '{print $NF}' | tr -d '%')
$ if (( $(echo "$COVERAGE < 80" | bc -l) )); then
    echo "Coverage below 80%!"; exit 1
  fi
```

---

## Notes

- Source-based coverage is more accurate than gcov (line-level vs instruction-level).
- GCC equivalent: `-coverage` flag + `gcov` + `lcov` for HTML.
- Aim for 80%+ line coverage and 70%+ branch coverage.
- 100% coverage doesn't mean bug-free - it means all paths are exercised.
- Exclude test files and third-party code: `llvm-cov show --ignore-filename-regex='test_.*'`.
