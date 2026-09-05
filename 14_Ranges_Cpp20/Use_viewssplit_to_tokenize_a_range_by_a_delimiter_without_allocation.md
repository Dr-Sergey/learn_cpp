# Use views::split to tokenize a range by a delimiter without allocation

**Category:** Ranges Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/split_view>  

---

## Topic Overview

`views::split` tokenizes a range by a delimiter, producing **subranges** (views) into the original data - **zero heap allocation**. This is a meaningful upgrade over traditional approaches like `std::stringstream` or manual `find`/`substr` loops, both of which allocate a fresh `std::string` for each token they produce.

### Why Zero-Allocation Matters

It's worth being concrete about what "zero allocation" means here. Traditional splitting copies characters; `views::split` does not.

```cpp
Traditional: "a,b,c" -> stringstream -> getline -> creates 3 std::string objects (3 allocations)

views::split: "a,b,c" -> 3 subranges pointing into original string (0 allocations)
```

The tokens are **views** (iterator pairs) into the source. No characters are copied, and the original string is never modified.

### Converting Tokens

Since tokens come back as subranges rather than strings, you'll usually need to convert them to something useful. Here are the common options:

| Target type | How to convert |
| --- | --- |
| `std::string_view` | `std::string_view(token.begin(), token.end())` |
| `std::string` | `std::string(token.begin(), token.end())` |
| `int` / `double` | `std::from_chars(...)` on the subrange |
| Keep as view | Just use the subrange directly |

### Working with Non-String Ranges

`views::split` works on **any forward range**, not just strings. You can split a vector of integers on a sentinel value just as easily as splitting a string on a comma.

```cpp
vector<int> data = {1, 0, 2, 3, 0, 4};
auto groups = data | views::split(0);
// [[1], [2,3], [4]]
```

---

## Self-Assessment

### Q1: Split a string by ',' using `views::split` and collect substrings without `std::string` allocation

This example shows the three most common patterns you'll reach for: viewing tokens as `string_view`, searching through them, and parsing integers - all without a single heap allocation.

```cpp
#include <iostream>
#include <ranges>
#include <string_view>

int main() {
    std::string_view input = "apple,banana,cherry,date";

    // Split by comma - zero allocation
    std::cout << "Tokens (as string_view, no allocation):\n";
    for (auto token : input | std::views::split(',')) {
        // Construct string_view from the subrange - no heap allocation
        std::string_view sv(token.begin(), token.end());
        std::cout << "  [" << sv << "] length=" << sv.size() << '\n';
    }

    // Use tokens directly in comparisons - no string needed
    std::cout << "\nSearching for 'banana': ";
    for (auto token : input | std::views::split(',')) {
        std::string_view sv(token.begin(), token.end());
        if (sv == "banana") {
            std::cout << "found!\n";
            break;
        }
    }

    // Parse integers without allocation
    std::string_view nums = "10,20,30,40,50";
    int sum = 0;
    for (auto token : nums | std::views::split(',')) {
        std::string_view sv(token.begin(), token.end());
        int val = 0;
        std::from_chars(sv.data(), sv.data() + sv.size(), val);
        sum += val;
    }
    std::cout << "Sum of parsed ints: " << sum << '\n';
}
// Expected output:
// Tokens (as string_view, no allocation):
//   [apple] length=5
//   [banana] length=6
//   [cherry] length=6
//   [date] length=4
//
// Searching for 'banana': found!
// Sum of parsed ints: 150
```

The `std::from_chars` call at the bottom is particularly nice: it takes a pointer and a length and parses an integer directly from the source memory, never needing an intermediate `std::string`. The whole split-and-parse pipeline genuinely allocates nothing on the heap.

### Q2: Explain why `views::split` returns subranges of the original range, not copies

The design is deliberate, and it follows from the core philosophy of the ranges library:

1. **Views are non-owning by design.** The ranges library philosophy is that views never own data - they provide different "windows" into existing data.

2. **O(1) construction per token.** Each token is just two iterators (begin and end within the source range). Creating a `std::string` copy would be O(N) per token.

3. **Composability.** Subranges can be piped into further views without materialization. The following example chains two levels of splitting with no intermediate allocations at all.

```cpp
#include <algorithm>
#include <iostream>
#include <ranges>
#include <string_view>

int main() {
    std::string_view csv = "Alice:95,Bob:87,Charlie:92";

    // Split, then for each token, split again by ':'
    for (auto record : csv | std::views::split(',')) {
        std::string_view rec(record.begin(), record.end());

        // Second-level split on the token itself - still zero allocation
        auto fields = rec | std::views::split(':');
        auto it = fields.begin();

        std::string_view name((*it).begin(), (*it).end());
        ++it;
        std::string_view score_sv((*it).begin(), (*it).end());

        std::cout << name << " scored " << score_sv << '\n';
    }
}
// Expected output:
// Alice scored 95
// Bob scored 87
// Charlie scored 92
```

Because each token from the outer split is already a subrange into the original `string_view`, you can apply another `split` to it directly. Every level of splitting operates on views into the original string - no `std::string` objects are ever created anywhere in this pipeline.

### Q3: Compare `views::split` with `std::stringstream` tokenization for performance

The table below lays out the differences concisely. The key column is "Heap allocations" - that's where `stringstream` pays a cost that `views::split` avoids entirely.

| Aspect | `views::split` | `std::stringstream` + `getline` |
| --- | --- | --- |
| **Heap allocations** | 0 (subranges into source) | 1 per token (`std::string`) |
| **Copies** | 0 | Full copy of each token |
| **Lazy** | Yes - tokens found on demand | No - `getline` reads immediately |
| **Works with string_view** | Yes | No - requires `std::string` input |
| **Multi-char delimiter** | Yes - `split("::"sv)` | No - `getline` takes single char |
| **Composable** | Yes - pipes into other views | No - must extract to variables |

The code below shows all three approaches: `stringstream`, `views::split`, and a composable pipeline that `stringstream` simply cannot express.

```cpp
#include <iostream>
#include <ranges>
#include <sstream>
#include <string>
#include <string_view>

int main() {
    std::string input = "one,two,three,four,five";

    // Approach 1: stringstream (allocates per token)
    std::cout << "stringstream: ";
    std::istringstream ss(input);
    std::string token;
    while (std::getline(ss, token, ',')) {
        std::cout << token << ' ';  // 'token' is a heap-allocated std::string
    }
    std::cout << '\n';

    // Approach 2: views::split (zero allocation)
    std::cout << "views::split: ";
    for (auto part : std::string_view(input) | std::views::split(',')) {
        std::string_view sv(part.begin(), part.end());
        std::cout << sv << ' ';  // 'sv' is a zero-cost view
    }
    std::cout << '\n';

    // Approach 3: Composable pipeline (impossible with stringstream)
    auto long_tokens = std::string_view(input)
        | std::views::split(',')
        | std::views::filter([](auto token) {
              return std::ranges::distance(token) > 3;
          });

    std::cout << "Long tokens:  ";
    for (auto part : long_tokens) {
        std::string_view sv(part.begin(), part.end());
        std::cout << sv << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// stringstream: one two three four five
// views::split: one two three four five
// Long tokens:  three four five
```

The third approach - filtering tokens by length - is the one that really shows the composability advantage. With `stringstream` you'd need a manual loop with an `if` statement; with `views::split` you just add `| views::filter(...)` to the pipeline. For tokenizing strings in performance-sensitive code, `views::split` is typically faster because it avoids `std::string` allocation per token. The difference grows with the number of tokens and the frequency of calls.

---

## Notes

- When splitting a `std::string` (not `string_view`), ensure the string outlives the split view - the view holds references.
- `views::split` works on any `forward_range`, not just strings: `vector<int>{1,0,2,0,3} | views::split(0)` yields `[[1],[2],[3]]`.
- For regex-based splitting, `views::split` is not sufficient - use `<regex>` or a third-party library.
- C++23 `ranges::to<string>()` makes materializing tokens easier: `token | ranges::to<string>()`.
