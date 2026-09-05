# Use mutation testing to measure test suite effectiveness

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/mull-project/mull>  

---

## Topic Overview

**Mutation testing** injects small bugs (mutations) into your code and checks whether your test suite catches them. If a test fails, the mutant is **killed** (good). If all tests pass, the mutant **survived** (your tests have a gap).

The intuition behind mutation testing is that if a small, deliberate change to your code does not cause any test to fail, then either your tests do not cover that code or - worse - they cover it but never check the result. Mutation testing finds both problems.

### How Mutation Testing Works

```cpp
Original code: if (a < b) return true;
                    │
        ┌───────────┼───────────────┐
        ▼           ▼               ▼
  Mutant 1:     Mutant 2:       Mutant 3:
  if (a <= b)   if (a > b)      if (a < b) return false;
        │           │               │
   Run tests   Run tests       Run tests
        │           │               │
   Tests fail   Tests fail     Tests pass!
   (killed)     (killed)       (survived)
                                -> need better test
```

### Mutation Score

$$\text{Mutation Score} = \frac{\text{Killed Mutants}}{\text{Total Mutants}} \times 100\%$$

| Score | Quality |
| :---: | --- |
| >90%  | Excellent test suite |
| 70-90% | Good, but gaps exist |
| <70%  | Significant blind spots |

### Common Mutation Operators

Each row in the table below represents one type of mutation the tool can apply. The most revealing ones in practice are the relational operators - changing `<` to `<=` - because they expose missing boundary tests, which are the most common source of off-by-one bugs.

| Operator | Original | Mutant |
| --- | --- | --- |
| Relational | `<` | `<=`, `>`, `>=`, `==`, `!=` |
| Arithmetic | `+` | `-`, `*`, `/` |
| Logical | `&&` | `\|\|` |
| Constant | `0` | `1` |
| Return | `return x` | `return 0` |
| Negate | `if (cond)` | `if (!cond)` |

---

## Self-Assessment

### Q1: Run a mutation testing tool (Mull or mutmut) on a simple function and see the mutation score

The function below classifies a BMI value into one of four categories. The tests cover typical values in each range, which gives 100% line coverage. But watch what happens when Mull looks for boundary tests.

```cpp
// Source: classify.cpp
#include <string>

std::string classify_bmi(double bmi) {
    if (bmi < 18.5) return "underweight";
    if (bmi < 25.0) return "normal";
    if (bmi < 30.0) return "overweight";
    return "obese";
}
```

```cpp
// Tests: test_classify.cpp
#include <gtest/gtest.h>

extern std::string classify_bmi(double);

TEST(BMI, Underweight) { EXPECT_EQ(classify_bmi(15.0), "underweight"); }
TEST(BMI, Normal)      { EXPECT_EQ(classify_bmi(22.0), "normal"); }
TEST(BMI, Overweight)  { EXPECT_EQ(classify_bmi(27.0), "overweight"); }
TEST(BMI, Obese)       { EXPECT_EQ(classify_bmi(35.0), "obese"); }
```

```bash
# Run Mull
# Step 1: Compile with Mull's LLVM pass
clang++ -fpass-plugin=/usr/lib/mull-ir-frontend.so \
        -g -O0 classify.cpp test_classify.cpp \
        -lgtest -lgtest_main -o test_bmi

# Step 2: Run Mull
mull-runner ./test_bmi

# Output:
# Mutation score: 75% (12/16 mutants killed)
#
# Survived mutants:
#   classify.cpp:4:  Replaced < with <= (bmi < 18.5 -> bmi <= 18.5)
#   classify.cpp:5:  Replaced < with <= (bmi < 25.0 -> bmi <= 25.0)
#   classify.cpp:6:  Replaced < with <= (bmi < 30.0 -> bmi <= 30.0)
#   classify.cpp:4:  Replaced 18.5 with 0.0
```

The report tells you exactly where your test suite is weak: none of your tests ever check what happens at exactly 18.5, 25.0, or 30.0. That is a gap in the specification you thought you were testing.

### Q2: Identify surviving mutants (bugs not caught by tests) and write tests to kill them

Each surviving mutant from Q1 points to a missing boundary test. The fix is mechanical: test the exact threshold value from both sides.

```cpp
// The surviving mutants show our tests lack BOUNDARY tests

// Mutant: bmi < 18.5 -> bmi <= 18.5
// This survives because we never test bmi = 18.5 exactly
// With the mutant, classify_bmi(18.5) returns "underweight" instead of "normal"

// Kill the mutants with boundary tests
TEST(BMI, BoundaryUnderweightNormal) {
    EXPECT_EQ(classify_bmi(18.5), "normal");      // Kills: < -> <=
    EXPECT_EQ(classify_bmi(18.49), "underweight"); // Kills: < -> >=
}

TEST(BMI, BoundaryNormalOverweight) {
    EXPECT_EQ(classify_bmi(25.0), "overweight");   // Kills: < -> <=
    EXPECT_EQ(classify_bmi(24.99), "normal");
}

TEST(BMI, BoundaryOverweightObese) {
    EXPECT_EQ(classify_bmi(30.0), "obese");        // Kills: < -> <=
    EXPECT_EQ(classify_bmi(29.99), "overweight");
}

TEST(BMI, ZeroBMI) {
    EXPECT_EQ(classify_bmi(0.0), "underweight");   // Kills: 18.5 -> 0.0
}

// After adding these:
// Mutation score: 100% (16/16 mutants killed)

// Key insight: mutation testing found that our tests NEVER tested
// boundary values - the most common source of off-by-one bugs!
```

This is the feedback loop mutation testing is designed to create: run the tool, read the survived list, add a test for each survivor, re-run. When the survived list is empty (or only contains equivalent mutants), you have a genuinely thorough test suite.

### Q3: Explain the difference between code coverage and mutation score as quality metrics

This is a fundamental conceptual distinction that every developer who takes testing seriously eventually has to confront. High code coverage is necessary but not sufficient. You can achieve 100% line coverage with tests that never assert anything.

```cpp
// Code coverage lies, mutation score doesn't

#include <cassert>

int abs_value(int x) {
    if (x < 0)
        return -x;     // Line A
    return x;           // Line B
}

// Test with 100% LINE COVERAGE but TERRIBLE quality
void test_abs_coverage() {
    abs_value(-5);   // Exercises line A -> coverage counts it
    abs_value(3);    // Exercises line B -> coverage counts it
    // 100% line coverage! But we never CHECK the results!
}

// Mutation testing EXPOSES the weakness
// Mutant: return -x -> return x
//   test_abs_coverage() still passes (no assertions!) -> SURVIVED
// Mutant: return x -> return -x
//   test_abs_coverage() still passes -> SURVIVED
// Mutation score: 0% despite 100% line coverage!

// Fix: add actual assertions
void test_abs_proper() {
    assert(abs_value(-5) == 5);  // Kills: return -x -> return x
    assert(abs_value(3)  == 3);  // Kills: return x -> return -x
    assert(abs_value(0)  == 0);  // Kills: x < 0 -> x <= 0
}
// Now mutation score: 100%
```

| Metric | What It Measures | False Positives | Finds Missing Assertions? |
| --- | --- | :---: | :---: |
| **Line coverage** | Lines executed | Many - executed does not mean tested | No |
| **Branch coverage** | Decision branches taken | Some - direction does not mean tested | No |
| **Mutation score** | Tests that detect code changes | Few - directly measures quality | **Yes** |

**Summary:** Coverage tells you what code *ran*. Mutation score tells you what code is *actually tested*. A test suite with 100% coverage but no assertions has 0% mutation score.

---

## Notes

- **Mull** operates at LLVM IR level - no source modification, works with any Clang-compiled code
- Mutation testing is slow: for 1000 mutations × 100 tests = 100,000 test runs total
- Start with critical code paths only - don't mutate the entire codebase
- Equivalent mutants (semantically identical) can't be killed - they inflate "survived" count
- Combine: aim for >90% line coverage FIRST, then use mutation testing to find quality gaps
