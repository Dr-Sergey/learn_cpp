# Use std::partition and std::stable_partition for in-place filtering

**Category:** Standard Library Algorithms  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/partition>  

---

## Topic Overview

This file focuses on the **practical filtering patterns** using `std::partition` and `std::stable_partition` - separating elements in-place by a predicate, understanding complexity trade-offs, and implementing multi-way partitioning.

### Quick Reference

The basic usage is straightforward: call `partition` or `stable_partition`, get back an iterator to the dividing line between the two groups:

```cpp
#include <algorithm>

// Partition: elements satisfying pred come first
auto mid = partition(v.begin(), v.end(), pred);
// [v.begin(), mid) - satisfy pred (unordered)
// [mid, v.end())   - don't satisfy pred (unordered)

// Stable partition: same but preserves relative order
auto mid = stable_partition(v.begin(), v.end(), pred);
```

### Why In-Place Filtering

You might wonder why you'd choose `partition` over a simple `copy_if` into a new container. The table below shows what you're trading off. The unique thing about partition is that it keeps **both** groups in the same container - something none of the other approaches do:

| Approach | Time | Extra Space | Preserves Order |
| --- | --- | --- | --- |
| Copy + `copy_if` | O(n) | O(n) | Yes |
| `partition` | O(n) | O(1) | No |
| `stable_partition` | O(n) / O(n log n) | O(n) / O(1) | Yes |
| `erase` + `remove_if` | O(n) | O(1) | Yes, but removes |

---

## Self-Assessment

### Q1: Use partition to separate even and odd numbers in a vector in O(n) time

After the call, `mid` points to the boundary. Everything before it is even, everything from it onward is odd - and you can count, iterate, or otherwise operate on each group using that iterator:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    std::cout << "Before: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // Partition: even numbers first, O(n) time, O(1) space
    auto mid = std::partition(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });

    std::cout << "After:  ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // e.g. 10 2 8 4 6 5 7 3 9 1  (even first, but within groups order varies)

    std::cout << "Even count: " << std::distance(v.begin(), mid) << "\n";  // 5

    // Use partition_point on an already-partitioned range (O(log n) for random access)
    auto pp = std::partition_point(v.begin(), v.end(), [](int n) {
        return n % 2 == 0;
    });
    std::cout << "partition_point == mid: " << std::boolalpha << (pp == mid) << "\n";

    // partition_copy: non-modifying alternative
    std::vector<int> original = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::vector<int> evens, odds;
    std::partition_copy(original.begin(), original.end(),
                        std::back_inserter(evens),
                        std::back_inserter(odds),
                        [](int n) { return n % 2 == 0; });

    std::cout << "Evens: ";
    for (int x : evens) std::cout << x << " ";  // 2 4 6 8 10
    std::cout << "\nOdds:  ";
    for (int x : odds)  std::cout << x << " ";  // 1 3 5 7 9
    std::cout << "\n";

    return 0;
}
```

`partition_copy` is worth knowing when you want the split without touching the original. It writes both groups to separate output iterators in a single pass.

### Q2: Show that stable_partition preserves relative order within each partition

The reason people reach for `stable_partition` is correctness in scenarios where the order of elements within each group is meaningful - like a task queue where high-priority items must stay in the order they were submitted:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

struct Task {
    std::string name;
    int priority;  // 1 = high, 2 = medium, 3 = low
};

int main() {
    std::vector<Task> tasks = {
        {"Deploy",  3}, {"Fix bug",   1}, {"Write docs", 2},
        {"Review",  1}, {"Refactor",  2}, {"Test",       1}
    };

    auto print = [](const std::vector<Task>& ts) {
        for (auto& t : ts)
            std::cout << "  [" << t.priority << "] " << t.name << "\n";
    };

    std::cout << "Original order:\n";
    print(tasks);

    // stable_partition: high priority (1) first, preserving insertion order
    auto mid = std::stable_partition(tasks.begin(), tasks.end(),
        [](const Task& t) { return t.priority == 1; });

    std::cout << "\nAfter stable_partition (priority==1 first):\n";
    print(tasks);
    // [1] Fix bug     <- relative order among priority-1 preserved
    // [1] Review
    // [1] Test
    // [3] Deploy      <- relative order among the rest preserved
    // [2] Write docs
    // [2] Refactor

    std::cout << "\nHigh priority tasks: "
              << std::distance(tasks.begin(), mid) << "\n";  // 3

    // === Compare with unstable partition ===
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8};

    auto d1 = data;
    std::partition(d1.begin(), d1.end(), [](int n) { return n <= 4; });
    std::cout << "\npartition:        ";
    for (int x : d1) std::cout << x << " ";  // Order not guaranteed
    std::cout << "\n";

    auto d2 = data;
    std::stable_partition(d2.begin(), d2.end(), [](int n) { return n <= 4; });
    std::cout << "stable_partition: ";
    for (int x : d2) std::cout << x << " ";  // 1 2 3 4 5 6 7 8
    std::cout << "\n";

    return 0;
}
```

With `stable_partition`, the high-priority tasks arrive in the same relative order they had in the original list. With plain `partition`, you'd still get all three at the front, but their order relative to each other would be unspecified.

### Q3: Implement a multi-way partition using two successive partition calls

The pattern for splitting into more than two groups is to apply `stable_partition` repeatedly on the suffix that hasn't been classified yet. Each call carves off one more group from the front:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    // === Three-way partition: negative / zero / positive ===
    std::vector<int> data = {3, -1, 0, -5, 2, 0, -3, 4, 0, 1, -2};

    std::cout << "Before: ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";

    // Step 1: negatives first
    auto zero_start = std::stable_partition(data.begin(), data.end(),
        [](int n) { return n < 0; });

    // Step 2: among the non-negatives, zeros first (then positives)
    auto pos_start = std::stable_partition(zero_start, data.end(),
        [](int n) { return n == 0; });

    std::cout << "After three-way partition:\n";
    std::cout << "  Negative: ";
    for (auto it = data.begin(); it != zero_start; ++it) std::cout << *it << " ";
    std::cout << "\n  Zero:     ";
    for (auto it = zero_start; it != pos_start; ++it) std::cout << *it << " ";
    std::cout << "\n  Positive: ";
    for (auto it = pos_start; it != data.end(); ++it) std::cout << *it << " ";
    std::cout << "\n";
    // Negative: -1 -5 -3 -2
    // Zero:     0 0 0
    // Positive: 3 2 4 1

    // === Practical: four-way classification ===
    enum class Grade { A, B, C, F };
    struct Student {
        std::string name;
        Grade grade;
    };

    std::vector<Student> students = {
        {"Alice", Grade::B}, {"Bob", Grade::A}, {"Carol", Grade::F},
        {"Dave", Grade::C}, {"Eve", Grade::A}, {"Frank", Grade::B},
        {"Grace", Grade::C}, {"Hank", Grade::F}
    };

    auto b_start = std::stable_partition(students.begin(), students.end(),
        [](const Student& s) { return s.grade == Grade::A; });
    auto c_start = std::stable_partition(b_start, students.end(),
        [](const Student& s) { return s.grade == Grade::B; });
    auto f_start = std::stable_partition(c_start, students.end(),
        [](const Student& s) { return s.grade == Grade::C; });

    std::cout << "\nGrade groups:\n";
    for (auto& s : students)
        std::cout << "  " << s.name << " (" << static_cast<int>(s.grade) << ")\n";
    // Bob(A), Eve(A), Alice(B), Frank(B), Dave(C), Grace(C), Carol(F), Hank(F)

    return 0;
}
```

The key insight in the multi-way pattern is that each successive call operates only on the not-yet-classified tail. The boundary iterators returned by each call (`zero_start`, `pos_start`) define the three groups precisely.

---

## Notes

- **`partition` does NOT sort** - it only guarantees that elements satisfying the predicate come before those that don't. Within each partition, the order is unspecified.
- **Multi-way partition** with `stable_partition` is a powerful pattern: partition into N groups by calling `stable_partition` N-1 times on successively smaller suffixes.
- **`partition_copy`** is the non-mutating alternative - splits into two output ranges without modifying the input.
- `std::ranges::partition` (C++20) supports projections: `std::ranges::partition(people, &Person::is_active)`.
- Partition-based algorithms are at the heart of **quicksort** and **quickselect**.
