# Implement the Entity-Component-System (ECS) pattern for game or simulation engines

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**ECS** completely separates three concerns that OOP typically mixes together: identity (entities), data (components), and behavior (systems). An entity is nothing but an integer ID. A component is a plain data struct with no methods. A system is a function that iterates over every entity that has a particular combination of components and does something with the data.

The reason this matters for games and simulations is performance. In a traditional OOP game engine, a `GameObject` might be a large heap-allocated object with dozens of fields. The update loop jumps around memory following pointers to scattered objects. CPUs hate that pattern - they load a cache line, use a few bytes, then have to load another one from somewhere completely different. ECS stores all `Position` components in one contiguous array, all `Velocity` components in another, and so on. The movement system walks through two flat arrays in sequence, and the CPU prefetcher is happy. That is the data-oriented design payoff.

Composition is the other big win. In OOP you might have a deep inheritance hierarchy: `MovingObject` extends `GameObject`, `Enemy` extends `MovingObject`, `FlyingEnemy` extends `Enemy`. Adding a "flying" ability to a `Player` gets messy. In ECS, you just attach an `AIController` component to enemies and leave it off the player. Systems that process `AIController` automatically skip the player. There is no hierarchy to untangle.

### ECS vs Traditional OOP

| Aspect | OOP Inheritance | ECS |
| --- | --- | --- |
| Entity | Object with methods | Just an ID |
| Data | Mixed with behavior | Separate components (SOA) |
| Behavior | Virtual methods | Systems iterate components |
| Cache | Poor (scattered objects) | Excellent (contiguous arrays) |
| Composition | Inheritance hierarchy | Mix-and-match components |
| Parallelism | Difficult | Natural (systems on disjoint data) |

---

## Self-Assessment

### Q1: Implement a basic ECS framework

The framework has three moving parts. The `ComponentPool<T>` stores all components of one type in a contiguous vector and maintains an index from entity ID to position in that vector. The `World` owns all pools and all entity-to-component bitmasks. The bitmask is how `each<Ts...>` efficiently finds entities that have all the required components.

**Answer:**

```cpp
#include <cstdint>
#include <vector>
#include <unordered_map>
#include <bitset>
#include <memory>
#include <typeindex>
#include <cassert>
#include <algorithm>

// === Entity: just an ID ===
using Entity = uint32_t;
constexpr Entity NULL_ENTITY = 0;

// === Component ID system ===
inline uint32_t next_component_id() {
    static uint32_t id = 0;
    return id++;
}

template<typename T>
uint32_t component_id() {
    static uint32_t id = next_component_id();
    return id;
}

constexpr size_t MAX_COMPONENTS = 64;
using ComponentMask = std::bitset<MAX_COMPONENTS>;

// === Component pool: contiguous storage for one type ===
class IComponentPool {
public:
    virtual ~IComponentPool() = default;
    virtual void remove(Entity e) = 0;
};

template<typename T>
class ComponentPool : public IComponentPool {
public:
    T& add(Entity e, T component = {}) {
        assert(entity_to_index_.find(e) == entity_to_index_.end());
        size_t index = data_.size();
        data_.push_back(std::move(component));
        entities_.push_back(e);
        entity_to_index_[e] = index;
        return data_[index];
    }

    void remove(Entity e) override {
        auto it = entity_to_index_.find(e);
        if (it == entity_to_index_.end()) return;

        size_t index = it->second;
        size_t last = data_.size() - 1;

        if (index != last) {
            // Swap with last to maintain contiguity
            data_[index] = std::move(data_[last]);
            entities_[index] = entities_[last];
            entity_to_index_[entities_[index]] = index;
        }

        data_.pop_back();
        entities_.pop_back();
        entity_to_index_.erase(e);
    }

    T* get(Entity e) {
        auto it = entity_to_index_.find(e);
        return it != entity_to_index_.end() ? &data_[it->second] : nullptr;
    }

    bool has(Entity e) const {
        return entity_to_index_.count(e) > 0;
    }

    // Direct array access for systems
    std::vector<T>& data() { return data_; }
    const std::vector<Entity>& entities() const { return entities_; }
    size_t size() const { return data_.size(); }

private:
    std::vector<T> data_;           // Contiguous component data
    std::vector<Entity> entities_;  // Parallel entity array
    std::unordered_map<Entity, size_t> entity_to_index_;
};

// === World: manages entities and components ===
class World {
public:
    Entity create() {
        Entity e = next_entity_++;
        masks_[e] = {};
        return e;
    }

    void destroy(Entity e) {
        for (auto& [type, pool] : pools_) {
            pool->remove(e);
        }
        masks_.erase(e);
    }

    template<typename T>
    T& add(Entity e, T component = {}) {
        auto& pool = get_or_create_pool<T>();
        masks_[e].set(component_id<T>());
        return pool.add(e, std::move(component));
    }

    template<typename T>
    void remove(Entity e) {
        get_pool<T>().remove(e);
        masks_[e].reset(component_id<T>());
    }

    template<typename T>
    T* get(Entity e) {
        return get_pool<T>().get(e);
    }

    template<typename T>
    bool has(Entity e) const {
        return masks_.at(e).test(component_id<T>());
    }

    // Query: iterate entities with specific components
    template<typename... Ts>
    void each(std::function<void(Entity, Ts&...)> fn) {
        ComponentMask required;
        (required.set(component_id<Ts>()), ...);

        for (auto& [entity, mask] : masks_) {
            if ((mask & required) == required) {
                fn(entity, *get<Ts>(entity)...);
            }
        }
    }

private:
    template<typename T>
    ComponentPool<T>& get_or_create_pool() {
        auto id = component_id<T>();
        if (pools_.find(id) == pools_.end()) {
            pools_[id] = std::make_unique<ComponentPool<T>>();
        }
        return static_cast<ComponentPool<T>&>(*pools_[id]);
    }

    template<typename T>
    ComponentPool<T>& get_pool() {
        return static_cast<ComponentPool<T>&>(*pools_.at(component_id<T>()));
    }

    Entity next_entity_ = 1;
    std::unordered_map<Entity, ComponentMask> masks_;
    std::unordered_map<uint32_t, std::unique_ptr<IComponentPool>> pools_;
};
```

The swap-with-last trick in `ComponentPool::remove` is worth pausing on. Erasing from the middle of a vector is O(n) because it shifts everything down. Swapping the target element with the last element and then popping the last is O(1). The trade-off is that the order of components in the array changes, which is fine for ECS because systems iterate all of them regardless of order.

### Q2: Define components and systems

Components are pure data - no methods, no inheritance, no virtual functions. Systems are plain functions that ask the world to call them for every entity matching a component signature. The separation is strict: the data owns nothing, and the logic knows nothing about individual entities beyond what is in its component slots.

**Answer:**

```cpp
// === Components: plain data, no behavior ===
struct Position { float x, y, z; };
struct Velocity { float dx, dy, dz; };
struct Health { int current, max; };
struct Sprite { int texture_id; int width, height; };
struct Collider { float radius; };
struct AIController { float aggro_range; Entity target; };

// === Systems: pure functions operating on component sets ===

// Movement system: processes (Position, Velocity)
void movement_system(World& world, float dt) {
    world.each<Position, Velocity>(
        [dt](Entity e, Position& pos, Velocity& vel) {
            pos.x += vel.dx * dt;
            pos.y += vel.dy * dt;
            pos.z += vel.dz * dt;
        });
}

// Collision system: processes (Position, Collider)
void collision_system(World& world) {
    struct ColliderData {
        Entity entity;
        Position* pos;
        Collider* col;
    };
    std::vector<ColliderData> colliders;

    world.each<Position, Collider>(
        [&](Entity e, Position& pos, Collider& col) {
            colliders.push_back({e, &pos, &col});
        });

    // O(n^2) broad phase (use spatial hash in production)
    for (size_t i = 0; i < colliders.size(); ++i) {
        for (size_t j = i + 1; j < colliders.size(); ++j) {
            float dx = colliders[i].pos->x - colliders[j].pos->x;
            float dy = colliders[i].pos->y - colliders[j].pos->y;
            float dist_sq = dx * dx + dy * dy;
            float r = colliders[i].col->radius + colliders[j].col->radius;
            if (dist_sq < r * r) {
                handle_collision(colliders[i].entity, colliders[j].entity);
            }
        }
    }
}

// Render system: processes (Position, Sprite)
void render_system(World& world, Renderer& renderer) {
    world.each<Position, Sprite>(
        [&](Entity e, Position& pos, Sprite& sprite) {
            renderer.draw(sprite.texture_id,
                         pos.x, pos.y, sprite.width, sprite.height);
        });
}

// === Game loop ===
void game_loop(World& world, Renderer& renderer) {
    float dt = 1.0f / 60.0f;
    while (running) {
        movement_system(world, dt);
        collision_system(world);
        render_system(world, renderer);
    }
}
```

Each system only sees the components it asks for. The movement system has no idea that some entities also have `AIController` or `Health`. The render system does not care about physics. This makes adding a new system trivially safe - it cannot accidentally affect code in other systems unless they share a component.

### Q3: Creating entities by composing components

Entity creation in ECS is just a sequence of component additions. There is no class to inherit from, no constructor hierarchy to satisfy. A player and an enemy differ only in which components they have. A static tree has even fewer components than either one.

**Answer:**

```cpp
// === Entity creation through composition (no inheritance!) ===

Entity create_player(World& world, float x, float y) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Velocity>(e, {0, 0, 0});
    world.add<Health>(e, {100, 100});
    world.add<Sprite>(e, {TEXTURE_PLAYER, 32, 32});
    world.add<Collider>(e, {16.0f});
    // No AIController - player is human-controlled
    return e;
}

Entity create_enemy(World& world, float x, float y) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Velocity>(e, {0, 0, 0});
    world.add<Health>(e, {50, 50});
    world.add<Sprite>(e, {TEXTURE_ENEMY, 32, 32});
    world.add<Collider>(e, {16.0f});
    world.add<AIController>(e, {200.0f, NULL_ENTITY});
    return e;
}

Entity create_projectile(World& world, float x, float y,
                          float dx, float dy) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Velocity>(e, {dx, dy, 0});
    world.add<Sprite>(e, {TEXTURE_BULLET, 8, 8});
    world.add<Collider>(e, {4.0f});
    // No Health - projectiles are destroyed on collision
    return e;
}

// Static prop: only Position + Sprite (no physics)
Entity create_tree(World& world, float x, float y) {
    Entity e = world.create();
    world.add<Position>(e, {x, y, 0});
    world.add<Sprite>(e, {TEXTURE_TREE, 64, 96});
    return e;
}

// The movement_system automatically skips trees (no Velocity component)
// The AI system automatically skips players (no AIController component)
// Composition > inheritance!
```

That last comment is the payoff of the whole section. The movement system skips trees for free, the AI system skips players for free. In an OOP hierarchy you would need virtual functions returning false, or abstract methods, or null checks. Here, absence of the component IS the answer, and no explicit code is needed to express it.

---

## Notes

- Components are data only - no methods, no virtual functions, no inheritance.
- Systems are functions - they iterate over matching component sets.
- Contiguous component storage = excellent cache performance (data-oriented design).
- Popular C++ ECS libraries: **EnTT**, **flecs**, **entityx**.
- Archetype-based ECS (like flecs) stores entities with the same component set together for even better cache performance.
- Systems with disjoint component access can run in parallel (movement || render).
- ECS scales to millions of entities; OOP inheritance hierarchies don't.
