# Use std::hive (C++26) in depth: stable addresses, iteration, and erasure

**Category:** Standard Library New Cpp23 26  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/container/hive>  

---

## Topic Overview

`std::hive` (C++26) is a **bucket-based container** optimized for workloads with frequent **random insertion and erasure** - common in game engines (entity pools), simulations (particle systems), and GUI frameworks (widget pools).

The big idea is this: when you erase an element from a `std::vector`, everything after it shifts. When you erase from a `std::list`, the pointer structure adjusts. Both approaches cost you something - either shuffled data or cache-hostile pointer chasing. `std::hive` takes a different approach entirely: it leaves a "hole" where the erased element was, tracks those holes with a compact skipfield, and skips them automatically during iteration. You get O(1) erase and cache-friendly iteration over survivors without ever compacting the storage.

### Key Properties

| Property                | Description                                                 |
| --- | --- |
| **Stable addresses**    | Pointers/references to elements remain valid after insert/erase |
| **Automatic hole skipping** | Iteration skips erased slots without user intervention  |
| **No compaction needed** | Unlike vector: erase leaves a "hole", no element shifting  |
| **Cache-friendly**      | Elements stored in contiguous blocks (chunks)              |
| **Unordered**           | No guaranteed element ordering; new inserts may fill holes  |

### How It Works

```cpp
Block 0: [A][B][ ][D][E]     <- slot 2 erased, becomes a "hole"
Block 1: [F][ ][ ][I][J]     <- slots 1,2 are holes
                  |
Iteration: A -> B -> D -> E -> F -> I -> J   (holes skipped automatically)
Insert: new element fills an existing hole (O(1))
```

---

## Self-Assessment

### Q1: Insert and erase elements in a std::hive and verify that surviving element addresses remain stable

The stability guarantee is the defining feature of `std::hive`. This example makes that concrete by capturing raw pointers to elements before an erase and checking that those pointers are still valid afterward. Pay attention to the contrast with `std::vector` in the comment at the end - that comparison is the core reason `std::hive` exists.

**Answer:**

<!-- compile: needs `hive` (not in the CI standard library yet) -->
```cpp
// C++26 — requires <hive> support
#include <hive>
#include <iostream>
#include <vector>
#include <cassert>

int main() {
    std::hive<int> h;

    // Insert elements and record their addresses
    auto it1 = h.insert(10);
    auto it2 = h.insert(20);
    auto it3 = h.insert(30);

    int* addr1 = &*it1;  // Address of element 10
    int* addr3 = &*it3;  // Address of element 30

    std::cout << "Before erase:\n";
    std::cout << "  *addr1 = " << *addr1 << " (at " << addr1 << ")\n";
    std::cout << "  *addr3 = " << *addr3 << " (at " << addr3 << ")\n";

    // Erase the middle element
    h.erase(it2);  // Removes 20, creates a hole

    // KEY GUARANTEE: addresses of surviving elements are UNCHANGED
    std::cout << "After erasing 20:\n";
    std::cout << "  *addr1 = " << *addr1 << " (at " << addr1 << ")\n";  // Still 10
    std::cout << "  *addr3 = " << *addr3 << " (at " << addr3 << ")\n";  // Still 30
    assert(*addr1 == 10);  // Stable!
    assert(*addr3 == 30);  // Stable!

    // Insert new element — may reuse the hole left by 20
    auto it4 = h.insert(40);
    assert(*addr1 == 10);  // Still stable after insert
    assert(*addr3 == 30);

    // Print all elements (holes are skipped)
    std::cout << "All elements: ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // Possible output: 10 40 30  (40 may fill the hole where 20 was)

    // Contrast with std::vector:
    // std::vector<int> v = {10, 20, 30};
    // int* vaddr = &v[0];
    // v.erase(v.begin() + 1);
    // *vaddr may now be 20 (shifted!) — NOT STABLE
}
```

Notice that the hole left by erasing 20 becomes available for reuse - the new element 40 may land right in that spot. The memory is not wasted, and no compaction pass is needed. The skipfield handles the bookkeeping entirely behind the scenes.

### Q2: Show that std::hive iteration skips holes (erased slots) automatically without repack

This example erases every other element and then iterates to show that the holes are transparently skipped. The conceptual diagram in the comments is worth reading carefully - the skipfield representation is the core data structure that makes this work in O(1) per hole.

**Answer:**

<!-- compile: needs `hive` (not in the CI standard library yet) -->
```cpp
#include <hive>
#include <iostream>

int main() {
    std::hive<int> h;

    // Insert 10 elements: 0, 1, 2, ..., 9
    std::vector<std::hive<int>::iterator> iters;
    for (int i = 0; i < 10; ++i)
        iters.push_back(h.insert(i));

    std::cout << "Before erasure (" << h.size() << " elements): ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 0 1 2 3 4 5 6 7 8 9

    // Erase every other element -> creates holes at indices 1, 3, 5, 7, 9
    for (int i = 1; i < 10; i += 2)
        h.erase(iters[i]);

    std::cout << "After erasing odd indices (" << h.size() << " elements): ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 0 2 4 6 8  — holes skipped automatically!

    // No repack/compaction happened — elements 0,2,4,6,8 are at their
    // ORIGINAL memory addresses. The iteration logic uses a skipfield
    // to jump over erased slots efficiently.

    // Internal structure (conceptual):
    // Memory:    [0][ ][2][ ][4][ ][6][ ][8][ ]
    // Skipfield: [0][1][0][1][0][1][0][1][0][1]
    //               ^     ^     ^     ^     ^  = holes (skipped)

    // Erasing more elements just adds more holes
    h.erase(h.begin());  // Erase element 0
    std::cout << "After erasing first: ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 2 4 6 8

    // Inserting fills holes first (no allocation if holes exist)
    h.insert(99);
    std::cout << "After inserting 99: ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 99 2 4 6 8  (99 may occupy a hole)
}
```

The skipfield stores, for each slot, how many consecutive slots to jump over to reach the next live element. This lets the iterator advance in O(1) per live element regardless of how many holes surround it - unlike a naive approach that would scan slot-by-slot.

### Q3: Benchmark std::hive vs std::list and std::deque for a workload with frequent random erasure

The benchmark below captures the performance story in concrete numbers. `std::hive` typically wins when the workload mixes erasure with iteration over survivors, because it avoids both the pointer-chasing of `std::list` and the O(n) shifting of `std::vector`.

**Answer:**

<!-- compile: needs `hive` (not in the CI standard library yet) -->
```cpp
#include <hive>
#include <list>
#include <deque>
#include <vector>
#include <chrono>
#include <iostream>
#include <random>
#include <algorithm>

template<typename Container>
double benchmark_insert_erase(int n_elements, int n_erasures) {
    Container c;
    std::vector<typename Container::iterator> iters;
    iters.reserve(n_elements);

    // Phase 1: Insert N elements
    for (int i = 0; i < n_elements; ++i)
        iters.push_back(c.insert(c.end(), i));

    // Phase 2: Randomly erase half the elements
    std::mt19937 rng(42);
    std::shuffle(iters.begin(), iters.end(), rng);

    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < n_erasures && !iters.empty(); ++i) {
        c.erase(iters.back());
        iters.pop_back();
    }

    // Phase 3: Iterate over surviving elements (sum)
    volatile int sum = 0;
    for (auto& x : c) sum += x;

    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
    constexpr int N = 100'000;
    constexpr int E = N / 2;

    // Note: std::hive<int> requires C++26 compiler support
    // Here we show the expected performance characteristics
    double hive_ms = benchmark_insert_erase<std::hive<int>>(N, E);
    double list_ms = benchmark_insert_erase<std::list<int>>(N, E);
    // std::deque doesn't support stable erase-by-iterator in the same way
    // Included for reference with sequential erasure

    std::cout << "Erase " << E << " of " << N << " + iterate survivors:\n";
    std::cout << "  std::hive: " << hive_ms << " ms\n";
    std::cout << "  std::list: " << list_ms << " ms\n";

    /*
    Expected results (100K elements, erase 50K):
    ┌──────────────┬───────────┬───────────────────────────────┐
    │ Container    │ Time      │ Why                           │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::hive    │ ~2-4 ms   │ O(1) erase + cache-friendly   │
    │              │           │ iteration with skipfield       │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::list    │ ~8-15 ms  │ O(1) erase but pointer-chasing│
    │              │           │ on iteration (cache misses)    │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::vector  │ ~50+ ms   │ O(n) erase (shifts elements)  │
    │              │           │ but fast iteration             │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::deque   │ ~30+ ms   │ O(n) erase for random pos     │
    │              │           │ moderate iteration speed       │
    └──────────────┴───────────┴───────────────────────────────┘

    std::hive wins when:

    - Frequent random erasure (entities dying, particles expiring)
    - External pointers to elements must remain valid
    - Iteration over survivors is a hot-path operation

    */
}
```

`std::list` beats `std::vector` on erase but then loses on iteration because every node is a separate heap allocation scattered across memory. `std::hive` gets the best of both: erases are O(1) and do not move anything, and iteration still walks through contiguous blocks, only jumping over the holes tracked by the skipfield.

---

## Notes

- `std::hive` was previously known as `plf::colony` (by Matthew Bentley) - battle-tested in game engines well before standardization.
- Elements are stored in blocks; erased slots become holes tracked by a **skipfield** (O(1) skip per hole during iteration).
- `hive::reshape()` controls the min/max block sizes for tuning memory overhead vs fragmentation.
- Not random-access: no `operator[]`. Use iterators only.
- Ideal for **entity-component-system (ECS)** pools where entities are created and destroyed frequently.
