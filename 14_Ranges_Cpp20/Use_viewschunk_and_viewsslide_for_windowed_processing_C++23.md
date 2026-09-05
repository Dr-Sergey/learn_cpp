# Use views::chunk and views::slide for windowed processing (C++23)

**Category:** Ranges Cpp20  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/chunk_view>  

---

## Topic Overview

`views::chunk(n)` and `views::slide(n)` partition a range into sub-ranges of size `n`, but differ in whether windows **overlap**. This is the single most important thing to keep straight between these two: `chunk` partitions (no element appears twice), while `slide` creates a sliding window (most elements appear in multiple windows).

### chunk vs slide Visual Comparison

Here is a side-by-side picture to make the difference concrete:

```cpp
Source:  [1] [2] [3] [4] [5] [6] [7] [8] [9]

chunk(3) - non-overlapping:
  [1,2,3]  [4,5,6]  [7,8,9]

slide(3) - overlapping (sliding window):
  [1,2,3]  [2,3,4]  [3,4,5]  [4,5,6]  [5,6,7]  [6,7,8]  [7,8,9]

chunk(3) on 10 elements - last chunk may be smaller:
  [1,2,3]  [4,5,6]  [7,8,9]  [10]

slide(3) on 10 elements - always exactly size 3:
  [1,2,3]  [2,3,4]  ...  [8,9,10]    (8 windows)
```

### Summary Table

If the table feels like a lot, just remember: `chunk` = pagination, `slide` = moving average.

| Property | `chunk(n)` | `slide(n)` |
| --- | --- | --- |
| Overlap | None | n-1 elements shared |
| Window count | ceil(size/n) | size - n + 1 |
| Last window size | May be < n | Always n |
| Required | `input_range` | `forward_range` |
| Result element | Sub-range / take_view | Sub-range of original |

### Related: `chunk_by` (C++23)

`views::chunk_by(pred)` splits whenever `pred(prev, curr)` returns `false` - for grouping by a condition rather than fixed size. This is the tool you reach for when you want to group runs of equal (or otherwise related) adjacent elements.

---

## Self-Assessment

### Q1: Split a range into fixed-size chunks using `views::chunk` and process each chunk

Here is the basic usage pattern. Notice how the last chunk can be smaller than `n` when the range doesn't divide evenly.

```cpp
#include <iostream>
#include <numeric>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Split into chunks of 3
    std::cout << "Chunks of 3:\n";
    for (auto chunk : data | std::views::chunk(3)) {
        std::cout << "  [";
        bool first = true;
        for (int x : chunk) {
            if (!first) std::cout << ", ";
            std::cout << x;
            first = false;
        }
        std::cout << "]\n";
    }

    // Process each chunk: compute the sum of each chunk
    std::cout << "Chunk sums: ";
    for (auto chunk : data | std::views::chunk(4)) {
        int sum = 0;
        for (int x : chunk) sum += x;
        std::cout << sum << ' ';
    }
    std::cout << '\n';

    // Batch processing: simulate sending data in packets of 3
    int batch = 1;
    for (auto packet : data | std::views::chunk(3)) {
        std::cout << "Batch " << batch++ << ": ";
        for (int x : packet) std::cout << x << ' ';
        std::cout << '\n';
    }
}
// Expected output:
// Chunks of 3:
//   [1, 2, 3]
//   [4, 5, 6]
//   [7, 8, 9]
//   [10]
// Chunk sums: 10 26 19
// Batch 1: 1 2 3
// Batch 2: 4 5 6
// Batch 3: 7 8 9
// Batch 4: 10
```

The final chunk `[10]` is shorter than 3 - that is expected and intentional behaviour. Each chunk is itself a range, so you can iterate it, sum it, pass it to an algorithm, or forward it as a "packet" to another system.

- `views::chunk(3)` yields sub-ranges of at most 3 elements each.
- The last chunk contains only 1 element (10) since 10 is not divisible by 3.
- Each chunk is itself a range that can be iterated or passed to algorithms.
- Common use: batch processing, pagination, or splitting data for parallel processing.

### Q2: Use `views::slide` to compute a moving average over a range

`slide` is the go-to for signal processing style work. Each window overlaps with the previous one by `n-1` elements, so you can compute things like running averages or detect trends between adjacent values.

```cpp
#include <iostream>
#include <numeric>
#include <ranges>
#include <vector>

int main() {
    std::vector<double> prices = {100.0, 102.5, 101.0, 105.0, 103.5, 107.0, 106.0};

    constexpr int window = 3;

    // Compute 3-day moving average
    std::cout << "Prices:          ";
    for (double p : prices) std::cout << p << ' ';
    std::cout << '\n';

    std::cout << "Moving avg (" << window << "): ";
    for (auto win : prices | std::views::slide(window)) {
        double sum = 0.0;
        int count = 0;
        for (double x : win) { sum += x; ++count; }
        std::cout << (sum / count) << ' ';
    }
    std::cout << '\n';

    // Detect consecutive increases using slide(2)
    std::cout << "Consecutive increases: ";
    for (auto pair : prices | std::views::slide(2)) {
        auto it = std::ranges::begin(pair);
        double a = *it++;
        double b = *it;
        if (b > a) std::cout << a << "->" << b << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// Prices:          100 102.5 101 105 103.5 107 106
// Moving avg (3): 101.167 102.833 103.167 105.167 105.5
// Consecutive increases: 100->102.5 101->105 103.5->107
```

With 7 prices and a window of 3, you get 5 averages (7 - 3 + 1 = 5). The `slide(2)` trick for adjacent pairs is especially handy - you're essentially zipping the range with a shifted copy of itself, but without actually making copies.

- `views::slide(3)` produces 5 overlapping windows of exactly 3 elements each.
- Each window is a sub-range into the original vector (no copies).
- The moving average divides the sum of each window by the window size.
- `slide(2)` gives adjacent pairs - useful for detecting trends, computing differences, etc.

### Q3: Explain the difference between chunk (non-overlapping) and slide (overlapping windows)

| Aspect | `chunk(n)` | `slide(n)` |
| --- | --- | --- |
| **Windows** | Non-overlapping partitions | Overlapping with stride 1 |
| **Element sharing** | Each element in exactly 1 chunk | Each element in up to `n` windows |
| **Window count** (for `N` elements) | ceil(N/n) | N - n + 1 |
| **Last window** | May have fewer than `n` elements | Always exactly `n` elements |
| **Use case** | Batch processing, pagination | Moving average, trend detection |
| **Minimum requirement** | `input_range` | `forward_range` |

The reason `slide` needs `forward_range` while `chunk` only needs `input_range` is fundamental: `chunk` visits each element exactly once (it can discard elements as it goes), while `slide` needs to revisit elements that belong to the next overlapping window, which requires being able to re-iterate part of the range.

Here is a concrete demonstration side by side:

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6};

    std::cout << "chunk(3) - non-overlapping:\n";
    for (auto c : data | std::views::chunk(3)) {
        std::cout << "  ";
        for (int x : c) std::cout << x << ' ';
        std::cout << '\n';
    }

    std::cout << "slide(3) - overlapping:\n";
    for (auto w : data | std::views::slide(3)) {
        std::cout << "  ";
        for (int x : w) std::cout << x << ' ';
        std::cout << '\n';
    }

    // chunk_by: split on condition changes
    std::vector<int> vals = {1, 1, 2, 2, 2, 3, 1, 1};
    std::cout << "chunk_by(equal):\n";
    for (auto group : vals | std::views::chunk_by(std::equal_to{})) {
        std::cout << "  ";
        for (int x : group) std::cout << x << ' ';
        std::cout << '\n';
    }
}
// Expected output:
// chunk(3) - non-overlapping:
//   1 2 3
//   4 5 6
// slide(3) - overlapping:
//   1 2 3
//   2 3 4
//   3 4 5
//   4 5 6
// chunk_by(equal):
//   1 1
//   2 2 2
//   3
//   1 1
```

`chunk` partitions the data (each element belongs to exactly one chunk), while `slide` creates overlapping windows (elements near the middle appear in `n` different windows). `chunk_by` is condition-based rather than fixed-size - it starts a new group every time consecutive elements stop satisfying the predicate.

---

## Notes

- `slide` requires `forward_range` because it needs to iterate the same elements in multiple windows. `chunk` works with `input_range` since each element is visited once.
- `chunk(1)` wraps each element in a single-element sub-range. `slide(1)` does the same.
- `chunk(n)` where `n >= size` produces a single chunk containing all elements.
- Both are **lazy** - sub-ranges are views into the original data.
- Also see `views::stride(n)` for taking every nth element (complementary to chunk).
