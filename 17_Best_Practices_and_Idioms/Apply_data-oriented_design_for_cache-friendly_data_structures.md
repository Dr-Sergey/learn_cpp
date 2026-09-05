# Apply data-oriented design for cache-friendly data structures

**Category:** Best Practices and Idioms  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/performance>  

---

## Topic Overview

**Data-Oriented Design (DoD)** focuses on how data flows through the CPU cache hierarchy. The core idea is simple: modern CPUs are fast, but memory is slow. When the CPU needs a value that isn't in cache, it stalls and waits - sometimes for hundreds of cycles. By organizing data for sequential access patterns, you minimize those stalls and give the compiler room to generate SIMD (vectorized) instructions.

The traditional OOP way of thinking organizes data around objects: each object bundles all its fields together. DoD asks a different question - how is the data *processed*? If you only ever look at one field at a time across many objects, bundling all the other fields alongside it wastes cache space.

### AoS vs SoA

The diagram below makes the memory layout difference immediately visible:

```cpp
Array of Structs (AoS):                Struct of Arrays (SoA):
┌───────────────────────┐         ┌───────────────────────┐
│ [x,y,z,vx,vy,vz,mass] │         │ x:  [x0 x1 x2 x3 ...]  │
│ [x,y,z,vx,vy,vz,mass] │         │ y:  [y0 y1 y2 y3 ...]  │
│ [x,y,z,vx,vy,vz,mass] │         │ z:  [z0 z1 z2 z3 ...]  │
│ ...                    │         │ vx: [vx0 vx1 ...]      │
└───────────────────────┘         └───────────────────────┘

If you only need x values:
AoS: loads x,y,z,vx,vy,vz,mass per cache line (6/7 wasted)
SoA: loads x0,x1,x2,...  per cache line (all useful)
```

In AoS, when you fetch `x[0]` from memory you pull a whole cache line - but that line contains `y0`, `z0`, `vx0`, and all the other fields you don't need right now. In SoA, that same cache line holds `x0` through `x15`: sixteen useful values instead of one.

### Cache Line Utilization

The numbers make the efficiency gap concrete:

| Layout | Bytes per particle | Bytes used per cache line (64B) | Utilization |
| --- | --- | --- | --- |
| AoS (access x only) | 28 (7x4) | 4 out of 28 per struct = ~14% | Poor |
| SoA (access x only) | 4 | 64 out of 64 = 100% | Optimal |

---

## Self-Assessment

### Q1: Transform an array of structs (AoS) to a struct of arrays (SoA) and measure cache miss improvement

This benchmark runs the same position update - `x += vx * dt` - over 10 million particles in both layouts. The only difference is memory layout; the arithmetic is identical:

```cpp
#include <chrono>
#include <iostream>
#include <vector>
#include <cmath>

// AoS: Array of Structs
struct ParticleAoS {
    float x, y, z;
    float vx, vy, vz;
    float mass;
};

// SoA: Struct of Arrays
struct ParticlesSoA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;

    void resize(size_t n) {
        x.resize(n); y.resize(n); z.resize(n);
        vx.resize(n); vy.resize(n); vz.resize(n);
        mass.resize(n);
    }
};

int main() {
    constexpr size_t N = 10'000'000;
    constexpr float dt = 0.01f;

    // AoS setup
    std::vector<ParticleAoS> aos(N);
    for (size_t i = 0; i < N; ++i) {
        aos[i] = {float(i), float(i), float(i),
                  1.0f, 2.0f, 3.0f, 1.0f};
    }

    // SoA setup
    ParticlesSoA soa;
    soa.resize(N);
    for (size_t i = 0; i < N; ++i) {
        soa.x[i] = float(i); soa.y[i] = float(i); soa.z[i] = float(i);
        soa.vx[i] = 1.0f; soa.vy[i] = 2.0f; soa.vz[i] = 3.0f;
        soa.mass[i] = 1.0f;
    }

    // Benchmark AoS: update positions
    auto t1 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < N; ++i) {
        aos[i].x += aos[i].vx * dt;
        aos[i].y += aos[i].vy * dt;
        aos[i].z += aos[i].vz * dt;
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark SoA: update positions
    auto t3 = std::chrono::high_resolution_clock::now();
    for (size_t i = 0; i < N; ++i) {
        soa.x[i] += soa.vx[i] * dt;
        soa.y[i] += soa.vy[i] * dt;
        soa.z[i] += soa.vz[i] * dt;
    }
    auto t4 = std::chrono::high_resolution_clock::now();

    auto aos_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto soa_ms = std::chrono::duration<double, std::milli>(t4 - t3).count();

    std::cout << "AoS: " << aos_ms << " ms\n";
    std::cout << "SoA: " << soa_ms << " ms\n";
    std::cout << "Speedup: " << (aos_ms / soa_ms) << "x\n";
}
// Typical output (10M particles, -O2):
// AoS: ~45 ms
// SoA: ~20 ms
// Speedup: ~2.2x
```

The roughly 2x speedup comes entirely from better cache utilization - the CPU is spending less time waiting for memory and more time doing arithmetic. On AVX-capable hardware with auto-vectorization, the gap can be even larger.

### Q2: Explain why SoA is faster when processing only one field of a large struct array

**Cache line analysis:**

```cpp
Cache line = 64 bytes = 16 floats

AoS: sizeof(ParticleAoS) = 28 bytes (7 floats)
  Processing only 'x' field:
  Cache line loads: [x0,y0,z0,vx0,vy0,vz0,mass0, x1,y1,...]
  Useful bytes: 4/28 per particle = 14% utilization
  For 10M particles: ~280MB touched, only ~40MB useful

SoA: x array is contiguous floats
  Processing only 'x' field:
  Cache line loads: [x0,x1,x2,...,x15]  (16 x values per line)
  Useful bytes: 64/64 = 100% utilization
  For 10M particles: ~40MB touched, all useful
```

That's not just a cache story - it's also a memory bandwidth story. SoA reads 7x less data from RAM for this access pattern, which matters a lot on systems where memory bandwidth is the bottleneck.

**Additional benefits of SoA:**

- **SIMD vectorization:** Compiler can auto-vectorize contiguous float arrays (4 floats per SSE, 8 per AVX)
- **Prefetching:** Sequential access lets hardware prefetcher work perfectly
- **Reduced memory bandwidth:** Less data transferred from RAM

**When AoS is better:**

- When you always access ALL fields of each struct together
- When objects are accessed randomly (not sequentially)
- When object identity matters (passing single particles around)

### Q3: Apply DoD to a particle system and show SIMD-friendly memory layout

Here's a production-style particle system using SoA layout, with `alignas(32)` to hint to the compiler that AVX-width loads are safe. Notice how `update_positions` and `apply_gravity` each touch only the arrays they need - nothing else pollutes the cache:

```cpp
#include <cstddef>
#include <iostream>
#include <vector>
#include <cstdlib>

// SIMD-friendly particle system using SoA
struct alignas(32) ParticleSystem {
    // Position arrays (aligned for AVX)
    std::vector<float> x, y, z;
    // Velocity arrays
    std::vector<float> vx, vy, vz;
    // Properties (separate: not always needed)
    std::vector<float> mass;
    std::vector<bool>  alive;

    size_t count = 0;

    void init(size_t n) {
        count = n;
        x.resize(n); y.resize(n); z.resize(n);
        vx.resize(n); vy.resize(n); vz.resize(n);
        mass.resize(n); alive.resize(n);
        for (size_t i = 0; i < n; ++i) {
            x[i] = float(i); y[i] = 0; z[i] = 0;
            vx[i] = 1.0f; vy[i] = 0.5f; vz[i] = 0;
            mass[i] = 1.0f; alive[i] = true;
        }
    }

    // SIMD-friendly: compiler auto-vectorizes these tight loops
    void update_positions(float dt) {
        for (size_t i = 0; i < count; ++i) x[i] += vx[i] * dt;
        for (size_t i = 0; i < count; ++i) y[i] += vy[i] * dt;
        for (size_t i = 0; i < count; ++i) z[i] += vz[i] * dt;
    }

    // Apply gravity (only needs vy)
    void apply_gravity(float g, float dt) {
        for (size_t i = 0; i < count; ++i)
            vy[i] += g * dt;  // touches only vy array
    }
};

int main() {
    ParticleSystem ps;
    ps.init(100);

    for (int frame = 0; frame < 60; ++frame) {
        ps.apply_gravity(-9.8f, 1.0f / 60.0f);
        ps.update_positions(1.0f / 60.0f);
    }

    std::cout << "Particle 0: (" << ps.x[0] << ", " << ps.y[0] << ")\n";
    std::cout << "Particle 99: (" << ps.x[99] << ", " << ps.y[99] << ")\n";
}
// Expected output (approx):
// Particle 0: (1, -4.9)
// Particle 99: (100, -4.9)
```

The three separate loops in `update_positions` look odd at first - why not one loop that updates x, y, and z together? The answer is that each separate loop iterates over a single contiguous array, which gives the auto-vectorizer a clean, dependency-free stream of data. A merged loop that interleaves x, y, and z updates may confuse the vectorizer or produce less efficient code.

---

## Notes

- DoD doesn't mean "never use objects" - it means organize data for how it's **processed**, not how it's **conceptualized**.
- Profile first: AoS may be fine if struct fits in a cache line and all fields are used together.
- Use `perf stat` (Linux) or VTune to measure actual cache miss rates.
- Alignment with `alignas(32)` helps AVX instructions.
- SoA + SIMD can give 4-8x speedup on float-heavy computations.
