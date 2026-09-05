# Understand flat_map and flat_set (C++23) and when they beat std::map

**Category:** Standard Library Containers  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/container/flat_map>  

---

## Topic Overview

`std::flat_map` and `std::flat_set` (C++23) store their elements in **sorted contiguous containers** (by default, two `std::vector`s for keys and values in `flat_map`). This gives them **excellent cache locality** for lookups (binary search on a sorted array) while providing the same ordered-map API as `std::map`.

The reason this matters is CPU cache behavior. `std::map` is a red-black tree: every lookup follows a chain of pointer dereferences to nodes scattered across the heap. Each pointer dereference is a potential cache miss. `flat_map` stores keys in a sorted vector and does a binary search on contiguous memory, so the CPU prefetcher can do its job. For read-heavy workloads that fit in cache, this difference is dramatic.

### Internal Structure Comparison

Here is a side-by-side picture of how the two containers lay out their data:

```cpp
std::map (red-black tree):
  Each node is a separate heap allocation:
  [Node: key|val|left|right|parent|color]
     /         \
  [Node]      [Node]
  Pointer chasing -> cache misses on every comparison

std::flat_map (sorted vectors):
  Keys:   [alice, bob, carol, dave]   <- contiguous
  Values: [95,    87,  92,    88]     <- contiguous
  Binary search -> sequential memory access -> cache-friendly
```

The key insight is that `flat_map` separates keys and values into two parallel vectors. That means during a binary search you only touch the keys vector - the values never enter the cache until you've found the entry you want.

### Performance Comparison

| Operation          | `std::map`      | `std::flat_map`         |
| --- | --- | --- |
| `find()` / lookup | O(log n), many cache misses | O(log n), **cache-friendly** |
| `insert()`        | O(log n) (alloc node) | **O(n)** (shift elements) |
| `erase()`         | O(log n)        | **O(n)** (shift elements) |
| Iteration          | O(n), pointer chasing | O(n), **contiguous** |
| Memory overhead    | ~3-4 pointers/node | Near-zero overhead |
| Memory locality    | Poor (scattered nodes) | **Excellent** |

Notice that insert and erase are O(n) for `flat_map` - that's the cost of shifting elements in a vector to keep the sorted order. This is the trade-off: fast lookups and iteration, but expensive structural modifications.

### When to Use flat_map

- **Read-heavy, write-infrequent:** Build once, query many times.
- **Small to medium sizes:** (< ~100K elements) where cache effects dominate.
- **Memory constrained:** Much lower memory overhead than tree nodes.
- **Sorted iteration needed:** Iterating a flat_map is like iterating a sorted vector.

### When NOT to Use flat_map

- **Frequent insertions/deletions:** O(n) insert/erase due to element shifting.
- **Very large collections:** The crossover point where tree O(log n) insertion beats flat_map O(n).
- **Need stable iterators:** flat_map invalidates all iterators on every insert/erase.

### Core API (C++23)

The API is intentionally nearly identical to `std::map`, so switching is low-friction. Notice the extra `keys()` and `values()` accessors that give you direct access to the underlying vectors:

<!-- compile: needs `flat_map` (not in the CI standard library yet) -->
```cpp
#include <iostream>
#include <flat_map>  // C++23
#include <string>

int main() {
    std::flat_map<std::string, int> scores;

    scores["alice"] = 95;
    scores["bob"] = 87;
    scores.insert({"carol", 92});
    scores.try_emplace("dave", 88);

    // Lookup (binary search on sorted keys vector)
    if (auto it = scores.find("bob"); it != scores.end()) {
        std::cout << "bob: " << it->second << "\n";
    }
    // Output: bob: 87

    // Iteration (contiguous, sorted)
    for (const auto& [name, score] : scores) {
        std::cout << name << ": " << score << "\n";
    }
    // Output (sorted):
    // alice: 95
    // bob: 87
    // carol: 92
    // dave: 88

    // Access underlying containers
    const auto& keys = scores.keys();    // const reference to sorted vector of keys
    const auto& vals = scores.values();  // const reference to vector of values

    return 0;
}
```

The `keys()` and `values()` accessors are unique to `flat_map` - they give you direct read access to the underlying vectors when you need to pass them to algorithms that expect contiguous ranges.

### Important Notes

- `std::flat_map` is an **adaptor** over two sequence containers (default: `std::vector`). You can customize: `std::flat_map<K, V, Compare, KeyContainer, ValueContainer>`.
- All iterators are invalidated by insert/erase (because vectors may reallocate or shift).
- `flat_set` is similar but stores only keys (single sorted vector).
- Available in GCC 13+, Clang 17+, MSVC 17.8+ (with `/std:c++latest`).

---

## Self-Assessment

### Q1: Explain why flat_map has better cache locality than std::map for small-to-medium sizes

This example simulates a `flat_map` with a sorted vector so you can benchmark the difference in pre-C++23 environments. The important thing to watch is the lookup time, not the build time:

```cpp
#include <iostream>
#include <map>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>
#include <string>
#include <numeric>
#include <utility>

// Simulating flat_map with a sorted vector for pre-C++23
template <typename K, typename V>
class SimpleFlatMap {
    std::vector<std::pair<K, V>> data_;
public:
    void insert(const K& k, const V& v) {
        auto it = std::lower_bound(data_.begin(), data_.end(), k,
            [](const auto& p, const K& key) { return p.first < key; });
        if (it != data_.end() && it->first == k) {
            it->second = v;
        } else {
            data_.insert(it, {k, v});  // O(n) due to shifting
        }
    }

    const V* find(const K& k) const {
        auto it = std::lower_bound(data_.begin(), data_.end(), k,
            [](const auto& p, const K& key) { return p.first < key; });
        if (it != data_.end() && it->first == k) return &it->second;
        return nullptr;
    }

    size_t size() const { return data_.size(); }
};

int main() {
    constexpr int N = 10'000;
    constexpr int LOOKUPS = 100'000;

    std::mt19937 rng(42);
    std::vector<int> keys(N);
    std::iota(keys.begin(), keys.end(), 0);
    std::shuffle(keys.begin(), keys.end(), rng);

    // Build both containers
    std::map<int, int> tree_map;
    SimpleFlatMap<int, int> flat_map;

    for (int k : keys) {
        tree_map[k] = k * 10;
        flat_map.insert(k, k * 10);
    }

    // Generate random lookup keys
    std::uniform_int_distribution<int> dist(0, N - 1);
    std::vector<int> lookup_keys(LOOKUPS);
    for (auto& k : lookup_keys) k = dist(rng);

    // Benchmark: std::map lookups
    {
        volatile int sum = 0;
        auto start = std::chrono::high_resolution_clock::now();
        for (int k : lookup_keys) {
            auto it = tree_map.find(k);
            if (it != tree_map.end()) sum += it->second;
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "std::map        " << LOOKUPS << " lookups: " << dur.count() << " us\n";
    }

    // Benchmark: flat_map lookups
    {
        volatile int sum = 0;
        auto start = std::chrono::high_resolution_clock::now();
        for (int k : lookup_keys) {
            auto* val = flat_map.find(k);
            if (val) sum += *val;
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "flat_map (vec)  " << LOOKUPS << " lookups: " << dur.count() << " us\n";
    }
    // Expected: flat_map 2-5x faster for N=10,000

    std::cout << "\nWhy flat_map wins:\n";
    std::cout << "  std::map: each node is a separate allocation.\n";
    std::cout << "    Binary search traverses nodes scattered in memory.\n";
    std::cout << "    Each comparison = pointer dereference = potential cache miss.\n";
    std::cout << "  flat_map: keys stored contiguously in a sorted vector.\n";
    std::cout << "    Binary search accesses sequential memory locations.\n";
    std::cout << "    CPU prefetcher can predict access patterns.\n";
    std::cout << "    Fewer cache misses per lookup.\n";

    return 0;
}
```

At N=10,000, you should see `flat_map` win by a factor of 2-5x. The gap narrows as N grows and the dataset stops fitting in L1/L2 cache.

**Explanation:**

- **`std::map`:** Implemented as a red-black tree. Each node is a separate heap allocation at a random memory address. During `find()`, each comparison requires following a pointer to a child node -> potential L1/L2 cache miss.
- **`flat_map`:** Keys are stored in a sorted `std::vector`. Binary search accesses elements at `data[mid]`, `data[mid/2]`, etc. - all within the same contiguous array. The CPU prefetcher can predict and preload these locations.
- For `N < 100K`, the cache advantage of contiguous storage typically outweighs the O(n) insertion cost, making flat_map 2-5x faster for lookups.

### Q2: Show the performance crossover point between flat_map and std::map for lookup-heavy workloads

The crossover is the key design question: at what N does `std::map` catch up? The answer depends on key size, value size, and how often you insert vs. look up. This benchmark sweeps across multiple sizes to give you a feel for the curve:

```cpp
#include <iostream>
#include <map>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>
#include <numeric>

template <typename K, typename V>
class FlatMap {
    std::vector<std::pair<K, V>> data_;
public:
    void reserve(size_t n) { data_.reserve(n); }

    void sorted_insert(const K& k, const V& v) {
        auto it = std::lower_bound(data_.begin(), data_.end(), k,
            [](const auto& p, const K& key) { return p.first < key; });
        data_.insert(it, {k, v});
    }

    bool find(const K& k) const {
        auto it = std::lower_bound(data_.begin(), data_.end(), k,
            [](const auto& p, const K& key) { return p.first < key; });
        return it != data_.end() && it->first == k;
    }
};

void benchmark_size(int N) {
    std::mt19937 rng(42);
    std::vector<int> keys(N);
    std::iota(keys.begin(), keys.end(), 0);
    std::shuffle(keys.begin(), keys.end(), rng);

    // Build containers
    std::map<int, int> m;
    FlatMap<int, int> fm;
    fm.reserve(N);
    for (int k : keys) {
        m[k] = k;
        fm.sorted_insert(k, k);
    }

    constexpr int REPS = 10;
    int lookups = std::max(100'000, N * 10);
    std::uniform_int_distribution<int> dist(0, N - 1);
    std::vector<int> lk(lookups);
    for (auto& k : lk) k = dist(rng);

    // Benchmark map
    auto t_map = [&] {
        auto start = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < REPS; ++r)
            for (int k : lk) m.find(k);
        auto end = std::chrono::high_resolution_clock::now();
        return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    }();

    // Benchmark flat_map
    auto t_flat = [&] {
        auto start = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < REPS; ++r)
            for (int k : lk) fm.find(k);
        auto end = std::chrono::high_resolution_clock::now();
        return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    }();

    double ratio = static_cast<double>(t_map) / t_flat;
    std::cout << "N=" << N
              << "\tmap: " << t_map << " us"
              << "\tflat: " << t_flat << " us"
              << "\tratio: " << ratio << "x"
              << (ratio > 1.0 ? " (flat wins)" : " (map wins)") << "\n";
}

int main() {
    std::cout << "Lookup performance: std::map vs flat_map\n\n";
    for (int n : {100, 500, 1000, 5000, 10'000, 50'000, 100'000, 500'000}) {
        benchmark_size(n);
    }
    // Expected pattern:
    // N=100:    flat wins by 5-10x
    // N=1000:   flat wins by 3-5x
    // N=10000:  flat wins by 2-3x
    // N=100000: flat wins by 1.5-2x
    // N=500000: roughly equal or map starts winning
    // (exact crossover depends on key/value size and hardware)

    std::cout << "\nCrossover factors:\n";
    std::cout << "  - Key size: larger keys shift crossover to smaller N\n";
    std::cout << "  - Value size: larger values make insertion shifting more expensive\n";
    std::cout << "  - Insert frequency: frequent inserts favor map\n";
    std::cout << "  - L1/L2/L3 cache sizes: larger caches favor flat_map at higher N\n";

    return 0;
}
```

On modern desktop hardware you'll typically see the crossover for lookup-only workloads somewhere around N=500K. If you mix in frequent inserts, the crossover drops dramatically - sometimes as low as N=1K-10K.

**How it works:**

- For **small N** (< 1K), flat_map dominates: the entire sorted array fits in L1/L2 cache.
- For **medium N** (1K-100K), flat_map still wins: binary search on contiguous data beats tree pointer chasing.
- For **large N** (> 100K-500K), the advantage narrows. At some point, the O(n) insertion cost of flat_map makes it impractical if insertions are frequent.
- The **crossover point for lookups alone** is typically around 500K-1M elements on modern hardware.
- The **crossover for insert-heavy workloads** is much lower (around 1K-10K).

### Q3: Explain the invalidation rules for flat_map iterators after insertions

This is one of the most important behavioral differences from `std::map`. Because `flat_map` is backed by vectors, any structural modification can reallocate or shift memory, invalidating every iterator you held. The example walks through the correct pattern for re-acquiring iterators:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

// Simulating flat_map to demonstrate iterator invalidation
template <typename K, typename V>
class FlatMap {
public:
    std::vector<K> keys;
    std::vector<V> values;

    using iterator = typename std::vector<K>::iterator;

    void insert(const K& k, const V& v) {
        auto it = std::lower_bound(keys.begin(), keys.end(), k);
        auto idx = std::distance(keys.begin(), it);
        keys.insert(it, k);
        values.insert(values.begin() + idx, v);
    }

    void erase(const K& k) {
        auto it = std::lower_bound(keys.begin(), keys.end(), k);
        if (it != keys.end() && *it == k) {
            auto idx = std::distance(keys.begin(), it);
            keys.erase(it);
            values.erase(values.begin() + idx);
        }
    }
};

int main() {
    FlatMap<std::string, int> fm;
    fm.insert("alice", 95);
    fm.insert("bob", 87);
    fm.insert("carol", 92);

    // Get an iterator to "bob"
    auto it = std::lower_bound(fm.keys.begin(), fm.keys.end(), std::string("bob"));
    std::cout << "Before insert: *it = " << *it << "\n";
    // Output: Before insert: *it = bob

    // Insert "dave" — this may reallocate the vectors!
    fm.insert("dave", 88);

    // 'it' is NOW INVALID if the vector reallocated
    // Accessing *it is UNDEFINED BEHAVIOR

    // Correct pattern: re-acquire iterator after modification
    it = std::lower_bound(fm.keys.begin(), fm.keys.end(), std::string("bob"));
    std::cout << "After re-acquire: *it = " << *it << "\n";
    // Output: After re-acquire: *it = bob

    // === Invalidation rules for flat_map ===
    std::cout << "\nIterator invalidation rules:\n";
    std::cout << "  insert(): ALL iterators invalidated\n";
    std::cout << "    - Vector may reallocate (invalidates all)\n";
    std::cout << "    - Elements shift right (changes positions)\n";
    std::cout << "  erase(): ALL iterators invalidated\n";
    std::cout << "    - Elements shift left (changes positions)\n";
    std::cout << "  find()/lookups: No invalidation\n";
    std::cout << "  operator[]: May insert -> invalidates all if key is new\n";

    // === Comparison with std::map ===
    std::cout << "\nComparison:\n";
    std::cout << "Container      | insert     | erase        | find\n";
    std::cout << "---------------|------------|--------------|--------\n";
    std::cout << "std::map       | No inval*  | Only erased  | No inval\n";
    std::cout << "std::flat_map  | ALL inval  | ALL inval    | No inval\n";
    std::cout << "std::unord_map | May rehash | Only erased  | No inval\n";
    std::cout << "  * map: no iterator invalidation on insert\n";

    // === Safe iteration pattern ===
    std::cout << "\nSafe pattern for modifying flat_map during iteration:\n";
    std::cout << "  Collect indices/keys to modify, then apply changes after loop.\n";

    // Example: collect keys to erase, then erase after iteration
    std::vector<std::string> to_erase;
    for (size_t i = 0; i < fm.keys.size(); ++i) {
        if (fm.values[i] < 90) {
            to_erase.push_back(fm.keys[i]);
        }
    }
    for (const auto& k : to_erase) {
        fm.erase(k);
    }

    std::cout << "\nAfter erasing scores < 90:\n";
    for (size_t i = 0; i < fm.keys.size(); ++i) {
        std::cout << "  " << fm.keys[i] << ": " << fm.values[i] << "\n";
    }
    // Output:
    //   alice: 95
    //   carol: 92

    return 0;
}
```

The collect-then-erase pattern at the end is the safe idiom: finish your read pass first, then apply all modifications once you no longer need the iterators.

**Explanation:**

- `std::flat_map` stores keys and values in `std::vector`s. Any structural modification (insert/erase) can:
  1. **Reallocate** the vectors (when capacity is exceeded) -> all iterators, pointers, references invalidated.
  2. **Shift elements** within the vectors (to maintain sorted order) -> iterators to shifted elements point to different elements.
- This is fundamentally different from `std::map`, where tree nodes are independent allocations - inserting a new node doesn't affect existing node addresses.
- **Safe pattern:** Never hold iterators across insert/erase calls. Re-acquire iterators after any modification. Or collect modifications and apply them batch-wise.

---

## Notes

- **`std::flat_map` is ideal for:** Configuration data, lookup tables, dictionaries built once and queried frequently.
- **Underlying containers can be customized:** `std::flat_map<K, V, Compare, std::deque<K>, std::deque<V>>` uses deques instead of vectors.
- **`sorted_unique_t` constructor:** `std::flat_map fm(std::sorted_unique, keys_vec, vals_vec)` constructs from pre-sorted data in O(1) (no re-sorting).
- **`extract()` (C++23):** Returns the underlying containers for direct manipulation, then `replace()` puts them back.
- **`std::flat_multimap` and `std::flat_multiset`:** Allow duplicate keys, also in C++23.
- **Tip:** If you build a flat_map from unsorted data, use `flat_map(first, last)` which sorts once at construction - much faster than N individual `insert()` calls.
