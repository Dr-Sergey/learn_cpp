# Use views::split for tokenizing ranges lazily

**Category:** Ranges Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/split_view>  

---

## Topic Overview

C++20 provides two split views: `views::split` (eager, forward subranges) and `views::lazy_split` (truly lazy, input-compatible). The history here is a bit bumpy - C++20 initially shipped only one `split` that turned out to be surprisingly hard to use in practice. The paper P2210R2 then renamed the original to `lazy_split` and introduced a more ergonomic `split` to replace it. If you see confusing behavior in early C++20 toolchains, that's almost certainly why.

### split vs lazy_split

The table below lays out the essential differences. If it feels like a lot, the short version is: use `split` for strings and string views, use `lazy_split` only when you're dealing with a single-pass input source like a stream.

| Property | `views::split` | `views::lazy_split` |
| --- | --- | --- |
| Requires | `forward_range` | `input_range` |
| Inner range type | `subrange` (contiguous for contiguous source) | Custom inner range (forward at best) |
| Converting to string_view | Easy: construct from subrange | Difficult: must iterate char by char |
| Use when | Working with strings, string_views | Working with input streams |

### How split Works

The diagram below shows what `split` actually produces. Each token you get back is not a copy of the characters - it's a window (a subrange) into the original string, pointing at exactly the right slice.

```cpp
Source:    "one,two,,four"
Delimiter: ','

split(",") produces:
  ["one"]  ["two"]  [""]  ["four"]
   ^^^^    ^^^^    ^^    ^^^^^^
   subrange into original string
```

Each token is a **subrange** (view) into the original string - no allocation.

### Delimiter Options

You have a few choices for how to express the delimiter:

- Single character: `split(',')`
- String pattern: `split(", "sv)` (splits on the two-character sequence, not each character individually)
- Single-element range: works with any forward range as delimiter

---

## Self-Assessment

### Q1: Split a `string_view` by delimiter using `views::split` and collect the parts

Here's a walkthrough of the most common usage patterns: splitting on a single character, collecting into a vector, and splitting on a multi-character delimiter.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <string_view>
#include <vector>

int main() {
    std::string_view csv = "Alice,Bob,Charlie,Diana";

    // Split by comma
    std::cout << "Parts: ";
    for (auto part : csv | std::views::split(',')) {
        // Convert subrange to string_view
        std::string_view token(part.begin(), part.end());
        std::cout << '"' << token << "\" ";
    }
    std::cout << '\n';

    // Collect into a vector<string>
    std::vector<std::string> names;
    for (auto part : csv | std::views::split(',')) {
        names.emplace_back(part.begin(), part.end());
    }

    std::cout << "Collected " << names.size() << " names: ";
    for (const auto& n : names) std::cout << n << ' ';
    std::cout << '\n';

    // Split by multi-character delimiter
    std::string_view data = "one::two::three";
    std::cout << "Split by '::': ";
    for (auto part : data | std::views::split(std::string_view("::"))) {
        std::string_view token(part.begin(), part.end());
        std::cout << '"' << token << "\" ";
    }
    std::cout << '\n';
}
// Expected output:
// Parts: "Alice" "Bob" "Charlie" "Diana"
// Collected 4 names: Alice Bob Charlie Diana
// Split by '::': "one" "two" "three"
```

Notice that constructing a `string_view` from `part.begin()` and `part.end()` is a zero-cost operation - those iterators already point into the original `string_view`, so no characters are ever copied. The multi-character delimiter `"::"` matches the whole two-character sequence, not each colon individually.

### Q2: Explain when `lazy_split` is preferred over `split`

**`lazy_split` works with `input_range`** (single-pass), while `split` requires `forward_range`. In most real-world code you want `split`. `lazy_split` is specifically for sources you can only read once - like a network stream or an `std::istream`.

| Use case | Which to use | Why |
| --- | --- | --- |
| Splitting `string` / `string_view` | `split` | Produces usable subranges, easy to convert to string_view |
| Splitting input from `std::istream` | `lazy_split` | Input iterators are single-pass; `split` won't compile |
| Splitting a generator/coroutine output | `lazy_split` | Generators are typically input_only |
| Performance-critical string parsing | `split` | Subranges are contiguous; faster conversion to string_view |

The code below shows both approaches side by side. Notice how `lazy_split` forces you to loop over individual characters to reconstruct a word, while `split` lets you grab the token as a `string_view` in one step.

```cpp
#include <iostream>
#include <ranges>
#include <sstream>
#include <string>

int main() {
    // lazy_split works with input streams (input_range)
    std::istringstream stream("Hello World Test");
    auto words = std::ranges::istream_view<char>(stream)
        | std::views::lazy_split(' ');

    // Note: lazy_split inner ranges are harder to use
    // Each inner range must be iterated char by char
    for (auto word : words) {
        for (char c : word)
            std::cout << c;
        std::cout << ' ';
    }
    std::cout << '\n';

    // split is much easier for string_view:
    std::string_view text = "Hello World Test";
    for (auto word : text | std::views::split(' ')) {
        std::string_view sv(word.begin(), word.end());
        std::cout << sv << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// Hello World Test
// Hello World Test
```

Both produce the same output, but the `split` version is much cleaner. The `lazy_split` version requires iterating character by character because its inner range type doesn't guarantee contiguous storage.

### Q3: Show the difference in behavior when the delimiter appears at the start or end

This is the kind of edge case that bites people. The behavior is consistent and well-defined, but it can surprise you if you haven't seen it before. The key rule: every delimiter creates a boundary, and boundaries always produce two sides - even if one side is empty.

```cpp
#include <iostream>
#include <ranges>
#include <string_view>

void show_split(std::string_view input, char delim) {
    std::cout << "\"" << input << "\" split by '" << delim << "': ";
    int count = 0;
    for (auto part : input | std::views::split(delim)) {
        std::string_view token(part.begin(), part.end());
        std::cout << '[' << token << ']';
        ++count;
    }
    std::cout << " (" << count << " parts)\n";
}

int main() {
    show_split("a,b,c", ',');     // normal
    show_split(",a,b", ',');      // delimiter at start
    show_split("a,b,", ',');      // delimiter at end
    show_split(",", ',');         // only delimiter
    show_split("", ',');          // empty string
    show_split(",,", ',');        // consecutive delimiters
    show_split("abc", ',');       // no delimiter found
}
// Expected output:
// "a,b,c" split by ',': [a][b][c] (3 parts)
// ",a,b" split by ',': [][a][b] (3 parts)
// "a,b," split by ',': [a][b][] (3 parts)
// "," split by ',': [][] (2 parts)
// "" split by ',': [] (1 parts)
// ",," split by ',': [][][] (3 parts)
// "abc" split by ',': [abc] (1 parts)
```

The key observations are:

- **Delimiter at start** produces an **empty first token**: `",a,b"` -> `["", "a", "b"]`.
- **Delimiter at end** produces an **empty last token**: `"a,b,"` -> `["a", "b", ""]`.
- **Empty string** produces **one empty token** (the whole empty range).
- **Consecutive delimiters** produce **empty tokens** between them: `",,"` -> `["", "", ""]`.
- This matches the behavior of Python's `str.split(',')` (not `str.split()` which collapses whitespace).

---

## Notes

- `views::split` was significantly improved by P2210R2. Pre-P2210 implementations (early C++20) had different, harder-to-use semantics.
- `split` with a `string_view` delimiter matches the **exact** sequence, not individual characters.
- For splitting on any of several delimiters (like `strtok`), use `views::split` per delimiter or a custom predicate-based approach.
- `views::split` is lazy - tokens are found on demand during iteration, not all at once upfront.
