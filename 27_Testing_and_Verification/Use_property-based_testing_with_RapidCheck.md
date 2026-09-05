# Use property-based testing with RapidCheck

**Category:** Testing and Verification  
**Standard:** C++17  
**Reference:** <https://github.com/emil-e/rapidcheck>  

---

## Topic Overview

**Property-based testing** flips the testing model: instead of writing specific input/output pairs, you describe **invariants** (properties) that must hold for ALL inputs, and the framework generates random test cases automatically. The benefit is that you stop thinking about "what inputs should I try?" and start thinking about "what is always true about my function?" - a much more powerful lens.

### Property-Based vs Example-Based

If the table below feels abstract, the short version is: example-based tests check correctness for inputs you thought to check; property-based tests check correctness for inputs you forgot to check.

| Aspect | Example-Based | Property-Based |
| --- | :---: | :---: |
| Input selection | Manual (you pick) | Random (framework generates) |
| Coverage | Only chosen examples | Thousands of random inputs |
| Edge case discovery | You must think of them | Framework finds them |
| Test clarity | Very readable | Requires invariant thinking |
| Debugging failures | Easy (known input) | Shrinking gives minimal input |

### RapidCheck Key API

Here is the API you will use most often. The `*` dereference syntax is how RapidCheck generates a value inside a property lambda - it looks odd at first but becomes natural quickly.

| Function | Purpose |
| --- | --- |
| `rc::check(...)` | Run a property with random inputs |
| `rc::prop(...)` | Define property for GTest integration |
| `*rc::gen::arbitrary<T>()` | Generate random T |
| `*rc::gen::inRange(lo, hi)` | Random int in [lo, hi) |
| `*rc::gen::container<C>(gen)` | Random container with generated elements |
| `RC_ASSERT(expr)` | Property assertion (triggers shrinking on failure) |
| `RC_CLASSIFY(cond, label)` | Label distribution of test cases |

---

## Self-Assessment

### Q1: Write a property test that verifies sort(sort(v)) == sort(v) for any vector of integers

**Answer:**

The idea here is to express things we know are mathematically true about sorting - not just "sort([3,1,2]) gives [1,2,3]" but universal laws. Sorting twice should give the same result as sorting once, the output should be a permutation of the input, and so on. RapidCheck will test each property against 100 random vectors:

<!-- compile: needs third-party library header `rapidcheck.h` -->
```cpp
#include <rapidcheck.h>
#include <algorithm>
#include <vector>
#include <numeric>

int main() {
    // Property: sorting is idempotent
    rc::check("sort is idempotent", [](std::vector<int> v) {
        auto sorted_once = v;
        std::sort(sorted_once.begin(), sorted_once.end());

        auto sorted_twice = sorted_once;
        std::sort(sorted_twice.begin(), sorted_twice.end());

        RC_ASSERT(sorted_once == sorted_twice);
    });

    // Property: sorted output is a permutation of input
    rc::check("sort preserves elements", [](std::vector<int> v) {
        auto original = v;
        std::sort(v.begin(), v.end());

        // Same size
        RC_ASSERT(v.size() == original.size());

        // Same multiset of elements
        std::sort(original.begin(), original.end());
        RC_ASSERT(v == original);
    });

    // Property: sorted output is non-decreasing
    rc::check("sort produces non-decreasing sequence", [](std::vector<int> v) {
        std::sort(v.begin(), v.end());
        for (size_t i = 1; i < v.size(); ++i) {
            RC_ASSERT(v[i-1] <= v[i]);
        }
    });

    // Property: sort(v) has same min/max
    rc::check("sort preserves min and max", [](std::vector<int> v) {
        RC_PRE(!v.empty());  // Precondition: skip empty vectors
        auto min_before = *std::min_element(v.begin(), v.end());
        auto max_before = *std::max_element(v.begin(), v.end());

        std::sort(v.begin(), v.end());

        RC_ASSERT(v.front() == min_before);
        RC_ASSERT(v.back()  == max_before);
    });

    return 0;
}

// Output:
// Using configuration: seed=12345
// sort is idempotent                      OK, passed 100 tests
// sort preserves elements                 OK, passed 100 tests
// sort produces non-decreasing sequence   OK, passed 100 tests
// sort preserves min and max              OK, passed 100 tests
```

All four properties pass. Notice `RC_PRE(!v.empty())` on the last one - that tells RapidCheck to skip any generated empty vector rather than fail on the `front()`/`back()` calls.

### Q2: Use rc::gen::inRange and rc::gen::container to generate constrained inputs

**Answer:**

When `rc::gen::arbitrary<T>()` generates too many invalid inputs for your function, use constrained generators to aim the fuzzing at the interesting region. For a percentage calculator, you want `part` to be between 0 and `total`, not just any random integer:

<!-- compile: needs third-party library header `rapidcheck.h` -->
```cpp
#include <rapidcheck.h>
#include <vector>
#include <string>
#include <cmath>

// Function under test: percentage calculator
double percentage(int part, int total) {
    if (total == 0) throw std::invalid_argument("total cannot be zero");
    return (static_cast<double>(part) / total) * 100.0;
}

int main() {
    // gen::inRange: constrained integers
    rc::check("percentage is between 0 and 100 for valid inputs",
    []() {
        auto total = *rc::gen::inRange(1, 10001);   // [1, 10000]
        auto part  = *rc::gen::inRange(0, total + 1); // [0, total]

        double pct = percentage(part, total);
        RC_ASSERT(pct >= 0.0);
        RC_ASSERT(pct <= 100.0);
    });

    // gen::container: constrained vector
    rc::check("sum of non-negative ints is non-negative",
    []() {
        // Generate vector of 1-100 elements, each in [0, 1000]
        auto vec = *rc::gen::container<std::vector<int>>(
            rc::gen::inRange(0, 1001)
        );
        RC_PRE(!vec.empty());

        int sum = 0;
        for (int x : vec) sum += x;
        RC_ASSERT(sum >= 0);
    });

    // gen::container with size bounds
    rc::check("matrix row sums",
    []() {
        // Generate a "matrix" = vector of vectors, 1-10 rows x 1-10 cols
        auto rows = *rc::gen::inRange(1, 11);
        auto cols = *rc::gen::inRange(1, 11);

        auto matrix = *rc::gen::container<std::vector<std::vector<double>>>(
            rows,
            rc::gen::container<std::vector<double>>(
                cols,
                rc::gen::arbitrary<double>()
            )
        );

        RC_ASSERT(static_cast<int>(matrix.size()) == rows);
        for (auto& row : matrix) {
            RC_ASSERT(static_cast<int>(row.size()) == cols);
        }
    });

    // Custom generator: valid email-like strings
    rc::check("custom generator",
    []() {
        auto username = *rc::gen::container<std::string>(
            rc::gen::inRange(1, 20),  // 1-19 chars
            rc::gen::elementOf('a', 'b', 'c', 'x', 'y', 'z')
        );
        auto domain = *rc::gen::elementOf(
            std::string("gmail.com"),
            std::string("test.org"),
            std::string("example.net")
        );

        std::string email = username + "@" + domain;
        RC_ASSERT(email.find('@') != std::string::npos);
        RC_ASSERT(email.size() > 3);

        RC_CLASSIFY(domain == "gmail.com", "gmail");  // Track distribution
        RC_CLASSIFY(domain == "test.org", "test.org");
    });

    return 0;
}
```

The nested `gen::container` for the matrix is worth studying: you first generate the number of rows, then generate a container where each element is itself a container of `cols` doubles. RapidCheck composes generators naturally.

### Q3: Show a counterexample: RapidCheck finds and minimizes a failing input automatically

**Answer:**

This is where property-based testing really shines. When a property fails, RapidCheck does not just dump the giant random input on you - it *shrinks* the counterexample down to the smallest input that still fails. Watch what happens when we test a buggy `remove_duplicates` that only removes adjacent duplicates:

<!-- compile: needs third-party library header `rapidcheck.h` -->
```cpp
#include <rapidcheck.h>
#include <vector>
#include <algorithm>
#include <numeric>
#include <iostream>

// Buggy function
int index_of_max(const std::vector<int>& v) {
    // BUG: doesn't handle duplicates correctly
    // Returns FIRST index with max value, but...
    int max_idx = 0;
    for (size_t i = 1; i < v.size(); ++i) {
        if (v[i] > v[max_idx])  // BUG: should be >=? No, > is correct for "first"
            max_idx = i;
    }
    return max_idx;
}

// Another buggy function
std::vector<int> remove_duplicates(std::vector<int> v) {
    // BUG: only removes ADJACENT duplicates
    auto it = std::unique(v.begin(), v.end());
    v.erase(it, v.end());
    return v;
}

int main() {
    // Property test finds bug in remove_duplicates
    rc::check("remove_duplicates has no duplicates",
    [](std::vector<int> v) {
        auto result = remove_duplicates(v);

        // Check: no duplicates in output
        for (size_t i = 0; i < result.size(); ++i) {
            for (size_t j = i + 1; j < result.size(); ++j) {
                RC_ASSERT(result[i] != result[j]);
            }
        }
    });
    // Output:
    // Falsifiable after 5 tests and 2 shrinks
    //
    // Counterexample:
    //   v = [0, 1, 0]    <- MINIMAL failing input!
    //
    // RapidCheck first found something like [42, -7, 13, 42, 8, -7]
    // Then shrunk it to [0, 1, 0] - the simplest case showing the bug
    // remove_duplicates([0, 1, 0]) -> [0, 1, 0] (unique only removes ADJACENT)

    // Shrinking process
    /*
    Initial failure:    [42, -7, 13, 42, 8, -7]   (6 elements)
    Shrink attempt 1:   [42, -7, 13, 42]           (drop last 2) -> still fails
    Shrink attempt 2:   [42, -7, 42]               (drop 13)     -> still fails
    Shrink attempt 3:   [42, 42]                   (drop -7)     -> passes! (adjacent dup removed)
    Shrink attempt 4:   [42, -7, 42]               (backtrack)   -> fails
    Shrink attempt 5:   [1, 0, 1]                  (minimize values) -> still fails
    Shrink attempt 6:   [0, 1, 0]                  (minimize further) -> still fails
    Shrink attempt 7:   [0, 0]                     -> passes (adjacent dup)
    RESULT:             [0, 1, 0]                  <- minimal counterexample
    */

    return 0;
}
```

The shrunk counterexample `[0, 1, 0]` immediately makes the bug obvious: the two `0` values are not adjacent, so `std::unique` leaves them both in. If you had found this manually, you would probably have tested `[1, 1]` or `[1, 1, 2]` first and never tried non-adjacent duplicates. That is precisely the kind of thing RapidCheck excels at finding.

---

## Notes

- RapidCheck is header-only and CMake `FetchContent` friendly; it integrates with GTest and Catch2.
- The default is 100 random inputs per property; configure with `RC_PARAMS(maxSuccess=1000)` for more thorough runs.
- `RC_PRE(cond)` skips inputs that don't satisfy preconditions (similar to Hypothesis's `assume()` in Python).
- `RC_TAG("label")` and `RC_CLASSIFY` help verify that your generators are producing a useful distribution of inputs.
- Shrinking is automatic - RapidCheck always tries to find the SMALLEST input that triggers a failure before reporting it.
