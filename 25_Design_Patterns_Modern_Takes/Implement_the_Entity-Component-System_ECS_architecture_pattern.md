# Implement the Entity-Component-System (ECS) architecture pattern

**Category:** Design Patterns Modern Takes  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/performance>  

---

## Topic Overview

This third ECS file focuses on **Structure of Arrays (SoA) layout for cache performance** and **component queries** (archetypes) - the two pillars that make ECS faster than traditional OOP hierarchies for data-heavy game and simulation workloads.

The core performance insight is about cache lines. A modern CPU cache line is 64 bytes. When you iterate over a million entities updating their positions, you want each cache line to be full of position data - not a mixture of position, health, name, and whatever else the entity struct holds. SoA achieves exactly that.

### AoS vs SoA Memory Layout

The diagram below shows why the layout choice matters. When you run a movement system that only touches position and velocity, you don't want health and name data polluting your cache lines.

```cpp
AoS (Array of Structs):             SoA (Struct of Arrays):
  Entity[0]: {pos, vel, health}       positions:  [p0, p1, p2, p3, ...]
  Entity[1]: {pos, vel, health}       velocities: [v0, v1, v2, v3, ...]
  Entity[2]: {pos, vel, health}       health:     [h0, h1, h2, h3, ...]

  Cache line loads pos+vel+health     Cache line loads pos0+pos1+pos2+...
  for ONE entity                      for MANY entities
  Bad for: iterate all positions      Good for: iterate all positions
```

---

## Self-Assessment

### Q1: Define entities as integer IDs, components as plain data structs, and systems as functions over components

**Answer:**

The fundamental ECS insight is the separation of concerns: entities are just IDs with no behavior, components are just data with no behavior, and systems are just functions that operate on component arrays. Nothing is hidden behind a virtual function table. This is what lets the compiler and CPU optimize the inner loops aggressively.

```cpp
#include <iostream>
#include <vector>
#include <cstdint>
#include <cmath>

// Entity = just an ID
using Entity = uint32_t;

// Components = plain data
struct Position { float x, y; };
struct Velocity { float dx, dy; };
struct Health   { int hp, max_hp; };

// SoA component storage
struct World {
    std::vector<Entity> entities;
    std::vector<Position> positions;
    std::vector<Velocity> velocities;
    std::vector<Health> health;

    Entity create(Position pos, Velocity vel, Health hp) {
        Entity id = static_cast<Entity>(entities.size());
        entities.push_back(id);
        positions.push_back(pos);
        velocities.push_back(vel);
        health.push_back(hp);
        return id;
    }
    size_t count() const { return entities.size(); }
};

// Systems = free functions over components
void movement_system(World& w, float dt) {
    for (size_t i = 0; i < w.count(); ++i) {
        w.positions[i].x += w.velocities[i].dx * dt;
        w.positions[i].y += w.velocities[i].dy * dt;
    }
}

void damage_system(World& w, int amount) {
    for (size_t i = 0; i < w.count(); ++i) {
        w.health[i].hp = std::max(0, w.health[i].hp - amount);
    }
}

void print_system(const World& w) {
    for (size_t i = 0; i < w.count(); ++i) {
        std::cout << "Entity " << w.entities[i]
                  << ": pos=(" << w.positions[i].x << "," << w.positions[i].y
                  << ") hp=" << w.health[i].hp << '\n';
    }
}

int main() {
    World world;
    world.create({0, 0}, {1, 0.5f}, {100, 100});
    world.create({10, 5}, {-0.5f, 1}, {80, 80});

    movement_system(world, 2.0f);
    damage_system(world, 15);
    print_system(world);
    // Entity 0: pos=(2,1) hp=85
    // Entity 1: pos=(9,7) hp=65
}
```

Notice that `movement_system` only touches `positions` and `velocities` - it never looks at `health`. In the SoA layout, those two arrays are laid out contiguously in memory, so iterating through them is extremely cache-friendly.

### Q2: Use a SoA layout for component storage and show cache performance vs AoS

**Answer:**

The benchmark below makes the cache difference measurable. The AoS entity struct has extra fields (`name`, `faction`, `armor`, `damage`) that the position update loop doesn't need. Every time the CPU loads a cache line to read a position, it also loads those unused fields - wasting cache capacity. The SoA version packs only position and velocity into tightly contiguous arrays, so the useful data density per cache line is much higher.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <cmath>

struct Vec2 { float x, y; };

// AoS: Array of Structs
struct AoS_Entity {
    Vec2 position;
    Vec2 velocity;
    float health;
    float armor;
    float damage;
    char name[32];        // Padding / unused data pollutes cache line
    int faction;
};

void update_positions_aos(std::vector<AoS_Entity>& entities, float dt) {
    for (auto& e : entities) {
        e.position.x += e.velocity.x * dt;
        e.position.y += e.velocity.y * dt;
    }
}

// SoA: Struct of Arrays
struct SoA_World {
    std::vector<float> pos_x, pos_y;
    std::vector<float> vel_x, vel_y;
    // health, armor, etc. in separate arrays
};

void update_positions_soa(SoA_World& w, float dt, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        w.pos_x[i] += w.vel_x[i] * dt;
        w.pos_y[i] += w.vel_y[i] * dt;
    }
}

int main() {
    constexpr size_t N = 1'000'000;

    // Setup AoS
    std::vector<AoS_Entity> aos(N);
    for (size_t i = 0; i < N; ++i) {
        aos[i].position = {0, 0};
        aos[i].velocity = {1.0f, 0.5f};
    }

    // Setup SoA
    SoA_World soa;
    soa.pos_x.resize(N, 0); soa.pos_y.resize(N, 0);
    soa.vel_x.resize(N, 1.0f); soa.vel_y.resize(N, 0.5f);

    // Benchmark AoS
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int frame = 0; frame < 100; ++frame)
        update_positions_aos(aos, 0.016f);
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark SoA
    auto t3 = std::chrono::high_resolution_clock::now();
    for (int frame = 0; frame < 100; ++frame)
        update_positions_soa(soa, 0.016f, N);
    auto t4 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "AoS: " << ms(t2 - t1).count() << " ms\n";
    std::cout << "SoA: " << ms(t4 - t3).count() << " ms\n";
    // Typical: SoA is 2-5x faster due to cache utilization
    // AoS loads 56 bytes per entity (most unused)
    // SoA loads 8 bytes per entity (pos_x + vel_x, tightly packed)
}
```

The reason this trips people up is that the AoS version *looks* clean and object-oriented. The struct groups related data nicely. But when a tight loop only needs two of the ten fields in that struct, the other eight fields are dead weight that fills up cache lines and forces more memory traffic.

### Q3: Implement component queries that return all entities with a given set of components

**Answer:**

Not every entity has every component. A camera entity has a position but no health. A trigger zone has no position in the physical sense but triggers events. Component queries let a system ask "give me all entities that have both Position and Velocity" - only the entities that can meaningfully participate in the movement system. The component mask using `std::bitset` makes that check a single bitwise AND operation.

```cpp
#include <iostream>
#include <vector>
#include <bitset>
#include <cstdint>
#include <optional>

// Component type IDs
enum ComponentType : size_t {
    POSITION = 0,
    VELOCITY = 1,
    HEALTH   = 2,
    RENDER   = 3,
    MAX_COMPONENTS = 64
};

using ComponentMask = std::bitset<MAX_COMPONENTS>;
using Entity = uint32_t;

struct Position { float x, y; };
struct Velocity { float dx, dy; };
struct Health   { int hp; };

// World with component masks
struct World {
    std::vector<ComponentMask> masks;
    std::vector<std::optional<Position>> positions;
    std::vector<std::optional<Velocity>> velocities;
    std::vector<std::optional<Health>>   health;

    Entity create() {
        Entity id = static_cast<Entity>(masks.size());
        masks.emplace_back();
        positions.emplace_back();
        velocities.emplace_back();
        health.emplace_back();
        return id;
    }

    void add_position(Entity e, Position p) {
        masks[e].set(POSITION);
        positions[e] = p;
    }
    void add_velocity(Entity e, Velocity v) {
        masks[e].set(VELOCITY);
        velocities[e] = v;
    }
    void add_health(Entity e, Health h) {
        masks[e].set(HEALTH);
        health[e] = h;
    }

    // Query: find entities matching a component mask
    std::vector<Entity> query(ComponentMask required) const {
        std::vector<Entity> result;
        for (size_t i = 0; i < masks.size(); ++i) {
            if ((masks[i] & required) == required) {
                result.push_back(static_cast<Entity>(i));
            }
        }
        return result;
    }
};

int main() {
    World w;

    // Entity 0: has Position + Velocity + Health
    auto e0 = w.create();
    w.add_position(e0, {0, 0});
    w.add_velocity(e0, {1, 0});
    w.add_health(e0, {100});

    // Entity 1: has Position + Velocity (no Health)
    auto e1 = w.create();
    w.add_position(e1, {5, 5});
    w.add_velocity(e1, {0, -1});

    // Entity 2: has Position + Health (no Velocity)
    auto e2 = w.create();
    w.add_position(e2, {10, 10});
    w.add_health(e2, {50});

    // Query: entities with Position AND Velocity
    ComponentMask movable;
    movable.set(POSITION);
    movable.set(VELOCITY);
    auto results = w.query(movable);

    std::cout << "Movable entities: ";
    for (auto e : results) std::cout << e << " ";
    std::cout << '\n';  // 0 1

    // Query: entities with Health
    ComponentMask has_health;
    has_health.set(HEALTH);
    auto alive = w.query(has_health);

    std::cout << "Entities with health: ";
    for (auto e : alive) std::cout << e << " ";
    std::cout << '\n';  // 0 2
}
```

The query `(masks[i] & required) == required` checks whether every required bit is set in the entity's mask. If the entity has extra components beyond what was requested, that's fine - the check still passes. Entity 0 has all three components and shows up in both queries as expected.

---

## Notes

- **SoA wins** when systems touch few components per entity - the movement system only reads position and velocity, so AoS wastes cache loading health, name, and other unused fields.
- **AoS wins** when you frequently access all components of one entity at once - serialization, deep copy, or UI inspection of a single selected object.
- Component masks (`bitset`) enable O(1) per-entity matching. Production ECS frameworks use **archetypes** - groups of entities with identical masks - for even faster iteration, since all entities in an archetype can be iterated without any per-entity mask check.
- Real ECS frameworks (EnTT, flecs) use sparse sets for O(1) component add/remove and cache-friendly iteration over archetypes.
