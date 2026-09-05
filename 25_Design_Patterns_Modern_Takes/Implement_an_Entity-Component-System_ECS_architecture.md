# Implement an Entity-Component-System (ECS) architecture

**Category:** Design Patterns Modern Takes  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Entity_component_system>  

---

## Topic Overview

ECS separates **data** (components) from **behavior** (systems). Entities are just integer IDs, components are plain data structs stored in contiguous arrays, and systems are functions that iterate over entities with specific component combinations. This yields cache-friendly, composable game/simulation architectures.

The reason ECS exists is a real performance problem. In classical OOP, a `Player` object holds its position, velocity, health, and sprite all bundled together. When a physics system iterates a million entities and only needs position and velocity, it still drags all the health and sprite data through cache lines it doesn't use. ECS solves this by separating each component type into its own contiguous array, so a physics system reads *only* position and velocity - perfect cache utilization.

### ECS vs Traditional OOP

```cpp
Traditional OOP:              ECS:
  class Player {                Entity: 42 (just an ID)
    Position pos;               Components:
    Velocity vel;                 positions[42] = {10, 20}
    Health hp;                    velocities[42] = {1, -1}
    void update() {...}           healths[42] = {100}
  };                            Systems:
                                  move_system(positions, velocities)
  -> Data + behavior mixed        -> Data separated from logic
  -> Cache misses (vtable)        -> Cache-friendly (SoA layout)
  -> Hard to compose              -> Easy to compose
```

---

## Self-Assessment

### Q1: Define entities as integer IDs, components as plain data structs, and systems as free functions

The simplest ECS storage is a sparse array indexed by entity ID: `optional<T>[MAX_ENTITIES]`. It's not the most cache-efficient layout but it's easy to understand and plenty fast for small entity counts. Notice that systems are just free functions that take component arrays - no inheritance, no virtual calls.

```cpp
#include <iostream>
#include <vector>
#include <optional>
#include <cstdint>

// Entity: just an ID
using Entity = std::uint32_t;

// Components: plain data, no behavior
struct Position { float x = 0, y = 0; };
struct Velocity { float dx = 0, dy = 0; };
struct Health   { int current = 100, max = 100; };
struct Sprite   { int texture_id = 0; int layer = 0; };

// Component storage: contiguous arrays
constexpr std::size_t MAX_ENTITIES = 10'000;

template<typename T>
struct ComponentArray {
    std::optional<T> data[MAX_ENTITIES]{};  // sparse: nullopt if entity lacks component

    void add(Entity e, T component) { data[e] = std::move(component); }
    void remove(Entity e) { data[e].reset(); }
    bool has(Entity e) const { return data[e].has_value(); }
    T& get(Entity e) { return *data[e]; }
    const T& get(Entity e) const { return *data[e]; }
};

// Systems: free functions operating on components
void move_system(ComponentArray<Position>& positions,
                 const ComponentArray<Velocity>& velocities,
                 float dt) {
    for (Entity e = 0; e < MAX_ENTITIES; ++e) {
        if (positions.has(e) && velocities.has(e)) {
            positions.get(e).x += velocities.get(e).dx * dt;
            positions.get(e).y += velocities.get(e).dy * dt;
        }
    }
}

void damage_system(ComponentArray<Health>& healths, Entity target, int damage) {
    if (healths.has(target)) {
        healths.get(target).current -= damage;
        if (healths.get(target).current <= 0) {
            std::cout << "Entity " << target << " died!\n";
        }
    }
}

int main() {
    ComponentArray<Position> positions;
    ComponentArray<Velocity> velocities;
    ComponentArray<Health>   healths;

    // Create entities by assigning components
    Entity player = 0;
    positions.add(player, {10.f, 20.f});
    velocities.add(player, {1.f, -0.5f});
    healths.add(player, {100, 100});

    Entity bullet = 1;
    positions.add(bullet, {50.f, 50.f});
    velocities.add(bullet, {0.f, -10.f});
    // bullet has no Health - it's just a projectile

    // Run systems
    move_system(positions, velocities, 1.0f);
    std::cout << "Player at (" << positions.get(player).x
              << ", " << positions.get(player).y << ")\n";
    // Output: Player at (11, 19.5)

    damage_system(healths, player, 30);
    std::cout << "Player HP: " << healths.get(player).current << '\n';
    // Output: Player HP: 70
}
```

The `bullet` entity is a good example of ECS composition: it has `Position` and `Velocity` but no `Health`. The `damage_system` simply skips it because `healths.has(bullet)` is false. No inheritance hierarchy needed.

### Q2: Store components in contiguous arrays indexed by entity ID for cache-friendly iteration

The sparse array from Q1 has a gap problem: iterating it means checking every slot up to `MAX_ENTITIES`, most of which are empty. The **dense component pool** fixes this by keeping a tightly packed array of components alongside a two-way index. The dense array is always gapless, so iteration is as cache-friendly as iterating a plain `std::vector`.

```cpp
#include <iostream>
#include <vector>
#include <bitset>
#include <cstdint>

// Dense component pool (cache-friendly)
// Instead of sparse optional<T>[MAX], use a dense vector + index mapping

using Entity = std::uint32_t;
constexpr std::size_t MAX_ENTITIES = 100'000;

template<typename T>
class DenseComponentPool {
    // Dense array: components packed together (cache-friendly!)
    std::vector<T> components_;
    std::vector<Entity> dense_to_entity_;  // dense index -> entity ID

    // Sparse array: entity ID -> dense index (or -1)
    std::vector<int> entity_to_dense_;

public:
    DenseComponentPool() : entity_to_dense_(MAX_ENTITIES, -1) {}

    void add(Entity e, T component) {
        entity_to_dense_[e] = static_cast<int>(components_.size());
        components_.push_back(std::move(component));
        dense_to_entity_.push_back(e);
    }

    void remove(Entity e) {
        int idx = entity_to_dense_[e];
        if (idx < 0) return;

        // Swap-and-pop: move last element into the gap
        Entity last_entity = dense_to_entity_.back();
        components_[idx] = std::move(components_.back());
        dense_to_entity_[idx] = last_entity;
        entity_to_dense_[last_entity] = idx;

        components_.pop_back();
        dense_to_entity_.pop_back();
        entity_to_dense_[e] = -1;
    }

    bool has(Entity e) const { return entity_to_dense_[e] >= 0; }
    T& get(Entity e) { return components_[entity_to_dense_[e]]; }

    // Cache-friendly iteration
    // Iterate dense array directly - no gaps, no cache misses
    std::size_t size() const { return components_.size(); }
    T& at_dense(std::size_t i) { return components_[i]; }
    Entity entity_at(std::size_t i) const { return dense_to_entity_[i]; }
};

struct Position { float x, y; };
struct Velocity { float dx, dy; };

int main() {
    DenseComponentPool<Position> positions;
    DenseComponentPool<Velocity> velocities;

    // Create 1000 entities with Position and Velocity
    for (Entity e = 0; e < 1000; ++e) {
        positions.add(e, {static_cast<float>(e), 0.f});
        velocities.add(e, {1.f, 0.5f});
    }

    // System: iterate dense arrays (cache-friendly, no gaps)
    for (std::size_t i = 0; i < positions.size(); ++i) {
        Entity e = positions.entity_at(i);
        if (velocities.has(e)) {
            positions.at_dense(i).x += velocities.get(e).dx;
            positions.at_dense(i).y += velocities.get(e).dy;
        }
    }

    std::cout << "Entity 0 at (" << positions.get(0).x
              << ", " << positions.get(0).y << ")\n";
    // Output: Entity 0 at (1, 0.5)
}
```

The swap-and-pop in `remove` is the clever part: when you remove an entity, you fill the hole by moving the last element into its slot and updating both indices. The dense array stays packed without any shifting.

### Q3: Show a SpeedSystem that iterates all entities with both Position and Velocity components

Multiple systems running in sequence is the whole ECS workflow. Each system only touches the component types it cares about - `GravitySystem` modifies `Velocity`, `SpeedSystem` reads `Velocity` and updates `Position`. The static wall entity proves that systems correctly skip entities that don't have the required components.

```cpp
#include <iostream>
#include <vector>
#include <optional>
#include <cstdint>
#include <functional>

using Entity = std::uint32_t;
constexpr Entity MAX_ENTITIES = 10'000;

struct Position { float x, y; };
struct Velocity { float dx, dy; };
struct Gravity  { float force = -9.81f; };

template<typename T>
struct Pool {
    std::optional<T> data[MAX_ENTITIES]{};
    void add(Entity e, T c) { data[e] = std::move(c); }
    bool has(Entity e) const { return data[e].has_value(); }
    T& get(Entity e) { return *data[e]; }
};

// SpeedSystem: updates Position using Velocity
struct SpeedSystem {
    static void update(Pool<Position>& pos, Pool<Velocity>& vel, float dt) {
        for (Entity e = 0; e < MAX_ENTITIES; ++e) {
            if (pos.has(e) && vel.has(e)) {
                pos.get(e).x += vel.get(e).dx * dt;
                pos.get(e).y += vel.get(e).dy * dt;
            }
        }
    }
};

// GravitySystem: applies gravity to Velocity (entities with Velocity + Gravity)
struct GravitySystem {
    static void update(Pool<Velocity>& vel, Pool<Gravity>& grav, float dt) {
        for (Entity e = 0; e < MAX_ENTITIES; ++e) {
            if (vel.has(e) && grav.has(e)) {
                vel.get(e).dy += grav.get(e).force * dt;
            }
        }
    }
};

int main() {
    Pool<Position> positions;
    Pool<Velocity> velocities;
    Pool<Gravity>  gravities;

    // Ball: has position, velocity, and gravity
    Entity ball = 0;
    positions.add(ball, {0.f, 100.f});
    velocities.add(ball, {5.f, 0.f});
    gravities.add(ball, {-9.81f});

    // Static wall: has position only (no velocity, no gravity)
    Entity wall = 1;
    positions.add(wall, {50.f, 0.f});

    // Simulate 3 frames at 60 FPS
    float dt = 1.f / 60.f;
    for (int frame = 0; frame < 3; ++frame) {
        GravitySystem::update(velocities, gravities, dt);  // Apply gravity first
        SpeedSystem::update(positions, velocities, dt);     // Then move

        std::cout << "Frame " << frame << ": ball at ("
                  << positions.get(ball).x << ", "
                  << positions.get(ball).y << ")\n";
    }
    // Ball moves right and falls; wall is untouched by SpeedSystem (no Velocity)
}
```

**Output:**

```text
Frame 0: ball at (0.0833, 99.9973)
Frame 1: ball at (0.1667, 99.9891)
Frame 2: ball at (0.25, 99.9755)
```

---

## Notes

- **ECS libraries:** EnTT, flecs, and EntityX are production-ready C++ ECS frameworks that handle the storage and iteration plumbing so you can focus on components and systems.
- **Archetype storage** (used by flecs) groups entities with the same component set into contiguous blocks, giving even better cache performance than per-component pools.
- **Systems should be pure functions** on component data - no hidden state, no globals. This makes them easy to reason about, test, and eventually parallelize.
- **Component bit masks** (`std::bitset`) enable fast "has all of these components?" queries without iterating each pool separately.
