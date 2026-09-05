# Use views::flat_map (transform + join) to flatmap ranges without intermediates

**Category:** Ranges Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/join_view>  

---

## Topic Overview

C++ doesn't have a dedicated `flat_map` view (until C++23's `join_with` improvements), but the composition of **`transform` + `join`** achieves the same effect as `flatMap` in other languages.

If you have ever used `flatMap` in Java streams, Rust iterators, or Haskell's `concatMap`, you already know what this does: apply a function that returns a collection to each element, then stitch all those collections together into one flat sequence. In C++ you build that from two simpler pieces.

### What is flatMap

flatMap applies a function that returns a range to each element, then **flattens** all results into a single range:

```cpp
flatMap(f, [a, b, c])  =  concat(f(a), f(b), f(c))

Example: flatMap(repeat_n, [1, 2, 3])
  f(1) = [1]
  f(2) = [2, 2]
  f(3) = [3, 3, 3]
  result = [1, 2, 2, 3, 3, 3]
```

### C++ Equivalent: transform | join

In C++20 you express the same idea as a two-stage pipeline:

```cpp
auto result = source
    | views::transform(f)   // range of ranges
    | views::join;           // flatten to single range
```

`transform` produces a range-of-ranges (each element of `source` maps to a sub-range), and `join` then flattens that one level down into a single stream.

### Comparison with Other Languages

| Language | Syntax |
| --- | --- |
| Haskell | `concatMap f xs` or `xs >>= f` |
| Rust | `iter.flat_map(f)` |
| Java | `stream.flatMap(f)` |
| C++ | `src \| views::transform(f) \| views::join` |

### Laziness

Both `transform` and `join` are **lazy**: no intermediate range-of-ranges is materialized. Elements flow through the pipeline one at a time. You will see this proved directly in Q3.

---

## Self-Assessment

### Q1: Expand each word in a list to its characters using transform then join into a flat character range

A `vector<string>` is already a range of ranges (each `string` is a range of `char`), so `join` alone is enough to flatten it. The `transform` step comes into play when you want to map each element to a different range first - in this case, an uppercase version.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    std::vector<std::string> words = {"Hello", "World", "C++"};

    // Each string IS a range of chars. transform is unnecessary here -
    // views::join directly flattens a range of ranges:
    auto all_chars = words | std::views::join;

    std::cout << "Joined: ";
    for (char c : all_chars)
        std::cout << c;
    std::cout << '\n';

    // With transform: expand each word to its uppercase version
    auto upper_chars = words
        | std::views::transform([](const std::string& w) {
              return w | std::views::transform([](char c) {
                  return static_cast<char>(std::toupper(static_cast<unsigned char>(c)));
              });
          })
        | std::views::join;  // flatten range-of-ranges into flat char range

    std::cout << "Upper:  ";
    for (char c : upper_chars)
        std::cout << c;
    std::cout << '\n';
}
// Expected output:
// Joined: HelloWorldC++
// Upper:  HELLOWORLDC++
```

The outer `transform` maps each word to a lazy uppercase view of its characters. The result is a range-of-lazy-ranges-of-char. Then `join` flattens that entire structure into a single stream of characters.

- `words` is a `vector<string>` - a range of ranges (each `string` is a range of `char`).
- `views::join` flattens it into a single `char` range: `H, e, l, l, o, W, o, r, l, d, ...`.
- The `transform` example maps each word to an uppercase view of its characters, producing a range-of-ranges-of-char, which `join` then flattens.

### Q2: Show that the combination of transform and join is the C++ equivalent of flatMap

Here is the textbook flatMap scenario: a function that maps each number to a repeated sequence of that number. The transform produces `[[1], [2,2], [3,3,3], [4,4,4,4]]` and join stitches it all together.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

// "Expand" function: maps each int to a range of ints
auto repeat_n(int n) {
    return std::views::repeat(n) | std::views::take(n);
}

int main() {
    std::vector<int> data = {1, 2, 3, 4};

    // flatMap: transform each element to a range, then join all ranges
    auto flat_mapped = data
        | std::views::transform([](int n) { return repeat_n(n); })
        | std::views::join;

    // 1 -> [1], 2 -> [2,2], 3 -> [3,3,3], 4 -> [4,4,4,4]
    // joined: [1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
    std::cout << "flatMap result: ";
    for (int x : flat_mapped)
        std::cout << x << ' ';
    std::cout << '\n';

    // Another example: split paths into components
    std::vector<std::string> paths = {"src/main.cpp", "include/lib.h"};

    auto all_segments = paths
        | std::views::transform([](const std::string& p) {
              return p | std::views::split('/');
          })
        | std::views::join;  // flatten: range of split_views -> single range of segments

    std::cout << "Path segments: ";
    for (auto segment : all_segments) {
        for (char c : segment) std::cout << c;
        std::cout << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// flatMap result: 1 2 2 3 3 3 4 4 4 4
// Path segments: src main.cpp include lib.h
```

The path example shows a real-world use: splitting each path string by `/` gives a range of `split_view`s, which join then flattens into one range of path segments. No intermediate `vector<vector<string>>` is built anywhere.

- `transform(repeat_n)` maps `[1,2,3,4]` to `[[1], [2,2], [3,3,3], [4,4,4,4]]` (a range of ranges).
- `join` flattens this into `[1, 2, 2, 3, 3, 3, 4, 4, 4, 4]`.
- This is exactly what `flatMap` does in Java/Rust/Haskell: apply a function that returns a collection, then concatenate all results.
- The path example demonstrates real-world use: splitting each path by `/` produces a range of split_views, which `join` flattens.

### Q3: Measure the laziness: no characters are produced until the outer range is iterated

This example is worth studying carefully because it makes the demand-driven nature visible. We instrument the `transform` to print when it is called, so you can see that the pipeline does nothing until you actually start pulling values out of it.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    int transform_calls = 0;

    std::vector<std::string> words = {"Hello", "World", "Test"};

    auto pipeline = words
        | std::views::transform([&](const std::string& w) {
              ++transform_calls;
              std::cout << "  transform called for \"" << w << "\"\n";
              return w | std::views::transform([](char c) {
                  return static_cast<char>(std::toupper(static_cast<unsigned char>(c)));
              });
          })
        | std::views::join;

    std::cout << "Pipeline created. transform_calls = " << transform_calls << '\n';
    std::cout << "\nStarting iteration:\n";

    int char_count = 0;
    for (char c : pipeline) {
        ++char_count;
        if (char_count > 7) break;  // stop early
    }

    std::cout << "\nProduced " << char_count << " chars.\n";
    std::cout << "transform_calls = " << transform_calls << '\n';
}
// Expected output:
// Pipeline created. transform_calls = 0
//
// Starting iteration:
//   transform called for "Hello"
//   transform called for "World"
//
// Produced 7 chars.
// transform_calls = 2
```

The reason `transform` was called only twice (not three times) is that `join` only asks the outer transform for the next inner range when it has exhausted the current one. We stopped after 7 characters - "HELLOW" (5 from "Hello") plus "O" (the first two characters of "WORLD") - so `join` needed the transforms for "Hello" and "World" but never asked for "Test".

- After creating the pipeline, `transform_calls` is **0** - no computation occurred.
- During iteration, `transform` is called lazily as the `join` view moves from one inner range to the next.
- We stopped after 7 characters ("HELLOW" + "O"), so `transform` was called only twice (for "Hello" and "World"), never for "Test".
- This proves the entire pipeline is demand-driven: only the work needed to produce requested elements is performed.

---

## Notes

- `join_view` caches `begin()` for the inner range, so calling `begin()` twice on the outer view doesn't restart the inner range.
- The `transform | join` pattern is the idiomatic C++20 flatMap. A dedicated `views::flat_map` is not in the standard.
- `join` on a range of prvalue ranges (e.g., `transform` returning `std::string`) requires the inner range to be materializable - this works because `join_view` stores the inner range.
- `views::join_with(delim)` (C++23) inserts a delimiter between joined inner ranges - useful for building strings.
