# Use std::merge and std::inplace_merge for sorted sequence merging

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/merge>  

---

## Topic Overview

### std::merge

Merges two **sorted** ranges into a single sorted output range. Requires a separate output buffer.

```cpp
std::merge(first1, last1, first2, last2, dest);
std::merge(first1, last1, first2, last2, dest, comp);
```

- **Complexity:** O(n + m) - exactly `n + m - 1` comparisons at most.
- **Stability:** If equal elements exist in both ranges, elements from the first range come first.
- **Output:** Must not overlap with either input range.

### std::inplace_merge

Merges two consecutive **sorted** sub-ranges within a single container, in place. The key thing to notice is the three-iterator signature: `[first, middle)` and `[middle, last)` are the two sorted halves that live back-to-back in the same container.

```cpp
std::inplace_merge(first, middle, last);
std::inplace_merge(first, middle, last, comp);
```

- `[first, middle)` and `[middle, last)` must both be sorted.
- **Complexity:** O(n) with O(n) extra memory. Falls back to O(n log n) with O(1) extra memory.

### Comparison

| Feature | `merge` | `inplace_merge` |
| --- | --- | --- |
| Output | Separate buffer | In-place |
| Memory | O(n+m) for output | O(n) or O(1) fallback |
| Complexity | O(n+m) always | O(n) or O(n log n) |
| Use case | Merging separate containers | Bottom-up merge sort |

---

## Self-Assessment

### Q1: Use std::merge to combine two sorted vectors into a new sorted vector

`std::merge` is the straightforward choice when your two sorted sequences are in separate containers and you want to produce a third. The stability guarantee means that if the same value appears in both input ranges, the one from the first range arrives first in the output - which matters for event logs, records, and any data where ties need a defined tie-breaking order.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <iterator>
#include <string>

int main() {
    // === Basic merge of two sorted vectors ===
    std::vector<int> a = {1, 3, 5, 7, 9};
    std::vector<int> b = {2, 4, 6, 8, 10};

    std::vector<int> result;
    std::merge(a.begin(), a.end(),
               b.begin(), b.end(),
               std::back_inserter(result));

    std::cout << "Merged: ";
    for (int x : result) std::cout << x << " ";
    std::cout << "\n";
    // Merged: 1 2 3 4 5 6 7 8 9 10

    // === Merge with overlapping values (stability) ===
    std::vector<int> x = {1, 3, 5, 5};
    std::vector<int> y = {2, 5, 5, 8};

    std::vector<int> merged;
    std::merge(x.begin(), x.end(), y.begin(), y.end(), std::back_inserter(merged));
    std::cout << "With duplicates: ";
    for (int v : merged) std::cout << v << " ";
    std::cout << "\n";
    // 1 2 3 5 5 5 5 8  (stable: x's 5s come before y's 5s)

    // === Merge with pre-allocated output ===
    std::vector<int> c = {10, 30, 50};
    std::vector<int> d = {20, 40, 60};
    std::vector<int> out(c.size() + d.size());
    std::merge(c.begin(), c.end(), d.begin(), d.end(), out.begin());
    std::cout << "Pre-allocated: ";
    for (int v : out) std::cout << v << " ";
    std::cout << "\n";
    // 10 20 30 40 50 60

    // === Merge with custom comparator (descending) ===
    std::vector<int> da = {9, 7, 5, 3};
    std::vector<int> db = {8, 6, 4, 2};
    std::vector<int> desc_merged;
    std::merge(da.begin(), da.end(), db.begin(), db.end(),
               std::back_inserter(desc_merged), std::greater<>{});
    std::cout << "Descending merge: ";
    for (int v : desc_merged) std::cout << v << " ";
    std::cout << "\n";
    // 9 8 7 6 5 4 3 2

    // === Practical: merge sorted event streams ===
    struct Event {
        int timestamp;
        std::string source;
    };
    auto cmp = [](const Event& a, const Event& b) {
        return a.timestamp < b.timestamp;
    };

    std::vector<Event> log1 = {{100, "server"}, {300, "server"}, {500, "server"}};
    std::vector<Event> log2 = {{200, "client"}, {400, "client"}};
    std::vector<Event> combined;
    std::merge(log1.begin(), log1.end(), log2.begin(), log2.end(),
               std::back_inserter(combined), cmp);

    std::cout << "Merged log:\n";
    for (const auto& e : combined)
        std::cout << "  t=" << e.timestamp << " src=" << e.source << "\n";

    return 0;
}
```

The merged-log example at the end is a typical real-world use: two sorted event streams (one from a server, one from a client) combined into a single timeline without copying either source unnecessarily.

### Q2: Use inplace_merge to merge two sorted halves of a single vector in place

`inplace_merge` shines when you already have two sorted halves living in the same container and want to combine them without allocating a separate output buffer. The bottom-up merge sort implementation here is a classic application - it repeatedly doubles the "sorted window" size using `inplace_merge` until the entire array is sorted.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>

int main() {
    // === Two sorted halves in one vector ===
    std::vector<int> v = {1, 4, 7, 10,    // first sorted half
                          2, 3, 8, 9};     // second sorted half

    auto middle = v.begin() + 4;  // partition point

    std::cout << "Before inplace_merge: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 1 4 7 10 2 3 8 9

    std::inplace_merge(v.begin(), middle, v.end());

    std::cout << "After inplace_merge:  ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 7 8 9 10

    // === Bottom-up merge sort implementation ===
    auto merge_sort = [](std::vector<int>& arr) {
        size_t n = arr.size();
        for (size_t width = 1; width < n; width *= 2) {
            for (size_t i = 0; i < n; i += 2 * width) {
                auto left = arr.begin() + i;
                auto mid = arr.begin() + std::min(i + width, n);
                auto right = arr.begin() + std::min(i + 2 * width, n);
                std::inplace_merge(left, mid, right);
            }
        }
    };

    std::vector<int> unsorted = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    merge_sort(unsorted);

    std::cout << "Merge sorted: ";
    for (int x : unsorted) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5 6 7 8 9

    // === Incremental sorted insert pattern ===
    // Keep data sorted, append new batch, sort new batch, inplace_merge
    std::vector<int> sorted_data = {1, 3, 5, 7, 9};
    std::vector<int> new_batch = {2, 6, 4, 8};

    size_t old_size = sorted_data.size();
    sorted_data.insert(sorted_data.end(), new_batch.begin(), new_batch.end());
    std::sort(sorted_data.begin() + old_size, sorted_data.end());  // sort new part
    std::inplace_merge(sorted_data.begin(),
                       sorted_data.begin() + old_size,
                       sorted_data.end());  // merge

    std::cout << "After batch merge: ";
    for (int x : sorted_data) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5 6 7 8 9

    return 0;
}
```

The incremental batch-insert pattern at the end is useful when you receive data in chunks: sort each new chunk as it arrives, then merge it into the already-sorted prefix with a single `inplace_merge` call.

### Q3: Show that inplace_merge is O(n log n) with O(1) extra memory when no buffer is available

The complexity of `inplace_merge` depends on whether the implementation can allocate a temporary buffer. This is one of the rare places in the standard library where the standard explicitly specifies two different complexity guarantees based on memory availability. Understanding the fallback helps you predict performance in constrained environments.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>
#include <numeric>

int main() {
    // === inplace_merge complexity depends on available memory ===
    //
    // When the implementation can allocate a temporary buffer of size n:
    //   - O(n) comparisons, O(n) moves — optimal
    //
    // When no extra memory is available (allocation fails or constrained):
    //   - Falls back to an in-place algorithm
    //   - O(n log n) comparisons — slower but still correct
    //   - O(1) extra memory
    //
    // The standard guarantees:
    //   "If there is enough memory, O(n) comparisons.
    //    Otherwise, O(n log n) comparisons."

    // === Demonstrate inplace_merge correctness at different sizes ===
    auto test_inplace_merge = [](size_t n) {
        std::vector<int> v(2 * n);
        std::iota(v.begin(), v.begin() + n, 0);             // [0, 2, 4, ...]
        std::iota(v.begin() + n, v.end(), 0);               // [1, 3, 5, ...]
        for (size_t i = 0; i < n; ++i) v[i] = v[i] * 2;
        for (size_t i = n; i < 2*n; ++i) v[i] = v[i-n] * 2 + 1;

        std::inplace_merge(v.begin(), v.begin() + n, v.end());

        return std::is_sorted(v.begin(), v.end());
    };

    for (size_t n : {10, 100, 1000, 10000, 100000}) {
        bool ok = test_inplace_merge(n);
        std::cout << "n=" << n << ": " << (ok ? "correct" : "FAILED") << "\n";
    }

    // === Timing comparison: merge (with buffer) vs inplace_merge ===
    constexpr size_t N = 1'000'000;
    std::vector<int> half1(N), half2(N);
    std::iota(half1.begin(), half1.end(), 0);
    std::iota(half2.begin(), half2.end(), 0);
    for (auto& x : half1) x *= 2;       // even numbers
    for (auto& x : half2) x = x * 2 + 1; // odd numbers

    // std::merge (always O(n), needs output buffer)
    std::vector<int> output(2 * N);
    auto t1 = std::chrono::high_resolution_clock::now();
    std::merge(half1.begin(), half1.end(),
               half2.begin(), half2.end(), output.begin());
    auto t2 = std::chrono::high_resolution_clock::now();
    auto ms_merge = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    // std::inplace_merge (O(n) with buffer, O(n log n) without)
    std::vector<int> combined;
    combined.reserve(2 * N);
    combined.insert(combined.end(), half1.begin(), half1.end());
    combined.insert(combined.end(), half2.begin(), half2.end());

    t1 = std::chrono::high_resolution_clock::now();
    std::inplace_merge(combined.begin(), combined.begin() + N, combined.end());
    t2 = std::chrono::high_resolution_clock::now();
    auto ms_inplace = std::chrono::duration_cast<std::chrono::microseconds>(t2 - t1).count();

    std::cout << "\nmerge:          " << ms_merge << " us\n";
    std::cout << "inplace_merge:  " << ms_inplace << " us\n";
    std::cout << "Both correct: "
              << std::boolalpha << (output == combined) << "\n";

    return 0;
}
```

`inplace_merge` tries to allocate a temporary buffer. If it succeeds, you get O(n) performance. If not, it falls back to a rotation-based algorithm: find the merge point, rotate, then recursively merge two smaller subproblems. In practice the O(n) path is almost always taken on a typical desktop - the O(n log n) fallback is a safety net, not the common case.

---

## Notes

- **Both require sorted inputs.** Merging unsorted ranges is undefined behavior.
- **`std::merge` is stable:** Equal elements from the first range appear before those from the second.
- **`std::inplace_merge` is also stable.**
- **C++20 ranges:** `std::ranges::merge(a, b, out)` - cleaner syntax, supports projections.
- **For k-way merge:** Use a priority queue (min-heap) that pulls from k sorted sources. No standard algorithm for this.
- **`std::merge` with `std::istream_iterator`:** Can merge sorted files without loading them entirely into memory.
