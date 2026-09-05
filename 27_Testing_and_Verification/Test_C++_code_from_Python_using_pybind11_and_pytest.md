# Test C++ code from Python using pybind11 and pytest

**Category:** Testing and Verification  
**Standard:** C++20  
**Reference:** <https://pybind11.readthedocs.io/>  

---

## Topic Overview

This file focuses on **property-based testing with Python's Hypothesis library** against C++ code and **understanding pybind11 call overhead**. (See files #767 and #687 for basic pybind11 bindings, parametrize, and embedded interpreter usage.)

Property-based testing is a step up from example-based testing. Instead of writing specific inputs and expected outputs, you describe *properties* that must always hold - things like "to_lower is idempotent" or "split then join is a round-trip" - and Hypothesis generates hundreds of inputs trying to find a counterexample. When it finds one, it automatically shrinks the input to the smallest case that still fails, which makes debugging much easier. Running this against C++ code via pybind11 is particularly effective because Hypothesis can expose edge cases that you'd never think to write by hand.

### pybind11 Call Overhead

Every call from Python to C++ through pybind11 has a fixed overhead for the binding dispatch and any type conversion. It's worth understanding this so you know when it matters and when it doesn't.

```cpp
Python  --call-->  pybind11 wrapper  --call-->  C++ function
        ~100-300ns      type conversion          native speed
        overhead        + GIL release
```

| Call Pattern             | Overhead per Call | Acceptable For |
| --- | :---: | :---: |
| Simple scalar return     | ~100 ns | Yes - any test |
| Vector conversion        | ~1 us / 1000 elems | Yes - unit tests |
| Large NumPy buffer copy  | ~10 us / 1M elems | Yes - integration tests |
| Thousands of tiny calls  | measurable | Batch if possible |

---

## Self-Assessment

### Q1: Write a pybind11 module that exposes a C++ function and write a pytest test for it

**Answer:**

The example uses a string utilities library - a good fit because string operations are easy to reason about and produce clean property-based tests in Q2. The binding is straightforward: `pybind11/stl.h` handles the `std::vector<std::string>` return from `split`, and a default argument for `delim` makes the Python API feel natural.

```cpp
// string_utils.hpp
#pragma once
#include <string>
#include <algorithm>
#include <cctype>
#include <sstream>
#include <vector>

namespace strutil {

std::string to_upper(const std::string& s) {
    std::string result = s;
    std::transform(result.begin(), result.end(), result.begin(),
                   [](unsigned char c) { return std::toupper(c); });
    return result;
}

std::string to_lower(const std::string& s) {
    std::string result = s;
    std::transform(result.begin(), result.end(), result.begin(),
                   [](unsigned char c) { return std::tolower(c); });
    return result;
}

bool is_palindrome(const std::string& s) {
    auto clean = to_lower(s);
    clean.erase(std::remove_if(clean.begin(), clean.end(),
                [](unsigned char c) { return !std::isalnum(c); }),
                clean.end());
    return std::equal(clean.begin(), clean.begin() + clean.size() / 2,
                      clean.rbegin());
}

std::vector<std::string> split(const std::string& s, char delim) {
    std::vector<std::string> tokens;
    std::istringstream iss(s);
    std::string token;
    while (std::getline(iss, token, delim))
        if (!token.empty()) tokens.push_back(token);
    return tokens;
}

} // namespace strutil
```

```cpp
// bindings.cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include "string_utils.hpp"

namespace py = pybind11;

PYBIND11_MODULE(string_utils, m) {
    m.doc() = "C++ string utilities";
    m.def("to_upper", &strutil::to_upper);
    m.def("to_lower", &strutil::to_lower);
    m.def("is_palindrome", &strutil::is_palindrome);
    m.def("split", &strutil::split, py::arg("s"), py::arg("delim") = ' ');
}
```

```python
# test_string_utils.py
import string_utils
import pytest

class TestToUpper:
    def test_basic(self):
        assert string_utils.to_upper("hello") == "HELLO"

    def test_mixed(self):
        assert string_utils.to_upper("Hello World") == "HELLO WORLD"

    def test_empty(self):
        assert string_utils.to_upper("") == ""

    def test_already_upper(self):
        assert string_utils.to_upper("ABC") == "ABC"

class TestPalindrome:
    @pytest.mark.parametrize("s", [
        "racecar", "A man a plan a canal Panama",
        "Was it a car or a cat I saw", "madam",
    ])
    def test_true_palindromes(self, s):
        assert string_utils.is_palindrome(s)

    @pytest.mark.parametrize("s", ["hello", "python", "ab"])
    def test_false_palindromes(self, s):
        assert not string_utils.is_palindrome(s)

class TestSplit:
    def test_default_space(self):
        assert string_utils.split("a b c") == ["a", "b", "c"]

    def test_custom_delimiter(self):
        assert string_utils.split("a,b,c", ',') == ["a", "b", "c"]

    def test_empty_string(self):
        assert string_utils.split("") == []
```

The parametrized palindrome tests are a good example of the sweet spot for example-based testing: a handful of known-correct cases that document the expected behavior clearly.

### Q2: Use Python's hypothesis library for property-based tests of the exposed C++ function

**Answer:**

Property-based testing with Hypothesis works like this: you describe an invariant that should always hold, and Hypothesis generates random inputs to try to break it. The key is identifying the right properties. For string utilities, good properties are things like idempotence (`to_lower(to_lower(s)) == to_lower(s)`), structural invariants (length is preserved), and round-trips (`split(join(parts)) == parts`).

The reason this is especially powerful against C++ code is that Hypothesis will find inputs your hand-written tests wouldn't cover - unusual Unicode characters, empty strings, very long strings, strings that are all the same byte. When it finds a failure, it shrinks the input to the minimal reproducing case automatically.

```python
# test_hypothesis.py
# Property-based testing: Hypothesis generates random inputs
# and checks that invariants always hold

import string_utils
from hypothesis import given, assume, settings, example
from hypothesis import strategies as st

# Property: to_upper(to_lower(s)) == to_upper(s)
@given(st.text(alphabet=st.characters(whitelist_categories=('L', 'N', 'Z'))))
def test_upper_lower_roundtrip(s):
    """to_upper is idempotent after to_lower."""
    assert string_utils.to_upper(string_utils.to_lower(s)) == string_utils.to_upper(s)


# Property: to_lower is idempotent
@given(st.text())
def test_lower_idempotent(s):
    result = string_utils.to_lower(s)
    assert string_utils.to_lower(result) == result


# Property: palindrome reversal
@given(st.text(alphabet=st.characters(whitelist_categories=('L',)), min_size=1))
def test_palindrome_of_mirror(s):
    """s + reverse(s) is always a palindrome."""
    mirrored = s + s[::-1]
    assert string_utils.is_palindrome(mirrored)


# Property: split then join roundtrip
@given(st.lists(st.text(
    alphabet=st.characters(blacklist_characters=','),
    min_size=1
), min_size=1))
def test_split_join_roundtrip(parts):
    """split(join(parts, ','), ',') == parts"""
    joined = ",".join(parts)
    result = string_utils.split(joined, ',')
    assert result == parts


# Property: to_upper preserves length
@given(st.text())
def test_upper_preserves_length(s):
    assert len(string_utils.to_upper(s)) == len(s)


# Targeted example: Hypothesis can shrink failing inputs
@given(st.text(min_size=0, max_size=1000))
@example("")             # Explicit edge case
@example("a")            # Single char
@example("\x00\xff")     # Null and high bytes
@settings(max_examples=500)
def test_lower_upper_length_always_preserved(s):
    assert len(string_utils.to_lower(s)) == len(s)
    assert len(string_utils.to_upper(s)) == len(s)
```

The `@example` decorator is useful for pinning known edge cases alongside the property. Even if Hypothesis never generates `"\x00\xff"` on its own, it will always test that specific case because you declared it explicitly.

### Q3: Explain the round-trip cost of pybind11 calls and when it makes test latency acceptable

**Answer:**

Understanding call overhead helps you decide when to restructure your bindings. The benchmark below measures the per-call cost of pybind11 dispatch versus a pure Python equivalent, which gives you a concrete sense of what you're paying.

```python
# Benchmark: measuring pybind11 overhead
import string_utils
import time

def benchmark_call_overhead():
    """Measure per-call overhead of pybind11 dispatch."""
    N = 100_000
    s = "hello"

    # pybind11 calls
    start = time.perf_counter_ns()
    for _ in range(N):
        string_utils.to_upper(s)
    elapsed_cpp = (time.perf_counter_ns() - start) / N

    # Pure Python equivalent
    start = time.perf_counter_ns()
    for _ in range(N):
        s.upper()
    elapsed_py = (time.perf_counter_ns() - start) / N

    print(f"pybind11 to_upper: {elapsed_cpp:.0f} ns/call")
    print(f"Python   .upper(): {elapsed_py:.0f} ns/call")
    print(f"Overhead:          {elapsed_cpp - elapsed_py:.0f} ns/call")

# Typical results:
#   pybind11 to_upper: ~250 ns/call
#   Python   .upper(): ~80 ns/call
#   Overhead:          ~170 ns/call
```

The overhead is fixed per call, not per byte. That makes the math straightforward: for most testing scenarios the overhead is completely negligible. The only case where it becomes a problem is when you make a very large number of tiny individual calls.

| Scenario | Calls/Test | Overhead | Verdict |
| --- | :---: | :---: | :---: |
| Unit test: one function call | 1 | ~200 ns | Negligible |
| Parametrized: 100 inputs | 100 | ~20 us | Negligible |
| Hypothesis: 500 examples | 500 | ~100 us | Fine |
| Stress test: 1M calls | 1M | ~200 ms | Noticeable |
| Data pipeline: large buffers | 1 | ~1 us (buffer copy) | Amortized |

The fix for the "many tiny calls" case is batching: instead of calling a C++ function once per element, expose a C++ function that accepts a whole vector and loops internally. One call with N elements instead of N calls with one element each.

```python
# BAD: many tiny calls
# for x in big_list:
#     result.append(string_utils.to_upper(x))  # 200ns overhead * N

# GOOD: batch call
# results = string_utils.to_upper_batch(big_list)  # One call, C++ loops internally
```

---

## Notes

- Hypothesis shrinks failing inputs to minimal reproducing cases - invaluable for finding edge cases in C++ parsers
- `pip install hypothesis` - no C++ changes needed
- For NumPy buffers, use `pybind11/numpy.h` for zero-copy access (no conversion overhead)
- Run Hypothesis with `--hypothesis-show-statistics` to see how many examples were generated
- Combine with sanitizers: build the C++ module with `-fsanitize=address` and Hypothesis will find memory bugs
