# Use std::list only when stable iterators are required

**Category:** Standard Library Containers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/container/list>  

---

## Topic Overview

`std::list` is a doubly-linked list. Every element is a separate heap allocation connected by prev/next pointers. This gives it unique properties - but at a **severe performance cost** for most workloads. The honest summary: reach for `std::list` only when you have a specific need for stable iterators, stable references, or O(1) splice. Otherwise, `std::vector` will almost certainly be faster.

### The Hard Truth About std::list

In nearly all cases, `std::vector` is faster than `std::list` - even for operations where `list` has better theoretical complexity (like mid-insertion). The reason is **CPU cache behavior**. The reason this trips people up is that the O(1) insert advantage sounds compelling on paper, but physical memory access patterns dominate in practice:

```cpp
std::vector<int> (contiguous):
  Memory: [1][2][3][4][5][6][7][8]  <- all in 1-2 cache lines
  Sequential read: prefetcher loads next elements automatically
  Cache hit rate: ~99%

std::list<int> (scattered):
  Memory: [prev|1|next]  ...gap...  [prev|2|next]  ...gap...  [prev|3|next]
             ^                         ^                         ^
           Heap alloc A             Heap alloc B              Heap alloc C
  Each node accessed = likely cache miss
  Cache hit rate: very low for large lists
```

### When to Use std::list (Rare!)

| Use case | Why list works | Why vector doesn't |
| --- | --- | --- |
| Stable iterators across insert/erase | Node addresses never change | Reallocation invalidates all iterators |
| Stable references/pointers to elements | Same reason | Same problem |
| O(1) splice between lists | Just relink pointers | Would require O(n) copy |
| Frequent mid-list insert/erase with iterator | O(1) unlink/link | O(n) shift, but often still faster due to cache |
| Elements are very large and non-movable | No element movement on insert | Must move/copy on reallocation |

### Memory Overhead per Element

| Container | Per-element overhead (64-bit) |
| --- | --- |
| `vector<int>` | 0 bytes (just the int) |
| `list<int>` | ~32-48 bytes (prev + next pointers + allocator overhead) |
| `forward_list<int>` | ~16-24 bytes (next pointer + allocator overhead) |

For a list of 1M ints: vector uses ~4 MB, list uses ~48 MB - **12x more memory**.

---

## Self-Assessment

### Q1: Show a benchmark where std::list is slower than std::vector for sequential traversal due to cache misses

The numbers here are the important part. The theoretical O(n) traversal cost is the same for both, but the constant factor is enormous because of cache misses:

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <chrono>
#include <numeric>

int main() {
    constexpr int N = 1'000'000;

    // === Build both containers ===
    std::vector<int> vec(N);
    std::iota(vec.begin(), vec.end(), 0);

    std::list<int> lst(vec.begin(), vec.end());

    // === Benchmark: sequential sum traversal ===

    // vector traversal
    {
        volatile long long sum = 0;
        auto start = std::chrono::steady_clock::now();
        for (int i = 0; i < 10; ++i) {
            long long s = 0;
            for (int x : vec) s += x;
            sum = s;
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "vector sum (10 iters): " << ms << " ms\n";
    }

    // list traversal
    {
        volatile long long sum = 0;
        auto start = std::chrono::steady_clock::now();
        for (int i = 0; i < 10; ++i) {
            long long s = 0;
            for (int x : lst) s += x;
            sum = s;
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "list sum (10 iters):   " << ms << " ms\n";
    }

    // === Benchmark: random access pattern ===
    // (list doesn't support random access, so we compare iteration to nth element)
    {
        auto start = std::chrono::steady_clock::now();
        volatile int val = 0;
        for (int i = 0; i < 10000; ++i) {
            val = vec[N / 2];  // O(1) random access
        }
        auto end = std::chrono::steady_clock::now();
        auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count();
        std::cout << "\nvector[N/2] x 10K: " << ns / 1000 << " us\n";
    }

    {
        auto start = std::chrono::steady_clock::now();
        volatile int val = 0;
        for (int i = 0; i < 10000; ++i) {
            auto it = lst.begin();
            std::advance(it, N / 2);  // O(n) walk!
            val = *it;
        }
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "list advance(N/2) x 10K: " << ms << " ms\n";
    }

    // === Memory comparison ===
    size_t vec_mem = N * sizeof(int);
    size_t lst_node_size = sizeof(int) + 2 * sizeof(void*); // value + prev + next
    size_t lst_mem = N * (lst_node_size + 16); // ~16 bytes allocator overhead per node

    std::cout << "\nMemory:\n";
    std::cout << "  vector: " << vec_mem / (1024 * 1024) << " MB\n";
    std::cout << "  list:   ~" << lst_mem / (1024 * 1024) << " MB\n";

    return 0;
}
// Typical output:
// vector sum (10 iters): ~15 ms
// list sum (10 iters):   ~120 ms  (8x slower!)
//
// vector[N/2] x 10K: ~1 us
// list advance(N/2) x 10K: ~5000 ms
//
// Memory:
//   vector: 3 MB
//   list:   ~38 MB
```

The ~8x slowdown for sequential traversal is typical. For random access the gap becomes catastrophic - `std::advance` on a list is O(n), so 10K calls at the midpoint means 10K full half-traversals. Even for mid-list insertion where list is O(1) and vector is O(n) for shifting, vector often wins for small-to-medium sizes because the shift is a fast `memcpy` in contiguous memory.

### Q2: Demonstrate a use case where list splice() is O(1) and eliminates the need for index shifting

The LRU cache is the textbook scenario where `std::list` genuinely earns its place. Every other option forces O(n) work somewhere:

```cpp
#include <iostream>
#include <list>
#include <vector>
#include <unordered_map>
#include <string>

// === LRU Cache - THE canonical std::list use case ===
// O(1) get, O(1) put, O(1) eviction
// Only possible with list::splice + unordered_map

template <typename K, typename V>
class LRUCache {
    size_t capacity_;
    std::list<std::pair<K, V>> items_;  // front = most recent
    std::unordered_map<K, typename std::list<std::pair<K, V>>::iterator> map_;

public:
    explicit LRUCache(size_t cap) : capacity_(cap) {}

    V* get(const K& key) {
        auto it = map_.find(key);
        if (it == map_.end()) return nullptr;

        // Move to front: O(1) splice!
        items_.splice(items_.begin(), items_, it->second);
        return &it->second->second;
    }

    void put(const K& key, const V& value) {
        auto it = map_.find(key);
        if (it != map_.end()) {
            it->second->second = value;
            items_.splice(items_.begin(), items_, it->second);
            return;
        }

        if (items_.size() >= capacity_) {
            // Evict least recently used (back of list): O(1)!
            auto& back = items_.back();
            map_.erase(back.first);
            items_.pop_back();
        }

        items_.emplace_front(key, value);
        map_[key] = items_.begin();
    }

    void print() const {
        std::cout << "Cache (MRU->LRU): ";
        for (const auto& [k, v] : items_)
            std::cout << k << "=" << v << " ";
        std::cout << "\n";
    }
};

int main() {
    LRUCache<std::string, int> cache(3);

    cache.put("a", 1);
    cache.put("b", 2);
    cache.put("c", 3);
    cache.print();  // c=3 b=2 a=1

    cache.get("a");  // Promotes "a" to front via splice
    cache.print();   // a=1 c=3 b=2

    cache.put("d", 4);  // Evicts "b" (LRU)
    cache.print();       // d=4 a=1 c=3

    // Try to get evicted key
    auto* val = cache.get("b");
    std::cout << "get('b'): " << (val ? std::to_string(*val) : "null") << "\n";
    // Output: get('b'): null

    // === Why vector can't do this ===
    // With vector: promoting an element to front = O(n) shift of all elements.
    // With vector: evicting from back = O(1), but maintaining order is O(n).
    // With list + splice: ALL operations are O(1).

    // === Comparison: vector-based "LRU" ===
    std::cout << "\nVector-based LRU would require:\n";
    std::cout << "  get()  -> find O(n) or O(1) via map, then shift to front O(n)\n";
    std::cout << "  put()  -> insert at front O(n), evict from back O(1)\n";
    std::cout << "  Total: O(n) per operation\n\n";
    std::cout << "List-based LRU:\n";
    std::cout << "  get()  -> map lookup O(1), splice to front O(1)\n";
    std::cout << "  put()  -> push_front O(1), pop_back O(1)\n";
    std::cout << "  Total: O(1) per operation\n";

    return 0;
}
```

The crucial ingredient here is that `unordered_map` stores iterators into the list, and those iterators stay valid across `splice` calls. Because `splice` only relinks pointers without moving any node in memory, the stored iterators always point to the right elements even after multiple promotions.

### Q3: Explain when forward_list is preferred over list (memory overhead: single vs double link)

`std::forward_list` is the singly-linked cousin of `std::list`. You give up backward traversal and a few convenience methods in exchange for roughly one-third less overhead per node:

```cpp
#include <iostream>
#include <list>
#include <forward_list>
#include <vector>

int main() {
    // === Memory comparison ===
    struct Node_List {
        int value;        // 4 bytes
        // padding:       // 4 bytes (alignment)
        void* prev;       // 8 bytes
        void* next;       // 8 bytes
    };  // Total: ~24 bytes per node (+ allocator overhead)

    struct Node_ForwardList {
        int value;        // 4 bytes
        // padding:       // 4 bytes
        void* next;       // 8 bytes
    };  // Total: ~16 bytes per node (+ allocator overhead)

    std::cout << "Per-node overhead (approximate, 64-bit):\n";
    std::cout << "  list<int>:         ~" << sizeof(Node_List)
              << " bytes + allocator overhead\n";
    std::cout << "  forward_list<int>: ~" << sizeof(Node_ForwardList)
              << " bytes + allocator overhead\n";
    std::cout << "  Savings:           ~33% per node\n\n";

    // === forward_list API differences ===
    std::forward_list<int> fl = {1, 2, 3, 4, 5};

    // No size()! - would be O(n) to compute, violating O(1) design goal
    // fl.size();  // ERROR: not available!
    auto count = std::distance(fl.begin(), fl.end());  // Manual O(n)
    std::cout << "forward_list elements: " << count << "\n";

    // No push_back! - would require O(n) traversal
    // fl.push_back(6);  // ERROR
    fl.push_front(0);    // O(1)

    // Insert AFTER (not before!) - because we only have next pointer
    auto it = fl.begin();
    fl.insert_after(it, 99);  // Insert 99 after first element
    // fl: 0, 99, 1, 2, 3, 4, 5

    // Erase AFTER
    fl.erase_after(it);  // Erase element after first
    // fl: 0, 1, 2, 3, 4, 5

    // splice_after (not splice)
    std::forward_list<int> fl2 = {10, 20, 30};
    fl.splice_after(fl.begin(), fl2);  // Insert fl2 after first element
    // fl: 0, 10, 20, 30, 1, 2, 3, 4, 5

    std::cout << "forward_list: ";
    for (int x : fl) std::cout << x << " ";
    std::cout << "\n";

    // === When to choose forward_list over list ===
    std::cout << "\nPrefer forward_list when:\n";
    std::cout << "  1. Only need forward traversal (never backwards)\n";
    std::cout << "  2. Memory is tight (1/3 less overhead)\n";
    std::cout << "  3. Only insert/erase at front or after known position\n";
    std::cout << "  4. Don't need size() - or can track it yourself\n";
    std::cout << "  5. Building a hash table chain (singly-linked buckets)\n\n";

    std::cout << "Prefer list when:\n";
    std::cout << "  1. Need bidirectional traversal\n";
    std::cout << "  2. Need push_back / pop_back\n";
    std::cout << "  3. Need splice (not just splice_after)\n";
    std::cout << "  4. Need size() in O(1)\n\n";

    std::cout << "Prefer vector over BOTH when:\n";
    std::cout << "  1. Sequential access patterns (cache matters)\n";
    std::cout << "  2. Random access needed\n";
    std::cout << "  3. Don't need stable iterators\n";
    std::cout << "  4. Memory efficiency matters (no per-node overhead)\n";
    std::cout << "  5. Almost always - vector is the default choice!\n";

    return 0;
}
```

The API shape of `forward_list` reflects its constraints: you can only insert or erase *after* a given position (since you cannot walk backwards), and there is no `size()` because storing a count would add overhead. If you find yourself needing any of the missing features, just use `std::list` instead. And if you find yourself using either for anything other than stable-iterator or splice scenarios, reach for `std::vector` first and benchmark.

---

## Notes

- **Bjarne Stroustrup's guideline:** "Use `vector` by default. Use `list` when you need stable iterators/references, or when `splice` is critical to your algorithm."
- **`std::list` has member sort/merge/unique/remove:** These are optimized for linked lists and exploit O(1) splice internally. Do not use `std::sort` on a list - it requires random access iterators.
- **Large elements:** If elements are very large (hundreds of bytes) and non-movable, list avoids reallocation moves. But consider `std::vector<std::unique_ptr<T>>` or `std::deque<T>` first.
- **`deque` as alternative:** `std::deque` provides stable references (not iterators) to elements and has better cache behavior than `list`. Consider it before reaching for `list`.
- **Benchmark before choosing list:** The theoretical O(1) insert advantage of list is almost always dominated by cache miss overhead. Profile with real data.
