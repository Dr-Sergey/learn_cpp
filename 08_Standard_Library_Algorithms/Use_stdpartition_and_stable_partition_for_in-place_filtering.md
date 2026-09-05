# Use std::partition and stable_partition for in-place filtering

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/partition>  

---

## Topic Overview

`std::partition` rearranges elements so that all elements satisfying a predicate come before those that don't. It returns an iterator to the **partition point** - the first element that does *not* satisfy the predicate. The key thing to understand is that `partition` makes no promises about the order of elements within each group - it only guarantees the two-group split.

```cpp
Before partition (is_even):  [3, 8, 2, 5, 7, 4, 1, 6]
After partition:             [6, 8, 2, 4 | 7, 5, 1, 3]
                                          ^ returned iterator (partition point)
```

### partition vs stable_partition

If you need the order within each group preserved, you want `std::stable_partition`. The trade-off is extra memory and potentially more work:

| Feature | `partition` | `stable_partition` |
| --- | --- | --- |
| Preserves relative order | No | Yes, within each group |
| Time complexity | O(n) | O(n) with extra memory, O(n log n) without |
| Space complexity | O(1) | O(n) or O(1) fallback |
| Swap count | <= n/2 | More swaps (rotation-based) |

### Related Functions

The `<algorithm>` header gives you a whole family of partition-related tools:

```cpp
#include <algorithm>
partition(first, last, pred);          // rearrange, return partition point
stable_partition(first, last, pred);   // order-preserving
partition_point(first, last, pred);    // find split on already-partitioned range
is_partitioned(first, last, pred);     // check if already partitioned
partition_copy(first, last, out_true, out_false, pred);  // copy to two outputs
```

---

## Self-Assessment

### Q1: Use std::partition to separate odd and even numbers and explain the iterator returned

The returned iterator is the dividing line between the two groups - everything before it satisfies the predicate, everything from it to the end does not. Watch how we use it to address each group independently:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {3, 8, 2, 5, 7, 4, 1, 6};

    std::cout << "Before: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // Partition: even numbers first
    auto it = std::partition(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });

    std::cout << "After:  ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // e.g. 6 8 2 4 7 5 1 3  (order within groups NOT preserved)

    // The returned iterator points to the first element that does NOT
    // satisfy the predicate - the partition point
    std::cout << "\nEven numbers: ";
    for (auto p = v.begin(); p != it; ++p) std::cout << *p << " ";
    std::cout << "\n";

    std::cout << "Odd numbers:  ";
    for (auto p = it; p != v.end(); ++p) std::cout << *p << " ";
    std::cout << "\n";

    std::cout << "Count of evens: " << std::distance(v.begin(), it) << "\n";  // 4

    // Verify: is_partitioned
    bool ok = std::is_partitioned(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });
    std::cout << "Is partitioned: " << std::boolalpha << ok << "\n";  // true

    return 0;
}
```

The `[v.begin(), it)` range holds all the evens and `[it, v.end())` holds all the odds - that split is exactly what `partition` guarantees, even though the elements within each group are in an unspecified order.

### Q2: Show that stable_partition preserves relative order within each partition

If the order within each group matters - for example, seniority order within a team - `stable_partition` is what you want. The extra cost is worth it when correctness depends on relative ordering:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

struct Employee {
    std::string name;
    int seniority;
    bool is_senior() const { return seniority >= 5; }
};

int main() {
    std::vector<Employee> team = {
        {"Alice", 8}, {"Bob", 2}, {"Carol", 6},
        {"Dave", 1}, {"Eve", 10}, {"Frank", 3}
    };

    auto print = [](const std::vector<Employee>& v, const char* label) {
        std::cout << label << ": ";
        for (auto& e : v)
            std::cout << e.name << "(" << e.seniority << ") ";
        std::cout << "\n";
    };

    // === partition (unstable) ===
    auto v1 = team;  // copy
    std::partition(v1.begin(), v1.end(), [](const Employee& e) {
        return e.is_senior();
    });
    print(v1, "partition      ");
    // Order within senior/junior groups is NOT guaranteed

    // === stable_partition ===
    auto v2 = team;  // copy
    auto mid = std::stable_partition(v2.begin(), v2.end(), [](const Employee& e) {
        return e.is_senior();
    });
    print(v2, "stable_partition");
    // Senior:  Alice(8) Carol(6) Eve(10)  - original order preserved!
    // Junior:  Bob(2) Dave(1) Frank(3)    - original order preserved!

    std::cout << "\nSenior team (" << std::distance(v2.begin(), mid) << "):\n";
    for (auto it = v2.begin(); it != mid; ++it)
        std::cout << "  " << it->name << " (seniority " << it->seniority << ")\n";

    // === Complexity notes ===
    // stable_partition with enough memory: O(n) time, O(n) space
    // stable_partition without memory (fallback): O(n log n) time, O(1) space
    // The implementation tries to allocate a buffer and falls back gracefully.

    return 0;
}
```

Notice that `stable_partition` tries to allocate a temporary buffer. If it succeeds, you get O(n) time. If the allocation fails, it falls back to an O(n log n) rotation-based approach. Either way, relative order is preserved - you just pay more without the buffer.

### Q3: Apply partition as a building block for a quicksort implementation

`std::partition` is the heart of quicksort - the core step of any quicksort is partitioning elements around a pivot. This example also shows a three-way partition for the Dutch National Flag problem:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <iterator>

// Quicksort using std::partition as the core operation
template <typename It>
void quicksort(It first, It last) {
    if (std::distance(first, last) <= 1) return;

    // Choose pivot: median-of-three
    auto mid = first + std::distance(first, last) / 2;
    auto prev_last = std::prev(last);
    if (*mid < *first) std::iter_swap(first, mid);
    if (*prev_last < *first) std::iter_swap(first, prev_last);
    if (*prev_last < *mid) std::iter_swap(mid, prev_last);
    auto pivot_val = *mid;

    // Partition into [< pivot | >= pivot]
    auto partition_point = std::partition(first, last, [&](const auto& elem) {
        return elem < pivot_val;
    });

    // Recurse on both halves
    quicksort(first, partition_point);
    quicksort(partition_point, last);
}

int main() {
    std::vector<int> data = {38, 27, 43, 3, 9, 82, 10};

    std::cout << "Before: ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";

    quicksort(data.begin(), data.end());

    std::cout << "After:  ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";
    // After: 3 9 10 27 38 43 82

    // === Three-way partition for Dutch National Flag ===
    std::vector<int> colors = {2, 0, 1, 2, 0, 1, 0, 2, 1};
    // partition twice: 0s first, then 1s
    auto ones_start = std::partition(colors.begin(), colors.end(),
        [](int c) { return c == 0; });
    std::partition(ones_start, colors.end(),
        [](int c) { return c == 1; });

    std::cout << "DNF:    ";
    for (int x : colors) std::cout << x << " ";
    std::cout << "\n";
    // DNF: 0 0 0 1 1 1 2 2 2

    return 0;
}
```

The Dutch National Flag trick is worth internalizing: you run `partition` twice on successively narrowing sub-ranges. The first call separates 0s from everything else; the second call separates 1s from 2s within the remainder. This pattern generalizes to any multi-way split.

---

## Notes

- **`partition_copy`** is useful when you don't want to modify the original - it writes matching and non-matching elements to separate output iterators.
- **`is_partitioned`** checks a precondition; **`partition_point`** performs binary search on a partitioned range (O(log n) if random-access).
- `std::partition` requires only `ForwardIterator`, but bidirectional iterators enable the efficient swap-from-both-ends algorithm.
- For `stable_partition`, if maintaining order matters (e.g., database-style filtering), always prefer it over `partition` + manual resorting.
