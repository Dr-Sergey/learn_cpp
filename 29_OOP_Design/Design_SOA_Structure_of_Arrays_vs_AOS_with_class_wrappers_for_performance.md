# Design SOA (Structure of Arrays) vs AOS with class wrappers for performance

**Category:** OOP Design  
**Standard:** Not version-specific  

---

## Topic Overview

**AOS (Array of Structures)** and **SOA (Structure of Arrays)** are two fundamental data layout strategies with dramatically different cache performance. AOS groups all fields of one entity together - which is the natural OOP way. SOA groups all values of the same field across all entities into their own array. Choosing the wrong layout for a hot loop can easily cost you 2-10x in runtime performance.

The reason this matters is the CPU cache. A cache line is 64 bytes. When your loop only touches one field (say, updating `x` positions), AOS loads every other field of the entity into cache for nothing - that's wasted bandwidth. SOA keeps all `x` values packed together, so every byte you load from memory is actually used.

### Layout Comparison

```cpp
AOS (Array of Structures):              SOA (Structure of Arrays):
┌─────────────────────────────┐         ┌────────────────────────┐
│ x₀ y₀ z₀ mass₀ │ x₁ y₁ z₁ │         │ x₀ x₁ x₂ x₃ ... xₙ  │  <- all x together
│ mass₁ │ x₂ y₂ z₂ mass₂ │..│         │ y₀ y₁ y₂ y₃ ... yₙ  │  <- all y together
└─────────────────────────────┘         │ z₀ z₁ z₂ z₃ ... zₙ  │  <- all z together
                                        │ m₀ m₁ m₂ m₃ ... mₙ  │  <- all mass together
                                        └────────────────────────┘
```

### When Each Layout Wins

| Scenario | AOS | SOA | Why |
| --- | --- | --- | --- |
| Process ALL fields of one entity | Best | Worse | AOS: one cache line has everything |
| Process ONE field across all entities | Worse | Best | SOA: perfect sequential access |
| SIMD vectorization | Hard | Best | SOA: contiguous same-type data |
| Random access by entity | Best | Worse | AOS: single pointer dereference |
| Insert/delete entities | Simpler | Complex | SOA: must keep arrays in sync |
| Iterate subset of fields (hot loop) | Wastes cache | Best | SOA: no irrelevant data loaded |

### Cache Line Utilization

The numbers here make the tradeoff concrete. With a 16-byte entity and 64-byte cache lines:

```cpp
Cache line = 64 bytes.  Entity = { float x, y, z, mass; } = 16 bytes

AOS: 4 entities per cache line. If loop only reads x -> 75% wasted bandwidth.
SOA: 16 floats per cache line. If loop only reads x -> 100% utilized.
```

That 75% waste is not hypothetical - it directly translates to 3-4x fewer useful bytes fetched per memory transaction, which means more stalls and lower throughput.

---

## Self-Assessment

### Q1: Implement both AOS and SOA particle systems and measure the difference

Here's a side-by-side implementation with a benchmark. Pay attention to what the `update_positions` and `sum_x` loops actually touch in memory for each layout.

**Answer:**

```cpp
#include <vector>
#include <chrono>
#include <iostream>
#include <cmath>
#include <random>

// AOS Layout
struct ParticleAOS {
    float x, y, z;
    float vx, vy, vz;
    float mass;
    float charge;     // Unused in velocity update -> wastes cache
    uint32_t flags;   // Unused in velocity update -> wastes cache
    float padding;    // Realistic: entities often have extra fields
};

class ParticleSystemAOS {
public:
    std::vector<ParticleAOS> particles;

    void update_positions(float dt) {
        for (auto& p : particles) {
            p.x += p.vx * dt;
            p.y += p.vy * dt;
            p.z += p.vz * dt;
        }
    }

    // Compute sum of x - only needs 1 field but loads entire struct
    float sum_x() const {
        float s = 0;
        for (const auto& p : particles)
            s += p.x;
        return s;
    }
};

// SOA Layout
class ParticleSystemSOA {
public:
    // Each field in its own contiguous array
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
    std::vector<float> charge;
    std::vector<uint32_t> flags;

    void resize(size_t n) {
        x.resize(n); y.resize(n); z.resize(n);
        vx.resize(n); vy.resize(n); vz.resize(n);
        mass.resize(n); charge.resize(n); flags.resize(n);
    }

    size_t size() const { return x.size(); }

    void update_positions(float dt) {
        const size_t n = x.size();
        // Compiler can auto-vectorize these tight loops
        for (size_t i = 0; i < n; ++i) x[i] += vx[i] * dt;
        for (size_t i = 0; i < n; ++i) y[i] += vy[i] * dt;
        for (size_t i = 0; i < n; ++i) z[i] += vz[i] * dt;
    }

    // Only touches x array - perfect cache utilization
    float sum_x() const {
        float s = 0;
        for (size_t i = 0; i < x.size(); ++i)
            s += x[i];
        return s;
    }
};

// Benchmark

template<typename F>
double benchmark(F&& func, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i)
        func();
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
    constexpr size_t N = 1'000'000;
    constexpr int ITERS = 100;
    std::mt19937 rng(42);
    std::uniform_real_distribution<float> dist(-1.0f, 1.0f);

    // Setup AOS
    ParticleSystemAOS aos;
    aos.particles.resize(N);
    for (auto& p : aos.particles) {
        p.x = dist(rng); p.y = dist(rng); p.z = dist(rng);
        p.vx = dist(rng); p.vy = dist(rng); p.vz = dist(rng);
        p.mass = 1.0f; p.charge = 0.0f; p.flags = 0;
    }

    // Setup SOA
    ParticleSystemSOA soa;
    soa.resize(N);
    rng.seed(42);  // Same data
    for (size_t i = 0; i < N; ++i) {
        soa.x[i] = dist(rng); soa.y[i] = dist(rng); soa.z[i] = dist(rng);
        soa.vx[i] = dist(rng); soa.vy[i] = dist(rng); soa.vz[i] = dist(rng);
    }

    // Benchmark position update
    double aos_time = benchmark([&] { aos.update_positions(0.016f); }, ITERS);
    double soa_time = benchmark([&] { soa.update_positions(0.016f); }, ITERS);
    std::cout << "Position update (ms/" << ITERS << " iters):\n";
    std::cout << "  AOS: " << aos_time << "\n";
    std::cout << "  SOA: " << soa_time << "\n";
    std::cout << "  Speedup: " << aos_time / soa_time << "x\n\n";

    // Benchmark single-field scan
    volatile float sink;
    double aos_sum = benchmark([&] { sink = aos.sum_x(); }, ITERS);
    double soa_sum = benchmark([&] { sink = soa.sum_x(); }, ITERS);
    std::cout << "Sum x (ms/" << ITERS << " iters):\n";
    std::cout << "  AOS: " << aos_sum << "\n";
    std::cout << "  SOA: " << soa_sum << "\n";
    std::cout << "  Speedup: " << aos_sum / soa_sum << "x\n";
}
```

**Typical results (1M particles, release build):**

- Position update: SOA 1.5-3x faster (less cache waste, better vectorization)
- Single-field scan: SOA 3-8x faster (loads only needed data)

The single-field scan shows the most dramatic gap because the AOS loop loads the full 40-byte struct per particle but uses only 4 bytes. The SOA version loads nothing it doesn't need.

### Q2: Design a C++ class wrapper that provides OOP ergonomics over an SOA layout

One downside of raw SOA is that code like `store.x[i] = 10.0f` doesn't feel natural if you're used to working with entity objects. A proxy type bridges the gap: the storage stays SOA, but the API feels like `entity.x() = 10.0f`.

**Answer:**

```cpp
#include <vector>
#include <iostream>
#include <cassert>

// SOA container with proxy accessor

class EntityStore {
public:
    // Proxy provides AOS-like syntax over SOA storage
    class Proxy {
        EntityStore& store_;
        size_t idx_;
    public:
        Proxy(EntityStore& store, size_t idx) : store_(store), idx_(idx) {}

        float& x()    { return store_.x_[idx_]; }
        float& y()    { return store_.y_[idx_]; }
        float& z()    { return store_.z_[idx_]; }
        float& mass() { return store_.mass_[idx_]; }
        int& type()   { return store_.type_[idx_]; }

        float x()    const { return store_.x_[idx_]; }
        float y()    const { return store_.y_[idx_]; }
        float z()    const { return store_.z_[idx_]; }
        float mass() const { return store_.mass_[idx_]; }
        int type()   const { return store_.type_[idx_]; }
    };

    class ConstProxy {
        const EntityStore& store_;
        size_t idx_;
    public:
        ConstProxy(const EntityStore& store, size_t idx) : store_(store), idx_(idx) {}
        float x()    const { return store_.x_[idx_]; }
        float y()    const { return store_.y_[idx_]; }
        float z()    const { return store_.z_[idx_]; }
        float mass() const { return store_.mass_[idx_]; }
        int type()   const { return store_.type_[idx_]; }
    };

    size_t size() const { return x_.size(); }

    size_t add(float x, float y, float z, float mass, int type) {
        size_t idx = x_.size();
        x_.push_back(x); y_.push_back(y); z_.push_back(z);
        mass_.push_back(mass); type_.push_back(type);
        return idx;
    }

    Proxy operator[](size_t idx) { return {*this, idx}; }
    ConstProxy operator[](size_t idx) const { return {*this, idx}; }

    // Bulk operations - the real performance wins
    // These operate directly on contiguous arrays
    void apply_gravity(float dt) {
        const size_t n = y_.size();
        for (size_t i = 0; i < n; ++i)
            y_[i] -= 9.81f * mass_[i] * dt;
    }

    // Filter by type - touches only type_ array
    std::vector<size_t> find_by_type(int target_type) const {
        std::vector<size_t> result;
        for (size_t i = 0; i < type_.size(); ++i) {
            if (type_[i] == target_type)
                result.push_back(i);
        }
        return result;
    }

    // Direct array access for hot paths
    float* x_data() { return x_.data(); }
    float* y_data() { return y_.data(); }
    float* z_data() { return z_.data(); }
    const float* x_data() const { return x_.data(); }
    const float* y_data() const { return y_.data(); }

private:
    std::vector<float> x_, y_, z_;
    std::vector<float> mass_;
    std::vector<int> type_;
};

int main() {
    EntityStore store;

    // AOS-like syntax via proxy
    store.add(1.0f, 2.0f, 3.0f, 1.0f, 0);
    store.add(4.0f, 5.0f, 6.0f, 2.0f, 1);
    store.add(7.0f, 8.0f, 9.0f, 1.5f, 0);

    // Individual access looks like AOS
    auto entity = store[1];
    std::cout << "Entity 1: (" << entity.x() << ", " << entity.y() << ")\n";
    entity.x() = 10.0f;  // Modify through proxy

    // Bulk operations are SOA-efficient
    store.apply_gravity(0.016f);

    // Type-based query - only touches type array
    auto type0 = store.find_by_type(0);
    std::cout << "Type-0 entities: " << type0.size() << "\n";  // 2
}
```

The proxy is a thin view into the SOA arrays - it holds a reference to the store and an index, and each accessor just indexes the right array. Hot path bulk operations like `apply_gravity` bypass the proxy entirely and loop directly over the contiguous arrays.

### Q3: Implement a hybrid AOSOA layout for SIMD-friendly processing

AOSOA (Array of Structure of Arrays) is the pattern used in production game and physics engines. The idea is to divide entities into fixed-size chunks (matching the SIMD register width, typically 8 floats for AVX2). Within a chunk the data is SOA, giving vectorizability. Across chunks it's AOS, giving entity locality. You get the best of both worlds.

**Answer:**

```cpp
#include <array>
#include <vector>
#include <iostream>
#include <cstdint>
#include <cassert>

// AOSOA: Array of Structure of Arrays
// Groups entities into SIMD-width chunks.
// Within each chunk, fields are contiguous (SOA).
// Chunks are stored in an array (AOS of chunks).

// This gives both:
// - Cache locality (chunk fits in cache line)
// - SIMD friendliness (contiguous same-type data in chunk)

constexpr size_t SIMD_WIDTH = 8;  // AVX2: 8 floats

struct ParticleChunk {
    // Each array is exactly SIMD_WIDTH - fits in 1-2 cache lines
    alignas(32) float x[SIMD_WIDTH];
    alignas(32) float y[SIMD_WIDTH];
    alignas(32) float z[SIMD_WIDTH];
    alignas(32) float vx[SIMD_WIDTH];
    alignas(32) float vy[SIMD_WIDTH];
    alignas(32) float vz[SIMD_WIDTH];
    alignas(32) float mass[SIMD_WIDTH];
    uint8_t active[SIMD_WIDTH];  // Mask for valid entities
    size_t count = 0;            // Number of active in this chunk
};

class ParticleSystemAOSOA {
public:
    void add(float x, float y, float z,
             float vx, float vy, float vz, float mass) {
        // Find chunk with space or create new one
        if (chunks_.empty() || chunks_.back().count == SIMD_WIDTH) {
            chunks_.emplace_back();
        }
        auto& chunk = chunks_.back();
        size_t i = chunk.count;
        chunk.x[i] = x; chunk.y[i] = y; chunk.z[i] = z;
        chunk.vx[i] = vx; chunk.vy[i] = vy; chunk.vz[i] = vz;
        chunk.mass[i] = mass;
        chunk.active[i] = 1;
        chunk.count++;
        total_++;
    }

    // Hot loop: processes SIMD_WIDTH elements at a time
    // Compiler can auto-vectorize due to alignment and contiguous layout
    void update_positions(float dt) {
        for (auto& chunk : chunks_) {
            // This inner loop processes exactly 8 elements
            // Perfect for AVX2 _mm256_fmadd_ps or auto-vectorization
            for (size_t i = 0; i < chunk.count; ++i) {
                chunk.x[i] += chunk.vx[i] * dt;
                chunk.y[i] += chunk.vy[i] * dt;
                chunk.z[i] += chunk.vz[i] * dt;
            }
        }
    }

    float total_kinetic_energy() const {
        float total = 0;
        for (const auto& chunk : chunks_) {
            // Inner loop: SIMD-friendly
            for (size_t i = 0; i < chunk.count; ++i) {
                float v2 = chunk.vx[i] * chunk.vx[i]
                         + chunk.vy[i] * chunk.vy[i]
                         + chunk.vz[i] * chunk.vz[i];
                total += 0.5f * chunk.mass[i] * v2;
            }
        }
        return total;
    }

    size_t size() const { return total_; }
    size_t chunk_count() const { return chunks_.size(); }

    // Access individual entity
    struct EntityRef {
        ParticleChunk& chunk;
        size_t idx;
        float& x() { return chunk.x[idx]; }
        float& y() { return chunk.y[idx]; }
        float& z() { return chunk.z[idx]; }
    };

    EntityRef at(size_t global_idx) {
        size_t chunk_idx = global_idx / SIMD_WIDTH;
        size_t local_idx = global_idx % SIMD_WIDTH;
        return {chunks_[chunk_idx], local_idx};
    }

private:
    std::vector<ParticleChunk> chunks_;
    size_t total_ = 0;
};

int main() {
    ParticleSystemAOSOA sys;

    // Add 100 particles
    for (int i = 0; i < 100; ++i) {
        sys.add(float(i), 0.0f, 0.0f,
                1.0f, 0.0f, 0.0f, 1.0f);
    }

    std::cout << "Particles: " << sys.size() << "\n";
    std::cout << "Chunks: " << sys.chunk_count()
              << " (" << SIMD_WIDTH << " per chunk)\n";

    sys.update_positions(0.016f);

    auto ref = sys.at(42);
    std::cout << "Particle 42 x: " << ref.x() << "\n";

    std::cout << "Total KE: " << sys.total_kinetic_energy() << "\n";
}
```

**AOSOA advantages:**

- Inner loops are exactly SIMD width, making auto-vectorization trivial for the compiler.
- Chunk data fits in L1 cache, minimizing cache misses.
- Entities within a chunk have some locality for multi-field access, which is better than pure SOA.
- This layout is used extensively in Unity DOTS, physics engines, and particle systems.

---

## Notes

- **Measure before optimizing** - AOS is simpler and faster for small N or full-entity access.
- SOA shines when N is large (>10K), loops touch a subset of fields, or SIMD is desired.
- AOSOA is the sweet spot for SIMD: contiguous same-type data within a cache-friendly chunk size.
- Use `alignas(32)` or `alignas(64)` for SIMD alignment in SOA/AOSOA arrays.
- `std::vector<float>` for SOA fields guarantees contiguous allocation.
- Consider `std::vector<std::array<float, N>>` for compile-time-sized SOA.
- Entity Component Systems (ECS) like EnTT use SOA/AOSOA internally.
- Proxy/reference objects (Q2) bridge OOP ergonomics and SOA performance.
- C++23 `std::mdspan` can provide multi-dimensional views over SOA data.
