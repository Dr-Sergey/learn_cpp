# Apply data-oriented design: prefer struct-of-arrays over array-of-structs

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/performance>  

---

## Topic Overview

This file complements **Item #209** (cache-friendly data structures) with a focus on the **AoS -> SoA transformation** - when to apply it, the code pattern, and a hybrid approach.

The key insight is that the right layout depends entirely on your access pattern. Neither AoS nor SoA is universally better. If you always process every field of every object together, AoS keeps related data co-located and wins. If you regularly scan one field across millions of objects, SoA puts all that field's data contiguously in memory and wins. The decision guide below makes this concrete.

### Decision Guide

Work through these questions top to bottom:

```cpp
Do you iterate over many objects?  --- No --> AoS is fine
         | Yes
         v
Do you access all fields?  --------- Yes --> AoS (all data per cache line)
         | No (hot/cold split)
         v
Use SoA (or AoSoA hybrid)
```

### AoSoA - Hybrid Layout

When neither pure SoA nor pure AoS fits perfectly, the AoSoA (Array of Struct of Arrays) layout splits data into small fixed-size blocks. Each block is SoA internally, so vectorization still works, while the blocks themselves have some spatial locality for multi-field access:

```cpp
AoSoA (Block size = 4):
Block 0: [x0 x1 x2 x3] [y0 y1 y2 y3] [z0 z1 z2 z3]
Block 1: [x4 x5 x6 x7] [y4 y5 y6 y7] [z4 z5 z6 z7]

Combines: SoA cache efficiency + AoS spatial locality
```

---

## Self-Assessment

### Q1: Transform an AoS to SoA and measure improvement on a single-field scan

This benchmark scans only the `health` field of 5 million entities. In AoS every health value is 24 bytes apart; in SoA they're 4 bytes apart. That gap is the entire reason for the speedup:

```cpp
#include <chrono>
#include <iostream>
#include <numeric>
#include <vector>

// AoS: 6 fields, 24 bytes per element
struct EntityAoS {
    float x, y, z;
    float health;
    int team;
    int id;
};

// SoA
struct EntitiesSoA {
    std::vector<float> x, y, z;
    std::vector<float> health;
    std::vector<int> team, id;
    void resize(size_t n) {
        x.resize(n); y.resize(n); z.resize(n);
        health.resize(n); team.resize(n); id.resize(n);
    }
};

int main() {
    constexpr size_t N = 5'000'000;

    std::vector<EntityAoS> aos(N);
    for (size_t i = 0; i < N; ++i)
        aos[i] = {float(i), float(i), float(i), 100.0f, int(i % 2), int(i)};

    EntitiesSoA soa;
    soa.resize(N);
    for (size_t i = 0; i < N; ++i) {
        soa.health[i] = 100.0f;
    }

    // Task: sum health values only
    auto t1 = std::chrono::high_resolution_clock::now();
    float sum_aos = 0;
    for (size_t i = 0; i < N; ++i)
        sum_aos += aos[i].health;  // loads 24 bytes per element, uses 4
    auto t2 = std::chrono::high_resolution_clock::now();

    auto t3 = std::chrono::high_resolution_clock::now();
    float sum_soa = 0;
    for (size_t i = 0; i < N; ++i)
        sum_soa += soa.health[i];  // loads 4 bytes per element, uses 4
    auto t4 = std::chrono::high_resolution_clock::now();

    auto ms_aos = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto ms_soa = std::chrono::duration<double, std::milli>(t4 - t3).count();

    std::cout << "AoS: " << ms_aos << " ms (sum=" << sum_aos << ")\n";
    std::cout << "SoA: " << ms_soa << " ms (sum=" << sum_soa << ")\n";
    std::cout << "Speedup: " << (ms_aos / ms_soa) << "x\n";
}
// Typical output:
// AoS: ~12 ms
// SoA: ~4 ms
// Speedup: ~3x
```

A 3x speedup from a layout change alone is typical for this kind of partial-field scan. No algorithm change, no parallelism - just putting the bytes closer together.

### Q2: Explain field-isolation advantage and when AoS wins

**SoA wins when accessing a subset of fields:**

- Processing `health` alone in SoA: 4 bytes/element x 5M = 20 MB touched
- Processing `health` alone in AoS: 24 bytes/element x 5M = 120 MB touched
- **6x less memory bandwidth** for SoA

**AoS wins when accessing all fields per element:**

```cpp
// All-field access: AoS keeps all data of one entity in the same cache line
void process_entity(EntityAoS& e) {
    e.x += e.health * 0.01f;  // x and health are co-located
    e.y += float(e.team);     // team is in the same cache line
}
// With SoA, these would be 4 separate cache misses (x[], health[], team[], y[])
```

When `process_entity` touches four fields, AoS fetches them all in (roughly) one or two cache lines because they live next to each other. SoA forces four separate cache line accesses - one for each array. Here AoS wins.

**Decision table:**

| Access pattern | Best layout | Why |
| --- | --- | --- |
| One field, many elements | SoA | Perfect cache utilization |
| All fields, many elements | AoS | Co-located, one cache line |
| Random single element | AoS | All fields in one fetch |
| Hot/cold field split | Hybrid | Hot fields in SoA, cold in AoS |

### Q3: Implement a hybrid hot/cold split for a particle system

The hot/cold split is the most practically useful DoD technique. The idea: separate the fields you touch every frame (hot) from fields you only need occasionally (cold). The hot fields get SoA layout so the frame-update loop is cache-friendly; the cold fields sit in their own AoS structure that you only touch during rare events like collision handling.

```cpp
#include <iostream>
#include <vector>

// Hybrid: hot data in SoA (updated every frame), cold data in AoS (rarely used)
struct ParticleSystem {
    // HOT data (SoA): accessed every frame
    std::vector<float> x, y, z;     // positions
    std::vector<float> vx, vy, vz;  // velocities

    // COLD data (AoS): accessed rarely (e.g., on collision)
    struct ColdData {
        int material_id;
        float spawn_time;
        int texture_index;
        char name[16];
    };
    std::vector<ColdData> cold;

    size_t count = 0;

    void add_particle(float px, float py, float pz, int mat) {
        x.push_back(px); y.push_back(py); z.push_back(pz);
        vx.push_back(0); vy.push_back(0); vz.push_back(0);
        cold.push_back({mat, 0.0f, 0, "particle"});
        ++count;
    }

    // Hot path: only touches position + velocity arrays
    void update(float dt) {
        for (size_t i = 0; i < count; ++i) {
            x[i] += vx[i] * dt;
            y[i] += vy[i] * dt;
            z[i] += vz[i] * dt;
        }
    }

    // Cold path: only on collision
    int material_at(size_t i) const { return cold[i].material_id; }
};

int main() {
    ParticleSystem ps;
    for (int i = 0; i < 5; ++i)
        ps.add_particle(float(i), 0, 0, i % 3);

    ps.vx[0] = 10.0f;
    ps.update(1.0f);

    std::cout << "Particle 0 at x=" << ps.x[0] << '\n';
    std::cout << "Material: " << ps.material_at(0) << '\n';
}
// Expected output:
// Particle 0 at x=10
// Material: 0
```

The `update` loop only touches six float arrays - position and velocity. The `cold` array is never loaded into cache during the frame update, so all that cache space is available for the data you actually need. When a collision happens and you call `material_at`, the cold array gets loaded then - but collisions are rare, so the cost is amortized.

---

## Notes

- The hot/cold split is the most practical DoD technique for real-world code.
- ECS (Entity Component System) frameworks (EnTT, flecs) are built on SoA/DoD.
- Use `-march=native` to enable auto-vectorization (SSE/AVX) on SoA loops.
- Measure with `perf stat -e LLC-load-misses` to verify cache improvement.
