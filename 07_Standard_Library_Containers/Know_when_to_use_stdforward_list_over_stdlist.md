# Know when to use std::forward_list over std::list

**Category:** Standard Library Containers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/container/forward_list>  

---

## Topic Overview

`std::forward_list` is a singly-linked list: each node stores one element and one pointer (to the next node). `std::list` is a doubly-linked list: each node has two pointers (next and prev). The trade-off is **memory savings** vs **API restrictions**.

The reason this matters is that linked-list nodes are small, and on a 64-bit system each pointer costs 8 bytes. If your element is also small - say a 4-byte `int` - that pointer overhead is a significant fraction of your per-node cost. `forward_list` cuts that overhead in half.

### Memory Layout Comparison

Here is a visual reminder of how the two containers differ internally:

```cpp
std::list (doubly-linked):
  [prev|data|next] <-> [prev|data|next] <-> [prev|data|next]
  Per-node overhead: 2 pointers (16 bytes on 64-bit)

std::forward_list (singly-linked):
  [data|next] -> [data|next] -> [data|next] -> nullptr
  Per-node overhead: 1 pointer (8 bytes on 64-bit)
```

One pointer saved per node. It sounds small, but across a million nodes it adds up fast.

### Key Differences

| Feature                     | `std::forward_list`       | `std::list`               |
| --- | --- | --- |
| Per-node overhead           | **1 pointer** (8 bytes)  | 2 pointers (16 bytes)    |
| Traversal                   | Forward only             | Forward and backward     |
| `push_front()`             | O(1)                     | O(1)                     |
| `push_back()`              | **N/A**                  | O(1)                     |
| `insert_after(it, val)`    | O(1)                     | N/A (uses `insert`)      |
| `insert(it, val)` (before) | **N/A**                  | O(1)                     |
| `erase_after(it)`          | O(1)                     | N/A (uses `erase`)       |
| `erase(it)`                | **N/A**                  | O(1)                     |
| `size()`                   | **O(n)** (not stored!)   | O(1) (stored since C++11)|
| `back()`                   | **N/A**                  | O(1)                     |
| `reverse()`                | O(n)                     | O(n)                     |
| `sort()`                   | O(n log n)               | O(n log n)               |
| `splice_after`/`splice`    | O(1)                     | O(1)                     |
| `before_begin()`           | Yes (pre-head sentinel)  | N/A                      |
| Reverse iteration           | **No**                   | Yes (rbegin/rend)        |

If the table feels like a lot, the practical summary is: `forward_list` trades away backward traversal, `push_back`, and O(1) `size()` in exchange for 50% less pointer overhead per node.

### Core Syntax

The `insert_after` / `erase_after` pattern is the part that trips people up most. Unlike `std::list` which inserts *before* a position, `forward_list` inserts *after* it - because a singly-linked node only knows its successor. Notice also `before_begin()`, the pre-head sentinel that lets you insert or erase at the very front using the same `_after` API:

```cpp
#include <iostream>
#include <forward_list>
#include <list>

int main() {
    std::forward_list<int> fl = {3, 1, 4, 1, 5};

    // push_front only (no push_back!)
    fl.push_front(0);  // [0, 3, 1, 4, 1, 5]

    // insert_after: insert AFTER the given position
    auto it = fl.begin();          // points to 0
    fl.insert_after(it, 99);       // [0, 99, 3, 1, 4, 1, 5]

    // before_begin: special iterator BEFORE the first element
    fl.insert_after(fl.before_begin(), -1);  // [-1, 0, 99, 3, 1, 4, 1, 5]

    // erase_after: erase the element AFTER the given position
    fl.erase_after(fl.begin());    // Erases 0: [-1, 99, 3, 1, 4, 1, 5]

    // Sort
    fl.sort();

    // Print
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: -1 1 1 3 4 5 99

    // Remove duplicates (after sorting)
    fl.unique();
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: -1 1 3 4 5 99

    return 0;
}
```

After sorting and calling `unique()`, each duplicate run is collapsed to one element. Both are O(n log n) and O(n) respectively, done in-place with no extra allocation.

### Important Notes

- **`before_begin()`** returns an iterator to a position before the first element - necessary because `insert_after` and `erase_after` operate on the element **following** the iterator.
- `std::forward_list` has **no `size()` member in O(1)** - it was intentionally omitted to keep the container as lightweight as possible. Use `std::distance(fl.begin(), fl.end())` if needed.
- `forward_list` saves 50% node overhead vs `list` but loses backward traversal and `push_back`.

---

## Self-Assessment

### Q1: Explain the memory savings of forward_list vs list and the API differences (insert_after)

The core insight here is that pointer overhead dominates for small element types. Watch the concrete numbers:

```cpp
#include <iostream>
#include <forward_list>
#include <list>
#include <cstddef>

struct SmallData {
    int value;
};

int main() {
    // === Memory comparison ===
    // On a 64-bit system:
    // sizeof(pointer) = 8 bytes

    // std::list node: [prev_ptr(8) | data | next_ptr(8)]
    // For SmallData (4 bytes): 8 + 4 + 8 = 20 bytes + padding = 24 bytes per node

    // std::forward_list node: [data | next_ptr(8)]
    // For SmallData (4 bytes): 4 + 8 = 12 bytes + padding = 16 bytes per node

    // Savings: 8 bytes per node (one pointer) = 33% reduction

    constexpr int N = 1'000'000;

    // Estimate memory usage
    size_t list_bytes = N * 24;          // ~24 MB for list
    size_t flist_bytes = N * 16;         // ~16 MB for forward_list
    size_t savings = list_bytes - flist_bytes;

    std::cout << "For " << N << " elements of SmallData:\n";
    std::cout << "  std::list:         ~" << list_bytes / (1024*1024) << " MB\n";
    std::cout << "  std::forward_list: ~" << flist_bytes / (1024*1024) << " MB\n";
    std::cout << "  Savings:           ~" << savings / (1024*1024) << " MB ("
              << (100 * savings / list_bytes) << "%)\n\n";
    // Output:
    //   std::list:         ~22 MB
    //   std::forward_list: ~15 MB
    //   Savings:           ~7 MB (33%)

    // === API differences: insert_after vs insert ===
    std::list<int> l = {1, 2, 3};
    auto lit = l.begin();
    ++lit;                         // Points to 2
    l.insert(lit, 99);             // Insert BEFORE 2: [1, 99, 2, 3]

    std::forward_list<int> fl = {1, 2, 3};
    auto flit = fl.begin();        // Points to 1
    fl.insert_after(flit, 99);     // Insert AFTER 1: [1, 99, 2, 3]
    // Same result, but different semantics!

    // Why "after"? In a singly-linked list, you can't go back to the
    // previous node to update its "next" pointer. So you MUST operate
    // on the node you have (modifying its "next" to point to a new node).

    std::cout << "list after insert(before 2):  ";
    for (int x : l) std::cout << x << " ";
    std::cout << "\n";
    // Output: list after insert(before 2):  1 99 2 3

    std::cout << "flist after insert_after(1): ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: flist after insert_after(1): 1 99 2 3

    // Erasing: same difference
    l.erase(std::next(l.begin()));   // Erase 99 by pointing at it
    fl.erase_after(fl.begin());      // Erase element AFTER first (99)

    // To insert at the front of forward_list:
    fl.insert_after(fl.before_begin(), 0);  // [0, 1, 2, 3]
    std::cout << "flist after insert_after(before_begin): ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: flist after insert_after(before_begin): 0 1 2 3

    return 0;
}
```

The final `insert_after(before_begin(), 0)` call is the canonical way to prepend an element when you only have the `_after` interface. `before_begin()` gives you a handle to a phantom node that sits conceptually one step before the head.

**Explanation:**

- Each `std::list` node stores 2 pointers (prev + next) = 16 bytes overhead on 64-bit systems.
- Each `std::forward_list` node stores 1 pointer (next only) = 8 bytes overhead.
- For small element types, the pointer overhead dominates - `forward_list` saves ~33% memory.
- `insert_after`/`erase_after` are necessary because a singly-linked node doesn't know its predecessor. You can only modify the "next" pointer of the node you hold.
- `before_begin()` provides a handle to insert/erase at the very front.

### Q2: Benchmark forward_list vs list for many small insertions with rare backward traversal

This benchmark exists to make the trade-off concrete. `forward_list` should pull ahead on `push_front` and forward iteration; `list` wins the moment you need `rbegin`/`rend`:

```cpp
#include <iostream>
#include <forward_list>
#include <list>
#include <chrono>
#include <random>

template <typename F>
long long measure_ms(F&& func) {
    auto start = std::chrono::high_resolution_clock::now();
    func();
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
}

int main() {
    constexpr int N = 2'000'000;

    // === Benchmark: push_front (common operation for both) ===
    long long t_fl, t_l;

    {
        std::forward_list<int> fl;
        t_fl = measure_ms([&] {
            for (int i = 0; i < N; ++i)
                fl.push_front(i);
        });
    }

    {
        std::list<int> l;
        t_l = measure_ms([&] {
            for (int i = 0; i < N; ++i)
                l.push_front(i);
        });
    }

    std::cout << "push_front x" << N << ":\n";
    std::cout << "  forward_list: " << t_fl << " ms\n";
    std::cout << "  list:         " << t_l << " ms\n\n";
    // Expected: forward_list slightly faster (smaller node allocation)

    // === Benchmark: forward iteration (summing all elements) ===
    {
        std::forward_list<int> fl;
        std::list<int> l;
        for (int i = 0; i < N; ++i) { fl.push_front(i); l.push_front(i); }

        volatile long long sum = 0;
        t_fl = measure_ms([&] {
            sum = 0;
            for (int x : fl) sum += x;
        });

        t_l = measure_ms([&] {
            sum = 0;
            for (int x : l) sum += x;
        });

        std::cout << "Forward iteration x" << N << ":\n";
        std::cout << "  forward_list: " << t_fl << " ms\n";
        std::cout << "  list:         " << t_l << " ms\n\n";
        // Expected: similar (both are pointer-chasing, but flist nodes are smaller)
    }

    // === When list wins: backward traversal needed ===
    {
        std::list<int> l;
        for (int i = 0; i < N; ++i) l.push_back(i);

        volatile long long sum = 0;
        auto t_reverse = measure_ms([&] {
            sum = 0;
            for (auto it = l.rbegin(); it != l.rend(); ++it)
                sum += *it;
        });

        std::cout << "Reverse iteration (list only): " << t_reverse << " ms\n";
        std::cout << "forward_list: NOT POSSIBLE (no rbegin/rend)\n\n";
    }

    std::cout << "Conclusion:\n";
    std::cout << "  Use forward_list when:\n";
    std::cout << "    - Memory matters (saves 33% node overhead)\n";
    std::cout << "    - Only forward iteration needed\n";
    std::cout << "    - Many small elements (pointer overhead > data size)\n";
    std::cout << "  Use list when:\n";
    std::cout << "    - Backward traversal is needed\n";
    std::cout << "    - push_back is needed\n";
    std::cout << "    - O(1) size() is needed\n";

    return 0;
}
```

The benchmark output will depend on your hardware and allocator, but the pattern is consistent: smaller nodes mean slightly less allocation time and slightly better cache density on forward traversal.

**How it works:**

- `forward_list` nodes are smaller -> slightly faster allocation and better cache density.
- Forward iteration speed is similar for both (both are pointer-chasing through heap nodes).
- `forward_list` wins on memory and insertion speed when only forward traversal is needed.
- `list` wins when you need `push_back`, backward iteration (`rbegin`/`rend`), or O(1) `size()`.

### Q3: Show that forward_list requires insert_after / erase_after due to single-direction linking

The reason this trips people up is that every other sequence container uses "insert before" semantics, but `forward_list` is physically incapable of doing that without knowing the predecessor. Let the diagram in the comments guide you:

```cpp
#include <iostream>
#include <forward_list>
#include <algorithm>

int main() {
    std::forward_list<int> fl = {10, 20, 30, 40, 50};

    // === Why insert_after, not insert? ===
    // In a singly-linked list, each node knows ONLY its successor:
    //
    //   [10|->] -> [20|->] -> [30|->] -> [40|->] -> [50|nullptr]
    //
    // If we have an iterator to node [30], we CAN:
    //   - Read node [30]'s "next" pointer -> [40]
    //   - Change [30]'s "next" to point to a new node
    //
    // But we CANNOT:
    //   - Find node [20] (the predecessor of [30]) without traversing from the start
    //   - Insert BEFORE [30] (requires modifying [20]'s "next")

    // --- Insert after a known position ---
    auto it = fl.begin();      // -> 10
    std::advance(it, 2);       // -> 30
    fl.insert_after(it, 35);   // [10, 20, 30, 35, 40, 50]
    //                          Node 30's "next" now -> 35, 35's "next" -> 40

    std::cout << "After insert_after(30, 35): ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: After insert_after(30, 35): 10 20 30 35 40 50

    // --- Insert at the very beginning ---
    // before_begin() returns an iterator to a phantom node before the first element
    fl.insert_after(fl.before_begin(), 5);
    // [5, 10, 20, 30, 35, 40, 50]

    std::cout << "After insert_after(before_begin, 5): ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: After insert_after(before_begin, 5): 5 10 20 30 35 40 50

    // --- Erase after a known position ---
    auto prev = fl.begin();     // -> 5
    std::advance(prev, 3);      // -> 30
    fl.erase_after(prev);       // Erases 35 (the node after 30)
    // [5, 10, 20, 30, 40, 50]

    std::cout << "After erase_after(30): ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: After erase_after(30): 5 10 20 30 40 50

    // --- Erase the first element ---
    fl.erase_after(fl.before_begin());  // Erases 5
    // [10, 20, 30, 40, 50]

    std::cout << "After erase_after(before_begin): ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: After erase_after(before_begin): 10 20 30 40 50

    // --- Pattern: track "previous" iterator for erasure ---
    // To erase an element matching a condition, track the predecessor:
    auto prev_it = fl.before_begin();
    for (auto curr = fl.begin(); curr != fl.end(); ) {
        if (*curr == 30) {
            curr = fl.erase_after(prev_it);  // Erase 30, returns iterator to 40
        } else {
            prev_it = curr;
            ++curr;
        }
    }

    std::cout << "After erasing 30: ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: After erasing 30: 10 20 40 50

    // --- Simpler: use remove() member function ---
    fl.remove(40);
    std::cout << "After remove(40): ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";
    // Output: After remove(40): 10 20 50

    return 0;
}
```

The "track previous" loop is the general technique for predicate-based erasure, but note the last line: if you just want to remove by value, `fl.remove(val)` handles the predecessor tracking for you internally.

**How it works:**

- A singly-linked node only knows its **next** neighbor. To insert/erase **before** a node, you'd need to modify the predecessor's "next" pointer - but you don't have access to the predecessor.
- Therefore, `std::forward_list` provides `insert_after()` and `erase_after()` instead of `insert()` and `erase()`.
- `before_begin()` provides a handle to the phantom pre-head position, enabling insertion/erasure at the very front.
- The manual "track previous" pattern is common when you need to erase elements matching a condition. The simpler alternative is the `remove()` / `remove_if()` member functions which handle predecessor tracking internally.

---

## Notes

- **Use `std::forward_list` when:** Memory is constrained, only forward traversal is needed, elements are small, and you primarily insert/remove from the front.
- **Use `std::list` when:** You need `push_back`, backward iteration, `splice` with counted transfer, or O(1) `size()`.
- **Neither for random access:** Both linked lists have O(n) access by index. If you need `[]`, use `std::vector` or `std::deque`.
- **`std::forward_list::sort()`** is an O(n log n) merge sort. It operates in-place (no extra memory allocation).
- **`splice_after`** transfers nodes between forward_lists in O(1) - no allocation, no copy.
- **C++20:** `std::erase_if(fl, predicate)` simplifies conditional removal without tracking predecessors.
