# Know the time complexity guarantees of all standard containers

**Category:** Standard Library Containers  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/container>  

---

## Topic Overview

The C++ standard mandates specific time complexity guarantees for all container operations. Choosing the right container depends on understanding these guarantees for your access patterns. If you remember one principle here: the standard guarantees complexity classes, not actual speed - a container with a "better" big-O can still be slower in practice for small sizes because of constant factors and cache behavior.

### Comprehensive Complexity Table

If the table feels like a lot, it boils down to: `vector` is fastest for everything except front insertion; `list`/`forward_list` have O(1) insert/erase if you already have an iterator; `map`/`set` give you O(log n) for everything ordered; `unordered_*` gives you O(1) average for lookups.

#### Sequence Containers

| Operation | `vector` | `deque` | `list` | `forward_list` | `array` |
| --- | --- | --- | --- | --- | --- |
| `push_back` | Amort. O(1) | Amort. O(1) | O(1) | N/A | N/A |
| `push_front` | **O(n)** | O(1) | O(1) | O(1) | N/A |
| `pop_back` | O(1) | O(1) | O(1) | N/A | N/A |
| `pop_front` | **O(n)** | O(1) | O(1) | O(1) | N/A |
| Insert middle | **O(n)** | **O(n)** | O(1)* | O(1)* | N/A |
| Erase middle | **O(n)** | **O(n)** | O(1)* | O(1)* | N/A |
| Access `[i]` | O(1) | O(1) | **O(n)** | **O(n)** | O(1) |
| `find` (unsorted) | O(n) | O(n) | O(n) | O(n) | O(n) |
| `sort` | O(n log n) | O(n log n) | O(n log n) | O(n log n) | O(n log n) |

*\* O(1) if you already have an iterator to the position; finding the position is O(n).*

#### Associative Containers (Tree-based, ordered)

| Operation | `map` / `set` | `multimap` / `multiset` |
| --- | --- | --- |
| `insert` | O(log n) | O(log n) |
| `erase` | O(log n) + amort. O(1) | O(log n) + O(k) |
| `find` | O(log n) | O(log n) |
| `lower_bound` / `upper_bound` | O(log n) | O(log n) |
| Iteration | O(n) total | O(n) total |
| `merge` | O(n log n) | O(n log n) |

#### Unordered Associative Containers (Hash-based)

| Operation | `unordered_map` / `unordered_set` | Worst case |
| --- | --- | --- |
| `insert` | O(1) amortized | **O(n)** |
| `erase` | O(1) amortized | **O(n)** |
| `find` | O(1) average | **O(n)** |
| `count` | O(1) average | O(n) |
| `rehash` | O(n) | O(n) |
| Iteration | O(n + bucket_count) | |

#### Container Adaptors

| Operation | `stack` | `queue` | `priority_queue` |
| --- | --- | --- | --- |
| Push | O(1)* | O(1)* | O(log n) |
| Pop | O(1)* | O(1)* | O(log n) |
| Top/Front | O(1) | O(1) | O(1) |

*\* Depends on underlying container (deque by default).*

#### C++23 Flat Containers

| Operation | `flat_map` / `flat_set` |
| --- | --- |
| `insert` | O(n) (must maintain sorted order) |
| `find` | **O(log n)** (binary search on contiguous memory) |
| `erase` | O(n) (shift elements) |
| Iteration | O(n) (cache-friendly) |

### Key Rules of Thumb

1. **Default to `std::vector`** - contiguous memory wins for most workloads.
2. **Need O(log n) lookup?** -> `std::map` / `std::set` (ordered) or `std::flat_map` (C++23, cache-friendly).
3. **Need O(1) lookup?** -> `std::unordered_map` / `std::unordered_set`.
4. **Need O(1) front/back insert?** -> `std::deque`.
5. **Need O(1) splice?** -> `std::list`.

### Important Notes

- "Amortized O(1)" means occasional O(n) operations (reallocation) averaged over many operations.
- Hash container worst case O(n) occurs when all keys hash to the same bucket.
- `std::vector::insert` at the end is amortized O(1); at the beginning is always O(n).
- Tree-based containers (`map`, `set`) guarantee O(log n) worst case - no pathological inputs.

---

## Self-Assessment

### Q1: List the O() complexity for insert, find, and erase in vector, list, map, unordered_map, and deque

Here's an empirical verification that makes the complexity differences visible in actual timing. Notice how the list middle-insert benchmark runs much faster per insert because we move the iterator to the middle once and then reuse it - that's the O(1) insert given an iterator that the table describes.

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <map>
#include <unordered_map>
#include <deque>
#include <chrono>
#include <string>

// Empirical verification of complexity guarantees
template <typename F>
long long measure_us(F&& func) {
    auto start = std::chrono::high_resolution_clock::now();
    func();
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
}

int main() {
    // === Complexity Summary ===
    std::cout << "Container      | Insert(mid) | Find     | Erase(mid)\n";
    std::cout << "---------------|-------------|----------|----------\n";
    std::cout << "vector         | O(n)        | O(n)*    | O(n)\n";
    std::cout << "deque          | O(n)        | O(n)*    | O(n)\n";
    std::cout << "list           | O(1)**      | O(n)     | O(1)**\n";
    std::cout << "map            | O(log n)    | O(log n) | O(log n)\n";
    std::cout << "unordered_map  | O(1) amort  | O(1) avg | O(1) avg\n";
    std::cout << "\n* O(n) linear scan; O(log n) if sorted + binary_search\n";
    std::cout << "** O(1) given iterator; finding position is O(n)\n\n";

    // === Empirical: insert at middle ===
    constexpr int N = 100'000;

    // Vector: O(n) insert at middle
    {
        std::vector<int> v(N);
        auto t = measure_us([&] {
            for (int i = 0; i < 1000; ++i)
                v.insert(v.begin() + v.size() / 2, i);
        });
        std::cout << "vector insert-middle x1000: " << t << " us\n";
    }

    // List: O(1) insert (but O(n) to find middle)
    {
        std::list<int> l(N, 0);
        auto mid = l.begin();
        std::advance(mid, N / 2);  // O(n) to find middle
        auto t = measure_us([&] {
            for (int i = 0; i < 1000; ++i)
                l.insert(mid, i);  // O(1) insert given iterator
        });
        std::cout << "list   insert-middle x1000: " << t << " us (O(1) each)\n";
    }

    // Map: O(log n) insert
    {
        std::map<int, int> m;
        auto t = measure_us([&] {
            for (int i = 0; i < N; ++i)
                m[i] = i;
        });
        std::cout << "map    insert x" << N << ": " << t << " us (O(log n) each)\n";
    }

    // Unordered_map: O(1) amortized insert
    {
        std::unordered_map<int, int> um;
        um.reserve(N);
        auto t = measure_us([&] {
            for (int i = 0; i < N; ++i)
                um[i] = i;
        });
        std::cout << "unordered_map insert x" << N << ": " << t << " us (O(1) each)\n";
    }

    // === Empirical: find ===
    {
        std::map<int, int> m;
        std::unordered_map<int, int> um;
        for (int i = 0; i < N; ++i) { m[i] = i; um[i] = i; }

        auto t_map = measure_us([&] {
            for (int i = 0; i < N; ++i) m.find(i);
        });
        auto t_umap = measure_us([&] {
            for (int i = 0; i < N; ++i) um.find(i);
        });
        std::cout << "\nmap    find x" << N << ": " << t_map << " us (O(log n) each)\n";
        std::cout << "umap   find x" << N << ": " << t_umap << " us (O(1) each)\n";
    }

    return 0;
}
```

**Explanation:**

- **`vector`:** Insert/erase at middle is O(n) because all subsequent elements must be shifted. `find` is O(n) linear scan (or O(log n) if sorted, using `std::lower_bound`).
- **`list`:** Insert/erase is O(1) *given an iterator*, but finding the position is O(n). No random access.
- **`map`:** All operations are O(log n) guaranteed - backed by a balanced BST (usually red-black tree).
- **`unordered_map`:** O(1) average, O(n) worst case. Worst case occurs with hash collisions.
- **`deque`:** Same complexity as vector for middle operations, but O(1) push_front/pop_front.

### Q2: Explain why unordered_map has O(1) amortized lookup but O(n) worst case

The worst case is not just a theoretical edge case - it's a real attack vector. This example shows both a contrived bad hash and what happens when `std::hash<int>` itself degrades on adversarial input.

```cpp
#include <iostream>
#include <unordered_map>
#include <chrono>
#include <string>

// Custom hash that causes all collisions
struct TerribleHash {
    size_t operator()(int key) const {
        return 0;  // Everything goes to bucket 0!
    }
};

int main() {
    constexpr int N = 10'000;

    // === Good hash: O(1) average ===
    {
        std::unordered_map<int, int> m;
        m.reserve(N);
        for (int i = 0; i < N; ++i) m[i] = i;

        auto start = std::chrono::high_resolution_clock::now();
        volatile int sum = 0;
        for (int i = 0; i < N; ++i) {
            sum += m.find(i)->second;
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

        size_t max_bucket = 0;
        for (size_t b = 0; b < m.bucket_count(); ++b)
            max_bucket = std::max(max_bucket, m.bucket_size(b));

        std::cout << "Good hash:\n";
        std::cout << "  Find " << N << " keys: " << dur.count() << " us\n";
        std::cout << "  Max bucket size: " << max_bucket << "\n\n";
        // Expected: ~100 us, max bucket 1-3
    }

    // === Terrible hash: O(n) per lookup ===
    {
        std::unordered_map<int, int, TerribleHash> m;
        for (int i = 0; i < N; ++i) m[i] = i;

        auto start = std::chrono::high_resolution_clock::now();
        volatile int sum = 0;
        for (int i = 0; i < N; ++i) {
            sum += m.find(i)->second;
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

        std::cout << "Terrible hash (all collisions):\n";
        std::cout << "  Find " << N << " keys: " << dur.count() << " us\n";
        std::cout << "  Max bucket size: " << m.bucket_size(0) << "\n";
        // Expected: ~100,000+ us (O(n) per lookup -> O(n^2) total)
        // Max bucket = N (all in one bucket)
    }

    std::cout << "\nWhy O(n) worst case:\n";
    std::cout << "1. hash(key) -> bucket index\n";
    std::cout << "2. Each bucket is a linked list of colliding elements\n";
    std::cout << "3. Good hash: ~1 element per bucket -> O(1) lookup\n";
    std::cout << "4. Bad hash: N elements in one bucket -> O(n) linear scan\n";

    return 0;
}
```

**Explanation:**

- `std::unordered_map` works by hashing the key to determine a bucket index.
- Each bucket is a linked list (separate chaining) of elements whose keys hash to that bucket.
- **O(1) average:** With a good hash and `load_factor <= 1.0`, each bucket has ~1 element. Lookup = hash + one comparison.
- **O(n) worst case:** If all keys hash to the same bucket (either by adversarial input or a poor hash function), lookup degrades to a linear scan through the entire linked list.
- **Rehashing** doesn't help if the hash function itself is the problem - all elements will still end up in the same bucket after rehashing.
- **Mitigation:** Use a high-quality hash function with good distribution. For string keys, consider `std::hash<std::string>` (usually good) or a randomized hash to defend against HashDoS.

### Q3: Show a pathological worst-case for unordered_map and how to mitigate it with a custom hash

The reason this trips people up is that `std::hash<int>` is often the identity function. That sounds fine until you realize that keys which are multiples of the bucket count all land in the same bucket. The SplitMix64 hash function is a well-known fix that mixes the bits properly before indexing.

```cpp
#include <iostream>
#include <unordered_map>
#include <chrono>
#include <random>
#include <cstdint>

// Exploit: keys that all hash to the same value under std::hash<int>
// On most implementations, std::hash<int> is the identity function!
// So keys that are multiples of bucket_count all land in bucket 0.

void demonstrate_pathological() {
    std::unordered_map<int, int> m;
    m.rehash(16);  // Force 16 buckets

    std::cout << "Bucket count: " << m.bucket_count() << "\n";

    // Insert multiples of bucket_count -> all go to bucket 0
    size_t bc = m.bucket_count();
    for (int i = 0; i < 1000; ++i) {
        m[static_cast<int>(i * bc)] = i;  // All hash to bucket 0!
    }

    std::cout << "Bucket 0 size: " << m.bucket_size(0) << "\n";
    // After rehashing, they may spread out. Let's check max again:
    size_t max_b = 0;
    for (size_t b = 0; b < m.bucket_count(); ++b)
        max_b = std::max(max_b, m.bucket_size(b));
    std::cout << "Max bucket after rehash: " << max_b << "\n";
    // Still bad: all multiples of any power-of-2 stay clustered
}

// Mitigation: a good mixing hash
struct SplitMix64Hash {
    size_t operator()(int key) const {
        uint64_t x = static_cast<uint64_t>(key);
        // SplitMix64 finalizer (used in java.util.SplittableRandom)
        x += 0x9e3779b97f4a7c15ULL;
        x ^= x >> 30;
        x *= 0xbf58476d1ce4e5b9ULL;
        x ^= x >> 27;
        x *= 0x94d049bb133111ebULL;
        x ^= x >> 31;
        return static_cast<size_t>(x);
    }
};

// Even better: per-instance randomized hash
struct RandomizedMixHash {
    uint64_t salt;

    RandomizedMixHash() {
        std::random_device rd;
        salt = rd();
    }

    size_t operator()(int key) const {
        uint64_t x = static_cast<uint64_t>(key) ^ salt;
        x ^= x >> 30;
        x *= 0xbf58476d1ce4e5b9ULL;
        x ^= x >> 27;
        x *= 0x94d049bb133111ebULL;
        x ^= x >> 31;
        return static_cast<size_t>(x);
    }
};

template <typename Hash>
void benchmark(const char* name, int n) {
    std::unordered_map<int, int, Hash> m;
    m.reserve(n);

    // Insert pathological keys (multiples of 64)
    for (int i = 0; i < n; ++i) m[i * 64] = i;

    size_t max_b = 0;
    for (size_t b = 0; b < m.bucket_count(); ++b)
        max_b = std::max(max_b, m.bucket_size(b));

    auto start = std::chrono::high_resolution_clock::now();
    volatile int sum = 0;
    for (int i = 0; i < n; ++i) sum += m[i * 64];
    auto end = std::chrono::high_resolution_clock::now();
    auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

    std::cout << name << ":\n";
    std::cout << "  Max bucket: " << max_b << ", Lookup " << n << " keys: " << dur.count() << " us\n";
}

int main() {
    demonstrate_pathological();
    std::cout << "\n=== Benchmark with pathological keys ===\n";
    constexpr int N = 50'000;
    benchmark<std::hash<int>>("std::hash<int>", N);
    benchmark<SplitMix64Hash>("SplitMix64Hash", N);
    benchmark<RandomizedMixHash>("RandomizedMixHash", N);

    // Expected:
    // std::hash<int>: max bucket very large, slow lookup
    // SplitMix64Hash: max bucket ~1-3, fast lookup
    // RandomizedMixHash: max bucket ~1-3, fast lookup, different each run

    return 0;
}
```

**How it works:**

- **Pathological case:** `std::hash<int>` is often the identity function. Keys that are multiples of the bucket count collide heavily because `(key % bucket_count) = 0` for all of them.
- **SplitMix64 hash:** Applies a well-known mixing/avalanche function that distributes any input pattern uniformly across buckets. Even pathological keys are distributed well.
- **Randomized hash:** Adds a per-instance random salt, making it impossible for an attacker to pre-compute colliding inputs. Different program invocations produce completely different hash distributions.
- **Result:** The mixing hash reduces max bucket size from O(n) to ~1-3, turning O(n) lookups back into O(1).

---

## Notes

- **The standard guarantees complexities, not constants.** An O(log n) `std::map` lookup may be slower than an O(1) `std::unordered_map` lookup on paper, but for small n (< ~100), the constant factors and cache behavior may reverse this.
- **`std::flat_map` (C++23)** provides O(log n) find with cache-friendly contiguous storage - excellent for read-heavy, write-infrequent workloads.
- **Iterator performance:** `std::unordered_map` iteration is O(n + bucket_count), not O(n). Many empty buckets slow iteration.
- **Amortized O(1)** for `vector::push_back` means occasional O(n) reallocations. The growth factor (usually 1.5x or 2x) ensures the total cost of n insertions is O(n).
- **`std::list::sort`** is O(n log n) but with larger constant than `std::sort` on vectors (cache-unfriendly merge sort vs. introsort).
