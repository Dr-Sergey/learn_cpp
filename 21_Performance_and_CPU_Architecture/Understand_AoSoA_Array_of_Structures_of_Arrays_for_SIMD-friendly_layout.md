# Understand AoSoA (Array of Structures of Arrays) for SIMD-friendly layout

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://en.cppreference.com/w/cpp/language/performance>  

---

## Topic Overview

AoSoA is a data layout that tries to get the best of two worlds that are normally in tension: the SIMD vectorizability of SoA (Struct of Arrays) and the cache locality of AoS (Array of Structs). Understanding why each layout has its own trade-off is the key to seeing why AoSoA exists.

With **AoS**, all fields of one element live together. That is great when you need all the data for a single particle at once, but it is terrible for SIMD - the eight `x` coordinates you want to load in one AVX register are 24 bytes apart, scattered across eight different structs.

With **SoA**, each field lives in its own contiguous array. The `x` coordinates are all adjacent, so a single `_mm256_load_ps` grabs eight at once. The problem is that accessing all fields of a single particle now touches six widely separated memory regions.

**AoSoA** breaks the data into fixed-width tiles whose width matches the SIMD register. Within a tile, each field is contiguous - so you get the same vectorizable layout as SoA - but all fields for those eight particles live within a single 192-byte block, which is good for cache line reuse when you process a full tile at once.

```cpp
AoS (Array of Structs):        SoA (Struct of Arrays):
[x y z] [x y z] [x y z] ...   x: [x x x x x x x x ...]
                                y: [y y y y y y y y ...]
                                z: [z z z z z z z z ...]

AoSoA (tile size = 8):
Tile 0:  x[8] y[8] z[8]   <- 8 x's contiguous, then 8 y's, then 8 z's
Tile 1:  x[8] y[8] z[8]
Tile 2:  x[8] y[8] z[8]
```

If the table below feels like a lot, the one-sentence summary is: use AoSoA when you process all fields of a group of elements in lockstep and your inner loop width matches the SIMD register size.

| Layout | SIMD efficiency | Single-element access | Cache locality |
| --- | --- | --- | --- |
| AoS | Poor (scattered) | Excellent (all fields contiguous) | Good per-element |
| SoA | Excellent (contiguous per field) | Poor (fields far apart) | Good per-field |
| **AoSoA** | **Excellent** | **Good (within tile)** | **Best overall** |

---

## Self-Assessment

### Q1: AoSoA as a compromise between AoS and SoA

Let's look at all three layouts side by side in code to make the trade-offs concrete. Pay attention to the comments on SIMD access patterns - they explain exactly why AoS fails for vectorization.

```cpp
#include <iostream>
#include <cstddef>
#include <array>

// AoS: all fields of one particle together
struct ParticleAoS {
    float x, y, z;
    float vx, vy, vz;
};
// Access pattern: p[i].x, p[i].y, p[i].z -- good per-element
// SIMD problem: x values are 24 bytes apart -- can't load 8 x's in one AVX load

// SoA: each field in its own array
struct ParticlesSoA {
    float* x;  float* y;  float* z;
    float* vx; float* vy; float* vz;
    size_t count;
};
// SIMD: x's are contiguous -- load 8 at once with _mm256_load_ps
// Problem: accessing all fields of particle i touches 6 cache lines

// AoSoA: tiles of SIMD-width elements
constexpr int TILE = 8;  // AVX has 8 floats per register

struct ParticleTile {
    float x[TILE], y[TILE], z[TILE];
    float vx[TILE], vy[TILE], vz[TILE];
};
// SIMD: tile.x is 8 contiguous floats -- perfect for _mm256_load_ps
// Locality: all 6 fields for 8 particles in one 192-byte block

int main() {
    constexpr int N = 1024;
    constexpr int num_tiles = N / TILE;

    ParticleTile tiles[num_tiles];

    // Initialize:
    for (int t = 0; t < num_tiles; ++t) {
        for (int i = 0; i < TILE; ++i) {
            int idx = t * TILE + i;
            tiles[t].x[i] = static_cast<float>(idx);
            tiles[t].y[i] = 0.0f;
            tiles[t].z[i] = 0.0f;
            tiles[t].vx[i] = 1.0f;
            tiles[t].vy[i] = 0.0f;
            tiles[t].vz[i] = 0.0f;
        }
    }

    std::cout << "Tile size: " << sizeof(ParticleTile) << " bytes\n"; // 192
    std::cout << "Tiles: " << num_tiles << '\n';                       // 128
    std::cout << "Total: " << sizeof(ParticleTile) * num_tiles << " bytes\n";
}
```

The 192-byte tile fits in three cache lines (64 bytes each). When you process a tile with SIMD instructions you load those three lines once and work entirely from cache for the duration of the tile.

### Q2: AoSoA particle system with SIMD-width tiles

Here the AoSoA tile size is set to `TILE_SIZE = 8` to match an AVX register, which holds exactly eight 32-bit floats. The `alignas(32)` ensures the tile base address is 32-byte aligned, which is required for the aligned load intrinsic `_mm256_load_ps`.

```cpp
#include <iostream>
#include <vector>
#include <immintrin.h>  // AVX intrinsics

constexpr int TILE_SIZE = 8;  // AVX: 256 bits / 32 bits = 8 floats

struct alignas(32) ParticleTile {
    float x[TILE_SIZE], y[TILE_SIZE], z[TILE_SIZE];
    float vx[TILE_SIZE], vy[TILE_SIZE], vz[TILE_SIZE];
};

// Update positions: pos += vel * dt
void update_particles(ParticleTile* tiles, int num_tiles, float dt) {
    __m256 dt_vec = _mm256_set1_ps(dt);

    for (int t = 0; t < num_tiles; ++t) {
        // Load 8 x positions at once:
        __m256 x = _mm256_load_ps(tiles[t].x);
        __m256 y = _mm256_load_ps(tiles[t].y);
        __m256 z = _mm256_load_ps(tiles[t].z);

        // Load 8 velocities:
        __m256 vx = _mm256_load_ps(tiles[t].vx);
        __m256 vy = _mm256_load_ps(tiles[t].vy);
        __m256 vz = _mm256_load_ps(tiles[t].vz);

        // pos += vel * dt (8 particles in parallel):
        x = _mm256_fmadd_ps(vx, dt_vec, x);
        y = _mm256_fmadd_ps(vy, dt_vec, y);
        z = _mm256_fmadd_ps(vz, dt_vec, z);

        // Store back:
        _mm256_store_ps(tiles[t].x, x);
        _mm256_store_ps(tiles[t].y, y);
        _mm256_store_ps(tiles[t].z, z);
    }
}

int main() {
    constexpr int N = 1024;
    constexpr int num_tiles = N / TILE_SIZE;

    std::vector<ParticleTile> tiles(num_tiles);
    // ... initialize ...

    float dt = 0.016f; // 60 FPS
    update_particles(tiles.data(), num_tiles, dt);

    std::cout << "Updated " << N << " particles in "
              << num_tiles << " tiles\n";
}
```

Notice that each iteration of the outer loop does six aligned loads, three fused multiply-adds, and three stores - operating on eight particles at once. Without the AoSoA layout you would need gather intrinsics (which are much slower) or you would fall back to scalar code.

### Q3: Benchmark AoS vs SoA vs AoSoA

This benchmark puts all three layouts through the same particle position update and measures the difference. The compiler can auto-vectorize the SoA and AoSoA loops because the data is contiguous; it struggles with AoS because the fields interleave.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <cmath>

constexpr int N = 1'000'000;
constexpr int TILE = 8;
constexpr int ITERS = 100;

// AoS:
struct PosAoS { float x, y, z, vx, vy, vz; };

void update_aos(std::vector<PosAoS>& p, float dt) {
    for (auto& e : p) {
        e.x += e.vx * dt;
        e.y += e.vy * dt;
        e.z += e.vz * dt;
    }
}

// SoA:
struct PosSoA {
    std::vector<float> x, y, z, vx, vy, vz;
};

void update_soa(PosSoA& p, float dt) {
    for (int i = 0; i < N; ++i) {
        p.x[i] += p.vx[i] * dt;
        p.y[i] += p.vy[i] * dt;
        p.z[i] += p.vz[i] * dt;
    }
}

// AoSoA:
struct Tile { float x[TILE], y[TILE], z[TILE], vx[TILE], vy[TILE], vz[TILE]; };

void update_aosoa(std::vector<Tile>& tiles, float dt) {
    for (auto& t : tiles) {
        for (int i = 0; i < TILE; ++i) {
            t.x[i] += t.vx[i] * dt;
            t.y[i] += t.vy[i] * dt;
            t.z[i] += t.vz[i] * dt;
        }
    }
}

int main() {
    float dt = 0.016f;

    // Setup
    std::vector<PosAoS> aos(N, {0,0,0,1,1,1});
    PosSoA soa; soa.x.resize(N,0); soa.y.resize(N,0); soa.z.resize(N,0);
    soa.vx.resize(N,1); soa.vy.resize(N,1); soa.vz.resize(N,1);
    std::vector<Tile> aosoa(N/TILE);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < ITERS; ++i) fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1-t0).count();
        std::cout << label << ": " << us/ITERS << " us/iter\n";
    };

    bench([&]{ update_aos(aos, dt); }, "AoS   ");
    bench([&]{ update_soa(soa, dt); }, "SoA   ");
    bench([&]{ update_aosoa(aosoa, dt); }, "AoSoA ");

    // Typical results (AVX2, 1M particles):
    // AoS:   ~800 us   (scattered, poor auto-vectorization)
    // SoA:   ~300 us   (excellent vectorization, 6 cache streams)
    // AoSoA: ~250 us   (excellent vectorization + fewer cache streams)
}
```

AoSoA edges out SoA here because SoA keeps six separate arrays, and iterating all six simultaneously means six independent cache streams. AoSoA colocates all six fields for a tile of eight particles, so a single set of cache line fills covers the whole tile.

---

## Notes

- Set tile size equal to the SIMD register width (AVX = 8 floats, SSE = 4, AVX-512 = 16).
- AoSoA combines the vectorization of SoA with the spatial locality of AoS.
- The inner loop over a tile processes exactly one SIMD register width per instruction.
- `alignas(32)` or `alignas(64)` ensures tiles are aligned for `_mm256_load_ps`.
- AoSoA is the standard layout in game engines, physics simulators, and HPC.
