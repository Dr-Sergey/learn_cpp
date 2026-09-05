# Implement mutation testing to evaluate test suite quality

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Mutation testing** evaluates how good your test suite actually is by introducing small, deliberate bugs (mutations) into the source code and checking whether your tests catch them. If a mutant survives - meaning your tests still pass despite the injected bug - you have found a gap in your tests. The **mutation score** (killed mutants divided by total mutants) is a much more honest quality metric than code coverage.

The reason code coverage can be misleading is that it only tells you which lines were *executed*, not whether any assertion actually verified the result on those lines. You can have 100% line coverage with tests that never call `EXPECT_EQ` - they would all pass even if every function returned the wrong value. Mutation testing exposes exactly those hollow tests.

### Mutation Testing vs Code Coverage

| Metric | Code Coverage | Mutation Score |
| --- | --- | --- |
| **Measures** | Which lines are executed | Whether tests detect changes |
| **False confidence** | High - 100% coverage doesn't mean good tests | Low - high scores indicate real verification |
| **Cost** | Cheap (single test run) | Expensive (many test runs) |
| **Typical target** | 80-90% line coverage | 70-80% mutation score is excellent |
| **Limitations** | Doesn't verify assertions exist | Slow, requires many test runs |

### Common Mutation Operators

These are the types of changes a mutation testing tool makes automatically. Notice how many of them are exactly the off-by-one and wrong-operator bugs that appear in real code:

| Operator | Original | Mutated |
| --- | --- | --- |
| **Arithmetic** | `a + b` | `a - b`, `a * b` |
| **Relational** | `a < b` | `a <= b`, `a > b`, `a == b` |
| **Logical** | `a && b` | `a \|\| b` |
| **Unary** | `-a` | `a` |
| **Return** | `return x;` | `return 0;`, `return x+1;` |
| **Void call** | `do_thing();` | (removed) |
| **Constant** | `42` | `0`, `1`, `43` |

---

## Self-Assessment

### Q1: Set up mutation testing with Mull for C++

**Answer:**

Mull is an LLVM-based mutation testing tool for C++. It works by instrumenting LLVM IR (the compiler's intermediate representation), which means it requires Clang and produces mutations at a level that can be more precise than source-level tools.

```bash
# === Install Mull (LLVM-based mutation testing) ===
# Ubuntu/Debian
sudo apt-get install mull-18  # Matches your LLVM version

# Verify
mull-runner-18 --version
```

```cmake
# === CMakeLists.txt - Prepare for mutation testing ===
# Mull works on LLVM bitcode, so we need Clang and special flags

option(ENABLE_MULL "Enable mutation testing with Mull" OFF)

if(ENABLE_MULL)
    # Mull requires specific Clang flags
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} \
        -fpass-plugin=/usr/lib/mull-ir-frontend-18 \
        -g -grecord-command-line")
endif()

add_library(my_math src/math_utils.cpp)
add_executable(math_tests tests/test_math.cpp)
target_link_libraries(math_tests my_math GTest::gtest_main)
```

Here is the production code we will use to demonstrate the difference between weak and strong tests:

```cpp
// === src/math_utils.h ===
#pragma once

int clamp(int value, int min, int max);
bool is_in_range(int value, int low, int high);
int safe_divide(int a, int b);
double calculate_discount(double price, int quantity);
```

```cpp
// === src/math_utils.cpp ===
#include "math_utils.h"
#include <stdexcept>

int clamp(int value, int min, int max) {
    if (value < min) return min;
    if (value > max) return max;
    return value;
}

bool is_in_range(int value, int low, int high) {
    return value >= low && value <= high;
}

int safe_divide(int a, int b) {
    if (b == 0) throw std::invalid_argument("Division by zero");
    return a / b;
}

double calculate_discount(double price, int quantity) {
    if (quantity >= 100) return price * 0.8;   // 20% off
    if (quantity >= 10)  return price * 0.9;   // 10% off
    return price;
}
```

### Q2: Write tests that survive mutation testing

**Answer:**

The contrast below shows why boundary tests are so important. The weak test only hits the middle case and completely misses the edges where mutants hide.

```cpp
// === tests/test_math.cpp ===
#include <gtest/gtest.h>
#include "math_utils.h"

// === WEAK tests: these let mutants survive ===
TEST(ClampWeak, BasicTest) {
    EXPECT_EQ(clamp(5, 0, 10), 5);  // Only tests middle path
    // Mutants that survive:
    // - change `value < min` to `value <= min` (not caught!)
    // - change `value > max` to `value >= max` (not caught!)
}

// === STRONG tests: kill all mutants ===
TEST(ClampStrong, ValueInRange) {
    EXPECT_EQ(clamp(5, 0, 10), 5);
}

TEST(ClampStrong, ValueBelowMin) {
    EXPECT_EQ(clamp(-5, 0, 10), 0);
}

TEST(ClampStrong, ValueAboveMax) {
    EXPECT_EQ(clamp(15, 0, 10), 10);
}

// Boundary tests kill `<` vs `<=` mutants
TEST(ClampStrong, ValueExactlyAtMin) {
    EXPECT_EQ(clamp(0, 0, 10), 0);  // Kills `<` -> `<=` mutant
}

TEST(ClampStrong, ValueExactlyAtMax) {
    EXPECT_EQ(clamp(10, 0, 10), 10);  // Kills `>` -> `>=` mutant
}

TEST(ClampStrong, ValueOneAboveMin) {
    EXPECT_EQ(clamp(1, 0, 10), 1);  // Kills off-by-one mutants
}

// === Discount tests: kill arithmetic mutants ===
TEST(DiscountStrong, NoDiscount) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 5), 100.0);
}

TEST(DiscountStrong, SmallDiscount) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 10), 90.0);
}

TEST(DiscountStrong, LargeDiscount) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 100), 80.0);
}

// Boundary kills `>=` vs `>` mutants
TEST(DiscountStrong, BoundaryAt10) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 9), 100.0);
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 10), 90.0);
}

TEST(DiscountStrong, BoundaryAt100) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 99), 90.0);
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 100), 80.0);
}

// === Range tests ===
TEST(IsInRangeStrong, Inside) {
    EXPECT_TRUE(is_in_range(5, 0, 10));
}

TEST(IsInRangeStrong, AtBoundaries) {
    EXPECT_TRUE(is_in_range(0, 0, 10));   // Kills >= vs > mutant
    EXPECT_TRUE(is_in_range(10, 0, 10));  // Kills <= vs < mutant
}

TEST(IsInRangeStrong, JustOutside) {
    EXPECT_FALSE(is_in_range(-1, 0, 10));  // Kills boundary mutants
    EXPECT_FALSE(is_in_range(11, 0, 10));
}

TEST(IsInRangeStrong, BothConditions) {
    // Kills `&&` -> `||` mutant: value outside one bound
    EXPECT_FALSE(is_in_range(-1, 0, 10));
    EXPECT_FALSE(is_in_range(11, 0, 10));
}
```

The pattern that kills the most relational operator mutants is always the same: test *at* the boundary value, test *one below* it, and test *one above* it. That is three tests per conditional, but it gives you confidence that the `<` cannot be changed to `<=` without anyone noticing.

### Q3: Run and interpret mutation testing results

**Answer:**

Here is how to actually run Mull and interpret what it tells you. The key is reading the list of surviving mutants - each one is a concrete hint about which assertion is missing from your test suite.

```bash
# === Build with Mull instrumentation ===
cmake -B build-mull \
    -DCMAKE_CXX_COMPILER=clang++-18 \
    -DENABLE_MULL=ON \
    -DCMAKE_BUILD_TYPE=Debug
cmake --build build-mull

# === Run mutation testing ===
mull-runner-18 ./build-mull/math_tests \
    --reporters=Elements \
    --report-dir=mull-report

# === Example output ===
# [info] Applying 47 mutations
# [info] Running mutants (47 total):
# Killed:    41 (87.2%)
# Survived:   4 (8.5%)
# Timeout:    2 (4.3%)
#
# Survived mutants:
#   src/math_utils.cpp:15: Replaced >= with > in is_in_range
#   src/math_utils.cpp:22: Replaced * with / in calculate_discount

# === mull.yml configuration ===
# Place in project root
```

When you see "Replaced >= with > in is_in_range survived", that tells you exactly what test to add: one that passes a value equal to `low` and expects `true`. If `>=` were changed to `>`, that specific test would fail and kill the mutant.

Configure Mull with a `mull.yml` to focus on the mutation operators that matter and exclude code you do not want to mutate:

```yaml
# === mull.yml ===
mutators:
  - cxx_arithmetic         # +, -, *, /
  - cxx_comparison         # <, <=, >, >=, ==, !=
  - cxx_logical            # &&, ||
  - cxx_remove_void_call   # Remove void function calls
  - cxx_replace_scalar     # Replace constants

excludePaths:
  - "third_party/.*"       # Don't mutate dependencies
  - "tests/.*"             # Don't mutate test code
  - ".*_generated\.cpp"    # Don't mutate generated code

timeout: 10000             # 10 seconds per mutant
```

---

## Notes

- Mutation score is more meaningful than code coverage for assessing test suite quality - it proves tests actually verify behavior, not just execute lines.
- Equivalent mutants are mutations that do not change observable behavior (for example, replacing `i++` with `++i`). They artificially inflate the denominator, so expect 5-10% of mutants to be equivalent and not killable.
- Start mutation testing on critical or complex modules, not the entire codebase - the runtime cost grows linearly with the number of mutants.
- Boundary tests are the single most effective technique for killing relational operator mutants. Always test at, one below, and one above each threshold value.
- When a mutant survives, ask: "What assertion would catch this change?" Then add that assertion.
- Mull works on LLVM IR and requires Clang. GCC users can try `mutate_cpp` or `dextool` as alternatives.
- Mutation testing is slow because it runs N full test suite executions for N mutants. Run it nightly or on critical paths only, not on every commit.
- Targeting 70-80% mutation score on critical code is a realistic and meaningful goal. 100% is impractical because of equivalent mutants.
