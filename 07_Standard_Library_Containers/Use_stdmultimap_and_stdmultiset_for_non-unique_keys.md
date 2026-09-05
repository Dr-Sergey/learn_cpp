# Use std::multimap and std::multiset for non-unique keys

**Category:** Standard Library Containers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/container/multimap>  

---

## Topic Overview

`std::multimap` and `std::multiset` are ordered associative containers that allow **duplicate keys**. Unlike `std::map`/`std::set`, which enforce unique keys, multi-containers store every inserted element even when keys collide. You use them when the one-entry-per-key constraint of their non-multi counterparts would throw away data you want to keep.

### multimap vs map

| Feature | `std::map<K,V>` | `std::multimap<K,V>` |
| --- | --- | --- |
| Duplicate keys | No (insert fails) | Yes (all kept) |
| `operator[]` | Yes | **No** (which value?) |
| `at()` | Yes | **No** |
| `insert_or_assign` | Yes (C++17) | **No** |
| `try_emplace` | Yes (C++17) | **No** |
| `count(key)` | 0 or 1 | 0, 1, 2, ... |
| `equal_range(key)` | 0 or 1 elements | range of all matches |
| Ordering | Sorted by key | Sorted by key (insertion order within equal keys) |

Notice that `operator[]` and `at()` are absent from `multimap`. The reason is simple: if a key maps to three values, there is no meaningful single value to return. You must use `equal_range` or `find` instead.

### multiset vs set

| Feature | `std::set<T>` | `std::multiset<T>` |
| --- | --- | --- |
| Duplicate values | No | Yes |
| `count(val)` | 0 or 1 | 0, 1, 2, ... |
| `erase(val)` | Removes 1 | **Removes ALL** matching |

### Core API

Here is the fundamental usage for both containers:

```cpp
#include <map>
#include <set>
#include <iostream>
#include <string>

int main() {
    // === multimap: multiple values per key ===
    std::multimap<std::string, int> scores;
    scores.insert({"Alice", 90});
    scores.insert({"Alice", 95});
    scores.insert({"Alice", 87});
    scores.insert({"Bob", 88});
    scores.emplace("Bob", 92);

    std::cout << "Alice entries: " << scores.count("Alice") << "\n";  // 3
    std::cout << "Bob entries: "   << scores.count("Bob")   << "\n";  // 2

    // Iterate all values for a key:
    auto [lo, hi] = scores.equal_range("Alice");
    for (auto it = lo; it != hi; ++it)
        std::cout << it->first << ": " << it->second << "\n";
    // Alice: 87  (sorted? NO - insertion order within equal keys is preserved)
    // Actually: multimap keeps key-equivalent elements in insertion order (C++11+)

    // === multiset: duplicate values ===
    std::multiset<int> ms = {5, 3, 5, 1, 3, 3};
    std::cout << "count(3): " << ms.count(3) << "\n";  // 3
    std::cout << "count(5): " << ms.count(5) << "\n";  // 2
    std::cout << "size: "     << ms.size()    << "\n";  // 6

    // erase ONE instance of 3:
    auto it = ms.find(3);
    if (it != ms.end()) ms.erase(it);  // removes single element
    std::cout << "count(3) after erase(it): " << ms.count(3) << "\n";  // 2

    // erase ALL instances of 5:
    ms.erase(5);  // removes by value -> removes ALL matching
    std::cout << "count(5) after erase(5): " << ms.count(5) << "\n";  // 0

    return 0;
}
```

The erase-by-iterator vs erase-by-value distinction is important to remember: `erase(iterator)` removes one element, while `erase(key)` removes every element with that key. Getting this wrong is a common source of bugs.

---

## Self-Assessment

### Q1: Use equal_range on a multimap to iterate all values for a given key

`equal_range` is the primary tool for working with multimap. It returns an iterator pair bounding all entries for a given key in one O(log n) call:

```cpp
#include <iostream>
#include <map>
#include <string>
#include <vector>

int main() {
    // === Student grade tracker: one student -> multiple grades ===
    std::multimap<std::string, int> grades;

    grades.insert({"Alice", 92});
    grades.insert({"Bob", 78});
    grades.insert({"Alice", 88});
    grades.insert({"Charlie", 95});
    grades.insert({"Alice", 76});
    grades.insert({"Bob", 85});

    // === Use equal_range to find all grades for "Alice" ===
    auto [begin, end] = grades.equal_range("Alice");

    std::cout << "Alice's grades: ";
    double sum = 0;
    int count = 0;
    for (auto it = begin; it != end; ++it) {
        std::cout << it->second << " ";
        sum += it->second;
        ++count;
    }
    std::cout << "\nAverage: " << sum / count << "\n";
    // Output:
    // Alice's grades: 92 88 76
    // Average: 85.3333

    // === Iterate ALL entries grouped by key ===
    std::cout << "\nAll students:\n";
    for (auto it = grades.begin(); it != grades.end(); ) {
        auto [range_begin, range_end] = grades.equal_range(it->first);
        std::cout << it->first << ": ";
        for (auto r = range_begin; r != range_end; ++r)
            std::cout << r->second << " ";
        std::cout << "\n";
        it = range_end;  // Jump past the group
    }
    // Output:
    // Alice: 92 88 76
    // Bob: 78 85
    // Charlie: 95

    // === Check for missing key ===
    auto [mb, me] = grades.equal_range("Dave");
    if (mb == me)
        std::cout << "\nDave has no grades.\n";
    // equal_range returns [end, end) for missing keys -> empty range

    return 0;
}
```

The grouped iteration pattern using `it = range_end` to jump past each key group is idiomatic for traversing a multimap without redundant lookups. When `equal_range` returns an empty range (`first == second`), the key simply doesn't exist.

### Q2: Show the difference between lower_bound/upper_bound iteration and equal_range on multimap

`equal_range` is sugar for a `{lower_bound, upper_bound}` pair, but `lower_bound` and `upper_bound` are individually useful for arbitrary range queries that do not start and end at the same key:

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::multimap<int, std::string> events;
    events.insert({10, "Start"});
    events.insert({20, "Login"});
    events.insert({20, "Upload"});
    events.insert({20, "Download"});
    events.insert({30, "Logout"});
    events.insert({40, "Stop"});

    int key = 20;

    // === Method 1: lower_bound + upper_bound ===
    // lower_bound(key): first element NOT LESS than key  (>=)
    // upper_bound(key): first element GREATER than key   (>)
    auto lb = events.lower_bound(key);
    auto ub = events.upper_bound(key);

    std::cout << "Method 1 (lower_bound/upper_bound):\n";
    for (auto it = lb; it != ub; ++it)
        std::cout << "  " << it->first << ": " << it->second << "\n";
    // 20: Login
    // 20: Upload
    // 20: Download

    // === Method 2: equal_range ===
    // equivalent to {lower_bound(key), upper_bound(key)}
    auto [er_begin, er_end] = events.equal_range(key);

    std::cout << "\nMethod 2 (equal_range):\n";
    for (auto it = er_begin; it != er_end; ++it)
        std::cout << "  " << it->first << ": " << it->second << "\n";
    // Same output

    // === They produce IDENTICAL results ===
    std::cout << "\nlower_bound == equal_range.first? "
              << (lb == er_begin ? "yes" : "no") << "\n";  // yes
    std::cout << "upper_bound == equal_range.second? "
              << (ub == er_end ? "yes" : "no") << "\n";    // yes

    // === When are they different? ===
    // Performance: equal_range does ONE traversal, finding both bounds together.
    // lower_bound + upper_bound = TWO separate traversals.
    //
    // However: in practice, the difference is negligible for small trees.
    // equal_range is clearer and preferred for "find all with this key".

    // === Bonus: use lower_bound alone for range queries ===
    // "All events from time 15 to 35"
    auto from = events.lower_bound(15);  // points to {20, "Login"}
    auto to   = events.lower_bound(35);  // points to {40, "Stop"}

    std::cout << "\nEvents from 15 to 35:\n";
    for (auto it = from; it != to; ++it)
        std::cout << "  " << it->first << ": " << it->second << "\n";
    // 20: Login
    // 20: Upload
    // 20: Download
    // 30: Logout

    return 0;
}
```

Use `equal_range` when you want all entries for exactly one key. Use `lower_bound` + `lower_bound` when you want everything between two key values. The range query at the bottom (`from time 15 to 35`) is a case where `equal_range` would not help because you are spanning multiple keys.

### Q3: Explain when unordered_multimap outperforms multimap and vice versa

```cpp
#include <iostream>
#include <map>
#include <unordered_map>
#include <string>
#include <chrono>
#include <vector>
#include <random>

// === Performance comparison ===
int main() {
    constexpr int N = 100'000;
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, N / 10);

    // Generate random keys (with duplicates)
    std::vector<int> keys(N);
    for (auto& k : keys) k = dist(rng);

    // === Insert benchmark ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        std::multimap<int, int> mm;
        for (int i = 0; i < N; ++i) mm.insert({keys[i], i});
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "multimap insert: " << ms << " us\n";
    }
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        std::unordered_multimap<int, int> umm;
        umm.reserve(N);
        for (int i = 0; i < N; ++i) umm.insert({keys[i], i});
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "unordered_multimap insert: " << ms << " us\n";
    }

    // === Lookup benchmark ===
    std::multimap<int, int> mm;
    std::unordered_multimap<int, int> umm;
    umm.reserve(N);
    for (int i = 0; i < N; ++i) {
        mm.insert({keys[i], i});
        umm.insert({keys[i], i});
    }
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        long sum = 0;
        for (int i = 0; i < N; ++i) sum += mm.count(keys[i]);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "multimap lookup: " << ms << " us (sum=" << sum << ")\n";
    }
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        long sum = 0;
        for (int i = 0; i < N; ++i) sum += umm.count(keys[i]);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "unordered_multimap lookup: " << ms << " us (sum=" << sum << ")\n";
    }

    return 0;
}
```

The benchmark shows raw lookup speed, but the right container depends on what your code needs to do with the data:

| Factor | `multimap` | `unordered_multimap` |
| --- | --- | --- |
| **Lookup** | O(log n) | O(1) average, O(n) worst |
| **Insert** | O(log n) | O(1) average, O(n) rehash |
| **Ordering** | Sorted by key | No ordering |
| **Range queries** | Excellent (lower_bound/upper_bound) | Not possible efficiently |
| **Cache behavior** | Poor (node-based, scattered memory) | Better (bucket array) |
| **Iterator stability** | All iterators valid after insert/erase | May invalidate on rehash |
| **Memory overhead** | 3 pointers per node + tree overhead | Hash table + bucket list |
| **Many duplicates** | O(log n + k) for equal_range (k = count) | All duplicates in same bucket -> O(k) |

**Choose `unordered_multimap` when:**

- You need only equality-based lookup (no ordering needed)
- Keys have a good hash function (integers, strings)
- Dataset is large and O(1) average lookup dominates

**Choose `multimap` when:**

- You need sorted iteration or range queries (`lower_bound`, `upper_bound`)
- You need stable iterators (unordered invalidates on rehash)
- Key type has no natural hash function but has `operator<`
- Dataset is small (tree overhead negligible, cache difference minimal)

---

## Notes

- **`erase(key)` on multimap/multiset removes ALL elements with that key.** To remove just one, use `erase(find(key))`.
- **`count(key)` is O(log n + k)** where k is the number of duplicates - it must traverse all matches.
- **C++17 `extract()`** works on multi-containers: `auto nh = mm.extract(mm.find(key));` extracts one node.
- **C++17 `merge()`** transfers nodes between containers without copying. Works across multi/non-multi containers.
- Elements with equal keys maintain **insertion order** (guaranteed since C++11).
- `multiset` can be used as a sorted bag or frequency counter: `count(x)` gives you the frequency of any value.
