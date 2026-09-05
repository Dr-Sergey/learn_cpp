# Use std::flat_map and flat_set in depth: construction, merge, and extract

**Category:** Standard Library New Cpp23 26  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/container/flat_map>  

---

## Topic Overview

`std::flat_map` and `std::flat_set` (C++23, `<flat_map>`, `<flat_set>`) are **sorted associative containers backed by contiguous storage** (typically `std::vector`). They give you the same interface as `std::map`/`std::set` but with a cache-friendly memory layout that can make a huge difference in read-heavy workloads.

The key tradeoff to understand up front is that flat containers pack keys and values into sorted vectors. That means lookups are fast binary searches over contiguous memory, but insertions and erasures are slower because elements have to be shifted. If you are writing a map that is built once and then read many times, `flat_map` is often the better choice. If elements are constantly being added and removed, `std::map` may win.

### flat_map vs map

| Aspect              | `std::map`                  | `std::flat_map`               |
| --- | --- | --- |
| **Storage**         | Node-based (per-element alloc) | Two sorted vectors (keys + values) |
| **Lookup**          | O(log n) tree traversal     | O(log n) binary search on contiguous memory |
| **Cache locality**  | Poor (pointer-chasing)      | Excellent (linear memory)     |
| **Insert/erase**    | O(log n) - no shifts        | O(n) - shifts elements        |
| **Iterator stability** | Stable                   | Invalidated on insert/erase   |
| **Best for**        | Frequent insert/erase       | Read-heavy, small-medium maps |

### Key Construction Tags

Two special tag values let you skip the internal sort when your data is already in order. This is one of the most valuable optimizations `flat_map` offers - if you know your data is sorted, tell the constructor so it does not waste time re-sorting it.

```cpp
std::sorted_unique   // keys are already sorted + unique -> skip internal sort
std::sorted_equivalent // for flat_multimap: already sorted
```

---

## Self-Assessment

### Q1: Construct a flat_map from a pre-sorted vector without a second sort pass using the sorted_unique tag

The four construction methods below range from the fastest (you guarantee sorted order) to the most convenient (you supply unsorted data and let the constructor handle it). Pay attention to `sorted_unique` - that tag is the payoff for keeping your data in order.

**Answer:**

<!-- compile: needs `flat_map` (not in the CI standard library yet) -->
```cpp
#include <flat_map>
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>

int main() {
    // Method 1: sorted_unique tag
    // Keys MUST already be sorted and unique — undefined behavior if not!
    std::vector<int> keys = {1, 3, 5, 7, 9};
    std::vector<std::string> vals = {"one", "three", "five", "seven", "nine"};

    // No sort performed — O(1) construction beyond the move
    std::flat_map<int, std::string> fm1(std::sorted_unique,
                                         std::move(keys),
                                         std::move(vals));

    for (const auto& [k, v] : fm1)
        std::cout << k << " -> " << v << '\n';
    // 1 -> one, 3 -> three, 5 -> five, 7 -> seven, 9 -> nine

    // Method 2: initializer_list (sorts internally)
    std::flat_map<int, std::string> fm2 = {
        {5, "five"}, {1, "one"}, {3, "three"}
    };
    // Internally sorts by key -> O(n log n)

    // Method 3: from unsorted vectors (sorts internally)
    std::vector<int> uk = {9, 3, 7, 1, 5};
    std::vector<std::string> uv = {"nine", "three", "seven", "one", "five"};
    std::flat_map<int, std::string> fm3(std::move(uk), std::move(uv));
    // Constructor sorts both vectors together by key

    // Method 4: custom comparator
    std::flat_map<int, std::string, std::greater<>> desc_map(
        std::sorted_unique,
        std::vector<int>{9, 7, 5, 3, 1},            // sorted descending!
        std::vector<std::string>{"nine", "seven", "five", "three", "one"}
    );
    for (const auto& [k, v] : desc_map)
        std::cout << k << " -> " << v << '\n';  // 9->nine, 7->seven, ...
}
```

Notice that `sorted_unique` requires you to guarantee the invariant - the constructor takes your word for it. If you pass keys that are not sorted and unique, the behavior is undefined. The payoff for keeping that promise is a significantly cheaper construction.

### Q2: Merge two flat_maps together and explain the invalidation behaviour

Merging is where `flat_map` reveals its most important gotcha: every insert or erase can invalidate all your iterators. This example walks through both the merge semantics and the invalidation rule. Do not skim the invalidation section - this is where bugs hide.

**Answer:**

<!-- compile: needs `flat_map` (not in the CI standard library yet) -->
```cpp
#include <flat_map>
#include <iostream>
#include <string>

int main() {
    std::flat_map<int, std::string> map1 = {
        {1, "alpha"}, {3, "gamma"}, {5, "epsilon"}
    };
    std::flat_map<int, std::string> map2 = {
        {2, "beta"}, {3, "GAMMA_v2"}, {4, "delta"}
    };

    // Merge: insert_range (C++23)
    // Elements from map2 are inserted into map1.
    // Duplicate keys (3) -> map1 keeps its existing value
    map1.insert(map2.begin(), map2.end());

    for (const auto& [k, v] : map1)
        std::cout << k << " -> " << v << '\n';
    // 1 -> alpha
    // 2 -> beta       (from map2)
    // 3 -> gamma      (map1's value preserved — NOT "GAMMA_v2")
    // 4 -> delta      (from map2)
    // 5 -> epsilon

    // Invalidation behavior
    // CRITICAL: All iterators, pointers, and references are invalidated
    // after any insert or erase because the underlying vector may reallocate.

    std::flat_map<int, std::string> fm = {{1, "a"}, {2, "b"}, {3, "c"}};
    auto it = fm.find(2);             // Iterator to key 2
    std::cout << it->second << '\n';  // "b" — valid here

    fm.insert({4, "d"});              // May reallocate the vector
    // it is NOW INVALID — do not dereference!
    // Must re-find: it = fm.find(2);

    // This is the fundamental tradeoff:
    // std::map: iterators survive insert/erase (node-based)
    // std::flat_map: iterators invalidated (vector-based)

    // merge() member function
    std::flat_map<int, std::string> src = {{10, "ten"}, {20, "twenty"}};
    std::flat_map<int, std::string> dst = {{15, "fifteen"}};
    dst.merge(src);
    // dst now has {10, 15, 20}; src is empty (elements moved out)
    for (const auto& [k, v] : dst)
        std::cout << k << " -> " << v << '\n';
}
```

The reason the invalidation rule trips people up is that `std::map` trains you to hold onto iterators across insertions. With `flat_map`, you need to re-query after any modification. The mental model is: treat it like a `std::vector` - any operation that could resize the storage is a potential invalidation.

### Q3: Extract the underlying sorted vector from a flat_map for bulk processing and reinsert it

Here is one of the most powerful things `flat_map` lets you do that `std::map` cannot: you can pull out the raw sorted vectors, do bulk processing on them directly (avoiding per-element overhead), then hand them back. This is the extract-modify-reinsert pattern.

**Answer:**

<!-- compile: needs `flat_map` (not in the CI standard library yet) -->
```cpp
#include <flat_map>
#include <vector>
#include <string>
#include <iostream>
#include <algorithm>

int main() {
    std::flat_map<int, std::string> fm = {
        {1, "one"}, {2, "two"}, {3, "three"}, {4, "four"}, {5, "five"}
    };

    // Extract: move out internal containers
    auto containers = std::move(fm).extract();
    // fm is now empty (moved-from state)

    auto& keys = containers.keys;     // std::vector<int>
    auto& values = containers.values; // std::vector<std::string>

    std::cout << "Extracted " << keys.size() << " entries\n";

    // Bulk processing on raw vectors
    // Much faster than individual flat_map operations for batch updates
    // Example: remove all entries where key is even
    auto should_remove = [](int k) { return k % 2 == 0; };

    // Erase corresponding values first (same indices)
    auto key_it = keys.begin();
    auto val_it = values.begin();
    while (key_it != keys.end()) {
        if (should_remove(*key_it)) {
            key_it = keys.erase(key_it);
            val_it = values.erase(val_it);
        } else {
            ++key_it;
            ++val_it;
        }
    }

    // Example: append new entries, then re-sort
    keys.push_back(10);
    values.push_back("ten");
    keys.push_back(7);
    values.push_back("seven");

    // Sort keys+values together by key
    // Use index-based sort to keep both vectors synchronized
    std::vector<size_t> indices(keys.size());
    std::iota(indices.begin(), indices.end(), 0);
    std::sort(indices.begin(), indices.end(),
              [&](size_t a, size_t b) { return keys[a] < keys[b]; });

    std::vector<int> sorted_keys;
    std::vector<std::string> sorted_vals;
    sorted_keys.reserve(keys.size());
    sorted_vals.reserve(values.size());
    for (size_t i : indices) {
        sorted_keys.push_back(std::move(keys[i]));
        sorted_vals.push_back(std::move(values[i]));
    }

    // Reinsert: construct from sorted data
    std::flat_map<int, std::string> fm2(
        std::sorted_unique,
        std::move(sorted_keys),
        std::move(sorted_vals)
    );

    for (const auto& [k, v] : fm2)
        std::cout << k << " -> " << v << '\n';
    // 1 -> one, 3 -> three, 5 -> five, 7 -> seven, 10 -> ten
    // (2 and 4 removed, 7 and 10 added)
}
```

The extract-and-reinsert pattern shines when you need to do something to a large fraction of the map's elements at once. Doing it through the normal map API would pay the O(n) shift cost on every individual operation. Working on the raw vectors and re-constructing with `sorted_unique` pays that cost just once at the end.

---

## Notes

- `std::flat_map` is in `<flat_map>` (C++23); available in MSVC 17.6+, libstdc++ 14, libc++ 18.
- Use the `sorted_unique` tag whenever your data is pre-sorted - it avoids the O(n log n) sort overhead.
- `extract()` returns a struct with `.keys` and `.values` - enables zero-copy bulk operations on the underlying vectors.
- For write-heavy workloads (more than around 30% inserts and erases), `std::map` may outperform `flat_map`.
- `std::flat_set` mirrors the API but stores only keys in a single sorted vector.
