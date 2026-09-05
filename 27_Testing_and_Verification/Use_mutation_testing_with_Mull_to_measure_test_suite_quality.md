# Use mutation testing with Mull to measure test suite quality

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/mull-project/mull>  

---

## Topic Overview

This file focuses on **practical Mull usage** - running it on a project, interpreting output, and understanding why 100% line coverage is insufficient. (See file #684 for mutation testing concepts, mutation operators, and the coverage-vs-mutation-score comparison.)

### Mull Architecture

Understanding how Mull works under the hood explains both why it is powerful and why it is slow. Mull does not modify your source code and recompile thousands of times. Instead, it compiles once to LLVM IR and injects mutations at that level, running each mutated version entirely in memory. That is much faster than source-level tools, but it still means running your full test suite once per mutant.

```cpp
Source code (.cpp)
      │
      ▼ Compile with -fpass-plugin=mull-ir-frontend
LLVM IR (.bc)
      │
      ▼ Mull injects mutations at IR level
Mutated binaries (in memory)
      │
      ▼ Each mutant runs the full test suite
Results: killed / survived / timeout
      │
      ▼
Mutation score report (console, JSON, HTML)
```

### Mull Configuration (mull.yml)

The configuration file lets you choose which mutation operators to enable and which paths to skip. Excluding test code from mutation is important - you do not want Mull mutating your `EXPECT_EQ` calls, because surviving those mutations tells you nothing useful about production code quality.

```yaml
mutators:
  - cxx_add_to_sub        # + -> -
  - cxx_mul_to_div        # * -> /
  - cxx_lt_to_le          # < -> <=
  - cxx_remove_void_call  # Remove void function calls
  - negate_mutator        # if(x) -> if(!x)

excludePaths:
  - "third_party/.*"
  - "test/.*"           # Don't mutate test code

timeout: 5000  # Kill slow mutants after 5s
```

---

## Self-Assessment

### Q1: Run Mull on a small project and interpret the mutation score percentage

The password validator below has five tests covering what looks like the full behavior. You would reasonably expect a good mutation score. Watch what Mull finds.

```cpp
// Source: password_validator.cpp
#include <string>
#include <algorithm>
#include <cctype>

struct ValidationResult {
    bool valid;
    std::string reason;
};

ValidationResult validate_password(const std::string& pw) {
    if (pw.length() < 8)
        return {false, "too short"};
    if (pw.length() > 64)
        return {false, "too long"};
    if (std::none_of(pw.begin(), pw.end(),
                      [](unsigned char c) { return std::isupper(c); }))
        return {false, "no uppercase"};
    if (std::none_of(pw.begin(), pw.end(),
                      [](unsigned char c) { return std::isdigit(c); }))
        return {false, "no digit"};
    return {true, "ok"};
}
```

```cpp
// Tests: test_password.cpp
#include <gtest/gtest.h>

extern ValidationResult validate_password(const std::string&);

TEST(Password, TooShort)    { EXPECT_FALSE(validate_password("Ab1").valid); }
TEST(Password, TooLong)     { EXPECT_FALSE(validate_password(std::string(65, 'A')).valid); }
TEST(Password, NoUppercase) { EXPECT_FALSE(validate_password("abcdefg1").valid); }
TEST(Password, NoDigit)     { EXPECT_FALSE(validate_password("Abcdefgh").valid); }
TEST(Password, Valid)       { EXPECT_TRUE(validate_password("Abcdefg1").valid); }
```

```bash
# Build and run Mull
clang++ -fpass-plugin=/usr/lib/mull-ir-frontend.so \
        -g -O0 password_validator.cpp test_password.cpp \
        -lgtest -lgtest_main -o test_pw

mull-runner --reporters=Elements ./test_pw

# Output:
# -----------------------------------------
# Mull Mutation Testing Report
# -----------------------------------------
# Total mutants:   24
# Killed:          18
# Survived:         4
# Timeout:          2
# -----------------------------------------
# Mutation Score:  75.0%
#
# Survived mutants:
#   password_validator.cpp:12  < -> <=  (pw.length() < 8 -> <= 8)
#   password_validator.cpp:14  > -> >=  (pw.length() > 64 -> >= 64)
#   password_validator.cpp:12  8 -> 0   (pw.length() < 8 -> < 0)
#   password_validator.cpp:14  64 -> 0  (pw.length() > 64 -> > 0)
```

All four survivors point to the same gap: the tests never check what happens at exactly the boundary values 8 and 64. The tests use 3-character and 65-character inputs, which are clearly inside the failing ranges, but nothing sits right on the threshold.

### Q2: Identify survived mutants (untested code paths) and write tests to kill them

Each survived mutant from Q1 maps to a specific missing test. The analysis below walks through all four and shows both the fix and - for the equivalent mutants - why they are acceptable to leave alone.

```cpp
// Analyze survived mutants and fix

// Survived: pw.length() < 8 -> pw.length() <= 8
// Meaning: our tests don't verify that a password of EXACTLY 8 chars is valid
// Kill it with:
TEST(Password, ExactlyMinLength) {
    EXPECT_TRUE(validate_password("Abcdef1x").valid);  // length = 8
    EXPECT_FALSE(validate_password("Abcde1x").valid);   // length = 7
}

// Survived: pw.length() > 64 -> pw.length() >= 64
// Meaning: we don't test exactly 64 chars
TEST(Password, ExactlyMaxLength) {
    std::string pw64(62, 'a');
    pw64 += "A1";  // length = 64
    EXPECT_TRUE(validate_password(pw64).valid);  // 64 is valid

    std::string pw65 = pw64 + "x";  // length = 65
    EXPECT_FALSE(validate_password(pw65).valid);
}

// Survived: 8 -> 0 (pw.length() < 0 is always false)
// This is an "equivalent mutant" - it changes behavior but
// only for negative-length strings, which can't exist.
// -> Equivalent mutants are acceptable; ignore them.

// Survived: 64 -> 0 (pw.length() > 0)
// This mutant makes ALL non-empty passwords fail the length check
// Kill with any test that expects valid:
TEST(Password, NonEmptyIsRejectedByMutant) {
    // Our existing Valid test already covers this...
    // BUT: the mutant changes > 64 to > 0, which means
    // validate_password("Abcdefg1") would return {false, "too long"}
    auto r = validate_password("Abcdefg1");
    EXPECT_TRUE(r.valid);
    EXPECT_EQ(r.reason, "ok");  // Also check the reason!
}

// After adding boundary tests:
// Mutation Score: 95.8% (23/24 killed, 1 equivalent mutant)
```

The distinction between a real surviving mutant and an equivalent mutant matters in practice. An equivalent mutant represents a change that is semantically impossible to observe - like changing a length comparison threshold to a value that can never be reached. Chasing these down will never improve your score, so you should learn to recognize them and move on.

### Q3: Explain why 100% line coverage does not imply a high mutation score

The `grade` function below shows the gap as starkly as possible. You can hit every line in the function without ever checking what the function returns. Coverage tools measure execution; mutation testing measures observation.

```cpp
// 100% line coverage, 0% mutation score
#include <string>

// Function under test
std::string grade(int score) {
    if (score >= 90) return "A";      // Line 1
    if (score >= 80) return "B";      // Line 2
    if (score >= 70) return "C";      // Line 3
    return "F";                        // Line 4
}

// "Tests" that achieve 100% line coverage
void test_grade() {
    grade(95);   // Hits line 1
    grade(85);   // Hits line 2
    grade(75);   // Hits line 3
    grade(50);   // Hits line 4
    // 100% line coverage! Every line executed!
    // But NO ASSERTIONS - we never check the return value!
}

// Mull generates these mutants:
// 1. score >= 90 -> score > 90          All tests PASS -> SURVIVED
// 2. score >= 90 -> score >= 80         All tests PASS -> SURVIVED
// 3. return "A" -> return ""            All tests PASS -> SURVIVED
// 4. return "B" -> return "A"           All tests PASS -> SURVIVED
// ... every single mutant SURVIVES
// Mutation score: 0%

// PROPER tests that actually verify behavior
#include <cassert>
void test_grade_proper() {
    assert(grade(95) == "A");     // Kills mutants on line 1
    assert(grade(90) == "A");     // Kills: >= -> > boundary mutant
    assert(grade(89) == "B");     // Kills: >= 90 -> >= 80
    assert(grade(80) == "B");
    assert(grade(79) == "C");
    assert(grade(70) == "C");
    assert(grade(69) == "F");
    assert(grade(0)  == "F");
}
// Now mutation score: ~100%
```

**Key takeaway:** Line coverage measures *execution*. Mutation score measures *observation*. You can execute every line without observing (asserting) a single result. Mutation testing catches this gap.

---

## Notes

- Mull requires Clang (uses LLVM IR) - doesn't work with GCC
- `--reporters=Elements` generates detailed per-mutant reports
- `--reporters=JSON` for machine-readable CI integration
- Typical project: start with critical business logic only (mutating everything is too slow)
- Equivalent mutants (same behavior as original) are a known limitation - expect ~5-10% false positives
- Mull alternatives: `mutmut` (Python), `Stryker` (JS/C#), `pitest` (Java)
