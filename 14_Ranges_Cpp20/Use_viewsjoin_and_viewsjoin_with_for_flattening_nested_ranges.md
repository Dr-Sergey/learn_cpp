# Use views::join and views::join_with for flattening nested ranges

**Category:** Ranges Cpp20  
**Standard:** C++20, C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/join_view>  

---

## Topic Overview

`views::join` flattens a **range of ranges** into a single flat range. `views::join_with` (C++23) does the same but inserts a **delimiter** between each inner range.

If you have ever written nested loops to process a `vector<vector<int>>` or concatenated strings with a separator, these two views are the lazy, composable equivalent.

### join vs join_with

Here is a quick picture of what each does to the same input:

```cpp
Source: [[1,2], [3], [4,5,6]]

views::join        ->  [1, 2, 3, 4, 5, 6]
views::join_with(0) ->  [1, 2, 0, 3, 0, 4, 5, 6]
```

### Supported Input Types

| Source type | Example | Result of join |
| --- | --- | --- |
| `vector<vector<int>>` | `{{1,2},{3,4}}` | flat range of `int` |
| `vector<string>` | `{"hello", "world"}` | flat range of `char` |
| `range of string_view` | from `views::split` | flat range of `char` |
| `range of views` | `transform` returning views | flat range of inner element type |

### Iterator Category Downgrade

`join_view` **downgrades** the iterator category because the outer iterator and inner iterator advance independently. This is one of the trickier aspects of `join` to internalize.

| Outer range | Inner range | join_view category |
| --- | --- | --- |
| bidirectional | bidirectional + common | **bidirectional** |
| forward | forward | **forward** |
| input | any | **input** |
| random_access | random_access | at best **bidirectional** (not random_access!) |

The key insight is that `join` is **never** random-access, because the inner ranges have variable sizes - you can't compute `it + n` in O(1). This catches people by surprise: even if both the outer and inner ranges are `vector` (random-access), the joined result is only bidirectional.

---

## Self-Assessment

### Q1: Flatten a `vector<vector<int>>` into a single range using `views::join`

This is the most common use case. A 2D structure becomes a 1D stream of elements, lazily, with no intermediate flat vector allocated.

```cpp
#include <iostream>
#include <ranges>
#include <vector>
#include <functional>

int main() {
    std::vector<std::vector<int>> matrix = {
        {1, 2, 3},
        {4, 5},
        {6, 7, 8, 9},
    };

    // Flatten into a single range
    auto flat = matrix | std::views::join;

    std::cout << "Flat: ";
    for (int x : flat)
        std::cout << x << ' ';
    std::cout << '\n';

    // Works with range algorithms
    auto sum = std::ranges::fold_left(flat, 0, std::plus{});
    std::cout << "Sum: " << sum << '\n';

    // Count elements
    std::cout << "Total elements: " << std::ranges::distance(flat) << '\n';

    // Join + transform pipeline: square all elements in a matrix
    auto squares = matrix
        | std::views::join
        | std::views::transform([](int n) { return n * n; });

    std::cout << "Squares: ";
    for (int s : squares)
        std::cout << s << ' ';
    std::cout << '\n';
}
// Expected output:
// Flat: 1 2 3 4 5 6 7 8 9
// Sum: 45
// Total elements: 9
// Squares: 1 4 9 16 25 36 49 64 81
```

The flattened view composes naturally with further adaptors like `transform`. There's no intermediate flat container at any point - everything flows through lazily.

- `matrix` is a `vector<vector<int>>` - a range of ranges.
- `views::join` iterates through each inner vector in sequence, yielding all elements as one flat stream.
- No copies - the join view yields references to elements in the original vectors.
- The flattened view can be piped into further adaptors like `transform`.

### Q2: Use `views::join_with` to concatenate strings with a delimiter without materializing intermediates

`join_with` is the clean solution for the classic "join strings with a separator" problem. The characters flow out lazily - no temporary `string` is built until you explicitly ask for one.

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <string_view>
#include <vector>

int main() {
    std::vector<std::string> words = {"Hello", "World", "from", "C++23"};

    // Join with a single delimiter character
    auto joined_chars = words | std::views::join_with(' ');
    std::cout << "Joined: ";
    for (char c : joined_chars)
        std::cout << c;
    std::cout << '\n';

    // Join with a delimiter string
    auto csv = words | std::views::join_with(std::string_view(", "));
    std::cout << "CSV: ";
    for (char c : csv)
        std::cout << c;
    std::cout << '\n';

    // Practical: build a path from segments
    std::vector<std::string> parts = {"usr", "local", "bin"};
    auto path = parts | std::views::join_with('/');
    std::cout << "Path: /";
    for (char c : path)
        std::cout << c;
    std::cout << '\n';

    // Materialize to string (C++23)
    std::string path_str = "/" + (parts
        | std::views::join_with('/')
        | std::ranges::to<std::string>());
    std::cout << "Path string: " << path_str << '\n';
}
// Expected output:
// Joined: Hello World from C++23
// CSV: Hello, World, from, C++23
// Path: /usr/local/bin
// Path string: /usr/local/bin
```

When you need a proper `std::string` in the end, `ranges::to<std::string>()` materializes the lazy view in one shot. That is still just one allocation for the whole result, rather than one per word.

- `join_with(' ')` inserts a space character between each string's characters.
- `join_with(", ")` inserts the two-character sequence `", "` between strings.
- No intermediate `string` is allocated - the view lazily interleaves delimiters and characters.
- To get a `std::string` result, use `ranges::to<std::string>()` to materialize.

### Q3: Explain the category downgrade: a join of random-access ranges is at best bidirectional

This is the most conceptually important point about `join`. It trips people up because it feels like two random-access ranges joined together ought to be random-access. The reason that doesn't work comes down to arithmetic.

**Why `join_view` is never random-access:**

Random-access requires `it + n` in O(1). But with `join`, elements are spread across inner ranges of **variable sizes**:

```cpp
[[1,2,3], [4], [5,6]]   <- inner ranges have sizes 3, 1, 2

To find element at position 4 (which is '5'):

  - Must know: size(inner[0]) = 3, size(inner[1]) = 1
  - Position 4 is in inner[2] at offset 0
  - This requires scanning inner range sizes -> NOT O(1)
```

Even if every inner range had the same known size, the standard does not provide a way for `join_view` to look that information up in O(1) in the general case - so random-access is simply off the table.

**Proof via static_assert:**

```cpp
#include <concepts>
#include <iostream>
#include <ranges>
#include <vector>
#include <string_view>

int main() {
    std::vector<std::vector<int>> vv = {{1,2}, {3,4}};

    // Inner and outer are both random-access
    static_assert(std::ranges::random_access_range<std::vector<int>>);
    static_assert(std::ranges::random_access_range<decltype(vv)>);

    // But join_view is only bidirectional!
    auto joined = vv | std::views::join;
    static_assert(std::ranges::bidirectional_range<decltype(joined)>);
    static_assert(!std::ranges::random_access_range<decltype(joined)>);
    std::cout << "vector<vector<int>> | join = bidirectional (not random_access)\n";

    // With forward-only inner ranges, join is forward
    auto split_result = std::string_view("a,b,c") | std::views::split(',');
    auto flat = split_result | std::views::join;
    static_assert(std::ranges::forward_range<decltype(flat)>);
    static_assert(!std::ranges::bidirectional_range<decltype(flat)>);
    std::cout << "split | join = forward only\n";
}
// Expected output:
// vector<vector<int>> | join = bidirectional (not random_access)
// split | join = forward only
```

The `split | join` result being only forward (not bidirectional) is because `split_view`'s inner ranges are themselves only forward.

**Category downgrade summary:**

| Property | Why |
| --- | --- |
| Never random-access | Can't compute position across variable-size inner ranges in O(1) |
| At best bidirectional | When both outer and inner are bidirectional + common, `--it` can move backward across boundaries |
| Forward | Default when all ranges are at least forward |
| Input | When outer range is input_only (single-pass) |

---

## Notes

- `join_view` caches `*outer_it` (the current inner range) internally, which is why `join` requires `forward_range` or stores the result for `input_range`.
- For joining with a delimiter pre-C++23: use a manual loop or `transform` + `join` with delimiter ranges interleaved.
- `join` on an empty outer range or on a range of empty inner ranges produces an empty range.
- `views::join` is the inverse of `views::chunk`/`views::split` - it reassembles what they take apart.
