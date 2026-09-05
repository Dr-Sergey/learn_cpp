# Use std::is_sorted and std::is_sorted_until for invariant checking

**Category:** Standard Library Algorithms  
**Standard:** C++11, C++20 (ranges)  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/is_sorted>  

---

## Topic Overview

`std::is_sorted` and `std::is_sorted_until` are essential tools for **invariant checking** - verifying that data structures maintain their sorted property after mutations.

### When Invariant Checking Matters

The distinction between precondition checking and invariant checking is worth pausing on. Precondition checking asks "is this data in the right state *before* I use it?" Invariant checking asks "did *my own code* leave the data in the right state *after* I modified it?" Both use the same functions, but the intent is different.

```cpp
Scenario: You maintain a sorted container and perform operations on it.
After each operation, the sorted invariant must hold.

insert(v, 5) -> assert(is_sorted(v))  // OK
update(v[3]) -> assert(is_sorted(v))  // violation caught here!
```

---

## Self-Assessment

### Q1: Use is_sorted as a debug precondition check before calling binary_search

The `SortedBuffer` class below wraps a `std::vector<int>` and asserts the sorted invariant after every mutation and before every query. In debug builds, any operation that accidentally breaks the order is caught at the exact operation that caused it - not somewhere downstream where the symptoms show up.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cassert>
#include <numeric>

// === Sorted container class with invariant checking ===
class SortedBuffer {
    std::vector<int> data_;

    void check_invariant() const {
        assert(std::is_sorted(data_.begin(), data_.end())
               && "SortedBuffer invariant violated: data not sorted!");
    }

public:
    void insert(int value) {
        auto pos = std::lower_bound(data_.begin(), data_.end(), value);
        data_.insert(pos, value);
        check_invariant();  // verify after mutation
    }

    bool contains(int value) const {
        check_invariant();  // verify before binary search
        return std::binary_search(data_.begin(), data_.end(), value);
    }

    void remove(int value) {
        auto pos = std::lower_bound(data_.begin(), data_.end(), value);
        if (pos != data_.end() && *pos == value) {
            data_.erase(pos);
        }
        check_invariant();  // verify after mutation
    }

    size_t size() const { return data_.size(); }

    void print() const {
        for (int x : data_) std::cout << x << " ";
        std::cout << "\n";
    }
};

int main() {
    SortedBuffer buf;
    buf.insert(5);
    buf.insert(3);
    buf.insert(8);
    buf.insert(1);
    buf.insert(5);  // duplicate

    std::cout << "Buffer: ";
    buf.print();
    // 1 3 5 5 8

    std::cout << "Contains 5: " << std::boolalpha << buf.contains(5) << "\n";  // true
    std::cout << "Contains 4: " << buf.contains(4) << "\n";  // false

    buf.remove(5);
    std::cout << "After remove(5): ";
    buf.print();
    // 1 3 5 8

    std::cout << "All invariant checks passed!\n";

    return 0;
}
```

The reason to check both before queries and after mutations is that these catch different bugs: a pre-query check catches cases where external code smuggled unsorted data in, while a post-mutation check catches bugs in the mutation logic itself.

### Q2: Use is_sorted_until to find the first out-of-order element and diagnose a sorting bug

When an invariant violation occurs, knowing *that* the container is unsorted is less useful than knowing *where* it broke. `is_sorted_until` gives you that information by returning an iterator to the first out-of-order element, along with enough context to identify the problematic value.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

// === Invariant checker that reports WHERE the violation occurs ===
template <typename Container>
void verify_sorted_invariant(const Container& c, const std::string& context) {
    auto it = std::is_sorted_until(c.begin(), c.end());
    if (it == c.end()) {
        std::cout << "[" << context << "] Invariant OK: sorted\n";
    } else {
        auto idx = std::distance(c.begin(), it);
        std::cout << "[" << context << "] INVARIANT VIOLATED at index " << idx
                  << ": value " << *it << " < previous " << *std::prev(it) << "\n";

        // Show surrounding context
        auto start = (idx > 2) ? std::next(c.begin(), idx - 2) : c.begin();
        auto end = (idx + 3 < static_cast<long>(c.size()))
                       ? std::next(c.begin(), idx + 3) : c.end();
        std::cout << "  Context: ...";
        for (auto curr = start; curr != end; ++curr) {
            if (curr == it)
                std::cout << " [" << *curr << "]";  // highlight bad element
            else
                std::cout << " " << *curr;
        }
        std::cout << " ...\n";
    }
}

int main() {
    // Simulate a sorted vector that gets corrupted
    std::vector<int> v = {1, 3, 5, 7, 9, 11, 13, 15};

    verify_sorted_invariant(v, "initial");
    // [initial] Invariant OK: sorted

    // Simulate a bug: direct mutation that breaks sorting
    v[4] = 2;  // Oops! Changed 9 to 2

    verify_sorted_invariant(v, "after mutation");
    // [after mutation] INVARIANT VIOLATED at index 4: value 2 < previous 7
    //   Context: ... 5 7 [2] 11 13 ...

    // === Check custom comparator invariant ===
    std::vector<int> desc = {10, 8, 6, 4, 2};
    auto it = std::is_sorted_until(desc.begin(), desc.end(), std::greater<>{});
    if (it == desc.end()) {
        std::cout << "Descending invariant: OK\n";
    }
    // Descending invariant: OK

    return 0;
}
```

The surrounding-context printout in `verify_sorted_invariant` is what makes this useful during debugging - you see the values on either side of the bad element, not just the index number.

### Q3: Show is_sorted with a custom comparator for reverse-sorted ranges

The default comparator for `is_sorted` is `<`, which checks ascending order. If your container is sorted in some other order - descending, by absolute value, by string length - you have to pass the matching comparator or `is_sorted` will return `false` even for a correctly ordered container.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
#include <functional>

int main() {
    // === Ascending (default comparator) ===
    std::vector<int> asc = {1, 3, 5, 7, 9};
    std::cout << "Ascending sorted: " << std::boolalpha
              << std::is_sorted(asc.begin(), asc.end()) << "\n";  // true

    // === Descending (need custom comparator) ===
    std::vector<int> desc = {9, 7, 5, 3, 1};

    // Without comparator: false (not ascending)
    std::cout << "Desc with default comp: "
              << std::is_sorted(desc.begin(), desc.end()) << "\n";  // false

    // With std::greater: true (sorted in descending order)
    std::cout << "Desc with greater<>: "
              << std::is_sorted(desc.begin(), desc.end(), std::greater<>{}) << "\n";  // true

    // === Custom comparator: sort by absolute value ===
    std::vector<int> abs_sorted = {-1, 2, -3, 4, -5};
    bool by_abs = std::is_sorted(abs_sorted.begin(), abs_sorted.end(),
        [](int a, int b) { return std::abs(a) < std::abs(b); });
    std::cout << "Sorted by |value|: " << by_abs << "\n";  // true

    // === Sort by string length ===
    std::vector<std::string> by_len = {"a", "bb", "ccc", "dddd"};
    bool len_sorted = std::is_sorted(by_len.begin(), by_len.end(),
        [](const std::string& a, const std::string& b) {
            return a.size() < b.size();
        });
    std::cout << "Sorted by length: " << len_sorted << "\n";  // true

    // === Practical: validate that a priority queue's backing store is a valid heap ===
    std::vector<int> heap = {9, 7, 5, 3, 1, 4, 2};
    // Note: a heap is NOT necessarily sorted, but is_sorted can verify
    // if you expect sorted output from a heap sort
    std::sort_heap(heap.begin(), heap.end());
    std::cout << "After sort_heap, sorted: "
              << std::is_sorted(heap.begin(), heap.end()) << "\n";  // true

    return 0;
}
```

The reason `{9, 7, 5, 3, 1}` returns `false` with the default comparator is that `is_sorted` checks whether `!(v[i+1] < v[i])` for all adjacent pairs - and for a descending sequence that check fails at the very first pair. Passing `std::greater<>{}` flips the comparison so it checks descending order instead.

---

## Notes

- **Invariant checking vs. precondition checking:** Preconditions check data BEFORE using it. Invariants check that YOUR operations maintain correctness AFTER mutations.
- **Performance:** O(n) per check. In hot loops, guard with `#ifndef NDEBUG` or use only in test builds.
- **`is_sorted` with equal elements:** Returns true. Equal adjacent elements are considered sorted (uses `<`, not `<=`).
- **C++20 ranges:** `std::ranges::is_sorted(v)` with projection support: `std::ranges::is_sorted(people, {}, &Person::age)`.
- **Relation to `adjacent_find`:** `is_sorted(first, last)` is equivalent to `adjacent_find(first, last, greater<>{}) == last`.
