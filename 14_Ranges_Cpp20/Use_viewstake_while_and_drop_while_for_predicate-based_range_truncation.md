# Use views::take_while and drop_while for predicate-based range truncation

**Category:** Ranges Cpp20  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/take_while_view>  

---

## Topic Overview

`views::take_while(pred)` yields elements from the front **while** `pred` is true, stopping at the first `false`. `views::drop_while(pred)` skips elements **while** `pred` is true, yielding everything after. Think of them as the predicate-based cousins of `take(n)` and `drop(n)`, except instead of counting elements you're watching for a condition to flip.

### Visual Comparison

The diagram below is worth staring at for a moment. Notice that the two views produce non-overlapping halves of the original range - `take_while` takes the prefix, `drop_while` takes everything from the boundary onward (the boundary element itself is included in `drop_while`'s output).

```cpp
Source:       [1] [2] [3] [7] [4] [5] [2]
                           ^
                           first element where (x < 5) is false

take_while(x < 5):  [1] [2] [3]              <- stops BEFORE 7
drop_while(x < 5):              [7] [4] [5] [2]  <- starts FROM 7

Note: take_while + drop_while = entire range (non-overlapping split)
```

### Key Properties

| View | Stops when | Includes the boundary element? |
| --- | --- | --- |
| `take_while(pred)` | `pred` returns `false` | No - the first failing element is excluded |
| `drop_while(pred)` | `pred` returns `false` | Yes - the first failing element starts the output |

### Comparison with take/drop (count-based)

| Count-based | Predicate-based |
| --- | --- |
| `take(5)` - first 5 elements | `take_while(pred)` - elements while pred is true |
| `drop(3)` - skip first 3 | `drop_while(pred)` - skip while pred is true |

---

## Self-Assessment

### Q1: Use `take_while` to read elements until a sentinel condition is met

`take_while` is particularly useful when you have data that signals its own end with a special value - like HTTP headers terminated by an empty line, or a C-style null-terminated pattern encoded in a container.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    // Simulate reading lines until an empty line
    std::vector<std::string> lines = {
        "Header: Content-Type",
        "Header: Accept",
        "Header: Auth",
        "",                      // empty line = end of headers
        "Body content here",
        "More body",
    };

    // Take lines while they are non-empty
    auto headers = lines | std::views::take_while(
        [](const std::string& line) { return !line.empty(); });

    std::cout << "Headers:\n";
    for (const auto& h : headers)
        std::cout << "  " << h << '\n';

    // Numeric example: take sorted prefix
    std::vector<int> data = {1, 3, 5, 7, 2, 4, 6};
    auto sorted_prefix = data | std::views::take_while(
        [prev = 0](int x) mutable {
            bool ascending = (x >= prev);
            prev = x;
            return ascending;
        });

    std::cout << "Sorted prefix: ";
    for (int x : sorted_prefix)
        std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Headers:
//   Header: Content-Type
//   Header: Accept
//   Header: Auth
// Sorted prefix: 1 3 5 7
```

The sorted-prefix example uses a **mutable lambda** with a captured `prev` variable to track state across calls. The reason this trips people up is that lambdas are `const` by default, so you can't modify captures without the `mutable` keyword. Here it's essential because the predicate needs to remember the last element it saw. `take_while` is lazy throughout - it evaluates the predicate one element at a time and stops the moment it returns `false`.

### Q2: Use `drop_while` to skip leading whitespace tokens in a tokenized range

`drop_while` is the mirror image: it keeps skipping as long as the predicate holds, then includes everything from the first non-matching element onward - even elements that would have matched the predicate.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <string_view>
#include <vector>

int main() {
    // Skip leading whitespace-only tokens
    std::vector<std::string> tokens = {
        "  ", "\t", "", "Hello", "World", " ", "End"
    };

    auto is_whitespace = [](const std::string& s) {
        return s.empty() || s.find_first_not_of(" \t\n") == std::string::npos;
    };

    auto content = tokens | std::views::drop_while(is_whitespace);

    std::cout << "After skipping whitespace tokens: ";
    for (const auto& t : content)
        std::cout << '[' << t << ']' << ' ';
    std::cout << '\n';

    // Numeric: skip leading zeros
    std::vector<int> digits = {0, 0, 0, 4, 2, 0, 7};
    auto significant = digits | std::views::drop_while(
        [](int d) { return d == 0; });

    std::cout << "After dropping leading zeros: ";
    for (int d : significant)
        std::cout << d;
    std::cout << '\n';
}
// Expected output:
// After skipping whitespace tokens: [Hello] [World] [ ] [End]
// After dropping leading zeros: 4207
```

Notice that the single-space token `" "` appears in the output even though it would match `is_whitespace`. That's expected: `drop_while` evaluates the predicate only on the leading run of elements. Once it finds the first non-matching element (`"Hello"`), it stops evaluating and includes everything from that point forward unconditionally.

### Q3: Combine take_while and drop_while to extract a middle segment matching a pattern

You can compose these two views to extract a contiguous run of matching elements from the middle of a sequence. The idea: first skip everything before the run with `drop_while`, then take the run with `take_while`.

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 10, 11, 12, 13, 4, 5, 6};
    //                                ^^^^^^^^^^^^^^^^
    //                                values >= 10 (the "middle segment")

    // Extract the run of values >= 10
    auto middle = data
        | std::views::drop_while([](int n) { return n < 10; })   // skip before
        | std::views::take_while([](int n) { return n >= 10; });  // take until end

    std::cout << "Middle segment (>= 10): ";
    for (int x : middle)
        std::cout << x << ' ';
    std::cout << '\n';

    // General pattern: extract between two markers
    std::vector<std::string> log = {
        "info: startup",
        "info: ready",
        "ERROR: disk full",
        "ERROR: retry failed",
        "ERROR: giving up",
        "info: shutdown",
    };

    auto errors = log
        | std::views::drop_while([](const std::string& s) {
              return s.substr(0, 5) != "ERROR";
          })
        | std::views::take_while([](const std::string& s) {
              return s.substr(0, 5) == "ERROR";
          });

    std::cout << "Error block:\n";
    for (const auto& line : errors)
        std::cout << "  " << line << '\n';

    // Equivalence: drop_while + take_while partitions into 3 segments
    // [before predicate] [matching segment] [after predicate]
    std::cout << "\nBefore: ";
    for (int x : data | std::views::take_while([](int n) { return n < 10; }))
        std::cout << x << ' ';
    std::cout << "\nMiddle: ";
    for (int x : middle)
        std::cout << x << ' ';
    std::cout << "\nAfter:  ";
    auto after = data
        | std::views::drop_while([](int n) { return n < 10; })
        | std::views::drop_while([](int n) { return n >= 10; });
    for (int x : after)
        std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Middle segment (>= 10): 10 11 12 13
// Error block:
//   ERROR: disk full
//   ERROR: retry failed
//   ERROR: giving up
//
// Before: 1 2 3
// Middle: 10 11 12 13
// After:  4 5 6
```

The three-segment breakdown at the bottom is a nice way to see the complete picture. `take_while(x < 10)` gives you the prefix, `drop_while(x < 10) | take_while(x >= 10)` gives you the middle run, and two chained `drop_while` calls give you the suffix. Together they account for every element exactly once.

---

## Notes

- `take_while` + `drop_while` are complementary: together they split a range exactly where the predicate first fails.
- `drop_while` caches `begin()` after the first call (O(N) first time, O(1) subsequently).
- For extracting **all** matching elements (not just a contiguous prefix), use `views::filter` instead.
- `take_while` and `drop_while` work with infinite ranges: `iota(0) | take_while(x < 100)` is finite.
