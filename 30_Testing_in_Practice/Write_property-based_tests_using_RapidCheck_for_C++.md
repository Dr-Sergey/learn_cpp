# Write property-based tests using RapidCheck for C++

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Property-based testing (PBT)** generates random inputs and verifies that certain **properties** (invariants) always hold, rather than checking specific example inputs. This finds edge cases that human-written examples miss. **RapidCheck** is the leading C++ PBT library, inspired by Haskell's QuickCheck.

The key mental shift with PBT is moving from "does this specific input produce this specific output?" to "does this operation always satisfy this general rule?". You cannot write an example-based test for every possible integer - but you can write a property that says "for any two integers, addition is commutative" and have the framework throw hundreds of random pairs at it. If any pair breaks the property, RapidCheck will automatically **shrink** the failing input down to the smallest reproduction case, which is invaluable for debugging.

### Example-Based vs Property-Based

| Example-Based Testing | Property-Based Testing |
| --- | --- |
| `sort({3,1,2}) == {1,2,3}` | For ANY vector: `sort(v)` is sorted |
| `reverse("abc") == "cba"` | For ANY string: `reverse(reverse(s)) == s` |
| Tests specific values | Tests universal properties |
| Human picks edge cases | Generator finds edge cases |
| 5-20 test cases typical | 100+ random cases per property |

### Common Properties to Test

| Property Type | Description | Example |
| --- | --- | --- |
| **Roundtrip** | encode then decode = identity | `deserialize(serialize(x)) == x` |
| **Idempotence** | Applying twice = applying once | `sort(sort(v)) == sort(v)` |
| **Invariant** | Some condition always holds | `size(filter(v, p)) <= size(v)` |
| **Equivalence** | Two implementations agree | `fast_sort(v) == std::sort(v)` |
| **Commutative** | Order doesn't matter | `a + b == b + a` |

---

## Self-Assessment

### Q1: Write property-based tests with RapidCheck

**Answer:**

Each of the properties below captures a different guarantee that `std::sort` should satisfy. They work together as a complete specification: the output is sorted, it contains the same elements as the input, sorting again produces the same result, and the size is preserved. No single example-based test could express all four of these at once.

```cpp
#include <rapidcheck.h>
#include <rapidcheck/gtest.h>  // Google Test integration
#include <gtest/gtest.h>
#include <algorithm>
#include <string>
#include <vector>
#include <numeric>

// === Property: sorting produces sorted output ===
RC_GTEST_PROP(SortProperties, OutputIsSorted, ()) {
    auto input = *rc::gen::container<std::vector<int>>(rc::gen::arbitrary<int>());

    std::sort(input.begin(), input.end());

    RC_ASSERT(std::is_sorted(input.begin(), input.end()));
}

// === Property: sorting preserves elements ===
RC_GTEST_PROP(SortProperties, PreservesElements, ()) {
    auto input = *rc::gen::container<std::vector<int>>(rc::gen::arbitrary<int>());
    auto original = input;

    std::sort(input.begin(), input.end());

    // Same elements, possibly different order
    std::sort(original.begin(), original.end());
    RC_ASSERT(input == original);
}

// === Property: sorting is idempotent ===
RC_GTEST_PROP(SortProperties, SortingTwiceIsSameAsOnce, ()) {
    auto input = *rc::gen::container<std::vector<int>>(rc::gen::arbitrary<int>());

    std::sort(input.begin(), input.end());
    auto once = input;
    std::sort(input.begin(), input.end());

    RC_ASSERT(input == once);
}

// === Property: sorting preserves size ===
RC_GTEST_PROP(SortProperties, PreservesSize, ()) {
    auto input = *rc::gen::container<std::vector<int>>(rc::gen::arbitrary<int>());
    auto size_before = input.size();

    std::sort(input.begin(), input.end());

    RC_ASSERT(input.size() == size_before);
}

// === Property: reverse is self-inverse ===
RC_GTEST_PROP(ReverseProperties, ReverseReverseIsIdentity, ()) {
    auto s = *rc::gen::arbitrary<std::string>();
    auto original = s;

    std::reverse(s.begin(), s.end());
    std::reverse(s.begin(), s.end());

    RC_ASSERT(s == original);
}

// === Property: string length preserved ===
RC_GTEST_PROP(ReverseProperties, PreservesLength, ()) {
    auto s = *rc::gen::arbitrary<std::string>();
    auto len = s.size();

    std::reverse(s.begin(), s.end());

    RC_ASSERT(s.size() == len);
}
```

The `*rc::gen::arbitrary<int>()` syntax is how you generate a random value and dereference it into an actual `int`. The `*` here is the RapidCheck dereference operator - it pulls the generated value out of the generator.

### Q2: Write custom generators for domain-specific types

**Answer:**

The built-in `arbitrary<std::string>()` generates completely random strings including non-ASCII characters and empty strings. For an `Email` type you need strings that look like valid emails, which means writing a custom generator. The specialization of `rc::Arbitrary<Email>` below tells RapidCheck how to build an `Email` from its component parts.

```cpp
#include <rapidcheck.h>
#include <rapidcheck/gtest.h>

// === Domain type ===
struct Email {
    std::string local_part;   // e.g., "user.name"
    std::string domain;       // e.g., "example.com"

    std::string to_string() const {
        return local_part + "@" + domain;
    }

    static Email from_string(const std::string& s) {
        auto at = s.find('@');
        if (at == std::string::npos) throw std::invalid_argument("No @");
        return {s.substr(0, at), s.substr(at + 1)};
    }
};

// === Custom generator ===
namespace rc {
template<>
struct Arbitrary<Email> {
    static Gen<Email> arbitrary() {
        return gen::build<Email>(
            gen::set(&Email::local_part,
                gen::nonEmpty(
                    gen::container<std::string>(
                        gen::elementOf('a', 'b', 'c', 'd', '.', '_', '1', '2')))),
            gen::set(&Email::domain,
                gen::map(
                    gen::pair(
                        gen::nonEmpty(gen::string<std::string>()),
                        gen::elementOf(
                            std::string("com"),
                            std::string("org"),
                            std::string("net"))),
                    [](auto pair) {
                        return pair.first + "." + pair.second;
                    }))
        );
    }
};
}  // namespace rc

// === Roundtrip property ===
RC_GTEST_PROP(EmailProperties, RoundtripParsing, ()) {
    auto email = *rc::gen::arbitrary<Email>();

    auto str = email.to_string();
    auto parsed = Email::from_string(str);

    RC_ASSERT(parsed.local_part == email.local_part);
    RC_ASSERT(parsed.domain == email.domain);
}

// === Invariant: always contains @ ===
RC_GTEST_PROP(EmailProperties, AlwaysContainsAt, ()) {
    auto email = *rc::gen::arbitrary<Email>();
    auto str = email.to_string();

    RC_ASSERT(str.find('@') != std::string::npos);
}

// === Custom generator for constrained values ===
RC_GTEST_PROP(BoundedValues, PortIsValid, ()) {
    // Generate port numbers in valid range
    auto port = *rc::gen::inRange(1, 65536);

    RC_ASSERT(port >= 1);
    RC_ASSERT(port <= 65535);
}

// === Stateful testing: model-based ===
RC_GTEST_PROP(VectorModel, MatchesReference, ()) {
    // Test our custom container against std::vector as a reference model
    std::vector<int> model;  // Reference implementation
    std::vector<int> sut;    // System under test (normally your custom container)

    auto commands = *rc::gen::container<std::vector<int>>(
        rc::gen::inRange(0, 3));  // 0=push, 1=pop, 2=check_size

    for (int cmd : commands) {
        if (cmd == 0) {
            auto val = *rc::gen::arbitrary<int>();
            model.push_back(val);
            sut.push_back(val);
        } else if (cmd == 1 && !model.empty()) {
            model.pop_back();
            sut.pop_back();
        }
        RC_ASSERT(model.size() == sut.size());
        RC_ASSERT(model == sut);
    }
}
```

The stateful (model-based) pattern at the bottom is the most powerful technique PBT offers for data structures. You run a random sequence of operations against both your implementation and a trusted reference, checking after every step that they agree. RapidCheck will find the shortest sequence of operations that triggers any disagreement.

### Q3: Find real bugs with property-based testing

**Answer:**

This is where PBT earns its keep. The two bugs below are easy to write and easy to overlook in code review - but a property test will find them because it generates inputs specifically designed to explore the edge cases of integer ranges and array boundaries.

```cpp
#include <rapidcheck.h>
#include <rapidcheck/gtest.h>

// === Buggy function: integer overflow ===
int safe_multiply(int a, int b) {
    return a * b;  // BUG: no overflow check!
}

// Property-based test finds the overflow
RC_GTEST_PROP(MultiplyProperties, ResultDivisibleByInputs, ()) {
    auto a = *rc::gen::inRange(1, 1000000);
    auto b = *rc::gen::inRange(1, 1000000);

    int result = safe_multiply(a, b);

    // This property fails when a*b overflows!
    RC_ASSERT(result / a == b);
    // RapidCheck will shrink to minimal failing case, e.g.:
    // a=46341, b=46341 (46341*46341 overflows int32)
}

// === Buggy function: off-by-one in binary search ===
int buggy_binary_search(const std::vector<int>& sorted, int target) {
    int lo = 0, hi = sorted.size();  // BUG: should be size()-1
    while (lo <= hi) {
        int mid = (lo + hi) / 2;  // BUG: can overflow; BUG: out of bounds when hi=size()
        if (sorted[mid] == target) return mid;
        if (sorted[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

// RapidCheck will find inputs that cause out-of-bounds access
RC_GTEST_PROP(BinarySearchProperties, FindsExistingElements, ()) {
    auto vec = *rc::gen::nonEmpty(
        rc::gen::container<std::vector<int>>(rc::gen::arbitrary<int>()));
    std::sort(vec.begin(), vec.end());
    vec.erase(std::unique(vec.begin(), vec.end()), vec.end());

    // Pick a random element that exists
    auto idx = *rc::gen::inRange<size_t>(0, vec.size());
    int target = vec[idx];

    int found = buggy_binary_search(vec, target);
    RC_ASSERT(found >= 0);
    RC_ASSERT(vec[found] == target);
}
```

The reason `46341 * 46341` is the shrunk minimal case for the overflow bug is that `46341^2 = 2147488281`, which just barely overflows a 32-bit signed integer. RapidCheck did not know that - it started with some large failing input and automatically minimized it to the smallest pair that still causes a failure. That is the shrinking feature in action.

---

## Notes

- RapidCheck **shrinks** failing inputs to the smallest reproduction case - invaluable for debugging.
- `RC_GTEST_PROP` integrates with Google Test - failures appear as gtest failures.
- Default: 100 test cases per property. Configure with `RC_PARAMS(maxSuccess=1000)`.
- Common properties: roundtrip (serialize/deserialize), idempotence, invariant preservation.
- Custom generators ensure inputs satisfy preconditions (e.g., sorted vectors, valid emails).
- **Stateful testing** (model-based) is the most powerful PBT technique for data structures.
- PBT complements example-based tests - use both. PBT finds edge cases; examples document intent.
- CMake integration: `FetchContent_Declare(rapidcheck GIT_REPOSITORY https://github.com/emil-e/rapidcheck.git)`.
