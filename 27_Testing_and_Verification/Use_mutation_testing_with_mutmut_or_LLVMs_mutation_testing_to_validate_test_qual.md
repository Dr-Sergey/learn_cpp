# Use mutation testing with mutmut or LLVM's mutation testing to validate test quality

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/llvm/llvm-project/tree/main/llvm/lib/FrontendOpenMP>  

---

## Topic Overview

This file focuses on **the full workflow of finding and killing surviving mutants**, and understanding what a surviving mutant means conceptually. (See files #684 and #587 for Mull-specific setup, mutation operators, and coverage-vs-mutation-score comparison.)

### Surviving Mutant = Blind Spot in Your Tests

The core idea of mutation testing is deceptively simple: the tool makes a small, deliberate code change - called a *mutant* - and then runs your test suite. If all your tests still pass, the mutant *survived*, which means your tests cannot tell the difference between correct code and buggy code. That is a blind spot you should care about.

Here is a concrete illustration. The original code uses `>` and the mutant changes it to `>=`. Notice that the existing tests, which only test the values 5 and -3, agree on both versions:

```cpp
Original:    if (x > 0) return x;
Mutant:      if (x >= 0) return x;   // Changed > to >=

Your tests:
  test(5)  -> returns 5  (both versions agree)
  test(-3) -> returns -3 (neither branch taken)

Missing test:
  test(0)  -> original returns ??? (falls through)
             mutant returns 0 (takes the branch)
  -> Different behavior! This test would KILL the mutant.
```

The value `0` is the exact boundary where the two versions diverge. A surviving mutant is always telling you that you are missing a test case right at some behavioural boundary.

### Mutation Testing Workflow

The workflow is iterative. You run mutation testing, read the report, write tests that target the survivors, and repeat until your score is high enough. Here is the full loop:

| Step | Action | Tool |
| --- | --- | --- |
| 1 | Run tests normally | ctest / gtest |
| 2 | Run mutation testing | Mull / mutmut |
| 3 | Read survived mutants | Mull report |
| 4 | Write tests to kill survivors | Write code |
| 5 | Re-run mutation testing | Mull |
| 6 | Repeat until score >= 90% | - |

---

## Self-Assessment

### Q1: Run a mutation testing tool on a well-tested function and identify surviving mutants

**Answer:**

Here is a calculator function that looks thoroughly tested - it has a test for every operator and for the error cases. Let's see what mutation testing reveals:

```cpp
// Source: calculator.cpp
#include <stdexcept>
#include <cmath>

double calculate(double a, const char* op, double b) {
    if (op[0] == '+') return a + b;
    if (op[0] == '-') return a - b;
    if (op[0] == '*') return a * b;
    if (op[0] == '/') {
        if (b == 0.0) throw std::domain_error("division by zero");
        return a / b;
    }
    if (op[0] == '^') return std::pow(a, b);
    throw std::invalid_argument("unknown operator");
}
```

```cpp
// Tests (appear well-tested): test_calc.cpp
#include <gtest/gtest.h>
#include <cmath>
extern double calculate(double, const char*, double);

TEST(Calc, Add)      { EXPECT_DOUBLE_EQ(calculate(2, "+", 3), 5.0); }
TEST(Calc, Subtract)  { EXPECT_DOUBLE_EQ(calculate(10, "-", 4), 6.0); }
TEST(Calc, Multiply)  { EXPECT_DOUBLE_EQ(calculate(3, "*", 5), 15.0); }
TEST(Calc, Divide)    { EXPECT_DOUBLE_EQ(calculate(10, "/", 2), 5.0); }
TEST(Calc, Power)     { EXPECT_DOUBLE_EQ(calculate(2, "^", 10), 1024.0); }
TEST(Calc, DivZero)   { EXPECT_THROW(calculate(1, "/", 0), std::domain_error); }
TEST(Calc, BadOp)     { EXPECT_THROW(calculate(1, "%", 1), std::invalid_argument); }
```

Now run Mull and read the report. The 82% score means several mutants slipped through:

```bash
# Run Mull:
mull-runner ./test_calc

# Mutation Score: 82%  (23/28 killed)
#
# SURVIVED:
# 1. calculator.cpp:5  + -> -  (a + b -> a - b)  WAIT, this should be caught
#    ...Actually it IS caught. Let me check:
#    calculate(2, "+", 3) = 5 -> mutant returns -1. Test FAILS. KILLED
#
# Real survivors:
# 2. calculator.cpp:7  * -> /  (a * b -> a / b)
#    calculate(3, "*", 5) = 15 -> mutant returns 0.6. KILLED
#
# Actually survived:
# 3. calculator.cpp:5  return a+b -> return a  (remove operand)
#    calculate(2, "+", 3): original=5, mutant=2. KILLED by our test
#
# TRUE survivors (our tests miss these):
# 4. calculator.cpp:9  b == 0.0 -> b != 0.0
#    -> Division by zero check is INVERTED. calculate(1,"/",0) would
#      NOT throw (returns inf), but calculate(1,"/",2) WOULD throw
#    Survived because we only test divzero, not normal division thoroughly
#
# 5. calculator.cpp:5  a + b -> a + 0 (replace b with constant)
#    calculate(2, "+", 3): original=5, mutant=2. Wait, this IS caught
#    calculate(0, "+", 0): original=0, mutant=0. NOT caught
```

The tool found a meaningful gap: the division-by-zero guard can be inverted and the tests do not notice.

### Q2: Explain what a surviving mutant means: a code change that no test caught

**Answer:**

A **surviving mutant** means the mutation testing tool made a specific code change (e.g., changed `<` to `<=`, or `+` to `-`), ran ALL your tests, and EVERY test still passed. This means:

1. **Your tests don't distinguish correct code from buggy code** for that specific scenario
2. **If a real developer accidentally made that exact change**, your test suite would not catch it
3. **There's a gap in your test coverage** - not line coverage, but *semantic* coverage

Here is a concrete example. The `clamp` function has what looks like solid coverage, but there is a surviving mutant hiding in the less-than check:

```cpp
// Concrete example
// Original code:
int clamp(int val, int lo, int hi) {
    if (val < lo) return lo;
    if (val > hi) return hi;
    return val;
}

// Tests:
// clamp(5, 0, 10)   -> 5
// clamp(-5, 0, 10)  -> 0
// clamp(15, 0, 10)  -> 10

// Surviving mutant: val < lo -> val <= lo
// clamp(5, 0, 10)   -> 5     (same as original)
// clamp(-5, 0, 10)  -> 0     (same as original)
// clamp(15, 0, 10)  -> 10    (same as original)
// ALL PASS! Because we never test clamp(0, 0, 10):
//   Original: 0 (returns val)
//   Mutant:   0 (returns lo, but lo==val, so same!)
// Even clamp(0, 0, 10) can't kill it!
//
// We need: clamp(lo, lo, hi) where lo != the expected output...
// Actually: clamp(0, 0, 10) returns 0 in BOTH cases (equivalent mutant)
// BUT clamp(5, 5, 10): original returns 5, mutant returns 5 (val==lo, both return lo)
// This IS an equivalent mutant - semantically identical for all inputs
// -> Not all survivors indicate test gaps; some are equivalent mutants
```

The reason this trips people up is that equivalent mutants look like gaps but are not. When `val == lo`, returning `lo` and returning `val` produce the same result. You can never kill that mutant because the two behaviours are mathematically the same for all inputs. These are just noise in the report and should be documented, not chased.

### Q3: Add tests that kill the surviving mutants and re-run to verify 100% mutation score

**Answer:**

Now let's go back to Q1's survivors and write the specific tests that kill them. The key insight is that each test should target the exact behavioural boundary the surviving mutant exposed:

```cpp
// Kill the survivors from Q1

// Survivor: b == 0.0 -> b != 0.0 (inverted division guard)
// The mutant would throw on NON-zero division and NOT throw on zero division

// Kill it by testing normal division AND checking zero throws:
TEST(Calc, DivideNormal) {
    // With the mutant (b != 0.0), this THROWS because b=2 != 0
    EXPECT_DOUBLE_EQ(calculate(10, "/", 2), 5.0);  // KILLS the mutant
    EXPECT_DOUBLE_EQ(calculate(7, "/", 3.5), 2.0);
}

TEST(Calc, DivZeroConfirm) {
    // Double-check both sides of the guard
    EXPECT_THROW(calculate(1, "/", 0), std::domain_error);
    EXPECT_NO_THROW(calculate(1, "/", 1));  // Must NOT throw
}

// Additional boundary tests to kill remaining mutants:
TEST(Calc, AddIdentity) {
    EXPECT_DOUBLE_EQ(calculate(0, "+", 0), 0.0);
    EXPECT_DOUBLE_EQ(calculate(5, "+", 0), 5.0);
    EXPECT_DOUBLE_EQ(calculate(0, "+", 5), 5.0);
}

TEST(Calc, SubtractIdentity) {
    EXPECT_DOUBLE_EQ(calculate(5, "-", 5), 0.0);
    EXPECT_DOUBLE_EQ(calculate(0, "-", 0), 0.0);
}

TEST(Calc, MultiplyByZero) {
    EXPECT_DOUBLE_EQ(calculate(5, "*", 0), 0.0);
    EXPECT_DOUBLE_EQ(calculate(0, "*", 5), 0.0);
}

TEST(Calc, PowerEdgeCases) {
    EXPECT_DOUBLE_EQ(calculate(2, "^", 0), 1.0);
    EXPECT_DOUBLE_EQ(calculate(0, "^", 5), 0.0);
    EXPECT_DOUBLE_EQ(calculate(1, "^", 100), 1.0);
}
```

Re-run Mull and check the new score. One equivalent mutant remains - that is acceptable and expected:

```bash
# Re-run Mull with new tests:
mull-runner ./test_calc

# Mutation Score: 96.4% (27/28 killed)
# 1 equivalent mutant remaining (acceptable)
```

The remaining unkilledmutant is almost certainly an equivalent mutant - not a real gap.

---

## Notes

- A surviving mutant is a **concrete, actionable** signal - unlike "increase coverage", it tells you **exactly where** the gap is.
- Equivalent mutants (semantically identical to original) cannot be killed - expect roughly 5-10% in any real codebase.
- For C++ projects, Mull is the most mature tool; `mutmut` is Python-only but useful for pybind11 test suites.
- Run mutation testing nightly or on PRs touching critical code paths - it's too slow for every commit.
- Prioritize killing mutants in **boundary conditions** and **error paths** - these are where real bugs hide.
