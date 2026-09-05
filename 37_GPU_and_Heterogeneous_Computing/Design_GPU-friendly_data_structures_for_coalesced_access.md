# Design GPU-Friendly Data Structures for Coalesced Access

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#coalesced-access-to-global-memory  

---

## Topic Overview

GPU performance depends critically on memory access patterns. A warp of 32 threads issues memory transactions collectively - when consecutive threads access consecutive memory addresses (coalesced access), the hardware can merge requests into a single wide transaction (128 bytes on modern NVIDIA GPUs). Non-coalesced access wastes bandwidth, requiring multiple transactions to serve the same warp, degrading throughput by up to 32x.

The most impactful design decision you'll make is choosing between Array-of-Structures (AoS) and Structure-of-Arrays (SoA) layouts. AoS is natural in CPU-centric C++ (a `std::vector<Particle>` where each Particle has `x, y, z, mass`), but on a GPU, when a kernel accesses only the `x` field, AoS forces loading all fields of adjacent particles, wasting 75% of bandwidth. SoA stores each field in a separate contiguous array, so accessing `x` across threads is perfectly coalesced. The reason this trips people up is that AoS feels more natural to write, but it fights the GPU's memory system at every turn.

Shared memory (scratchpad) serves as a programmer-managed L1 cache. Tiling algorithms load a tile of global memory into shared memory cooperatively, then perform computations on the fast shared memory. This is essential for non-trivial access patterns like matrix transpose, stencil computation, and reduction.

The diagram below shows concretely why AoS hurts. Each `[...]` block is one particle's worth of data. When thread 0 wants `x₀` and thread 1 wants `x₁`, those values are four floats apart in AoS - requiring four separate memory transactions where one would suffice in SoA:

```cpp
AoS vs SoA Memory Layout:

AoS:  [x0 y0 z0 m0] [x1 y1 z1 m1] [x2 y2 z2 m2] ...
       Thread 0 reads ------+  Thread 1 reads ------+
       -> Stride-4 access = 4 transactions per warp

SoA:  [x0 x1 x2 x3 ... x31] [y0 y1 ... y31] [z0 z1 ...] [m0 m1 ...]
       Thread 0--+  Thread 1--+
       -> Consecutive access = 1 transaction per warp (coalesced)
```

Here is a summary of how the major layout strategies compare. If the table feels like a lot, the short version is: SoA wins on the GPU, AoS wins for developer ergonomics, and AoSoA is the compromise that gets you close to SoA performance without throwing away locality entirely.

| Layout | Coalescing  | Cache efficiency | CPU ergonomics | GPU performance |
| --- | --- | --- | --- | --- |
| AoS    | Poor        | Loads unused fields | Natural (OOP)   | 3-10x slower   |
| SoA    | Perfect     | Only needed fields  | Verbose         | Optimal         |
| AoSoA  | Good        | Warp-width tiles    | Moderate        | Near-optimal    |
| Hybrid | Depends     | Grouped by access   | Moderate        | Good if tuned   |

---

## Self-Assessment

### Q1: Implement a particle simulation kernel using both AoS and SoA layouts and measure the performance difference

This benchmark makes the performance difference tangible. Run both kernels on 4M particles for 100 iterations and watch the numbers. The CUDA event timing gives you GPU-only time, which is exactly what you want here.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <cstdlib>

#define CUDA_CHECK(x) do{cudaError_t e=(x);if(e!=cudaSuccess){\
    printf("err %s:%d\n",__FILE__,__LINE__);exit(1);}}while(0)

// ===== AoS Layout =====
struct ParticleAoS {
    float x, y, z;
    float vx, vy, vz;
    float mass;
    float padding;  // Pad to 32 bytes for alignment
};

__global__ void update_aos(ParticleAoS* p, int n, float dt) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i >= n) return;
    // Each thread reads 32 bytes but only needs x,y,z,vx,vy,vz
    // Stride between consecutive threads = 32 bytes (uncoalesced)
    p[i].x += p[i].vx * dt;
    p[i].y += p[i].vy * dt;
    p[i].z += p[i].vz * dt;
}

// ===== SoA Layout =====
struct ParticleSoA {
    float* x;  float* y;  float* z;
    float* vx; float* vy; float* vz;
    float* mass;
    int n;
};

__global__ void update_soa(float* __restrict__ x,
                            float* __restrict__ y,
                            float* __restrict__ z,
                            const float* __restrict__ vx,
                            const float* __restrict__ vy,
                            const float* __restrict__ vz,
                            int n, float dt) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i >= n) return;
    // Consecutive threads read consecutive floats — coalesced
    x[i] += vx[i] * dt;
    y[i] += vy[i] * dt;
    z[i] += vz[i] * dt;
}

int main() {
    constexpr int N = 1 << 22;  // 4M particles
    constexpr float dt = 0.01f;
    constexpr int ITERS = 100;

    // === AoS ===
    ParticleAoS* d_aos;
    CUDA_CHECK(cudaMalloc(&d_aos, N * sizeof(ParticleAoS)));
    CUDA_CHECK(cudaMemset(d_aos, 0, N * sizeof(ParticleAoS)));

    cudaEvent_t start, stop;
    CUDA_CHECK(cudaEventCreate(&start));
    CUDA_CHECK(cudaEventCreate(&stop));

    int threads = 256;
    int blocks = (N + threads - 1) / threads;

    CUDA_CHECK(cudaEventRecord(start));
    for (int i = 0; i < ITERS; ++i)
        update_aos<<<blocks, threads>>>(d_aos, N, dt);
    CUDA_CHECK(cudaEventRecord(stop));
    CUDA_CHECK(cudaDeviceSynchronize());

    float aos_ms;
    CUDA_CHECK(cudaEventElapsedTime(&aos_ms, start, stop));

    // === SoA ===
    float *d_x, *d_y, *d_z, *d_vx, *d_vy, *d_vz;
    size_t bytes = N * sizeof(float);
    CUDA_CHECK(cudaMalloc(&d_x,  bytes));
    CUDA_CHECK(cudaMalloc(&d_y,  bytes));
    CUDA_CHECK(cudaMalloc(&d_z,  bytes));
    CUDA_CHECK(cudaMalloc(&d_vx, bytes));
    CUDA_CHECK(cudaMalloc(&d_vy, bytes));
    CUDA_CHECK(cudaMalloc(&d_vz, bytes));
    CUDA_CHECK(cudaMemset(d_x, 0, bytes));
    CUDA_CHECK(cudaMemset(d_y, 0, bytes));
    CUDA_CHECK(cudaMemset(d_z, 0, bytes));
    CUDA_CHECK(cudaMemset(d_vx, 0, bytes));
    CUDA_CHECK(cudaMemset(d_vy, 0, bytes));
    CUDA_CHECK(cudaMemset(d_vz, 0, bytes));

    CUDA_CHECK(cudaEventRecord(start));
    for (int i = 0; i < ITERS; ++i)
        update_soa<<<blocks, threads>>>(d_x, d_y, d_z,
                                         d_vx, d_vy, d_vz, N, dt);
    CUDA_CHECK(cudaEventRecord(stop));
    CUDA_CHECK(cudaDeviceSynchronize());

    float soa_ms;
    CUDA_CHECK(cudaEventElapsedTime(&soa_ms, start, stop));

    printf("AoS: %.2f ms (%d iters)\n", aos_ms, ITERS);
    printf("SoA: %.2f ms (%d iters)\n", soa_ms, ITERS);
    printf("SoA speedup: %.2fx\n", aos_ms / soa_ms);

    cudaFree(d_aos);
    cudaFree(d_x); cudaFree(d_y); cudaFree(d_z);
    cudaFree(d_vx); cudaFree(d_vy); cudaFree(d_vz);
    cudaEventDestroy(start); cudaEventDestroy(stop);
    return 0;
}
```

The SoA speedup is typically in the 3-10x range depending on GPU generation. The `__restrict__` qualifiers on the SoA kernel parameters are also doing real work here - they tell the compiler that no two pointers alias each other, enabling additional load/store optimizations.

### Q2: Implement a shared memory tiled matrix transpose that eliminates bank conflicts

Matrix transpose is the canonical example of a kernel that needs shared memory. The naive approach writes to global memory in a strided, non-coalesced pattern. The tiled version loads a tile into shared memory with coalesced reads, then writes it out in transposed order with coalesced writes - fast in both directions. The tricky part is the bank conflict, which the `+1` padding trick solves.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>

#define TILE 32
#define CUDA_CHECK(x) do{cudaError_t e=(x);if(e!=cudaSuccess){\
    printf("err %s:%d\n",__FILE__,__LINE__);exit(1);}}while(0)

// Naive transpose — uncoalesced writes
__global__ void transpose_naive(float* out, const float* in,
                                 int rows, int cols) {
    int r = blockIdx.y * blockDim.y + threadIdx.y;
    int c = blockIdx.x * blockDim.x + threadIdx.x;
    if (r < rows && c < cols)
        out[c * rows + r] = in[r * cols + c];  // Uncoalesced write
}

// Tiled transpose with shared memory — NO bank conflicts
__global__ void transpose_tiled(float* out, const float* in,
                                 int rows, int cols) {
    // +1 padding eliminates bank conflicts on shared memory
    __shared__ float tile[TILE][TILE + 1];

    int bx = blockIdx.x * TILE;
    int by = blockIdx.y * TILE;
    int x = bx + threadIdx.x;
    int y = by + threadIdx.y;

    // Coalesced read from global -> shared
    if (y < rows && x < cols)
        tile[threadIdx.y][threadIdx.x] = in[y * cols + x];
    __syncthreads();

    // Transposed indices for coalesced write
    int tx = by + threadIdx.x;  // Swapped block offsets
    int ty = bx + threadIdx.y;

    // Coalesced write from shared -> global
    if (ty < cols && tx < rows)
        out[ty * rows + tx] = tile[threadIdx.x][threadIdx.y];
}

int main() {
    constexpr int R = 4096, C = 4096;
    size_t bytes = R * C * sizeof(float);

    float *d_in, *d_out;
    CUDA_CHECK(cudaMalloc(&d_in, bytes));
    CUDA_CHECK(cudaMalloc(&d_out, bytes));

    // Initialize with sequential values
    std::vector<float> h(R * C);
    for (int i = 0; i < R * C; ++i) h[i] = static_cast<float>(i);
    CUDA_CHECK(cudaMemcpy(d_in, h.data(), bytes,
                          cudaMemcpyHostToDevice));

    dim3 threads(TILE, TILE);
    dim3 blocks((C + TILE - 1) / TILE, (R + TILE - 1) / TILE);

    cudaEvent_t start, stop;
    CUDA_CHECK(cudaEventCreate(&start));
    CUDA_CHECK(cudaEventCreate(&stop));

    // Benchmark naive
    CUDA_CHECK(cudaEventRecord(start));
    for (int i = 0; i < 100; ++i)
        transpose_naive<<<blocks, threads>>>(d_out, d_in, R, C);
    CUDA_CHECK(cudaEventRecord(stop));
    CUDA_CHECK(cudaDeviceSynchronize());
    float naive_ms;
    CUDA_CHECK(cudaEventElapsedTime(&naive_ms, start, stop));

    // Benchmark tiled
    CUDA_CHECK(cudaEventRecord(start));
    for (int i = 0; i < 100; ++i)
        transpose_tiled<<<blocks, threads>>>(d_out, d_in, R, C);
    CUDA_CHECK(cudaEventRecord(stop));
    CUDA_CHECK(cudaDeviceSynchronize());
    float tiled_ms;
    CUDA_CHECK(cudaEventElapsedTime(&tiled_ms, start, stop));

    printf("Naive transpose:  %.2f ms\n", naive_ms);
    printf("Tiled transpose:  %.2f ms\n", tiled_ms);
    printf("Speedup: %.2fx\n", naive_ms / tiled_ms);

    /*
     Bank conflict avoidance with +1 padding:
     Shared memory has 32 banks, 4 bytes each.
     Without padding: tile[TILE][TILE]
       Column access: threads 0..31 hit banks 0,1,2...31 — no conflict
       Row access:    threads 0..31 hit same bank — 32-way conflict!
     With padding:    tile[TILE][TILE+1]
       Row access:    stride=33, so bank = (tid*33)%32 = tid — no conflict!
    */

    cudaFree(d_in); cudaFree(d_out);
    cudaEventDestroy(start); cudaEventDestroy(stop);
    return 0;
}
```

The bank conflict explanation in the comment is the key piece of intuition here. Shared memory is divided into 32 banks, and threads in the same warp that access the same bank must serialize. Without the `+1` padding, reading a column of the tile causes all 32 threads to hit the same bank - a 32-way serialization that completely defeats the purpose of using shared memory.

### Q3: Implement the AoSoA (Array-of-Structures-of-Arrays) hybrid layout optimized for warp-width access

AoSoA is the best-of-both-worlds layout when you need multiple fields per particle in the same kernel. It groups particles into tiles of exactly warp size (32), and within each tile uses SoA ordering. That gives you coalesced access across a warp's threads (they hit consecutive addresses within the tile) while keeping all fields of the same warp's particles physically close together.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <cstdlib>

#define CUDA_CHECK(x) do{cudaError_t e=(x);if(e!=cudaSuccess){\
    printf("err %s:%d\n",__FILE__,__LINE__);exit(1);}}while(0)

// AoSoA: groups of WARP_SIZE elements stored as mini-SoA tiles
constexpr int WARP_SIZE = 32;

// Each tile holds WARP_SIZE particles in SoA within the tile
struct ParticleTile {
    float x[WARP_SIZE];
    float y[WARP_SIZE];
    float z[WARP_SIZE];
    float vx[WARP_SIZE];
    float vy[WARP_SIZE];
    float vz[WARP_SIZE];
};
// sizeof(ParticleTile) = 6 * 32 * 4 = 768 bytes
// All fields of one warp's worth of particles are close in memory

__global__ void update_aosoa(ParticleTile* tiles, int n_tiles,
                              float dt) {
    int tile_idx = blockIdx.x * blockDim.x / WARP_SIZE
                   + threadIdx.x / WARP_SIZE;

    int lane = threadIdx.x % WARP_SIZE;  // Lane within warp

    if (tile_idx >= n_tiles) return;

    ParticleTile& t = tiles[tile_idx];

    // Each lane accesses its element within the tile — coalesced
    // because t.x[0..31] are contiguous
    t.x[lane] += t.vx[lane] * dt;
    t.y[lane] += t.vy[lane] * dt;
    t.z[lane] += t.vz[lane] * dt;
}

int main() {
    constexpr int N = 1 << 22;
    constexpr int N_TILES = N / WARP_SIZE;
    constexpr float dt = 0.01f;

    ParticleTile* d_tiles;
    CUDA_CHECK(cudaMalloc(&d_tiles, N_TILES * sizeof(ParticleTile)));
    CUDA_CHECK(cudaMemset(d_tiles, 0,
                          N_TILES * sizeof(ParticleTile)));

    cudaEvent_t start, stop;
    CUDA_CHECK(cudaEventCreate(&start));
    CUDA_CHECK(cudaEventCreate(&stop));

    int threads = 256;  // 8 warps per block
    int blocks = (N + threads - 1) / threads;

    CUDA_CHECK(cudaEventRecord(start));
    for (int i = 0; i < 100; ++i)
        update_aosoa<<<blocks, threads>>>(d_tiles, N_TILES, dt);
    CUDA_CHECK(cudaEventRecord(stop));
    CUDA_CHECK(cudaDeviceSynchronize());

    float ms;
    CUDA_CHECK(cudaEventElapsedTime(&ms, start, stop));
    printf("AoSoA update: %.2f ms (100 iters, %d particles)\n", ms, N);

    /*
     AoSoA advantages:
     Layout          | Coalescing | Locality
     AoS             | Poor       | All fields
     SoA             | Perfect    | One field only
     AoSoA (warp=32) | Perfect    | All fields

     AoSoA gives coalescing (within tile, stride=1)
     AND locality (all fields of a warp in ~768 bytes,
     fits in a few cache lines).
    */

    cudaFree(d_tiles);
    cudaEventDestroy(start);
    cudaEventDestroy(stop);
    return 0;
}
```

The table in the comment sums up the tradeoff nicely. Full SoA gives perfect coalescing but poor locality when you need multiple fields - you load from six separate regions of memory. AoSoA keeps all six fields for one warp's worth of particles in 768 bytes, so they likely all fit in L1 cache together, while still giving you stride-1 access within each field array.

---

## Notes

- `__restrict__` on kernel pointer parameters enables the compiler to assume no aliasing, unlocking load/store reordering and vectorization.
- Shared memory bank conflict padding (`[TILE][TILE+1]`) is the canonical trick - never use `[TILE][TILE]` for 2D shared memory with column access.
- AoSoA layout is ideal when kernels access multiple fields per particle - it combines SoA coalescing with AoS locality.
- Use `nvprof` / `ncu` metrics `l1tex__t_sectors_pipe_lsu_mem_global_op_ld.sum` and `_st.sum` to quantify coalescing efficiency.
- Align allocations to 128 bytes (`cudaMalloc` does this automatically) to ensure transaction alignment.
- For sparse access patterns, consider texture memory or `__ldg()` intrinsic for read-only data that benefits from the texture cache.
