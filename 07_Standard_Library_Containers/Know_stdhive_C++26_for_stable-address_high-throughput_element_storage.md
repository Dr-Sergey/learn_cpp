# Know std::hive (C++26) for stable-address, high-throughput element storage

**Category:** Standard Library Containers  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/container/hive>  

---

## Topic Overview

`std::hive` (C++26, P0447) is a new container designed for scenarios requiring **stable element addresses** (pointers/references never invalidated by insertion or erasure of other elements) while maintaining **high iteration throughput** via cache-friendly memory layout. It fills a gap between `std::list` (stable addresses, poor cache) and `std::vector` (great cache, no address stability).

The reason this matters so much in practice is that there's a whole class of software - game engines, physics simulations, ECS systems, particle effects - where you need to iterate over thousands of objects every frame while simultaneously adding and removing objects from the middle. Vector invalidates pointers on growth. List is stable but cache-unfriendly. Hive gives you both.

### How std::hive Works Internally

The core innovation is the **skipfield**: a compact data structure that records which slots are occupied and which are erased, and lets the iterator jump over erased slots in O(1) without visiting each one individually.

```cpp
Block 1:        Block 2:        Block 3:
[E][_][E][E]    [E][E][_][E]    [E][E][E][_]
     ^               ^                    ^
  skipfield=1    skipfield=1          skipfield=1

E = occupied element    _ = erased (skipfield marks it)
```

- **Colony/hive pattern:** Allocates elements in contiguous **blocks** (chunks).
- **Skipfield:** An auxiliary data structure (jump-counting skipfield) marks erased slots, allowing O(1) skip during iteration.
- **No reallocation of existing blocks** on insert -> **stable addresses**.
- **Erased slots are reused** by future insertions -> low memory fragmentation.

### Key Properties

| Property | `std::hive` | `std::vector` | `std::list` |
| --- | --- | --- | --- |
| Address stability | **Yes** | No | **Yes** |
| Cache-friendly iteration | **Good** | **Excellent** | Poor |
| Insert (any position) | O(1) amortized | O(n) | O(1) |
| Erase (any position) | O(1) | O(n) | O(1) |
| Random access `[i]` | **No** | O(1) | O(n) |
| Memory overhead per element | ~1 byte (skipfield) | 0 | 2 pointers |
| Preserves insertion order | **No** (partially) | Yes | Yes |
| Contiguous memory | Per-block | Yes | No |

### Core API (C++26)

Since `std::hive` isn't in compilers yet (as of 2024), this example demonstrates the conceptual API and simulates the intent with a `vector` + commentary. In real code you'd use `plf::hive` as a polyfill.

```cpp
// NOTE: std::hive is C++26. As of 2024, use plf::hive (reference implementation)
// from https://github.com/mattreecebentley/plf_hive as a polyfill.

#include <iostream>
#include <vector>
// #include <hive>  // C++26

// Using plf::hive as stand-in demonstration:
// #include "plf_hive.h"

// Conceptual API:
// std::hive<T> h;
// auto it = h.insert(value);    // Returns iterator, O(1) amortized
// h.erase(it);                  // O(1), does not invalidate other iterators
// h.size();                     // Number of active elements
// h.capacity();                 // Total slots (active + erased)
// h.reshape(...);               // Set min/max block sizes
// for (auto& x : h) { ... }    // Iteration skips erased slots via skipfield

int main() {
    // Simulating hive behavior with a vector + free list (conceptual)
    // Real usage requires C++26 or plf::hive

    struct Entity {
        int id;
        float x, y;
        bool active;
    };

    // In real std::hive:
    // std::hive<Entity> entities;
    // auto it = entities.insert({1, 10.0f, 20.0f, true});
    // Entity* ptr = &(*it);  // Stable! Never invalidated by other inserts/erases
    // entities.erase(it);    // O(1), ptr is now dangling but OTHER pointers remain valid

    std::cout << "std::hive provides:\n";
    std::cout << "  - O(1) insert and erase\n";
    std::cout << "  - Stable pointers/references to non-erased elements\n";
    std::cout << "  - Cache-friendly iteration via skipfield\n";
    std::cout << "  - Automatic slot reuse for erased elements\n";

    return 0;
}
```

### Important Notes

- `std::hive` does **not** support random access (`operator[]`). It's a forward-iterable container.
- Erased elements leave "holes" that are tracked by the skipfield and reused on subsequent insertions.
- Iteration order is **not** insertion order - erased-then-reused slots appear at their memory position.
- Iterator invalidation: only the erased element's iterator is invalidated. All other iterators remain valid.
- Block sizing can be tuned with `reshape(min_block_size, max_block_size)` for specific workloads.

---

## Self-Assessment

### Q1: Explain why std::hive preserves element addresses on insertion unlike std::vector

This is the core property that makes hive valuable. The example below makes the vector problem visible before explaining how hive avoids it through its block-based design.

```cpp
#include <iostream>
#include <vector>

// Demonstrating the problem with vector (address instability)
// and why hive solves it

int main() {
    // --- The vector problem ---
    std::vector<int> v = {10, 20, 30};
    int* ptr = &v[1];  // Points to 20
    std::cout << "Before push_back: *ptr = " << *ptr << "\n";
    // Output: Before push_back: *ptr = 20

    v.push_back(40);  // May reallocate!
    // ptr is now DANGLING if reallocation occurred!
    // Accessing *ptr is undefined behavior

    std::cout << "After push_back: vector was reallocated, ptr is dangling!\n";

    // --- Why hive preserves addresses ---
    // std::hive allocates elements in fixed-size BLOCKS.
    // When a new element is inserted:
    //   1. If there's a reusable erased slot in an existing block -> use it
    //   2. If current block is full -> allocate a NEW block
    //   3. Existing blocks are NEVER moved or reallocated
    //
    // Therefore, pointers to elements in existing blocks remain valid.

    // Conceptual hive:
    // std::hive<int> h;
    // auto it1 = h.insert(10);
    // int* p1 = &(*it1);        // Pointer to 10
    // auto it2 = h.insert(20);  // New block may be allocated, but block with 10 stays
    // assert(*p1 == 10);        // ALWAYS valid!
    // h.erase(it2);             // Only it2 invalidated, p1 still valid
    // auto it3 = h.insert(30);  // May reuse it2's slot, p1 still valid

    // Memory layout concept:
    std::cout << "\nHive memory model:\n";
    std::cout << "Block A: [10][ ][30]  (slot 1 was erased, slot 2 reused)\n";
    std::cout << "Block B: [40][50][ ]  (new block, old blocks untouched)\n";
    std::cout << "Pointer to 10 -> always valid (Block A never moves)\n";

    return 0;
}
```

**Explanation:**

- **`std::vector`:** Stores all elements in one contiguous buffer. When capacity is exceeded, it allocates a larger buffer and **moves/copies all elements** -> all pointers, references, and iterators are invalidated.
- **`std::hive`:** Uses multiple fixed-size blocks. New blocks are allocated independently. Existing blocks are **never moved or reallocated**. Therefore, a pointer to an element remains valid as long as that specific element isn't erased.
- This is critical for systems like ECS (Entity Component Systems), physics engines, or any scenario where external structures hold pointers into the container.

### Q2: Show a use case in a game entity system where stable addresses avoid pointer invalidation

This is the canonical motivation for hive. The physics system holds raw pointers to entities. If the entity container ever moves memory, those pointers go stale. Notice how this example uses `std::list` as a stand-in because it also provides stable addresses - but the note at the end explains why hive is better in practice.

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <string>
#include <memory>
#include <algorithm>

// Simulating std::hive behavior with std::list (which also has stable addresses)
// In real code, use std::hive (C++26) or plf::hive for better cache performance

struct Transform {
    float x = 0, y = 0;
};

struct Health {
    int hp = 100;
};

struct Entity {
    int id;
    std::string name;
    Transform transform;
    Health health;
    bool alive = true;
};

// Physics system holds RAW POINTERS to entities
struct PhysicsSystem {
    std::vector<Entity*> tracked_entities;

    void track(Entity& e) {
        tracked_entities.push_back(&e);
    }

    void update() {
        for (Entity* e : tracked_entities) {
            if (e->alive) {
                e->transform.x += 1.0f;  // Requires stable address!
                e->transform.y += 0.5f;
            }
        }
    }

    void remove_dead() {
        std::erase_if(tracked_entities,
            [](Entity* e) { return !e->alive; });
    }
};

int main() {
    // Using std::list as stand-in for std::hive (both have stable addresses)
    // std::hive<Entity> entities;  // C++26
    std::list<Entity> entities;

    PhysicsSystem physics;

    // Spawn entities
    entities.push_back({1, "Player", {0, 0}, {100}});
    physics.track(entities.back());

    entities.push_back({2, "Enemy1", {10, 5}, {50}});
    physics.track(entities.back());

    entities.push_back({3, "Enemy2", {20, 10}, {30}});
    physics.track(entities.back());

    // Simulate: update physics (accesses entities via pointers)
    physics.update();

    // Kill Enemy1 - erase from container
    auto it = std::find_if(entities.begin(), entities.end(),
        [](const Entity& e) { return e.id == 2; });
    if (it != entities.end()) {
        it->alive = false;  // Mark dead
        entities.erase(it); // Remove from container
    }

    // Other entity pointers are STILL VALID (stable addresses!)
    physics.remove_dead();
    physics.update();  // Player and Enemy2 pointers still work!

    for (Entity* e : physics.tracked_entities) {
        std::cout << e->name << " at (" << e->transform.x
                  << ", " << e->transform.y << ")\n";
    }
    // Output:
    // Player at (2, 1)
    // Enemy2 at (22, 11)

    std::cout << "\nWith std::vector, erasing Enemy1 could have invalidated\n"
              << "the Player and Enemy2 pointers!\n";

    // With std::hive vs std::list:
    // - std::hive: iteration is ~3-5x faster (cache-friendly blocks + skipfield)
    // - std::list: each node is a separate allocation, poor cache behavior
    // - Both: O(1) erase, stable addresses

    return 0;
}
```

**How it works:**

- The `PhysicsSystem` stores raw pointers to entities in the container.
- When an entity is erased from the container, other entity pointers must remain valid.
- `std::vector` would invalidate all pointers on erase (shifts elements) or on growth (reallocation).
- `std::list` and `std::hive` both provide stable addresses - but `std::hive` iterates much faster due to its block-based layout with skipfields, making it ideal for game loops that iterate all entities every frame.

### Q3: Compare std::hive with a list of chunks (block list) for cache performance

This benchmark builds toward hive by showing how chunking already dramatically improves on `std::list`. The block list is essentially a manual hive without the skipfield optimization. The commentary at the end explains what the skipfield adds on top.

```cpp
#include <iostream>
#include <list>
#include <vector>
#include <array>
#include <chrono>
#include <numeric>

// Approach 1: std::list (node-based, poor cache)
// Approach 2: Block list (manual chunking, better cache)
// Approach 3: std::hive (automatic chunking + skipfield, best of both)

struct Data {
    float values[4];  // 16 bytes per element
};

template <std::size_t BlockSize>
class BlockList {
    struct Block {
        std::array<Data, BlockSize> data;
        std::array<bool, BlockSize> active{};
        std::size_t count = 0;
    };
    std::list<Block> blocks_;

public:
    void insert(const Data& d) {
        for (auto& block : blocks_) {
            for (std::size_t i = 0; i < BlockSize; ++i) {
                if (!block.active[i]) {
                    block.data[i] = d;
                    block.active[i] = true;
                    ++block.count;
                    return;
                }
            }
        }
        blocks_.push_back({});
        blocks_.back().data[0] = d;
        blocks_.back().active[0] = true;
        blocks_.back().count = 1;
    }

    float sum_iteration() const {
        float total = 0;
        for (const auto& block : blocks_) {
            for (std::size_t i = 0; i < BlockSize; ++i) {
                if (block.active[i]) {
                    total += block.data[i].values[0];
                }
            }
        }
        return total;
    }

    std::size_t size() const {
        std::size_t total = 0;
        for (const auto& b : blocks_) total += b.count;
        return total;
    }
};

int main() {
    constexpr int N = 100'000;

    // --- std::list (worst cache) ---
    {
        std::list<Data> container;
        for (int i = 0; i < N; ++i) {
            container.push_back(Data{{float(i), 0, 0, 0}});
        }

        auto start = std::chrono::high_resolution_clock::now();
        float sum = 0;
        for (int rep = 0; rep < 100; ++rep) {
            sum = 0;
            for (const auto& d : container) sum += d.values[0];
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "std::list iteration (100 reps): " << dur.count()
                  << " us, sum=" << sum << "\n";
    }
    // Expected: slowest due to pointer-chasing between individual nodes

    // --- Block list with 256-element blocks (manual hive-like) ---
    {
        BlockList<256> container;
        for (int i = 0; i < N; ++i) {
            container.insert(Data{{float(i), 0, 0, 0}});
        }

        auto start = std::chrono::high_resolution_clock::now();
        float sum = 0;
        for (int rep = 0; rep < 100; ++rep) {
            sum = container.sum_iteration();
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "BlockList<256> iteration (100 reps): " << dur.count()
                  << " us, sum=" << sum << "\n";
    }
    // Expected: 3-10x faster than std::list (cache-friendly blocks)

    // --- std::vector (best cache, no stability) ---
    {
        std::vector<Data> container(N);
        for (int i = 0; i < N; ++i) {
            container[i] = Data{{float(i), 0, 0, 0}};
        }

        auto start = std::chrono::high_resolution_clock::now();
        float sum = 0;
        for (int rep = 0; rep < 100; ++rep) {
            sum = 0;
            for (const auto& d : container) sum += d.values[0];
        }
        auto end = std::chrono::high_resolution_clock::now();
        auto dur = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
        std::cout << "std::vector iteration (100 reps): " << dur.count()
                  << " us, sum=" << sum << "\n";
    }
    // Expected: fastest (fully contiguous), but no address stability

    std::cout << "\nComparison:\n";
    std::cout << "Container       | Cache       | Address Stable | Erase Cost\n";
    std::cout << "----------------|-------------|----------------|----------\n";
    std::cout << "std::vector     | Excellent   | No             | O(n)\n";
    std::cout << "std::list       | Poor        | Yes            | O(1)\n";
    std::cout << "BlockList/hive  | Good        | Yes            | O(1)\n";

    return 0;
}
```

**How it works:**

- **`std::list`:** Each element is a separate heap allocation -> pointer chasing on every iteration -> cache miss per element. Very slow for iteration-heavy workloads.
- **Block list (manual):** Elements are packed into fixed-size arrays. Iteration within a block is contiguous and cache-friendly. Between blocks, there's one pointer indirection, but far fewer than `std::list`.
- **`std::hive`** (not yet available in compilers) improves on the block list by:
  1. Using a **jump-counting skipfield** instead of per-element bool - iteration skips erased slots in O(1) without checking each slot individually.
  2. Automatic block sizing and growth.
  3. Slot reuse for new insertions.
- Typical benchmark: `std::hive` iterates 3-10x faster than `std::list`, and within 1.5-2x of `std::vector` for dense workloads.

---

## Notes

- **`std::hive` is C++26** - not yet available in any major compiler (as of 2024). Use `plf::hive` (the reference implementation by Matt Bentley) as a polyfill.
- **Best use cases:** Game entity systems, particle systems, physics simulations, memory pools - any scenario with frequent insert/erase and iteration where pointers to elements must remain valid.
- **Not suitable for:** Sorted data (no random access), small collections (overhead not worth it), or when insertion order must be preserved.
- **Skipfield:** The key innovation. A jump-counting skipfield stores the number of consecutive erased slots, allowing the iterator to skip them in O(1) rather than checking each slot.
- **`reshape(min, max)`:** Tunes block size range. Smaller blocks waste less memory with few elements; larger blocks improve iteration throughput.
- **No `operator[]`:** Hive iterators are bidirectional, not random access. Use iterators or range-based for loops.
