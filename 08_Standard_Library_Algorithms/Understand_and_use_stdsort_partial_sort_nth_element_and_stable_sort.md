# Understand and use std::sort, partial_sort, nth_element, and stable_sort

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/sort>  

---

## Topic Overview

C++ gives you multiple sorting algorithms optimized for different use cases, and picking the right one can matter a lot for performance. The key insight is that full sorting is often more work than you actually need.

| Algorithm | What it does | Complexity | Stable? |
| --- | --- | --- | --- |
| `std::sort` | Fully sorts the range | O(n log n) guaranteed (introsort) | No |
| `std::stable_sort` | Fully sorts, preserving equal-element order | O(n log n) with memory, O(n log²n) without | **Yes** |
| `std::partial_sort` | Sorts only the first K elements | O(n log k) | No |
| `std::nth_element` | Puts the nth element in its sorted position | O(n) average | No |

### When to Use What

This decision tree covers the most common situations:

```cpp
Need full sorted order?
├── Yes --- Need stable? --- Yes -> stable_sort
│                          └── No  -> sort
├── Only top K elements? ---------> partial_sort
├── Only single element position? -> nth_element
└── Check if already sorted? -----> is_sorted
```

### How They Work Internally

Understanding the internals helps you set realistic performance expectations and explains why each algorithm has its particular tradeoffs.

- **`std::sort`**: Introsort = quicksort + heapsort fallback + insertion sort for small ranges. Quicksort is fast in practice but has O(n²) worst case; the heapsort fallback kicks in after too many recursive levels to ensure O(n log n) worst-case guaranteed.
- **`std::stable_sort`**: Merge sort. Needs O(n) extra memory; falls back to in-place O(n log²n) if allocation fails.
- **`std::partial_sort`**: Heapselect - builds a heap of the first K elements, then scans the rest to see if any element belongs in the top K.
- **`std::nth_element`**: Introselect - quickselect with a median-of-medians fallback to guarantee O(n) average.

---

## Self-Assessment

### Q1: Show when nth_element is preferable to full sort (finding the median in O(n))

The reason `nth_element` is so much faster for median-finding is that you only need one element in its final position - you do not care about the order of everything else. Full sort does a lot of unnecessary work ordering elements you will never look at.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>
#include <numeric>

int main() {
    constexpr int N = 1'000'000;

    // Generate random data
    std::vector<int> data(N);
    std::mt19937 rng(42);
    std::generate(data.begin(), data.end(), rng);

    // === Method 1: Full sort to find median — O(n log n) ===
    {
        auto v = data;  // copy
        auto t0 = std::chrono::high_resolution_clock::now();
        std::sort(v.begin(), v.end());
        int median = v[N / 2];
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
        std::cout << "Full sort:    median=" << median << "  time=" << ms << " ms\n";
    }

    // === Method 2: nth_element — O(n) average ===
    {
        auto v = data;  // copy
        auto t0 = std::chrono::high_resolution_clock::now();
        auto nth = v.begin() + N / 2;
        std::nth_element(v.begin(), nth, v.end());
        int median = *nth;
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0).count();
        std::cout << "nth_element:  median=" << median << "  time=" << ms << " ms\n";
    }
    // nth_element is typically 3-5x faster for finding the median

    // === Bonus: nth_element gives partition property ===
    std::vector<int> v = {9, 3, 7, 1, 5, 8, 2, 6, 4};
    auto nth = v.begin() + 4;  // position 4 = median of 9 elements
    std::nth_element(v.begin(), nth, v.end());

    std::cout << "\nMedian value: " << *nth << "\n";
    std::cout << "Array after nth_element: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // *nth = 5 (the value that would be at index 4 in sorted order)
    // Elements [0..3] are all <= 5
    // Elements [5..8] are all >= 5
    // Neither half is necessarily sorted!

    // === Use case: find the 3 smallest elements (unordered) ===
    v = {9, 3, 7, 1, 5, 8, 2, 6, 4};
    std::nth_element(v.begin(), v.begin() + 3, v.end());
    std::cout << "\n3 smallest (unordered): ";
    for (int i = 0; i < 3; ++i) std::cout << v[i] << " ";
    std::cout << "\n";
    // Some permutation of {1, 2, 3}

    return 0;
}
```

After `nth_element`, the element at position `nth` is exactly what it would be in a fully sorted array - everything before it is less than or equal, everything after is greater than or equal - but neither half is sorted. That partition property is often all you need: use `nth_element` for median, percentiles, top-K (unordered), and statistics.

### Q2: Explain why stable_sort preserves relative order and at what complexity cost

**Stability** means that elements comparing as equal retain their original relative ordering after sorting. The reason this matters in practice is multi-key sorting: sort by secondary key first, then use `stable_sort` by primary key, and the secondary order is preserved within each primary-key group.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

struct Employee {
    std::string name;
    std::string department;
    int hire_year;
};

int main() {
    std::vector<Employee> employees = {
        {"Alice",   "Eng",  2020},
        {"Bob",     "Eng",  2019},
        {"Charlie", "Sales", 2021},
        {"Diana",   "Eng",  2018},
        {"Eve",     "Sales", 2020},
        {"Frank",   "Sales", 2019},
    };

    // === First sort by hire_year ===
    std::ranges::sort(employees, {}, &Employee::hire_year);

    std::cout << "Sorted by year:\n";
    for (const auto& e : employees)
        std::cout << "  " << e.name << " " << e.department << " " << e.hire_year << "\n";

    // === Now stable_sort by department ===
    // Within each department, hire_year order is PRESERVED
    std::stable_sort(employees.begin(), employees.end(),
        [](const Employee& a, const Employee& b) {
            return a.department < b.department;
        });

    std::cout << "\nStable-sorted by department:\n";
    for (const auto& e : employees)
        std::cout << "  " << e.name << " " << e.department << " " << e.hire_year << "\n";
    // Eng:   Diana(2018), Bob(2019), Alice(2020)  — year order preserved!
    // Sales: Frank(2019), Eve(2020), Charlie(2021) — year order preserved!

    // === If we used std::sort instead: ===
    // Within "Eng" group, Diana/Bob/Alice might appear in ANY order
    // because sort is NOT stable — equal elements may swap

    // === Complexity comparison ===
    std::cout << "\n=== Complexity ===\n";
    std::cout << "sort:        O(n log n) — introsort, NOT stable\n";
    std::cout << "stable_sort: O(n log n) with O(n) extra memory (merge sort)\n";
    std::cout << "             O(n log²n) without extra memory\n";
    std::cout << "\nstable_sort is ~10-20% slower than sort due to merge overhead\n";
    std::cout << "and needs O(n) temporary memory for the merge buffers.\n";

    return 0;
}
```

The cost of stability is the merge-sort buffer: O(n) extra memory. If the allocator cannot provide it, the algorithm falls back to an in-place merge sort at O(n log²n) - still correct, just slower. For the multi-key sorting pattern shown above, `stable_sort` is exactly the right tool.

- **`std::sort`** uses introsort (hybrid quicksort). It may swap equal elements - not stable.
- **`std::stable_sort`** uses merge sort. Equal elements retain their original relative order - stable.
- **Use case:** Multi-key sorting. Sort by secondary key first, then `stable_sort` by primary key.

### Q3: Write a comparator for std::sort that sorts structs by multiple fields lexicographically

Multi-field sorting comparators are easy to get wrong because each field needs to fall through correctly when equal. Here are three approaches, from most explicit to most elegant.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
#include <tuple>

struct Student {
    std::string last_name;
    std::string first_name;
    double gpa;
    int age;
};

int main() {
    std::vector<Student> students = {
        {"Smith", "Alice", 3.8, 22},
        {"Jones", "Bob", 3.5, 21},
        {"Smith", "Charlie", 3.8, 20},
        {"Jones", "Alice", 3.9, 23},
        {"Smith", "Alice", 3.7, 22},
        {"Brown", "Eve", 3.8, 21},
    };

    // === Method 1: Manual comparator with if-else chains ===
    // Sort by last_name ASC, then first_name ASC, then GPA DESC, then age ASC
    std::sort(students.begin(), students.end(),
        [](const Student& a, const Student& b) {
            if (a.last_name != b.last_name) return a.last_name < b.last_name;
            if (a.first_name != b.first_name) return a.first_name < b.first_name;
            if (a.gpa != b.gpa) return a.gpa > b.gpa;  // DESC
            return a.age < b.age;
        });

    std::cout << "Method 1 (manual):\n";
    for (const auto& s : students)
        std::cout << "  " << s.last_name << ", " << s.first_name
                  << "  GPA=" << s.gpa << "  age=" << s.age << "\n";

    // === Method 2: std::tie for lexicographic comparison ===
    // Trick for DESC: negate the value (works for numeric types)
    std::sort(students.begin(), students.end(),
        [](const Student& a, const Student& b) {
            return std::tie(a.last_name, a.first_name, b.gpa, a.age)
                 < std::tie(b.last_name, b.first_name, a.gpa, b.age);
            // Note: b.gpa before a.gpa gives DESCENDING order for GPA
        });

    std::cout << "\nMethod 2 (std::tie):\n";
    for (const auto& s : students)
        std::cout << "  " << s.last_name << ", " << s.first_name
                  << "  GPA=" << s.gpa << "  age=" << s.age << "\n";

    // === Method 3: C++20 ranges with projection (single field) ===
    std::ranges::sort(students, std::greater{}, &Student::gpa);

    std::cout << "\nMethod 3 (ranges, GPA desc):\n";
    for (const auto& s : students)
        std::cout << "  " << s.last_name << ", " << s.first_name
                  << "  GPA=" << s.gpa << "\n";

    // === Method 4: C++20 spaceship operator (if you own the type) ===
    // struct Student {
    //     auto operator<=>(const Student&) const = default;
    //     // Gives lexicographic comparison of ALL members in declaration order
    // };

    // === IMPORTANT: strict weak ordering ===
    // A comparator must satisfy:
    // 1. Irreflexivity:  comp(a, a) == false
    // 2. Asymmetry:      if comp(a, b) then !comp(b, a)
    // 3. Transitivity:   if comp(a, b) && comp(b, c) then comp(a, c)
    // Violating these causes undefined behavior (crashes, infinite loops)!

    // WRONG: using <= instead of <
    // std::sort(v.begin(), v.end(), [](int a, int b) { return a <= b; }); // UB!

    return 0;
}
```

The `std::tie` trick in Method 2 is worth understanding: it creates temporary tuples using references and compares them lexicographically in one expression. The DESC trick (swapping `a.gpa` and `b.gpa` to get descending order) works because reversing the comparison reverses the sort direction. The critical rule for every comparator: it must implement **strict weak ordering**, meaning `comp(a, a)` is always false. Using `<=` instead of `<` breaks this and causes undefined behavior - the sort may crash or loop forever.

- **Method 1 (if-else chain):** Compare fields in order of priority. If a field differs, that determines the order. Otherwise, fall through to the next field. Flexible but verbose.
- **Method 2 (`std::tie`):** Creates tuples on the fly and uses tuple's built-in lexicographic comparison. Swap `a` and `b` for a field to reverse its order. Elegant and compact.
- **Method 3 (ranges projection):** Best for single-field sorting. Projections extract the field, comparator handles direction.

---

## Notes

- **`partial_sort(first, middle, last)`** sorts so that `[first, middle)` contains the smallest `(middle-first)` elements in sorted order. Elements in `[middle, last)` are in unspecified order. O(n log k) where k = middle-first.
- **Choosing the right sort:** Full sort -> `sort`. Stable -> `stable_sort`. Top-K sorted -> `partial_sort`. Single position -> `nth_element`. Already nearly sorted -> `std::sort` handles it well (insertion sort for small partitions internally).
- **`std::sort` implementation:** GCC/Clang use introsort: quicksort that falls back to heapsort after log2(n) recursion depth, plus insertion sort for partitions < 16 elements.
- **C++20 `std::ranges::sort`** accepts ranges directly (no begin/end) and supports projections for cleaner comparisons.
