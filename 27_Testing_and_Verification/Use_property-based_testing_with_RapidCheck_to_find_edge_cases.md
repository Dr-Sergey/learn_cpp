# Use property-based testing with RapidCheck to find edge cases

**Category:** Testing and Verification  
**Standard:** C++11  
**Reference:** <https://github.com/emil-e/rapidcheck>  

---

## Topic Overview

Property-based testing excels at **finding edge cases** that hand-written examples miss. RapidCheck generates hundreds of random inputs and, when a property fails, automatically **shrinks** the counterexample to the smallest, most informative failing input.

### Why Edge Cases Slip Through

The reason edge cases escape manual testing is not carelessness - it is that the space of interesting inputs is much larger than it feels. You remember to test a negative number, but do you remember `INT_MIN`? You test an empty string, but do you test a string with embedded NUL bytes? RapidCheck generates all of these naturally.

| Manual Testing Problem | RapidCheck Solution |
| --- | --- |
| You forget `INT_MIN`, `INT_MAX` | Generates extreme values naturally |
| Empty containers not tested | Generates `size == 0` |
| String with NUL bytes | Generates arbitrary `char` including `'\0'` |
| Off-by-one at boundaries | Tests thousands of boundary-adjacent values |
| Negative numbers ignored | Generates full `int` range |

### Shrinking: From Chaos to Clarity

When RapidCheck finds a failing input, it does not stop there. It runs a systematic reduction process to find the *smallest* input that still triggers the failure. This is what makes the counterexample actually useful for debugging - instead of a 50-element vector with seemingly random values, you get the three-element minimal case that exposes the bug.

```cpp
Initial failure:  [-847, 42, 0, -3, 999, 7, -1]   (random mess)
         | shrink vector size
              [-847, 42, 0]
         | shrink element values
              [-1, 1, 0]
         | try further
              [0, 1, 0]      <- minimal counterexample!
```

Shrinking is **automatic** for built-in types. RapidCheck tries:

1. Reduce container length
2. Reduce element magnitudes toward 0
3. Remove unnecessary elements
4. Simplify structure while keeping the failure

---

## Self-Assessment

### Q1: Write a property that reverse(reverse(v)) == v for arbitrary std::vector<int>

**Answer:**

A round-trip property is one of the cleanest patterns in property-based testing: if you do an operation and then undo it, you should get back what you started with. Reversing twice is a textbook example. Notice that RapidCheck automatically discovers the interesting edge cases - empty vectors, single-element vectors, vectors with duplicates, vectors with `INT_MIN` - all without you having to think of them:

<!-- compile: needs third-party library header `rapidcheck.h` -->
```cpp
#include <rapidcheck.h>
#include <algorithm>
#include <vector>
#include <string>

int main() {
    // Round-trip property: reverse is its own inverse
    rc::check("reverse(reverse(v)) == v",
    [](std::vector<int> v) {
        auto original = v;

        std::reverse(v.begin(), v.end());
        std::reverse(v.begin(), v.end());

        RC_ASSERT(v == original);
    });

    // Edge cases RapidCheck auto-discovers
    // In 100 runs it will test vectors of size:
    //   - 0 (empty)         -> trivially passes
    //   - 1 (single element) -> trivially passes
    //   - large (50+ elements)
    //   - containing INT_MIN, INT_MAX, 0, negative

    // Additional properties of reverse
    rc::check("reverse preserves length",
    [](std::vector<int> v) {
        auto sz = v.size();
        std::reverse(v.begin(), v.end());
        RC_ASSERT(v.size() == sz);
    });

    rc::check("reverse swaps first and last",
    [](std::vector<int> v) {
        RC_PRE(v.size() >= 2);
        auto first = v.front();
        auto last  = v.back();
        std::reverse(v.begin(), v.end());
        RC_ASSERT(v.front() == last);
        RC_ASSERT(v.back()  == first);
    });

    // Works for strings too
    rc::check("reverse(reverse(s)) == s for strings",
    [](std::string s) {
        auto original = s;
        std::reverse(s.begin(), s.end());
        std::reverse(s.begin(), s.end());
        RC_ASSERT(s == original);
    });

    return 0;
}

// Output:
// reverse(reverse(v)) == v            OK, passed 100 tests
// reverse preserves length            OK, passed 100 tests
// reverse swaps first and last        OK, passed 100 tests
// reverse(reverse(s)) == s for strings OK, passed 100 tests
```

The string version is particularly useful because `std::string` can contain NUL bytes and other tricky characters that you would never think to include in a hand-written test.

### Q2: Use rc::gen::container to generate arbitrary containers with size constraints

**Answer:**

`rc::gen::container` lets you build structured inputs rather than accepting whatever arbitrary generates. Here we use it to test a `second_largest` function that has specific preconditions (it needs at least two distinct elements), so we construct inputs that satisfy those constraints rather than filtering most of them out with `RC_PRE`:

<!-- compile: needs third-party library header `rapidcheck.h` -->
```cpp
#include <rapidcheck.h>
#include <vector>
#include <set>
#include <map>
#include <string>

// Function under test
int second_largest(const std::vector<int>& v) {
    // Requires at least 2 distinct elements
    std::set<int> unique(v.begin(), v.end());
    if (unique.size() < 2) throw std::invalid_argument("need 2+ distinct");
    auto it = unique.end();
    --it;  // largest
    --it;  // second largest
    return *it;
}

int main() {
    // Fixed-size container
    rc::check("vector of exactly 5 ints",
    []() {
        // Generate vector with EXACTLY 5 elements
        auto v = *rc::gen::container<std::vector<int>>(
            5,
            rc::gen::arbitrary<int>()
        );
        RC_ASSERT(v.size() == 5);
    });

    // Bounded-size container
    rc::check("non-empty vector up to 100 elements",
    []() {
        auto v = *rc::gen::container<std::vector<int>>(
            rc::gen::inRange(0, 1000)   // element generator
        );
        RC_PRE(!v.empty());  // Filter: at least 1 element
        RC_PRE(v.size() <= 100);  // Filter: at most 100
        // Or better - use rc::gen::resize:
        // auto v = *rc::gen::resize(100, rc::gen::container<std::vector<int>>(...));
        RC_ASSERT(v.size() >= 1);
    });

    // Set with unique elements
    rc::check("set always has unique elements",
    []() {
        auto s = *rc::gen::container<std::set<int>>(
            rc::gen::arbitrary<int>()
        );
        // set guarantees uniqueness - property holds trivially
        std::vector<int> as_vec(s.begin(), s.end());
        for (size_t i = 1; i < as_vec.size(); ++i) {
            RC_ASSERT(as_vec[i-1] < as_vec[i]);  // Strictly increasing
        }
    });

    // Map with constrained keys/values
    rc::check("map keys in [0, 100), values positive",
    []() {
        auto m = *rc::gen::container<std::map<int, int>>(
            rc::gen::pair(
                rc::gen::inRange(0, 100),    // keys [0, 100)
                rc::gen::inRange(1, 10000)   // values [1, 10000)
            )
        );
        for (auto& [k, v] : m) {
            RC_ASSERT(k >= 0 && k < 100);
            RC_ASSERT(v >= 1);
        }
    });

    // Edge-case hunting with second_largest
    rc::check("second_largest < max for 2+ distinct elements",
    []() {
        auto v = *rc::gen::container<std::vector<int>>(
            rc::gen::arbitrary<int>()
        );
        std::set<int> unique(v.begin(), v.end());
        RC_PRE(unique.size() >= 2);   // Need 2+ distinct

        int result = second_largest(v);
        int max_val = *std::max_element(v.begin(), v.end());
        RC_ASSERT(result < max_val);
        RC_ASSERT(std::find(v.begin(), v.end(), result) != v.end());
    });

    return 0;
}
```

The `second_largest` test is a good illustration of when `RC_PRE` is appropriate: the function has a clear mathematical precondition, so filtering out invalid inputs is exactly right. If you find that `RC_PRE` is rejecting too many generated inputs (causing lots of "discards" in the output), switch to a custom generator that only generates valid inputs in the first place.

### Q3: Explain shrinking: how the minimal failing example is automatically derived

**Answer:**

Shrinking is RapidCheck's killer feature. When a property fails, it doesn't just report the raw random input - it systematically reduces the counterexample to the **smallest input that still fails**.

The reason this matters so much is that the raw random input is usually too large and noisy to reason about. A vector of 40 seemingly random integers tells you almost nothing. The minimal counterexample `[0, 1, 0]` immediately shows you exactly what the bug is about.

**The Shrinking Algorithm:**

```cpp
                         [42, -93, 7, -93, 15]   <- initial failure
                                  |
                    ..............|..............
                    |             |              |
              remove elements  shrink values   combine
              [42, -93, 7]    [1, -1, 0, -1]   ...
                    |             |
                  fails?        fails?
                    |             |
              [42, -93]       [1, -1, 0, -1]
              passes!         fails! -> continue shrinking...
                                  |
                              [0, 1, 0]
                              fails!
                                  |
                              try smaller...
                              [0, 0] -> passes
                              [1, 0] -> passes
                              ================
                              MINIMAL: [0, 1, 0]
```

**Shrinking strategies by type:**

| Type | Shrink Direction | Examples |
| --- | --- | --- |
| `int` | Toward 0 | 42 -> 21 -> 10 -> 5 -> 2 -> 1 -> 0 |
| `bool` | Toward false | true -> false |
| `string` | Shorter + simpler chars | "xK9" -> "aA" -> "a" -> "" |
| `vector<T>` | Fewer elements + shrink each | [5,3,2] -> [5,3] -> [1,1] -> [0,1] |
| `pair<A,B>` | Shrink A, then B | (99, "hi") -> (0, "a") |
| `optional<T>` | Toward nullopt, then shrink T | opt(42) -> opt(0) -> nullopt |

Here is a demonstration. The `sum_ints` function has a signed integer overflow bug. The property we write checks that splitting the sum into halves gives the same result as summing the whole thing. When overflow occurs, the associativity breaks. RapidCheck finds it and shrinks to the minimal overflow trigger:

<!-- compile: needs third-party library header `rapidcheck.h` -->
```cpp
#include <rapidcheck.h>
#include <vector>
#include <iostream>
#include <numeric>

// Buggy: overflow for large sums
int sum_ints(const std::vector<int>& v) {
    int total = 0;
    for (int x : v) total += x;  // BUG: signed overflow for large inputs
    return total;
}

int main() {
    // Find overflow with property
    rc::check("sum is associative (splits give same result)",
    [](std::vector<int> v) {
        RC_PRE(v.size() >= 2);

        int full = sum_ints(v);

        // Split in half and sum separately
        size_t mid = v.size() / 2;
        std::vector<int> first(v.begin(), v.begin() + mid);
        std::vector<int> second(v.begin() + mid, v.end());

        int partial = sum_ints(first) + sum_ints(second);
        RC_ASSERT(full == partial);
    });
    // With overflow, this WILL fail.
    // RapidCheck shrinks to something like:
    //   v = [2147483647, 1]   <- minimal overflow!

    // Custom type with custom shrinking
    // For user-defined types, implement rc::Arbitrary<T>::shrink()
    // to get good shrinking behavior.

    return 0;
}
```

The shrunk counterexample `[2147483647, 1]` is perfect: it is the smallest vector that demonstrates that `INT_MAX + 1` overflows in `int`. You immediately know where to look.

**Key shrinking rules:**

1. **Automatic** for all built-in types and std containers.
2. **Compositional**: shrinking `vector<pair<int,string>>` shrinks each layer independently.
3. **Custom types** need `rc::Arbitrary<T>::shrink()` to get useful minimized output.
4. **Preconditions** (`RC_PRE`) are respected: shrunk inputs that violate preconditions are skipped automatically.
5. The algorithm uses **binary search** style reduction - it's fast even for large inputs.

---

## Notes

- Focus on edge cases: property-based testing shines at boundaries (empty, max-size, overflow, NaN).
- Complement example-based tests - don't replace them entirely; both serve different purposes.
- `RC_PRE()` filters invalid inputs but too many discards slow tests - prefer custom generators when possible.
- Seed control: `RC_PARAMS(seed=42)` for reproducibility, or `RC_PARAMS(seed=0)` for random seeding.
- RapidCheck generates edge cases early: 0, empty, INT_MIN, INT_MAX are tried before random values.
