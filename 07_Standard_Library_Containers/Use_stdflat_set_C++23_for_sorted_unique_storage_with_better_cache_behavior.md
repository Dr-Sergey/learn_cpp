# Use std::flat_set (C++23) for sorted unique storage with better cache behavior

**Category:** Standard Library Containers  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/container/flat_set>  

---

## Topic Overview

`std::flat_set` (C++23) is a sorted associative container adaptor that stores unique elements in a **sorted contiguous sequence** (by default `std::vector`). It provides the same interface as `std::set` but uses a flat sorted array instead of a red-black tree. The trade-off is simple: lookups and iteration get faster (sometimes dramatically so), while individual inserts and erases get slower.

### flat_set vs std::set - Internal Structure

The difference in structure explains every performance characteristic that follows. `std::set` builds a tree where each node is a separate heap allocation. `std::flat_set` keeps everything in one contiguous block:

```cpp
std::set<int> (red-black tree):        std::flat_set<int> (sorted vector):

      [5]                              Memory: [1][3][5][7][9]  <- contiguous!

     /   \
   [3]   [7]    <- each node is        Binary search over contiguous array
   / \   / \       a separate heap     -> cache-friendly lookups
 [1] [4][6] [9]   allocation           -> O(log n) with excellent cache behavior
```

### Performance Comparison

| Operation | `std::set` | `std::flat_set` | Why |
| --- | --- | --- | --- |
| Lookup | O(log n) - tree walk | O(log n) - binary search | flat_set faster: cache-friendly contiguous memory |
| Insert | O(log n) - tree rebalance | O(n) - shift elements | set faster: O(log n) vs O(n) |
| Erase | O(log n) - tree rebalance | O(n) - shift elements | set faster |
| Iteration | O(n) - pointer chasing | O(n) - sequential scan | flat_set much faster: no cache misses |
| Memory per element | ~40-64 bytes (node + pointers) | sizeof(T) only | flat_set: 8-16x less memory |
| Iterator stability | Stable (never invalidated by insert/erase of others) | ALL invalidated on insert/erase | set much better |

### When to Use flat_set

- **Read-heavy workloads:** Many lookups, few inserts or erases - flat_set wins here.
- **Build once, query many times:** Fill with data once, then only search - ideal for flat_set.
- **Small to medium sets (< ~10K elements):** O(n) insert is fast for small n.
- **Memory-constrained:** flat_set uses dramatically less memory than a tree-based set.
- **Iteration-heavy:** Sequential memory access is much faster than following pointers.

### When to Use std::set

- **Frequent insert/erase:** O(log n) vs O(n) makes a real difference above ~1000 elements.
- **Iterator stability required:** flat_set invalidates ALL iterators on mutation.
- **Need `extract()`/`merge()`:** Only available on node-based containers.

### Core Usage

The interface is intentionally identical to `std::set` - you can often swap one for the other with a single type change:

<!-- compile: needs `flat_set` (not in the CI standard library yet) -->
```cpp
// Note: requires C++23 compiler with <flat_set> support
#include <iostream>
#include <flat_set>
#include <vector>

int main() {
    // Basic usage - same API as std::set
    std::flat_set<int> fs = {5, 3, 8, 1, 9, 3};
    // Internal: sorted vector [1, 3, 5, 8, 9] (duplicate 3 removed)

    fs.insert(4);    // O(n): shifts elements to maintain sorted order
    fs.erase(8);     // O(n): shifts elements
    bool has = fs.contains(5);  // O(log n): binary search

    // Iterate (fast! contiguous memory)
    for (int x : fs)
        std::cout << x << " ";  // 1 3 4 5 9
    std::cout << "\n";

    // Access underlying container
    // const auto& vec = fs.extract();  // Get the sorted vector

    return 0;
}
```

---

## Self-Assessment

### Q1: Replace a std::set<int> with std::flat_set<int> and measure lookup speedup for hot workloads

Since `std::flat_set` may not be available on all compilers yet, this example simulates it with a sorted `std::vector` so you can observe the cache difference yourself:

```cpp
#include <iostream>
#include <set>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>

// Simulate flat_set with sorted vector (for compilers without C++23 <flat_set>)
template <typename T>
class SimpleFlatSet {
    std::vector<T> data_;
public:
    void insert(const T& val) {
        auto it = std::lower_bound(data_.begin(), data_.end(), val);
        if (it == data_.end() || *it != val)
            data_.insert(it, val);
    }

    bool contains(const T& val) const {
        return std::binary_search(data_.begin(), data_.end(), val);
    }

    size_t size() const { return data_.size(); }

    void reserve(size_t n) { data_.reserve(n); }

    // Build from sorted range (much faster than individual inserts)
    template <typename It>
    void assign_sorted(It first, It last) {
        data_.assign(first, last);
    }

    auto begin() const { return data_.begin(); }
    auto end() const { return data_.end(); }
};

int main() {
    constexpr int N = 100'000;
    constexpr int LOOKUPS = 1'000'000;

    // Generate random data
    std::mt19937 rng(42);
    std::vector<int> data(N);
    for (int& x : data) x = rng() % (N * 10);

    // Build std::set
    std::set<int> s(data.begin(), data.end());

    // Build flat_set (sorted vector)
    std::vector<int> sorted_data(data.begin(), data.end());
    std::sort(sorted_data.begin(), sorted_data.end());
    sorted_data.erase(std::unique(sorted_data.begin(), sorted_data.end()),
                      sorted_data.end());

    SimpleFlatSet<int> fs;
    fs.assign_sorted(sorted_data.begin(), sorted_data.end());

    std::cout << "Elements: " << s.size() << " (set) vs "
              << fs.size() << " (flat_set)\n";

    // Generate lookup keys (mix of hits and misses)
    std::vector<int> queries(LOOKUPS);
    for (int& q : queries) q = rng() % (N * 10);

    // === Benchmark: std::set lookups ===
    {
        auto start = std::chrono::steady_clock::now();
        volatile int found = 0;
        for (int q : queries) {
            if (s.count(q)) ++found;
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "std::set      lookups: " << ms << " ms\n";
    }

    // === Benchmark: flat_set lookups ===
    {
        auto start = std::chrono::steady_clock::now();
        volatile int found = 0;
        for (int q : queries) {
            if (fs.contains(q)) ++found;
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "flat_set      lookups: " << ms << " ms\n";
    }

    // === Memory comparison ===
    size_t set_mem = s.size() * (sizeof(int) + 3 * sizeof(void*) + sizeof(bool));
    // Approximate: each RB-tree node has value + left/right/parent pointers + color
    size_t flat_mem = fs.size() * sizeof(int);

    std::cout << "\nMemory estimate:\n";
    std::cout << "  std::set:  ~" << set_mem / 1024 << " KB\n";
    std::cout << "  flat_set:  ~" << flat_mem / 1024 << " KB\n";
    std::cout << "  Ratio:     " << set_mem / flat_mem << "x\n";

    return 0;
}
// Typical output:
// std::set      lookups: ~120 ms
// flat_set      lookups: ~35 ms  (3-4x faster!)
// Memory: set ~3200 KB, flat_set ~400 KB (8x less)
```

The 3-4x speedup comes entirely from cache behavior. Binary search touches O(log n) elements in both cases, but in a contiguous array those elements are nearby in physical memory, so they often land in the same or adjacent cache lines. In a tree, those O(log n) nodes are scattered across the heap - each one is a likely cache miss.

### Q2: Explain that flat_set uses a sorted contiguous sequence: O(n) insert for O(log n) cache-friendly lookup

Understanding exactly what happens on insert is important because it is where `flat_set` pays its price:

**Lookup - O(log n) but fast:**

Binary search over a contiguous array. Each comparison accesses memory near the previous comparison (within the same cache line or adjacent cache lines). A 64-byte cache line holds 16 ints - so a binary search through 100K ints touches ~17 elements across ~5-6 cache lines.

Compare with `std::set`: binary search through a tree touches ~17 nodes, each on a separate heap allocation - 17 cache misses.

**Insert - O(n) because of shifting:**

```cpp
flat_set: [1, 3, 5, 7, 9]     insert(4):

Step 1: Binary search -> position 2     O(log n)
Step 2: Shift elements right:           O(n)
         [1, 3, _, 5, 7, 9]
Step 3: Write value:
         [1, 3, 4, 5, 7, 9]

Total: O(n) per insert (dominated by shift)
```

**Batch construction is fast:**

When you need to build a large `flat_set`, do not insert elements one by one. Fill a vector, sort it, deduplicate, and construct from the sorted range. This avoids the O(n) per-insert penalty entirely:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>

int main() {
    constexpr int N = 1'000'000;

    // BAD: individual inserts -> O(n^2) total
    {
        std::vector<int> flat;
        auto start = std::chrono::steady_clock::now();
        for (int i = N; i >= 0; --i) {
            auto it = std::lower_bound(flat.begin(), flat.end(), i);
            flat.insert(it, i);  // O(n) shift each time!
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "Individual inserts: " << ms << " ms\n";
    }

    // GOOD: bulk fill + sort -> O(n log n) total
    {
        std::vector<int> flat(N);
        auto start = std::chrono::steady_clock::now();
        for (int i = 0; i < N; ++i) flat[i] = N - i;
        std::sort(flat.begin(), flat.end());
        flat.erase(std::unique(flat.begin(), flat.end()), flat.end());
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "Bulk sort:          " << ms << " ms\n";
    }

    // For flat_set: use the sorted_unique_t constructor
    // std::flat_set<int> fs(std::sorted_unique, sorted_vec);

    return 0;
}
// Typical output:
// Individual inserts: ~2000+ ms (O(n^2))
// Bulk sort:          ~80 ms (O(n log n))
```

### Q3: Show that flat_set invalidates all iterators on any insert or erase (unlike std::set)

This is the main gotcha with `flat_set`. The reason this trips people up is that `std::set` has stable iterators, so code that stores iterators across mutations "just works" with set - and then silently breaks when you swap in `flat_set`:

```cpp
#include <iostream>
#include <set>
#include <vector>
#include <algorithm>

// Simulate flat_set for demonstration
template <typename T>
class FlatSet {
    std::vector<T> data_;
public:
    using iterator = typename std::vector<T>::iterator;
    using const_iterator = typename std::vector<T>::const_iterator;

    std::pair<iterator, bool> insert(const T& val) {
        auto it = std::lower_bound(data_.begin(), data_.end(), val);
        if (it != data_.end() && *it == val) return {it, false};
        // THIS INVALIDATES ALL ITERATORS (vector::insert may reallocate!)
        it = data_.insert(it, val);
        return {it, true};
    }

    iterator erase(iterator pos) {
        // THIS INVALIDATES ALL ITERATORS AT/AFTER pos
        return data_.erase(pos);
    }

    iterator find(const T& val) {
        auto it = std::lower_bound(data_.begin(), data_.end(), val);
        if (it != data_.end() && *it == val) return it;
        return data_.end();
    }

    iterator begin() { return data_.begin(); }
    iterator end() { return data_.end(); }
    size_t size() const { return data_.size(); }
};

int main() {
    // === std::set: iterators are STABLE ===
    {
        std::set<int> s = {10, 20, 30, 40, 50};

        auto it30 = s.find(30);
        std::cout << "set: it30 points to " << *it30 << "\n";  // 30

        s.insert(25);  // Insert nearby
        s.erase(s.find(40));  // Erase another element

        // it30 is STILL VALID!
        std::cout << "set: after insert+erase, it30 = " << *it30 << "\n";  // 30

        // This stability allows patterns like:
        for (auto it = s.begin(); it != s.end(); ) {
            if (*it % 2 == 0) {
                it = s.erase(it);  // Erase returns next iterator
            } else {
                ++it;
            }
        }
        std::cout << "set after erase evens: ";
        for (int x : s) std::cout << x << " ";
        std::cout << "\n";
        // Output: 25
    }

    // === flat_set: ALL iterators invalidated on mutation ===
    {
        FlatSet<int> fs;
        for (int x : {10, 20, 30, 40, 50}) fs.insert(x);

        auto it30 = fs.find(30);
        std::cout << "\nflat_set: it30 points to " << *it30 << "\n";

        fs.insert(25);  // May reallocate vector -> ALL iterators INVALID!
        // *it30 is now UNDEFINED BEHAVIOR!

        // Must re-find after any mutation:
        it30 = fs.find(30);
        if (it30 != fs.end())
            std::cout << "flat_set: re-found 30 = " << *it30 << "\n";

        // Safe erase pattern for flat_set:
        // Erase in reverse order, or use erase-remove idiom
        std::cout << "flat_set safe erase (reverse): ";
        for (auto it = fs.end(); it != fs.begin(); ) {
            --it;
            if (*it % 2 == 0) {
                it = fs.erase(it);
                // After erase, 'it' may be invalid, but erase returns
                // iterator to element after erased (which == end or next)
            }
        }
        for (auto it = fs.begin(); it != fs.end(); ++it)
            std::cout << *it << " ";
        std::cout << "\n";
    }

    // === Summary ===
    std::cout << "\nIterator stability:\n";
    std::cout << "  std::set insert:    other iterators VALID\n";
    std::cout << "  std::set erase:     only erased iterator invalid\n";
    std::cout << "  flat_set insert:    ALL iterators INVALID (reallocation)\n";
    std::cout << "  flat_set erase:     iterators at/after erase point INVALID\n";

    return 0;
}
```

`std::set` uses a tree where each node is independently allocated, so inserting or erasing a node does not move anything else. `flat_set` uses a contiguous vector, so insert may trigger reallocation (invalidating every iterator) and always shifts elements after the insertion point. The practical rule: never cache a `flat_set` iterator across a mutation. Always re-find.

---

## Notes

- **C++23 header:** `#include <flat_set>`. Also `#include <flat_map>` for `std::flat_map`.
- **Custom underlying container:** `std::flat_set<int, std::less<>, std::deque<int>>` uses a deque instead of vector.
- **`sorted_unique_t` constructor:** `std::flat_set<int> fs(std::sorted_unique, already_sorted_vec)` skips sorting - O(1) if the data is already sorted and unique.
- **`std::flat_multiset`:** Allows duplicates, also in C++23.
- **Crossover point:** For most workloads, `flat_set` beats `std::set` up to ~100K-500K elements. Above that, O(n) insert becomes too expensive unless the workload is lookup-dominated.
- **Compiler support (2024):** MSVC 17.6+, libstdc++ (GCC 14+). Check your compiler's C++23 support status.
