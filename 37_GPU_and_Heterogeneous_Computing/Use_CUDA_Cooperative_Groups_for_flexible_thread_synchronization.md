# Use CUDA Cooperative Groups for Flexible Thread Synchronization

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** <https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#cooperative-groups>  

---

## Topic Overview

CUDA Cooperative Groups is a C++ API that generalizes thread synchronization beyond the traditional block-level `__syncthreads()`. It provides a hierarchy of named group abstractions - from sub-warp partitions to entire multi-GPU grids - each with `sync()`, `size()`, and collective operations. This enables algorithms that need synchronization granularities not possible with the legacy model: grid-level barriers for iterative algorithms, tiled partitions for warp-level reductions, and coalesced groups for divergent thread subsets.

The key group types form a hierarchy: `thread_block_tile<N>` creates compile-time-sized partitions (N must be a power of 2 <= warp size), enabling shuffle-based collectives without shared memory. `grid_group` enables all threads in a grid to synchronize - critical for iterative solvers (Jacobi, conjugate gradient) that previously required kernel re-launch for each iteration. `multi_grid_group` extends this across multiple GPUs for distributed algorithms.

The reason this API exists is that the old model forced you into one of two awkward choices: either fit your algorithm into block-sized synchronization units, or pay the overhead of re-launching a kernel every iteration just to sync across blocks. Cooperative Groups breaks that constraint cleanly.

Cooperative Groups integrates naturally with modern C++ patterns. Group objects are first-class values that can be passed to functions, stored in variables, and used with generic code. The `reduce()` and `inclusive_scan()` algorithms on `thread_block_tile` groups generate efficient warp shuffle instructions without manual `__shfl_down_sync` calls.

| Group Type | Scope | Sync Cost | Use Case |
| --- | --- | --- | --- |
| `coalesced_threads()` | Active lanes in warp | Free (warp) | Divergent code paths |
| `thread_block_tile<N>` | N threads in warp | Shuffle | Warp-level reduce/scan |
| `this_thread_block()` | Thread block | `__syncthreads` | Block-level algorithms |
| `this_grid()` | Entire grid | Grid barrier | Iterative solvers |
| `this_multi_grid()` | Multiple GPU grids | Multi-GPU sync | Distributed computation |

Here's how the groups nest inside each other, from the finest to the coarsest granularity:

```cpp
Cooperative Groups Hierarchy:
┌─────────────────────────────────────────────────────┐
│                  multi_grid_group                    │
│  ┌───────────────────────────────────────────────┐  │
│  │                grid_group                      │  │
│  │  ┌─────────────────┐  ┌─────────────────┐     │  │
│  │  │  thread_block    │  │  thread_block    │    │  │
│  │  │ ┌──────┐┌──────┐│  │ ┌──────┐┌──────┐│    │  │
│  │  │ │tile<8>││tile<8>││  │ │tile<8>││tile<8>││   │  │
│  │  │ └──────┘└──────┘│  │ └──────┘└──────┘│    │  │
│  │  └─────────────────┘  └─────────────────┘     │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: Implement a warp-level reduction using thread_block_tile and compare with manual __shfl_down_sync

The classic way to do a warp-level reduction is a manual loop of `__shfl_down_sync` calls. Cooperative Groups lets you replace that entire loop with a single `cg::reduce()` call that generates the same instructions. This example shows both side by side so you can see what you're trading:

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cooperative_groups.h>
#include <cooperative_groups/reduce.h>
#include <cstdio>

namespace cg = cooperative_groups;

// Manual warp reduction using shuffle intrinsics
__device__ float manual_warp_reduce(float val) {
    for (int offset = 16; offset > 0; offset >>= 1) {
        val += __shfl_down_sync(0xFFFFFFFF, val, offset);
    }
    return val;  // Result valid in lane 0
}

// Cooperative Groups warp reduction -- cleaner, composable
__device__ float cg_tile_reduce(cg::thread_block_tile<32> warp,
                                 float val) {
    // Single function call replaces manual shuffle loop
    return cg::reduce(warp, val, cg::plus<float>());
}

// Reduction using smaller tile (e.g., 8 threads)
__device__ float cg_small_tile_reduce(
        cg::thread_block_tile<8> tile, float val) {
    return cg::reduce(tile, val, cg::plus<float>());
}

__global__ void reduction_compare(const float* data, float* results,
                                   int n) {
    int tid = blockIdx.x * blockDim.x + threadIdx.x;
    float val = (tid < n) ? data[tid] : 0.0f;

    // Get the thread block and partition it
    auto block = cg::this_thread_block();
    auto warp = cg::tiled_partition<32>(block);
    auto tile8 = cg::tiled_partition<8>(block);

    // Method 1: Manual shuffle
    float manual_sum = manual_warp_reduce(val);

    // Method 2: CG warp reduce
    float cg_warp_sum = cg_tile_reduce(warp, val);

    // Method 3: CG tile<8> reduce -- 4 independent reductions per warp
    float cg_tile8_sum = cg_small_tile_reduce(tile8, val);

    // Lane 0 of each group stores result
    if (warp.thread_rank() == 0) {
        int warp_id = tid / 32;
        results[warp_id * 3 + 0] = manual_sum;
        results[warp_id * 3 + 1] = cg_warp_sum;
    }
    if (tile8.thread_rank() == 0) {
        // Each tile<8> group has its own lane 0
        int tile_id = tid / 8;
        results[tile_id] = cg_tile8_sum;
    }
}

int main() {
    constexpr int N = 256;
    float h_data[N];
    for (int i = 0; i < N; ++i) h_data[i] = 1.0f;

    float *d_data, *d_results;
    cudaMalloc(&d_data, N * sizeof(float));
    cudaMalloc(&d_results, N * sizeof(float));
    cudaMemcpy(d_data, h_data, N * sizeof(float),
               cudaMemcpyHostToDevice);

    reduction_compare<<<1, 256>>>(d_data, d_results, N);

    float h_results[N];
    cudaMemcpy(h_results, d_results, N * sizeof(float),
               cudaMemcpyDeviceToHost);

    printf("Warp 0 manual reduce:  %.0f (expected 32)\n", h_results[0]);
    printf("Warp 0 CG reduce:     %.0f (expected 32)\n", h_results[1]);

    cudaFree(d_data);
    cudaFree(d_results);
    return 0;
}
```

Notice that `tile<8>` gives you four independent reductions per warp, not one. Each group of 8 threads gets its own result, which is useful for algorithms that work on smaller logical units than a full warp.

### Q2: Use grid_group to implement a grid-wide barrier for an iterative Jacobi solver without kernel re-launch

This example is the most important use case for `grid_group`. Normally, an iterative solver like Jacobi requires a full host-device synchronization between iterations because no mechanism existed for all GPU blocks to sync without returning to the host. With `this_grid().sync()`, you can stay inside a single kernel for all iterations - no round-trip overhead per iteration.

The reason this trips people up is the launch requirements. You cannot use `<<<>>>` syntax for a cooperative kernel. You must use `cudaLaunchCooperativeKernel`, and you must carefully calculate the block count so that all blocks can be simultaneously resident on the GPU. If you launch more blocks than the hardware can hold at once, the grid barrier will deadlock.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cooperative_groups.h>
#include <cstdio>
#include <cmath>

namespace cg = cooperative_groups;

// Jacobi iteration with grid-level sync -- single kernel launch
// for all iterations (no host-device round-trip per iteration)
__global__ void jacobi_grid_sync(float* u_new, float* u_old,
                                  int nx, int ny, int iters) {
    auto grid = cg::this_grid();
    int idx = grid.thread_rank();
    int total = nx * ny;

    for (int iter = 0; iter < iters; ++iter) {
        // Each thread updates one cell
        if (idx < total) {
            int i = idx / ny;
            int j = idx % ny;

            if (i > 0 && i < nx - 1 && j > 0 && j < ny - 1) {
                u_new[idx] = 0.25f * (
                    u_old[(i - 1) * ny + j] +
                    u_old[(i + 1) * ny + j] +
                    u_old[i * ny + (j - 1)] +
                    u_old[i * ny + (j + 1)]
                );
            }
        }

        // Grid-wide barrier -- ALL threads across ALL blocks sync
        grid.sync();

        // Swap pointers (logically -- both are valid after sync)
        if (idx < total) {
            float tmp = u_new[idx];
            u_new[idx] = u_old[idx];
            u_old[idx] = tmp;
        }

        grid.sync();
    }
}

int main() {
    constexpr int NX = 512, NY = 512;
    constexpr int ITERS = 100;
    size_t bytes = NX * NY * sizeof(float);

    float *d_u, *d_u_new;
    cudaMalloc(&d_u, bytes);
    cudaMalloc(&d_u_new, bytes);
    cudaMemset(d_u, 0, bytes);
    cudaMemset(d_u_new, 0, bytes);

    // Set boundary conditions on host
    std::vector<float> h_u(NX * NY, 0.0f);
    for (int j = 0; j < NY; ++j) h_u[j] = 100.0f;  // Top row hot
    cudaMemcpy(d_u, h_u.data(), bytes, cudaMemcpyHostToDevice);
    cudaMemcpy(d_u_new, h_u.data(), bytes, cudaMemcpyHostToDevice);

    // Cooperative launch requires querying max active blocks
    int threads = 256;
    int dev = 0;
    int max_blocks;
    cudaOccupancyMaxActiveBlocksPerMultiprocessor(
        &max_blocks, jacobi_grid_sync, threads, 0);

    int sm_count;
    cudaDeviceGetAttribute(&sm_count,
                            cudaDevAttrMultiProcessorCount, dev);
    int blocks = std::min(max_blocks * sm_count,
                          (NX * NY + threads - 1) / threads);

    // Must use cooperative launch API
    void* args[] = {&d_u_new, &d_u, (void*)&NX, (void*)&NY,
                    (void*)&ITERS};
    cudaLaunchCooperativeKernel(
        (void*)jacobi_grid_sync, blocks, threads, args);
    cudaDeviceSynchronize();

    cudaMemcpy(h_u.data(), d_u, bytes, cudaMemcpyDeviceToHost);
    printf("Center temp after %d iters: %.4f\n",
           ITERS, h_u[NX / 2 * NY + NY / 2]);

    cudaFree(d_u);
    cudaFree(d_u_new);
    return 0;
}
```

The `cudaOccupancyMaxActiveBlocksPerMultiprocessor` query is not optional boilerplate - it's the mechanism that guarantees your block count fits within the simultaneously-resident capacity of the GPU. Get this wrong, and `grid.sync()` will hang forever.

### Q3: Use coalesced_threads() and tiled_partition to handle divergent code paths efficiently

Divergent code - where threads in a warp take different branches - is a classic GPU performance problem because the warp has to execute both paths serially. Cooperative Groups adds `coalesced_threads()`, which gives you a group object representing only the threads that are *currently active* in the warp. This lets you perform collectives (like reductions) among just the active threads, avoiding wasted work and incorrect results from inactive lanes.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cooperative_groups.h>
#include <cooperative_groups/reduce.h>
#include <cstdio>

namespace cg = cooperative_groups;

// Example: partition active threads in divergent branches
// and perform reductions only among active threads
__global__ void divergent_reduce(const int* data, int* even_sum,
                                  int* odd_sum, int n) {
    int tid = blockIdx.x * blockDim.x + threadIdx.x;
    if (tid >= n) return;

    int val = data[tid];

    if (val % 2 == 0) {
        // Only threads with even values are active here
        auto active = cg::coalesced_threads();

        // Reduce only among the active (even) threads
        int sum = cg::reduce(active, val, cg::plus<int>());

        // Thread rank 0 in the coalesced group writes
        if (active.thread_rank() == 0) {
            atomicAdd(even_sum, sum);
        }
    } else {
        auto active = cg::coalesced_threads();
        int sum = cg::reduce(active, val, cg::plus<int>());
        if (active.thread_rank() == 0) {
            atomicAdd(odd_sum, sum);
        }
    }
}

// Dynamic partitioning based on labels
__global__ void labeled_partition(const int* labels,
                                   const float* values,
                                   float* group_sums,
                                   int n) {
    auto block = cg::this_thread_block();
    int tid = block.group_index().x * block.group_dim().x
              + block.thread_rank();

    if (tid >= n) return;

    int label = labels[tid];
    float val = values[tid];

    // Partition warp by label value -- threads with same label
    // form a group
    auto warp = cg::tiled_partition<32>(block);
    auto labeled = cg::labeled_partition(warp, label);

    // Reduce within each label group
    float label_sum = cg::reduce(labeled, val, cg::plus<float>());

    // First thread in each label subgroup writes
    if (labeled.thread_rank() == 0) {
        atomicAdd(&group_sums[label], label_sum);
    }
}

int main() {
    constexpr int N = 1024;

    // Test divergent reduce
    int h_data[N];
    for (int i = 0; i < N; ++i) h_data[i] = i;

    int *d_data, *d_even, *d_odd;
    cudaMalloc(&d_data, N * sizeof(int));
    cudaMalloc(&d_even, sizeof(int));
    cudaMalloc(&d_odd, sizeof(int));
    cudaMemcpy(d_data, h_data, N * sizeof(int), cudaMemcpyHostToDevice);
    cudaMemset(d_even, 0, sizeof(int));
    cudaMemset(d_odd, 0, sizeof(int));

    divergent_reduce<<<(N+255)/256, 256>>>(d_data, d_even, d_odd, N);

    int h_even, h_odd;
    cudaMemcpy(&h_even, d_even, sizeof(int), cudaMemcpyDeviceToHost);
    cudaMemcpy(&h_odd, d_odd, sizeof(int), cudaMemcpyDeviceToHost);
    printf("Even sum: %d, Odd sum: %d, Total: %d\n",
           h_even, h_odd, h_even + h_odd);
    printf("Expected total: %d\n", N * (N - 1) / 2);

    // Test labeled partition
    constexpr int NUM_LABELS = 4;
    int h_labels[N];
    float h_values[N];
    for (int i = 0; i < N; ++i) {
        h_labels[i] = i % NUM_LABELS;
        h_values[i] = 1.0f;
    }

    int* d_labels;
    float *d_values, *d_gsums;
    cudaMalloc(&d_labels, N * sizeof(int));
    cudaMalloc(&d_values, N * sizeof(float));
    cudaMalloc(&d_gsums, NUM_LABELS * sizeof(float));
    cudaMemcpy(d_labels, h_labels, N * sizeof(int),
               cudaMemcpyHostToDevice);
    cudaMemcpy(d_values, h_values, N * sizeof(float),
               cudaMemcpyHostToDevice);
    cudaMemset(d_gsums, 0, NUM_LABELS * sizeof(float));

    labeled_partition<<<(N+255)/256, 256>>>(
        d_labels, d_values, d_gsums, N);

    float h_gsums[NUM_LABELS];
    cudaMemcpy(h_gsums, d_gsums, NUM_LABELS * sizeof(float),
               cudaMemcpyDeviceToHost);
    for (int i = 0; i < NUM_LABELS; ++i)
        printf("Label %d sum: %.0f (expected %d)\n",
               i, h_gsums[i], N / NUM_LABELS);

    cudaFree(d_data); cudaFree(d_even); cudaFree(d_odd);
    cudaFree(d_labels); cudaFree(d_values); cudaFree(d_gsums);
    return 0;
}
```

The `labeled_partition` at the end is especially powerful for algorithms like segmented reduction and graph processing, where threads naturally fall into data-dependent groups and you want to reduce within each group without knowing the groupings at compile time.

---

## Notes

- `cooperative_groups.h` and `cooperative_groups/reduce.h` must be included explicitly - they are not part of `cuda_runtime.h`.
- Grid-level sync (`this_grid().sync()`) requires `cudaLaunchCooperativeKernel` - regular `<<<>>>` launch does not support it, and the failure mode is a silent deadlock.
- Grid sync also requires that the total blocks fit in resident capacity - query `cudaOccupancyMaxActiveBlocksPerMultiprocessor` and cap your block count accordingly.
- `thread_block_tile<N>` with N <= 32 generates shuffle instructions; N > 32 falls back to shared memory + `__syncthreads`, which is slower.
- `labeled_partition` enables data-dependent grouping at warp level - powerful for histogram, segmented reduction, and graph algorithms.
- Compile with `nvcc -rdc=true -arch=sm_70` (or later) for cooperative launch support.
