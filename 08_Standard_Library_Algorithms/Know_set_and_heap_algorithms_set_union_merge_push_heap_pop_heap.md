# Know set and heap algorithms: set_union, merge, push_heap, pop_heap

**Category:** Standard Library Algorithms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm>  

---

## Topic Overview

The standard library gives you two distinct families of algorithms that go beyond basic sorting and searching. One family treats ranges as mathematical sets; the other lets you manage a heap structure directly. Both are O(n + m) or better - much faster than naive approaches - but they come with firm preconditions you need to respect.

### Set Algorithms (require sorted input)

| Algorithm | Description | Complexity |
| --- | --- | --- |
| `set_union` | Elements in either range | O(n + m) |
| `set_intersection` | Elements in both ranges | O(n + m) |
| `set_difference` | Elements in first but not second | O(n + m) |
| `set_symmetric_difference` | Elements in one but not both | O(n + m) |
| `merge` | Merge two sorted ranges into one sorted output | O(n + m) |
| `inplace_merge` | Merge two consecutive sorted subranges | O(n log n) worst, O(n) with extra memory |
| `includes` | Check if one sorted range contains another | O(n + m) |

All set algorithms require **sorted** inputs and produce **sorted** outputs. If you hand them unsorted data, the results are wrong - not just undefined behavior, but silently wrong values.

### Heap Algorithms

| Algorithm | Description | Complexity |
| --- | --- | --- |
| `make_heap` | Arrange range into max-heap | O(n) |
| `push_heap` | Add element at end to heap | O(log n) |
| `pop_heap` | Move max to end, re-heapify rest | O(log n) |
| `sort_heap` | Sort a heap (destroys heap property) | O(n log n) |
| `is_heap` | Check if range is a heap | O(n) |
| `is_heap_until` | Find first element violating heap property | O(n) |

A heap is a tree stored in a contiguous array where every parent is greater than or equal to its children (max-heap). The largest element always lives at index 0.

### Core Illustration

Here is a quick visual showing the difference between set operations and merge, and what a max-heap looks like in an array:

```cpp
Set operations on sorted ranges:
  A = {1, 3, 5, 7}
  B = {2, 3, 5, 8}

  set_union:                 {1, 2, 3, 5, 7, 8}
  set_intersection:          {3, 5}
  set_difference(A,B):       {1, 7}
  set_symmetric_difference:  {1, 2, 7, 8}
  merge:                     {1, 2, 3, 3, 5, 5, 7, 8}  (keeps duplicates)

Heap (max-heap in array):
  [9, 7, 8, 3, 5, 2, 6]
       9
      / \
     7   8
    / \ / \
   3  5 2  6
```

Notice that `merge` keeps every element from both ranges, including duplicates, while `set_union` behaves like a mathematical union and deduplicates.

---

## Self-Assessment

### Q1: Use std::set_intersection on two sorted vectors and explain the output requirements

The key discipline here is paying attention to the output side of these algorithms. They don't allocate anything for you - you supply the destination, and you choose whether to use a `back_inserter` (grows automatically) or a pre-sized vector (which requires trimming afterward).

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <iterator>

int main() {
    // Both inputs MUST be sorted
    std::vector<int> a = {1, 2, 3, 4, 5, 6, 7, 8};
    std::vector<int> b = {2, 4, 6, 8, 10, 12};

    // === set_intersection: elements in BOTH ranges ===
    std::vector<int> intersection;
    std::set_intersection(a.begin(), a.end(),
                          b.begin(), b.end(),
                          std::back_inserter(intersection));

    std::cout << "Intersection: ";
    for (int x : intersection) std::cout << x << " ";
    std::cout << "\n";
    // Output: 2 4 6 8

    // === set_union: elements in EITHER range ===
    std::vector<int> united;
    std::set_union(a.begin(), a.end(),
                   b.begin(), b.end(),
                   std::back_inserter(united));

    std::cout << "Union: ";
    for (int x : united) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 2 3 4 5 6 7 8 10 12

    // === set_difference: in A but NOT in B ===
    std::vector<int> diff;
    std::set_difference(a.begin(), a.end(),
                        b.begin(), b.end(),
                        std::back_inserter(diff));

    std::cout << "A - B: ";
    for (int x : diff) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 3 5 7

    // === set_symmetric_difference: in one but NOT both ===
    std::vector<int> sym_diff;
    std::set_symmetric_difference(a.begin(), a.end(),
                                   b.begin(), b.end(),
                                   std::back_inserter(sym_diff));

    std::cout << "Symmetric diff: ";
    for (int x : sym_diff) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 3 5 7 10 12

    // === Output requirements ===
    // 1. Output iterator must have enough space (use back_inserter or pre-sized vector)
    // 2. Output MUST NOT overlap with either input range
    // 3. Both inputs must be sorted with the SAME comparator
    // 4. All set algorithms return iterator past the last written element

    // Example: using pre-sized output
    std::vector<int> result(a.size() + b.size());  // max possible size
    auto end_it = std::set_union(a.begin(), a.end(),
                                  b.begin(), b.end(),
                                  result.begin());
    result.erase(end_it, result.end());  // trim excess
    std::cout << "Pre-sized union size: " << result.size() << "\n";  // 10

    // === With custom comparator ===
    std::vector<int> c = {8, 6, 4, 2};      // sorted descending
    std::vector<int> d = {12, 10, 8, 6, 4}; // sorted descending
    std::vector<int> desc_inter;
    std::set_intersection(c.begin(), c.end(),
                          d.begin(), d.end(),
                          std::back_inserter(desc_inter),
                          std::greater<>{});  // MUST match the sort order
    std::cout << "Descending intersection: ";
    for (int x : desc_inter) std::cout << x << " ";
    std::cout << "\n";
    // Output: 8 6 4

    return 0;
}
```

All set algorithms use a merge-like linear scan on two sorted ranges - O(n+m). The output iterator receives elements as they are produced; `back_inserter` handles growth automatically. If you use a pre-sized vector instead, the algorithm returns an iterator past the last written element, so you need to erase the remainder. One thing that trips people up: both inputs must be sorted with exactly the same comparator. If you sorted descending, pass `std::greater<>{}` as the last argument too.

### Q2: Build a heap with std::make_heap and simulate priority queue operations manually

Think of this as a manual `std::priority_queue`. The heap algorithms give you direct access to the underlying array, which matters in cases like the "decrease-key" operation that `priority_queue` doesn't expose.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

void print_heap(const std::vector<int>& h, const char* label) {
    std::cout << label << ": ";
    for (int x : h) std::cout << x << " ";
    std::cout << " (is_heap: " << std::is_heap(h.begin(), h.end()) << ")\n";
}

int main() {
    std::vector<int> h = {3, 1, 4, 1, 5, 9, 2, 6};

    // === make_heap: O(n) — rearranges into max-heap ===
    std::make_heap(h.begin(), h.end());
    print_heap(h, "After make_heap");
    // Output: 9 6 4 3 5 1 2 1  (max at front)

    // === "push": add element, then restore heap ===
    h.push_back(8);  // add at end (breaks heap temporarily)
    std::push_heap(h.begin(), h.end());  // sift up: O(log n)
    print_heap(h, "After push 8");
    // 9 is still at front (or 8 replaces if larger)

    h.push_back(10);
    std::push_heap(h.begin(), h.end());
    print_heap(h, "After push 10");
    // 10 is now at front

    // === "pop": remove max element ===
    std::pop_heap(h.begin(), h.end());  // moves max to back, re-heapifies: O(log n)
    int max_val = h.back();
    h.pop_back();
    std::cout << "Popped: " << max_val << "\n";  // 10
    print_heap(h, "After pop");

    // Pop next
    std::pop_heap(h.begin(), h.end());
    max_val = h.back();
    h.pop_back();
    std::cout << "Popped: " << max_val << "\n";  // 9

    // === sort_heap: extract all in sorted order ===
    std::sort_heap(h.begin(), h.end());
    std::cout << "After sort_heap: ";
    for (int x : h) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 1 2 3 4 5 6 8 (ascending)
    // Note: heap property is destroyed after sort_heap

    // === Min-heap with std::greater ===
    std::vector<int> minh = {3, 1, 4, 1, 5};
    std::make_heap(minh.begin(), minh.end(), std::greater<>{});
    std::cout << "\nMin-heap front: " << minh.front() << "\n";  // 1

    std::pop_heap(minh.begin(), minh.end(), std::greater<>{});
    int min_val = minh.back();
    minh.pop_back();
    std::cout << "Min popped: " << min_val << "\n";  // 1

    return 0;
}
```

The reason the pop step has two parts is subtle: `pop_heap` moves the maximum to the last position and re-heapifies the rest, but it intentionally leaves that element at the back rather than erasing it - that way the algorithm stays container-agnostic and you decide when to `pop_back`. For a min-heap, pass `std::greater<>{}` consistently to every heap function you call, not just `make_heap`.

- **`make_heap`** rearranges the range into a max-heap in O(n). Element at index 0 is the maximum.
- **`push_heap`** assumes the range `[first, last-1)` is a heap and the new element is at `last-1`. It sifts up in O(log n).
- **`pop_heap`** swaps the root (max) with the last element, then sifts down to restore the heap on `[first, last-1)`. The max is now at `last-1` - call `pop_back` to actually remove it.
- **`sort_heap`** repeatedly pops to produce a sorted range. This is essentially how heapsort works.

### Q3: Show a std::merge of two sorted files into a sorted output using istream_iterators

Here is where `std::merge` really shines: you can pipe two sorted input streams directly into a sorted output stream without loading either file fully into memory.

```cpp
#include <iostream>
#include <fstream>
#include <algorithm>
#include <iterator>
#include <vector>
#include <sstream>

int main() {
    // === Simulate two sorted "files" with stringstreams ===
    // (In real code, use std::ifstream)
    std::istringstream file1("1 3 5 7 9 11 13");
    std::istringstream file2("2 4 6 8 10 12");

    // === Merge using istream_iterators directly ===
    // No need to load entire files into memory!
    std::merge(
        std::istream_iterator<int>(file1), std::istream_iterator<int>(),
        std::istream_iterator<int>(file2), std::istream_iterator<int>(),
        std::ostream_iterator<int>(std::cout, " ")
    );
    std::cout << "\n";
    // Output: 1 2 3 4 5 6 7 8 9 10 11 12 13

    // === Merge into a vector ===
    std::istringstream f1("10 20 30 40");
    std::istringstream f2("15 25 35 45");

    std::vector<int> merged;
    std::merge(
        std::istream_iterator<int>(f1), std::istream_iterator<int>(),
        std::istream_iterator<int>(f2), std::istream_iterator<int>(),
        std::back_inserter(merged)
    );

    std::cout << "Merged vector: ";
    for (int x : merged) std::cout << x << " ";
    std::cout << "\n";
    // Output: 10 15 20 25 30 35 40 45

    // === inplace_merge: merge two sorted halves of a vector ===
    std::vector<int> v = {1, 3, 5, 7, 2, 4, 6, 8};
    //                     <-sorted->  <-sorted->
    auto mid = v.begin() + 4;
    std::inplace_merge(v.begin(), mid, v.end());

    std::cout << "Inplace merge: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 2 3 4 5 6 7 8

    // === Merge with custom comparator (descending) ===
    std::vector<int> desc1 = {9, 7, 5, 3};
    std::vector<int> desc2 = {8, 6, 4, 2};
    std::vector<int> desc_merged;
    std::merge(desc1.begin(), desc1.end(),
               desc2.begin(), desc2.end(),
               std::back_inserter(desc_merged),
               std::greater<>{});

    std::cout << "Descending merge: ";
    for (int x : desc_merged) std::cout << x << " ";
    std::cout << "\n";
    // Output: 9 8 7 6 5 4 3 2

    return 0;
}
```

The default-constructed `istream_iterator<int>()` acts as the end-of-stream sentinel - that is standard iterator idiom. This pattern processes files element by element, making it an excellent approach for merging large sorted files that do not fit in memory. `inplace_merge` merges two consecutive sorted subranges within a single container; it tries to allocate auxiliary memory for O(n) performance but falls back to O(n log n) if allocation fails.

`std::merge` keeps all duplicates from both inputs, which is the critical difference from `set_union`. If you have `{1,1,2}` and `{1,2,2}`, `merge` gives `{1,1,1,2,2,2}`, while `set_union` gives `{1,1,2,2}`.

---

## Notes

- **Set operations vs merge:** Set operations treat ranges as mathematical sets where duplicates have meaning. Merge treats them as sequences and keeps everything. For `{1,1,2}` and `{1,2,2}`: `set_union` = `{1,1,2,2}`, `merge` = `{1,1,1,2,2,2}`.
- **`includes(a, b)`** checks if sorted range `a` is a superset of sorted range `b`. O(n+m).
- **Heap vs `priority_queue`:** `priority_queue` is a wrapper around these heap algorithms. Use raw heap algorithms when you need access to all elements (for example, a decrease-key operation).
- **`is_heap_until`** returns an iterator to the first element that violates the heap property - useful for debugging.
- **C++20 ranges versions:** `std::ranges::merge`, `std::ranges::set_intersection`, etc., with projections support.
