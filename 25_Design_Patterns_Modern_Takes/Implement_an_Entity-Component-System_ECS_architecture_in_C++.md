# Implement an Entity-Component-System (ECS) architecture in C++

**Category:** Design Patterns Modern Takes  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Entity_component_system>  

---

## Topic Overview

This second ECS file focuses on **advanced storage strategies**: type-erased component arrays, the AoS vs SoA debate, and the **sparse set** data structure that enables O(1) component operations without hash maps.

If the first ECS file was about the idea, this one is about the engineering. Real ECS frameworks don't use `optional<T>[MAX_ENTITIES]` - they use carefully designed storage that minimizes indirection, maximizes cache hits, and keeps add/remove at O(1). Each of the three questions here addresses one of those concerns.

### AoS vs SoA Layout

```cpp
AoS (Array of Structs) - traditional OOP:
  entities[] = [ {pos, vel, hp}, {pos, vel, hp}, {pos, vel, hp} ]
  Cache line: |pos|vel|hp|pos|vel|hp|...
  Problem: iterating only positions loads vel+hp into cache too -> waste

SoA (Struct of Arrays) - ECS style:
  positions[]  = [ pos, pos, pos, pos, pos ]  <- contiguous
  velocities[] = [ vel, vel, vel, vel, vel ]  <- contiguous
  healths[]    = [ hp,  hp,  hp,  hp,  hp ]   <- contiguous
  Cache line: |pos|pos|pos|pos|...  <- only what you need
  Result: 3-5x faster iteration for systems that touch 1-2 components
```

---

## Self-Assessment

### Q1: Design an ECS where entities are integer IDs, components are stored in type-erased arrays, and systems iterate component views

The key architectural idea here is the `World` class acting as a registry. It holds one `ComponentPool<T>` per type, keyed by `std::type_index`. Because `ComponentPool<T>` inherits from `IComponentPool`, the world can store all pools in one map without knowing the concrete types at compile time.

```cpp
#include <iostream>
#include <memory>
#include <unordered_map>
#include <typeindex>
#include <vector>
#include <cstdint>
#include <optional>
#include <cassert>

using Entity = std::uint32_t;

// Type-erased component storage base
class IComponentPool {
public:
    virtual ~IComponentPool() = default;
    virtual void remove(Entity e) = 0;
    virtual bool has(Entity e) const = 0;
};

template<typename T>
class ComponentPool : public IComponentPool {
    std::unordered_map<Entity, std::size_t> entity_to_index_;
    std::vector<T> components_;
    std::vector<Entity> entities_;
public:
    void add(Entity e, T component) {
        entity_to_index_[e] = components_.size();
        components_.push_back(std::move(component));
        entities_.push_back(e);
    }

    void remove(Entity e) override {
        auto it = entity_to_index_.find(e);
        if (it == entity_to_index_.end()) return;
        std::size_t idx = it->second;
        // Swap-and-pop
        Entity last = entities_.back();
        components_[idx] = std::move(components_.back());
        entities_[idx] = last;
        entity_to_index_[last] = idx;
        components_.pop_back();
        entities_.pop_back();
        entity_to_index_.erase(it);
    }

    bool has(Entity e) const override { return entity_to_index_.count(e); }
    T& get(Entity e) { return components_[entity_to_index_.at(e)]; }

    // For system iteration
    std::size_t size() const { return components_.size(); }
    T& at(std::size_t i) { return components_[i]; }
    Entity entity_at(std::size_t i) const { return entities_[i]; }
};

// World: manages entities and type-erased pools
class World {
    Entity next_entity_ = 0;
    std::unordered_map<std::type_index, std::unique_ptr<IComponentPool>> pools_;

    template<typename T>
    ComponentPool<T>& get_pool() {
        auto key = std::type_index(typeid(T));
        if (!pools_.count(key))
            pools_[key] = std::make_unique<ComponentPool<T>>();
        return static_cast<ComponentPool<T>&>(*pools_[key]);
    }
public:
    Entity create() { return next_entity_++; }

    template<typename T>
    void add(Entity e, T component) { get_pool<T>().add(e, std::move(component)); }

    template<typename T>
    T& get(Entity e) { return get_pool<T>().get(e); }

    template<typename T>
    bool has(Entity e) { return get_pool<T>().has(e); }

    // View: iterate entities with specific components
    template<typename... Ts, typename Func>
    void each(Func&& func) {
        // Use the smallest pool for iteration
        auto& first_pool = get_pool<std::tuple_element_t<0, std::tuple<Ts...>>>();
        for (std::size_t i = 0; i < first_pool.size(); ++i) {
            Entity e = first_pool.entity_at(i);
            if ((get_pool<Ts>().has(e) && ...)) {  // Fold expression
                func(e, get_pool<Ts>().get(e)...);
            }
        }
    }
};

struct Position { float x, y; };
struct Velocity { float dx, dy; };
struct Health { int hp; };

int main() {
    World world;

    auto player = world.create();
    world.add(player, Position{0, 0});
    world.add(player, Velocity{1, 2});
    world.add(player, Health{100});

    auto bullet = world.create();
    world.add(bullet, Position{10, 10});
    world.add(bullet, Velocity{0, -5});

    // System: move all entities with Position + Velocity
    world.each<Position, Velocity>([](Entity e, Position& pos, Velocity& vel) {
        pos.x += vel.dx;
        pos.y += vel.dy;
    });

    std::cout << "Player: (" << world.get<Position>(player).x
              << ", " << world.get<Position>(player).y << ")\n";
    // Output: Player: (1, 2)
}
```

The `each<Position, Velocity>()` call uses a fold expression `(get_pool<Ts>().has(e) && ...)` to check all required component types in one shot. If any is missing, the entity is skipped. The lambda then receives references to each component directly - no indirection, no casts.

### Q2: Show why AoS is poor for ECS and how SoA component pools improve cache performance

This is the core performance argument for ECS. The move system needs `pos_x`, `pos_y`, `vel_x`, `vel_y` - 16 bytes per entity. In AoS layout, each entity struct is 28 bytes, so 43% of every cache line is wasted data the system doesn't use. In SoA layout, position and velocity each live in their own tightly-packed arrays, so every cache line holds only useful data. The compiler can even auto-vectorize the SoA loop using SIMD instructions.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <cstdint>

// AoS: Array of Structs (poor cache usage)
struct EntityAoS {
    float pos_x, pos_y;   // Position
    float vel_x, vel_y;   // Velocity
    int health;            // Health
    int texture_id;        // Sprite
    float ai_timer;        // AI state
    // 28 bytes per entity - but move system only needs 16 bytes (pos + vel)
};

void move_system_aos(std::vector<EntityAoS>& entities, float dt) {
    for (auto& e : entities) {
        e.pos_x += e.vel_x * dt;   // Load 28 bytes, use 16
        e.pos_y += e.vel_y * dt;   // Cache lines wasted on health, texture, ai_timer
    }
}

// SoA: Struct of Arrays (cache-friendly)
struct WorldSoA {
    std::vector<float> pos_x, pos_y;  // Contiguous position data
    std::vector<float> vel_x, vel_y;  // Contiguous velocity data
    std::vector<int> health;           // Separate, not loaded by move system
    std::vector<int> texture_id;       // Separate
    std::vector<float> ai_timer;       // Separate
};

void move_system_soa(WorldSoA& w, float dt) {
    std::size_t n = w.pos_x.size();
    for (std::size_t i = 0; i < n; ++i) {
        w.pos_x[i] += w.vel_x[i] * dt;  // Only loads pos + vel
        w.pos_y[i] += w.vel_y[i] * dt;  // Perfect cache utilization
    }
    // Compiler can auto-vectorize this with SIMD (SSE/AVX)!
}

int main() {
    constexpr int N = 1'000'000;

    // AoS setup
    std::vector<EntityAoS> aos(N, {1, 2, 0.5f, 0.3f, 100, 1, 0});

    // SoA setup
    WorldSoA soa;
    soa.pos_x.resize(N, 1); soa.pos_y.resize(N, 2);
    soa.vel_x.resize(N, 0.5f); soa.vel_y.resize(N, 0.3f);

    // Benchmark AoS
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 100; ++i) move_system_aos(aos, 0.016f);
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark SoA
    auto t3 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < 100; ++i) move_system_soa(soa, 0.016f);
    auto t4 = std::chrono::high_resolution_clock::now();

    auto aos_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto soa_ms = std::chrono::duration<double, std::milli>(t4 - t3).count();
    std::cout << "AoS: " << aos_ms << " ms\n";
    std::cout << "SoA: " << soa_ms << " ms\n";
    std::cout << "Speedup: " << aos_ms / soa_ms << "x\n";
    // Typical: SoA is 3-5x faster due to cache efficiency + SIMD
}
```

### Q3: Implement a sparse set for O(1) component add/remove/has without hash maps

The sparse set is worth slowing down on because the idea is genuinely clever. The reason a hash map isn't used here is that `std::unordered_map` has unpredictable memory access patterns - each lookup follows a pointer to a bucket. The sparse set avoids this by using two plain arrays. `sparse[entity]` gives you an index into `dense[]` in one step, and `dense[]` stays packed (no gaps) so iteration is just a sequential scan.

The reason this trips people up is the `remove` operation. You can't just set `sparse[e] = INVALID` and leave a hole in `dense[]` - that would break iteration. The trick is to swap the removed element with the last element of `dense[]`, pop the back, and update the sparse index for the entity that was just moved.

```cpp
#include <iostream>
#include <vector>
#include <cstdint>
#include <cassert>

using Entity = std::uint32_t;

// Sparse Set: O(1) add, remove, has, iteration
// Two arrays:
//   sparse[entity] -> index into dense (or invalid)
//   dense[index]   -> entity ID
// dense is always packed (no gaps) -> cache-friendly iteration

template<typename T>
class SparseSet {
    static constexpr Entity INVALID = ~Entity{0};

    std::vector<Entity> sparse_;  // entity ID -> dense index
    std::vector<Entity> dense_;   // dense index -> entity ID
    std::vector<T> components_;   // dense index -> component data

    void ensure_sparse(Entity e) {
        if (e >= sparse_.size())
            sparse_.resize(e + 1, INVALID);
    }

public:
    // O(1) add
    void add(Entity e, T component) {
        ensure_sparse(e);
        assert(!has(e));
        sparse_[e] = static_cast<Entity>(dense_.size());
        dense_.push_back(e);
        components_.push_back(std::move(component));
    }

    // O(1) remove (swap-and-pop)
    void remove(Entity e) {
        assert(has(e));
        Entity idx = sparse_[e];
        Entity last_entity = dense_.back();

        // Swap with last element
        dense_[idx] = last_entity;
        components_[idx] = std::move(components_.back());
        sparse_[last_entity] = idx;

        // Pop last
        dense_.pop_back();
        components_.pop_back();
        sparse_[e] = INVALID;
    }

    // O(1) has
    bool has(Entity e) const {
        return e < sparse_.size() &&
               sparse_[e] != INVALID &&
               sparse_[e] < dense_.size() &&
               dense_[sparse_[e]] == e;
    }

    // O(1) get
    T& get(Entity e) { return components_[sparse_[e]]; }

    // Cache-friendly iteration (packed array, no gaps)
    std::size_t size() const { return dense_.size(); }
    Entity entity_at(std::size_t i) const { return dense_[i]; }
    T& component_at(std::size_t i) { return components_[i]; }
};

struct Position { float x, y; };

int main() {
    SparseSet<Position> positions;

    positions.add(0, {10, 20});
    positions.add(5, {50, 60});   // Sparse: entity 5, no gap in dense
    positions.add(3, {30, 40});

    std::cout << std::boolalpha;
    std::cout << "Has 5: " << positions.has(5) << '\n';  // true
    std::cout << "Has 2: " << positions.has(2) << '\n';  // false

    // Remove entity 5 - O(1), maintains packed dense array
    positions.remove(5);
    std::cout << "Has 5 after remove: " << positions.has(5) << '\n'; // false
    std::cout << "Dense size: " << positions.size() << '\n';          // 2

    // Iterate - no gaps, cache-friendly
    for (std::size_t i = 0; i < positions.size(); ++i) {
        auto e = positions.entity_at(i);
        auto& p = positions.component_at(i);
        std::cout << "Entity " << e << " at (" << p.x << ", " << p.y << ")\n";
    }
}
```

**Output:**

```text
Has 5: true
Has 2: false
Has 5 after remove: false
Dense size: 2
Entity 0 at (10, 20)
Entity 3 at (30, 40)
```

---

## Notes

- **Sparse set** is the core data structure in EnTT, the most popular C++ ECS library. Understanding it gives you genuine insight into how production-grade ECS works.
- **Archetype-based ECS** (used by flecs) groups entities by their exact component combination, which allows even tighter cache packing for iteration.
- **SoA is essential for SIMD:** compilers can auto-vectorize SoA loops using SSE/AVX instructions but generally cannot vectorize AoS because the data isn't contiguous.
- **Trade-off:** the sparse set uses O(max_entity) memory for the sparse array, but all operations are O(1) and iteration is cache-friendly with no gaps.
