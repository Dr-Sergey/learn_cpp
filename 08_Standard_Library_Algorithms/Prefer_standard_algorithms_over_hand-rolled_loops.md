# Prefer standard algorithms over hand-rolled loops

**Category:** Standard Library Algorithms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm>  

---

## Topic Overview

The C++ Standard Library provides ~100+ algorithms in `<algorithm>`, `<numeric>`, and `<ranges>`. Using them instead of writing manual loops gives you correctness (tested implementations), clarity (names convey intent), and performance (optimized by library vendors). Scott Meyers called this out explicitly in Effective STL Item 43, and the advice is even more relevant now that C++17 execution policies and C++20 ranges exist.

### Why Prefer Algorithms

If the table below feels like a lot, the core message is simple: algorithm names document your intent, and a code reviewer who sees `std::sort` instantly knows what is happening, while a raw loop requires careful reading to figure out what invariant it maintains.

| Benefit | Manual loop | Standard algorithm |
| --- | --- | --- |
| **Intent** | Must read loop body to understand | Name says what it does |
| **Correctness** | Off-by-one, boundary bugs | Thoroughly tested |
| **Optimization** | Manual | Vendor-optimized (SIMD, branch prediction) |
| **Parallelism** | Rewrite entirely | Add execution policy argument |
| **Composition** | Nested loops | Pipe with ranges (C++20) |
| **Code review** | "What does this loop do?" | "sort, find, transform" - self-documenting |

### Common Replacements

The side-by-side comparison below shows how familiar loop patterns map directly to named algorithms. Once you train your eye to see these patterns, you will naturally reach for the algorithm version first.

```cpp
// LOOP: find max element
int max = v[0];
for (int x : v) if (x > max) max = x;
// ALGORITHM:
auto max = *std::max_element(v.begin(), v.end());

// LOOP: check if any element matches
bool found = false;
for (auto& x : v) if (x == target) { found = true; break; }
// ALGORITHM:
bool found = std::any_of(v.begin(), v.end(), [&](auto& x) { return x == target; });
// Or: std::find(v.begin(), v.end(), target) != v.end();

// LOOP: count matches
int count = 0;
for (auto& x : v) if (x > 5) ++count;
// ALGORITHM:
auto count = std::count_if(v.begin(), v.end(), [](int x) { return x > 5; });

// LOOP: transform
for (auto& x : v) x = x * 2;
// ALGORITHM:
std::transform(v.begin(), v.end(), v.begin(), [](int x) { return x * 2; });

// LOOP: copy-if
std::vector<int> result;
for (auto& x : v) if (x > 0) result.push_back(x);
// ALGORITHM:
std::copy_if(v.begin(), v.end(), std::back_inserter(result), [](int x) { return x > 0; });
```

---

## Self-Assessment

### Q1: Rewrite a manual max-finding loop using std::max_element and show the result is identical

Let's prove the results match, then look at the C++20 projection syntax that makes the struct case even cleaner.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cassert>
#include <string>

struct Student {
    std::string name;
    double gpa;
};

int main() {
    std::vector<int> scores = {72, 95, 88, 63, 91, 85, 97, 76};

    // === Manual loop ===
    int manual_max = scores[0];
    int manual_idx = 0;
    for (int i = 1; i < static_cast<int>(scores.size()); ++i) {
        if (scores[i] > manual_max) {
            manual_max = scores[i];
            manual_idx = i;
        }
    }
    std::cout << "Manual: max=" << manual_max << " at index " << manual_idx << "\n";
    // max=97 at index 6

    // === std::max_element ===
    auto it = std::max_element(scores.begin(), scores.end());
    int algo_max = *it;
    int algo_idx = static_cast<int>(std::distance(scores.begin(), it));
    std::cout << "Algorithm: max=" << algo_max << " at index " << algo_idx << "\n";
    // max=97 at index 6

    assert(manual_max == algo_max);
    assert(manual_idx == algo_idx);
    std::cout << "Results are identical\n";

    // === With custom comparator on structs ===
    std::vector<Student> students = {
        {"Alice", 3.8}, {"Bob", 3.2}, {"Charlie", 3.9}, {"Diana", 3.5}
    };

    // Manual:
    const Student* best_manual = &students[0];
    for (const auto& s : students)
        if (s.gpa > best_manual->gpa) best_manual = &s;

    // Algorithm:
    auto best_algo = std::max_element(students.begin(), students.end(),
        [](const Student& a, const Student& b) { return a.gpa < b.gpa; });

    // C++20 ranges with projection (cleanest):
    auto best_ranges = std::ranges::max_element(students, {}, &Student::gpa);

    std::cout << "\nBest student (manual): " << best_manual->name << " " << best_manual->gpa << "\n";
    std::cout << "Best student (algo):   " << best_algo->name << " " << best_algo->gpa << "\n";
    std::cout << "Best student (ranges): " << best_ranges->name << " " << best_ranges->gpa << "\n";
    // All: Charlie 3.9

    // === minmax_element: both in one pass ===
    auto [min_it, max_it] = std::minmax_element(scores.begin(), scores.end());
    std::cout << "\nMin: " << *min_it << ", Max: " << *max_it << "\n";  // 63, 97

    return 0;
}
```

`std::max_element` returns an iterator to the greatest element, which gives you both the value and the position. The algorithm version is shorter, has no off-by-one risk, and its intent is clear at a glance. The C++20 `std::ranges::max_element` with a projection eliminates the need for a custom comparator in most cases. `minmax_element` finds both extremes in a single pass - roughly 1.5n comparisons versus 2n for calling `min_element` and `max_element` separately.

### Q2: Explain how algorithms express intent better than raw loops for code reviewers

**Algorithms communicate WHAT, not HOW.** A code reviewer seeing `std::sort(v.begin(), v.end())` immediately knows the vector is being sorted - no need to trace through loop logic. Compare:

```cpp
// What does this loop do? Reader must trace every line:
for (size_t i = 0; i < v.size(); ) {
    if (v[i] % 2 == 0)
        v.erase(v.begin() + i);
    else
        ++i;
}

// vs. (C++20):
std::erase_if(v, [](int x) { return x % 2 == 0; });
// Intent is instantly clear: "remove even numbers"
```

The loop version is also subtly dangerous: erasing from a vector while iterating it manually is a classic source of bugs (the index-management logic is easy to get wrong). The algorithm version is correct by construction.

**Key advantages for code review:**

1. **Named intent:** `find_if` = "find something matching a condition". `partition` = "divide into two groups". No interpretation needed.
2. **Guaranteed correctness:** `stable_partition` preserves relative order - the algorithm guarantees it. A hand-rolled version might not, and you would not know until a tricky test case exposed it.
3. **Known complexity:** Reviewers know `sort` is O(n log n) without checking the implementation. A loop could be anything.
4. **Reduced surface area for bugs:** Algorithms don't have off-by-one errors, dangling iterator bugs, or early-termination mistakes.
5. **Trivial parallelism:** Adding `std::execution::par` to an algorithm is one extra argument - no restructuring needed.

### Q3: Show five algorithms that are non-trivial to implement correctly (e.g., stable_partition, inplace_merge)

This example makes the strongest case for preferring algorithms: these five operations have subtle internal implementations that are genuinely hard to get right from scratch, yet each one is a single standard call.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <random>
#include <string>

int main() {
    // === 1. std::stable_partition ===
    // Partitions while preserving relative order within each group.
    // O(n) with extra memory, O(n log n) without — complex buffer management.
    {
        std::vector<int> v = {1, 8, 3, 6, 2, 7, 4, 5};
        auto mid = std::stable_partition(v.begin(), v.end(),
            [](int x) { return x <= 4; });

        std::cout << "1. stable_partition (<=4 | >4): ";
        for (int x : v) std::cout << x << " ";
        std::cout << "  (pivot at index " << (mid - v.begin()) << ")\n";
        // Output: 1 3 2 4 8 6 7 5 — relative order preserved in BOTH groups
    }

    // === 2. std::inplace_merge ===
    // Merges two consecutive sorted subranges in-place.
    // Allocates buffer for O(n); falls back to O(n log n) rotate-merge.
    {
        std::vector<int> v = {1, 3, 5, 7, 2, 4, 6, 8};
        std::inplace_merge(v.begin(), v.begin() + 4, v.end());

        std::cout << "2. inplace_merge: ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n";
        // Output: 1 2 3 4 5 6 7 8
    }

    // === 3. std::nth_element ===
    // Places the nth element where it would be in sorted order.
    // All elements before nth are <=, all after are >=. O(n) average.
    // Based on Introselect (quickselect + median-of-medians fallback).
    {
        std::vector<int> v = {7, 2, 8, 1, 5, 3, 9, 4, 6};
        auto nth = v.begin() + 4;  // median position
        std::nth_element(v.begin(), nth, v.end());

        std::cout << "3. nth_element (median): " << *nth << "\n";
        // Output: 5 (the element that would be at index 4 if sorted)
        std::cout << "   Array: ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n";
        // First 4 elements are <= 5, last 4 are >= 5 (not necessarily sorted)
    }

    // === 4. std::rotate ===
    // Left-rotates elements so that `middle` becomes the first element.
    // O(n) with at most n swaps — the 3-reverse trick or juggling algorithm.
    {
        std::vector<int> v = {1, 2, 3, 4, 5, 6, 7};
        std::rotate(v.begin(), v.begin() + 3, v.end());

        std::cout << "4. rotate (by 3): ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n";
        // Output: 4 5 6 7 1 2 3
    }

    // === 5. std::partial_sort ===
    // Sorts only the first K elements. O(n log k) using a selection algorithm.
    // Getting this right without the full sort overhead requires heap operations.
    {
        std::vector<int> v = {9, 3, 7, 1, 8, 2, 6, 4, 5};
        std::partial_sort(v.begin(), v.begin() + 4, v.end());

        std::cout << "5. partial_sort (top 4): ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n";
        // Output: 1 2 3 4 | rest in unspecified order
        // Only the first 4 elements are sorted and correct
    }

    // === Why these are hard to implement correctly ===
    std::cout << "\n=== Why these are non-trivial ===\n";
    std::cout << "stable_partition: Must preserve order while partitioning in-place — requires O(n) buffer or complex O(n log n) rotation\n";
    std::cout << "inplace_merge:    Merging without extra memory needs block-rotation techniques\n";
    std::cout << "nth_element:      Introselect (quickselect + fallback) for guaranteed O(n) — subtle pivot selection\n";
    std::cout << "rotate:           Three reverses or juggling algorithm — easy to get boundary conditions wrong\n";
    std::cout << "partial_sort:     Heap-based selection of top-k — careful to maintain invariant\n";

    return 0;
}
```

These five algorithms have subtle implementations that are easy to get wrong when hand-coded:

1. **`stable_partition`** - Preserving relative order during in-place partitioning requires buffer management or O(n log n) rotation chains. A naive implementation loses the stability guarantee.
2. **`inplace_merge`** - Merging two sorted halves without extra space uses block rotation, which is tricky to implement correctly and rarely appears in textbooks.
3. **`nth_element`** - Introselect with a median-of-medians fallback guarantees O(n) but has complex pivot selection logic. Quickselect alone can degrade to O(n²) on adversarial input.
4. **`rotate`** - The three-reverse trick is elegant but boundary conditions are easy to mess up. The juggling algorithm is even more complex.
5. **`partial_sort`** - Uses a heap to efficiently find and sort the top-k elements without sorting the rest. Getting the heap invariant right at the boundaries is non-trivial.

---

## Notes

- **Performance optimizations:** Library implementations use SIMD, branchless comparisons, and architecture-specific tricks that hand-rolled loops cannot match without significant effort.
- **Execution policies (C++17):** `std::sort(std::execution::par, v.begin(), v.end())` parallelizes with one argument change. Hand-rolled loops require complete restructuring.
- **C++20 ranges:** `std::ranges::sort(v)` eliminates begin/end boilerplate. `v | views::filter(pred) | views::transform(fn)` replaces nested loops with composable pipelines.
- **When loops ARE better:** When the algorithm doesn't exist (for example, a complex state machine), when early exit with multiple conditions is needed, or when readability suffers from overly lambda-heavy algorithm usage.
- **Scott Meyers' rule (Effective STL #43):** "Prefer algorithm calls to hand-written loops."
