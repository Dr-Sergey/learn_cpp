# Know how std::unordered_map handles load factor and rehashing

**Category:** Standard Library Containers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/container/unordered_map>  

---

## Topic Overview

`std::unordered_map` is a hash table that stores key-value pairs in **buckets**. Performance depends critically on how elements are distributed across buckets. Two core mechanisms govern this: the **load factor** and **rehashing**. If you've ever noticed an unordered_map suddenly slow down during a burst of insertions, or wondered why pre-allocating dramatically improves throughput, this is the section that explains it.

### Hash Table Structure

Visualizing what's happening under the hood helps. Each bucket is essentially the head of a short linked list. A good hash spreads elements evenly, so most buckets hold at most one or two elements and lookups stay O(1). A bad hash piles everything into a few buckets and lookups degrade to O(n).

```cpp
Bucket 0: [key_a -> val] -> [key_x -> val] -> nullptr
Bucket 1: [key_b -> val] -> nullptr
Bucket 2: (empty)
Bucket 3: [key_c -> val] -> [key_d -> val] -> [key_e -> val] -> nullptr
...
Bucket N-1: [key_z -> val] -> nullptr
```

### Load Factor

The **load factor** = `size() / bucket_count()`. It measures how full the hash table is. Think of it like occupancy in a parking lot - once it gets too full, finding a spot (doing a lookup) takes longer because more collisions pile up in the same bucket.

| Member Function         | Description                                              |
| --- | --- |
| `load_factor()`        | Returns current load factor (float)                      |
| `max_load_factor()`    | Returns the threshold (default = 1.0)                    |
| `max_load_factor(f)`   | Sets the threshold                                       |
| `bucket_count()`       | Number of buckets currently allocated                    |
| `bucket_size(n)`       | Number of elements in bucket `n`                         |
| `bucket(key)`          | Returns the bucket index for a given key                 |
| `reserve(n)`           | Pre-allocates enough buckets for `n` elements            |
| `rehash(n)`            | Sets bucket count to at least `n`, triggers rehash       |

### Rehashing Process

Rehashing occurs **automatically** when an insertion would cause `load_factor() > max_load_factor()`. The process:

1. A new, larger bucket array is allocated (typically ~2x the previous size)
2. **Every existing element** is re-hashed and moved to a new bucket
3. The old bucket array is deallocated

**Cost of rehashing:** O(n) - every element must be rehashed and relocated.

The reason this trips people up is that rehashing is invisible unless you're watching bucket counts. Your map inserts look fine until the table crosses the load threshold, then one insert quietly triggers an O(n) operation. If your map grows to 1 million elements without a `reserve()`, you'll trigger roughly 20 of these rehashes, each more expensive than the last.

### reserve() vs rehash()

| Function      | Purpose                                                  |
| --- | --- |
| `reserve(n)` | Ensures capacity for `n` elements without rehash         |
| `rehash(n)`  | Sets bucket count to >= `n` (may trigger element moves)  |

`reserve(n)` calls `rehash(ceil(n / max_load_factor()))` internally. In other words, `reserve` thinks in terms of elements while `rehash` thinks in terms of buckets.

### Core Syntax and Patterns

Let's see the API in action. Pay attention to what `reserve(100)` does to the bucket count before any elements are inserted - this is the pre-allocation you want to call whenever you know the approximate final size upfront.

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

int main() {
    std::unordered_map<std::string, int> m;

    // Check initial state
    std::cout << "Initial bucket_count: " << m.bucket_count() << "\n";
    std::cout << "max_load_factor: " << m.max_load_factor() << "\n";
    // Output: max_load_factor: 1

    // Reserve space for 100 elements upfront
    m.reserve(100);
    std::cout << "After reserve(100), bucket_count: " << m.bucket_count() << "\n";
    // Output: bucket_count >= 100

    // Insert elements
    for (int i = 0; i < 50; ++i) {
        m["key_" + std::to_string(i)] = i;
    }

    std::cout << "Size: " << m.size() << "\n";
    std::cout << "load_factor: " << m.load_factor() << "\n";
    std::cout << "bucket_count: " << m.bucket_count() << "\n";

    // Inspect individual buckets
    for (size_t i = 0; i < m.bucket_count(); ++i) {
        if (m.bucket_size(i) > 2) {
            std::cout << "Bucket " << i << " has " << m.bucket_size(i) << " elements\n";
        }
    }

    // Lower max_load_factor for fewer collisions (more memory)
    m.max_load_factor(0.5f);
    // This triggers a rehash immediately if current load_factor > 0.5
    std::cout << "After max_load_factor(0.5), bucket_count: " << m.bucket_count() << "\n";

    return 0;
}
```

After calling `max_load_factor(0.5f)`, the map immediately rehashes if it needs to - it won't let itself stay above the threshold you just set. You're trading memory for fewer collisions, which means faster average lookups.

### Important Notes

- Default `max_load_factor()` is **1.0** for all standard unordered containers.
- **Always call `reserve(n)`** if you know how many elements you'll insert - avoids O(n) rehashes.
- Lowering `max_load_factor` reduces collisions but increases memory usage.
- Iterators, pointers, and references to elements are **not invalidated** by rehashing (they remain valid), but **iterator ordering may change**.
- `rehash(0)` is legal and shrinks to fit.

---

## Self-Assessment

### Q1: Trigger excessive rehashing by inserting beyond max_load_factor and measure the cost

This benchmark makes the difference concrete. Watch how the rehash count stays at zero with `reserve()` - every single one of those ~20 rehashes in the first version is an O(n) operation you're paying for.

```cpp
#include <iostream>
#include <unordered_map>
#include <chrono>
#include <string>

int main() {
    // WITHOUT reserve - triggers many rehashes
    {
        std::unordered_map<int, int> m;
        size_t prev_buckets = m.bucket_count();
        int rehash_count = 0;

        auto start = std::chrono::high_resolution_clock::now();

        for (int i = 0; i < 1'000'000; ++i) {
            m[i] = i;
            if (m.bucket_count() != prev_buckets) {
                ++rehash_count;
                prev_buckets = m.bucket_count();
            }
        }

        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

        std::cout << "Without reserve:\n";
        std::cout << "  Rehashes: " << rehash_count << "\n";
        std::cout << "  Final buckets: " << m.bucket_count() << "\n";
        std::cout << "  Time: " << dur.count() << " ms\n";
    }
    // Expected output (typical):
    //   Rehashes: ~20
    //   Final buckets: ~1048576
    //   Time: ~150-300 ms

    // WITH reserve - zero rehashes
    {
        std::unordered_map<int, int> m;
        m.reserve(1'000'000);
        size_t prev_buckets = m.bucket_count();
        int rehash_count = 0;

        auto start = std::chrono::high_resolution_clock::now();

        for (int i = 0; i < 1'000'000; ++i) {
            m[i] = i;
            if (m.bucket_count() != prev_buckets) {
                ++rehash_count;
                prev_buckets = m.bucket_count();
            }
        }

        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

        std::cout << "With reserve(1'000'000):\n";
        std::cout << "  Rehashes: " << rehash_count << "\n";
        std::cout << "  Final buckets: " << m.bucket_count() << "\n";
        std::cout << "  Time: " << dur.count() << " ms\n";
    }
    // Expected output (typical):
    //   Rehashes: 0
    //   Final buckets: ~1048576
    //   Time: ~80-150 ms (significantly faster)

    return 0;
}
```

**How it works:**

- Without `reserve()`, the map starts with a small bucket count (often 1 or 8).
- Each time `load_factor() > max_load_factor()` (1.0 by default), a rehash occurs.
- Each rehash roughly doubles bucket count, so inserting 1M elements triggers ~20 rehashes.
- Each rehash re-hashes **all existing elements** - O(n) per rehash.
- Total rehash cost: O(n) + O(2n) + O(4n) + ... approximately O(2n) amortized, but with large constant factors.
- With `reserve(1'000'000)`, all buckets are pre-allocated, so zero rehashes happen - measurably faster.

### Q2: Use reserve(n) before inserting n elements to avoid rehashing

Notice how `reserve` and `rehash` have subtly different semantics. `reserve(100)` means "I plan to insert 100 elements," while `rehash(100)` means "give me at least 100 buckets." When your `max_load_factor` is less than 1.0, the difference becomes significant - `reserve` will allocate more buckets than `rehash` with the same argument.

```cpp
#include <iostream>
#include <unordered_map>
#include <string>
#include <vector>

int main() {
    // Simulate loading data - we know we'll have ~500 entries
    constexpr size_t expected_size = 500;

    std::unordered_map<std::string, double> scores;
    scores.reserve(expected_size);  // Pre-allocate buckets

    std::cout << "After reserve(" << expected_size << "):\n";
    std::cout << "  bucket_count: " << scores.bucket_count() << "\n";
    std::cout << "  max_load_factor: " << scores.max_load_factor() << "\n";
    // bucket_count >= 500 (since max_load_factor is 1.0)

    // Insert data - no rehashing will occur
    for (size_t i = 0; i < expected_size; ++i) {
        scores["student_" + std::to_string(i)] = 50.0 + (i % 50);
    }

    std::cout << "After inserting " << scores.size() << " elements:\n";
    std::cout << "  load_factor: " << scores.load_factor() << "\n";
    std::cout << "  bucket_count: " << scores.bucket_count() << "\n";
    // load_factor <= 1.0, bucket_count unchanged

    // Difference between reserve and rehash:
    std::unordered_map<int, int> m;
    m.rehash(100);  // Sets bucket_count >= 100
    std::cout << "\nAfter rehash(100): bucket_count = " << m.bucket_count() << "\n";

    m.reserve(100);  // Sets buckets for 100 ELEMENTS
    // Internally: rehash(ceil(100 / 1.0)) = rehash(100)
    std::cout << "After reserve(100): bucket_count = " << m.bucket_count() << "\n";

    // With lower max_load_factor, reserve allocates MORE buckets
    m.max_load_factor(0.25f);
    m.reserve(100);  // Internally: rehash(ceil(100 / 0.25)) = rehash(400)
    std::cout << "After max_load_factor(0.25) + reserve(100): bucket_count = "
              << m.bucket_count() << "\n";
    // Expected: bucket_count >= 400

    return 0;
}
```

**How it works:**

- `reserve(n)` computes `ceil(n / max_load_factor())` and calls `rehash()` with that value.
- With default `max_load_factor()` of 1.0, `reserve(500)` allocates >=500 buckets.
- With `max_load_factor(0.25)`, `reserve(100)` allocates >=400 buckets (fewer collisions, more memory).
- After `reserve()`, inserting up to `n` elements is guaranteed to cause **zero rehashes**.

### Q3: Implement a worst-case hash collision attack and explain how to defend with a randomized hash

This is important for any server-side code that puts user-supplied data into an `unordered_map`. The attack is called HashDoS: craft input keys that all hash to the same bucket, turning every lookup into an O(n) scan. The randomized salt defeats this because the attacker cannot predict which keys will collide on any given run.

```cpp
#include <iostream>
#include <unordered_map>
#include <chrono>
#include <cstdint>
#include <random>
#include <functional>
#include <string>

// --- Attack: All keys hash to the same bucket ---
struct BadHash {
    size_t operator()(int key) const {
        return 42;  // Every key goes to bucket 42 % bucket_count
    }
};

// --- Defense: Randomized hash with per-instance salt ---
struct RandomizedHash {
    size_t salt;

    RandomizedHash() {
        // Generate random salt at construction time
        static std::mt19937_64 rng(std::random_device{}());
        salt = rng();
    }

    size_t operator()(int key) const {
        // Combine key with salt using a mixing function
        size_t h = static_cast<size_t>(key);
        h ^= salt;
        h ^= (h >> 16);
        h *= 0x45d9f3b;
        h ^= (h >> 16);
        return h;
    }
};

template <typename Hash>
void benchmark(const char* name, int n) {
    std::unordered_map<int, int, Hash> m;
    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < n; ++i) {
        m[i] = i;
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto dur = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    // Check max chain length
    size_t max_chain = 0;
    for (size_t b = 0; b < m.bucket_count(); ++b) {
        max_chain = std::max(max_chain, m.bucket_size(b));
    }

    std::cout << name << " (n=" << n << "):\n";
    std::cout << "  Time: " << dur.count() << " ms\n";
    std::cout << "  Max chain: " << max_chain << "\n";
    std::cout << "  Buckets: " << m.bucket_count() << "\n\n";
}

int main() {
    constexpr int N = 50'000;

    benchmark<BadHash>("BadHash (collision attack)", N);
    // Expected: Time ~seconds, Max chain = N (all in one bucket!) -> O(n^2) total

    benchmark<std::hash<int>>("std::hash<int> (default)", N);
    // Expected: Time ~ms, Max chain ~1-3

    benchmark<RandomizedHash>("RandomizedHash (salted)", N);
    // Expected: Time ~ms, Max chain ~1-3, different distribution each run

    return 0;
}
```

**How it works:**

- **Collision attack:** `BadHash` maps every key to the same bucket. With n elements in one bucket, each insertion requires a linear scan -> O(n) per insert -> O(n^2) total. This is a **HashDoS** attack.
- **Default `std::hash`:** Good distribution but **deterministic** - an attacker who knows the hash function can craft colliding keys.
- **Randomized hash:** Uses a per-instance random salt mixed into the hash. Different program invocations (or different map instances) produce different hash distributions, making collision attacks impractical.
- **Real-world defense:** Libraries like Boost.Unordered and abseil's `flat_hash_map` use randomized seeds by default. Python dicts adopted hash randomization after CVE-2012-1150.

---

## Notes

- **When to call `reserve()`:** Always, if you know the approximate number of elements. It's the single most impactful optimization for unordered containers.
- **`max_load_factor` trade-off:** Lower values -> fewer collisions, faster lookups, more memory. Higher values -> more collisions, slower lookups, less memory. Default of 1.0 is a good balance.
- **Pointer/reference stability:** Rehashing does NOT invalidate pointers or references to elements - only iterators may be invalidated (in practice, node-based implementations preserve them too, but the standard doesn't guarantee iteration order).
- **`rehash(0)` trick:** Call `rehash(0)` after erasing many elements to shrink the bucket array.
- **Hash quality matters more than load factor:** A poor hash function with load factor 0.5 can be slower than a good hash with load factor 2.0.
- **`bucket(key)`** is useful for debugging - check if your hash distributes evenly across buckets.
