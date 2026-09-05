# Understand iterator invalidation rules for all standard containers

**Category:** Standard Library Containers  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/container>  

---

## Topic Overview

Iterator invalidation is one of the most common sources of bugs in C++ programs. Each container type has specific rules about which operations invalidate iterators, pointers, and references to elements.

The reason this trips people up is that the rules differ significantly between containers, and the consequence of using an invalidated iterator is undefined behavior - which usually means a crash or silently wrong output, often far away from the actual bug. Understanding the rules upfront is much cheaper than debugging the aftermath.

### Master Invalidation Table

#### std::vector

| Operation                | Iterators invalidated                    | Refs/Ptrs invalidated |
| --- | --- | --- |
| `push_back` (no realloc)| Past-the-end only                        | None                  |
| `push_back` (realloc)   | **ALL**                                  | **ALL**               |
| `insert` (no realloc)   | At and after insertion point             | At and after          |
| `insert` (realloc)      | **ALL**                                  | **ALL**               |
| `erase`                 | At and after erasure point               | At and after          |
| `reserve`               | **ALL** (if capacity changes)            | **ALL**               |
| `clear`                 | **ALL**                                  | **ALL**               |
| `resize` (grow, realloc)| **ALL**                                  | **ALL**               |

#### std::deque

| Operation                | Iterators invalidated                    | Refs/Ptrs invalidated  |
| --- | --- | --- |
| `push_back` / `push_front` | **ALL iterators**                     | **None** (refs/ptrs stable!) |
| `insert` (middle)       | **ALL**                                  | **ALL**                |
| `erase` (front/back)    | Only erased + past-the-end               | Only erased            |
| `erase` (middle)        | **ALL**                                  | **ALL**                |

#### std::list / std::forward_list

| Operation     | Iterators invalidated | Refs/Ptrs invalidated |
| --- | --- | --- |
| `insert`     | **None**             | **None**             |
| `push_back`  | **None**             | **None**             |
| `push_front` | **None**             | **None**             |
| `erase`      | **Only erased**      | **Only erased**      |
| `splice`     | **None** (transferred elements keep iterators!) | **None** |

#### std::map / std::set / std::multimap / std::multiset

| Operation  | Iterators invalidated | Refs/Ptrs invalidated |
| --- | --- | --- |
| `insert`  | **None**             | **None**             |
| `emplace` | **None**             | **None**             |
| `erase`   | **Only erased**      | **Only erased**      |
| `merge`   | **None** (C++17)     | **None**             |

#### std::unordered_map / std::unordered_set

| Operation               | Iterators invalidated        | Refs/Ptrs invalidated |
| --- | --- | --- |
| `insert` (no rehash)   | **None**                    | **None**              |
| `insert` (rehash)      | **ALL**                     | **None** (!)          |
| `erase`                | **Only erased**             | **Only erased**       |
| `rehash` / `reserve`   | **ALL**                     | **None**              |

### Key Insight: References vs Iterators

For `std::unordered_map` and `std::deque`, **references and pointers** may be more stable than iterators. This is one of the more surprising facts in the table, and it's worth understanding why:

- Unordered containers: rehashing changes the bucket structure (which iterators track) but doesn't physically move elements in memory - so raw pointers and references to elements stay valid.
- Deque: pushing to front/back adds new memory blocks but doesn't relocate existing elements within their blocks - so a `&dq[i]` reference survives a `push_back`.

### Core Example

Here is a quick demonstration of the three broad categories - vector (dangerous), list (safest), and map (node-stable):

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <map>
#include <unordered_map>

int main() {
    // === Vector: most dangerous ===
    std::vector<int> v = {1, 2, 3, 4, 5};
    auto it = v.begin() + 2;  // Points to 3
    std::cout << "Before push_back: " << *it << "\n";  // Output: 3

    v.push_back(6);  // May reallocate!
    // 'it' is potentially INVALID here
    // std::cout << *it;  // UNDEFINED BEHAVIOR if realloc happened

    // === List: safest ===
    std::list<int> l = {1, 2, 3, 4, 5};
    auto lit = std::next(l.begin(), 2);  // Points to 3
    l.push_back(6);   // No invalidation!
    l.push_front(0);  // No invalidation!
    l.insert(l.begin(), -1);  // No invalidation!
    std::cout << "List after modifications: " << *lit << "\n";  // Output: 3 (still valid!)

    // === Map: node-stable ===
    std::map<std::string, int> m = {{"a", 1}, {"b", 2}, {"c", 3}};
    auto mit = m.find("b");
    m.insert({"d", 4});  // No invalidation!
    m.erase("a");         // Only "a"'s iterator invalidated
    std::cout << "Map after modifications: " << mit->second << "\n";  // Output: 2 (valid!)

    return 0;
}
```

The list example is the pleasant surprise: after three separate insertions from different ends, `lit` still correctly points to `3`. That stability is the whole reason to reach for `std::list` when you need long-lived iterators.

### Important Notes

- **When in doubt with vectors:** Re-acquire iterators after any modification.
- **`reserve()` before inserting** into a vector prevents reallocation-based invalidation.
- C++20 `std::erase` / `std::erase_if` handle iterator management internally - prefer them.
- Address sanitizers (ASan) can detect use of invalidated iterators at runtime.

---

## Self-Assessment

### Q1: List which operations invalidate iterators for vector, deque, list, map, and unordered_map

This demonstration goes through each container and shows a concrete example where invalidation does or does not occur. Pay close attention to the deque case - it is the most counterintuitive:

```cpp
#include <iostream>
#include <vector>
#include <deque>
#include <list>
#include <map>
#include <unordered_map>

int main() {
    std::cout << "=== Iterator Invalidation Summary ===\n\n";

    // --- vector ---
    std::cout << "std::vector:\n";
    std::cout << "  push_back (no realloc): end() invalidated\n";
    std::cout << "  push_back (realloc):    ALL invalidated\n";
    std::cout << "  insert:                 at+after insertion (ALL if realloc)\n";
    std::cout << "  erase:                  at+after erasure point\n";
    std::cout << "  reserve (cap change):   ALL invalidated\n\n";

    // Demonstrate vector invalidation
    {
        std::vector<int> v = {1, 2, 3};
        v.reserve(100);  // Ensure no realloc
        auto it = v.begin() + 1;
        std::cout << "  Before insert: *it = " << *it << "\n";  // 2

        v.insert(v.begin(), 0);  // Insert before it -> it now points to 1 (shifted)
        // it is invalidated! Re-acquire:
        it = v.begin() + 2;
        std::cout << "  After insert(begin): *it = " << *it << "\n";  // 2

        // Erase at begin -> shifts everything left
        v.erase(v.begin());  // [1, 2, 3]
        it = v.begin() + 1;
        std::cout << "  After erase(begin): *it = " << *it << "\n\n";  // 2
    }

    // --- deque ---
    std::cout << "std::deque:\n";
    std::cout << "  push_back/push_front:  ALL iterators invalidated, refs/ptrs STABLE\n";
    std::cout << "  insert (middle):       ALL invalidated\n";
    std::cout << "  erase (front/back):    erased + end() invalidated\n";
    std::cout << "  erase (middle):        ALL invalidated\n\n";

    {
        std::deque<int> dq = {1, 2, 3};
        int& ref = dq[1];  // Reference to 2
        dq.push_back(4);   // Iterators invalidated, but ref is STILL VALID
        std::cout << "  Deque ref after push_back: " << ref << "\n\n";  // 2
    }

    // --- list ---
    std::cout << "std::list:\n";
    std::cout << "  insert/push_back/push_front: NOTHING invalidated\n";
    std::cout << "  erase:                       ONLY erased element\n";
    std::cout << "  splice:                      NOTHING invalidated\n\n";

    // --- map/set ---
    std::cout << "std::map / std::set:\n";
    std::cout << "  insert/emplace: NOTHING invalidated\n";
    std::cout << "  erase:          ONLY erased element\n\n";

    // --- unordered_map/set ---
    std::cout << "std::unordered_map / std::unordered_set:\n";
    std::cout << "  insert (no rehash):  NOTHING invalidated\n";
    std::cout << "  insert (rehash):     ALL iterators invalidated, refs/ptrs STABLE\n";
    std::cout << "  erase:               ONLY erased element\n";
    std::cout << "  rehash/reserve:      ALL iterators invalidated, refs/ptrs STABLE\n";

    return 0;
}
```

The deque example is worth re-reading: `ref` remains valid after `push_back` even though all *iterators* were invalidated. That asymmetry exists because deque's iterator type carries block-boundary bookkeeping, while a raw reference is just a pointer to the element's actual storage location.

**Explanation:**

- **Vector:** Most aggressive invalidation. Any insert/erase can invalidate iterators at and after the point of modification. Reallocation invalidates everything.
- **Deque:** Iterators are fragile (push_back/push_front invalidate all), but **references and pointers to existing elements remain valid** for push_back/push_front.
- **List:** Safest - only the erased element's iterator is invalidated. All others survive any operation.
- **Map/Set:** Node-based, similar stability to list. Insert never invalidates existing iterators.
- **Unordered containers:** Rehashing invalidates all iterators but preserves references/pointers.

### Q2: Write a loop that erases elements from a vector while iterating and show the correct update pattern

This is the classic "erase while iterating" problem. The wrong approach is common enough that it deserves explicit documentation. All four correct patterns are shown so you can pick the one that fits your context:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    // === WRONG: Incrementing after erase ===
    {
        std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};
        std::cout << "WRONG approach (may skip elements or crash):\n";
        // DO NOT DO THIS:
        // for (auto it = v.begin(); it != v.end(); ++it) {
        //     if (*it % 2 == 0) {
        //         v.erase(it);  // it is invalidated! ++it is UB
        //     }
        // }
        std::cout << "  (skipped - undefined behavior)\n\n";
    }

    // === CORRECT Pattern 1: Use return value of erase() ===
    {
        std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};
        std::cout << "Pattern 1 - erase() return value:\n";

        for (auto it = v.begin(); it != v.end(); /* no ++ here */) {
            if (*it % 2 == 0) {
                it = v.erase(it);  // erase returns iterator to NEXT element
            } else {
                ++it;              // Only increment if we didn't erase
            }
        }

        std::cout << "  Result: ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n\n";
        // Output: Result: 1 3 5 7
    }

    // === CORRECT Pattern 2: Erase-remove idiom (pre-C++20) ===
    {
        std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};
        std::cout << "Pattern 2 - erase-remove idiom:\n";

        v.erase(
            std::remove_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; }),
            v.end()
        );

        std::cout << "  Result: ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n\n";
        // Output: Result: 1 3 5 7
    }

    // === CORRECT Pattern 3: std::erase_if (C++20, recommended) ===
    {
        std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};
        std::cout << "Pattern 3 - std::erase_if (C++20):\n";

        auto removed = std::erase_if(v, [](int x) { return x % 2 == 0; });
        std::cout << "  Removed " << removed << " elements\n";

        std::cout << "  Result: ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n\n";
        // Output:
        //   Removed 4 elements
        //   Result: 1 3 5 7
    }

    // === CORRECT Pattern 4: Reverse iteration ===
    {
        std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};
        std::cout << "Pattern 4 - reverse index iteration:\n";

        for (int i = static_cast<int>(v.size()) - 1; i >= 0; --i) {
            if (v[i] % 2 == 0) {
                v.erase(v.begin() + i);
            }
        }

        std::cout << "  Result: ";
        for (int x : v) std::cout << x << " ";
        std::cout << "\n";
        // Output: Result: 1 3 5 7
    }

    return 0;
}
```

For new code, reach for Pattern 3 first - it's the clearest and most efficient. Pattern 1 is useful when you need to do something with the erased element before removing it. Pattern 4 works when you need an index for other reasons and want to keep the logic simple.

**How it works:**

- **Pattern 1:** `erase()` returns an iterator to the element after the erased one. Assign it back to `it` and skip the `++it`. This is the classic correct pattern.
- **Pattern 2:** `std::remove_if` moves non-matching elements to the front, then `erase` removes the tail. More efficient (O(n) moves vs O(n²) shifts).
- **Pattern 3:** C++20's `std::erase_if` wraps Pattern 2 in a single call. **Recommended** for new code.
- **Pattern 4:** Iterating in reverse avoids the shifting problem - erased elements are always at or after the current position.

### Q3: Explain why list iterators are never invalidated by insert/erase (except for the erased element)

The key is understanding that `std::list` stores each element in its own independently allocated node on the heap. Insertions and erasures change which nodes point to which - they never move a node. So any iterator you hold (which is essentially a pointer to a node) stays valid as long as that particular node hasn't been erased:

```cpp
#include <iostream>
#include <list>

int main() {
    std::list<int> l = {10, 20, 30, 40, 50};

    // Get iterators to various elements
    auto it10 = l.begin();                    // -> 10
    auto it30 = std::next(l.begin(), 2);      // -> 30
    auto it50 = std::prev(l.end());           // -> 50

    std::cout << "Before modifications:\n";
    std::cout << "  *it10 = " << *it10 << "\n";  // 10
    std::cout << "  *it30 = " << *it30 << "\n";  // 30
    std::cout << "  *it50 = " << *it50 << "\n";  // 50

    // --- Insert elements ---
    l.push_front(5);              // [5, 10, 20, 30, 40, 50]
    l.push_back(60);              // [5, 10, 20, 30, 40, 50, 60]
    l.insert(it30, 25);           // [5, 10, 20, 25, 30, 40, 50, 60]

    // All previous iterators are STILL VALID!
    std::cout << "\nAfter 3 insertions:\n";
    std::cout << "  *it10 = " << *it10 << "\n";  // 10 (valid!)
    std::cout << "  *it30 = " << *it30 << "\n";  // 30 (valid!)
    std::cout << "  *it50 = " << *it50 << "\n";  // 50 (valid!)

    // --- Erase a different element ---
    auto it40 = std::next(it30);  // -> 40
    l.erase(it40);                // [5, 10, 20, 25, 30, 50, 60]
    // Only it40 is invalidated. All other iterators survive!

    std::cout << "\nAfter erasing 40:\n";
    std::cout << "  *it10 = " << *it10 << "\n";  // 10 (valid!)
    std::cout << "  *it30 = " << *it30 << "\n";  // 30 (valid!)
    std::cout << "  *it50 = " << *it50 << "\n";  // 50 (valid!)
    // it40 is now DANGLING - do not use!

    // --- Why this works: node-based storage ---
    //
    // std::list node layout:
    //   [prev_ptr | DATA | next_ptr]
    //
    // Each element is in its own heap-allocated node.
    // Inserting a new element allocates a NEW node and adjusts
    // neighbor pointers - existing nodes are NEVER moved.
    //
    // list.insert(it30, 25):
    //   Before: ... [20|->] [30|->] ...
    //   After:  ... [20|->] [25|->] [30|->] ...
    //   Node [30] did NOT move - only its prev pointer changed.
    //   it30 still points to the same node at the same address.
    //
    // list.erase(it40):
    //   Deallocates the node for 40.
    //   Adjusts [30]'s next -> [50] and [50]'s prev -> [30].
    //   No other nodes are moved or deallocated.

    // Print final list
    std::cout << "\nFinal list: ";
    for (int x : l) std::cout << x << " ";
    std::cout << "\n";
    // Output: Final list: 5 10 20 25 30 50 60

    return 0;
}
```

After three insertions and one erasure, `it10`, `it30`, and `it50` are all still valid. Only `it40` is dangling. This is exactly the property that makes `std::list` the right choice when you need to hold iterators into a container that is being modified concurrently - or simply modified in a complex way where re-acquiring iterators would be awkward.

**Explanation:**

- `std::list` is a doubly-linked list where each element lives in its own independently allocated node.
- **Insert:** A new node is allocated and linked between existing nodes by adjusting `prev`/`next` pointers. Existing nodes don't move -> their iterators (which are pointers to nodes) remain valid.
- **Erase:** Only the erased node is deallocated. Adjacent nodes are relinked. No other node is affected.
- Contrast with `std::vector`: elements are stored contiguously. Insert shifts elements right (invalidating iterators to shifted positions). Reallocation copies everything to a new buffer (invalidating all iterators).
- This makes `std::list` ideal when you hold long-lived iterators/pointers to elements and need them to survive modifications to the container.

---

## Notes

- **Golden rule:** When using `std::vector`, re-acquire iterators after any modification unless you can guarantee no reallocation occurred.
- **`reserve()` trick:** After `v.reserve(n)`, `push_back` won't invalidate iterators as long as `size() < n`. But `insert()` still invalidates iterators at/after the insertion point.
- **`std::deque` surprise:** `push_back`/`push_front` invalidate all **iterators** but preserve all **references and pointers** to elements. This is because deque's iterator needs to track block boundaries, but the elements themselves don't move.
- **`std::unordered_map` rehash:** Invalidates iterators but **not** references/pointers. This is safe: `int& ref = map[key]; /* rehash happens */ ref = 42;` - `ref` is still valid.
- **Container adaptors** (`stack`, `queue`, `priority_queue`) don't expose iterators, so invalidation is not a direct concern.
- **C++20 `std::erase`/`std::erase_if`:** Always prefer these for erasing by value/predicate - they handle iterator management correctly and efficiently.
