# Use Unified Shared Memory for Simplified GPU Programming

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17, C++20  
**Reference:** <https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#unified-memory-programming>  

---

## Topic Overview

Unified Shared Memory (USM) unifies the host and device address spaces so that a single pointer is valid on both CPU and GPU. This eliminates explicit `memcpy` calls for many workloads and dramatically simplifies code - especially for complex, pointer-rich data structures like linked lists, trees, and graphs that are difficult or impossible to marshal across address spaces manually.

CUDA calls this "Managed Memory" (`cudaMallocManaged`), while SYCL 2020 defines three USM allocation types: `malloc_device` (device-only), `malloc_host` (host-allocated, device-accessible via PCIe), and `malloc_shared` (automatically migrated). The key performance insight is that shared/managed memory relies on page-fault-driven migration, which introduces latency on first access. Prefetching (`cudaMemPrefetchAsync` / `sycl::queue::prefetch`) moves pages proactively, and memory advise hints (`cudaMemAdvise`) inform the runtime about access patterns (read-mostly, preferred location) to optimize placement.

Migration happens at page granularity (typically 4 KB on CPU, 64 KB on GPU). This is the detail that trips most people up: even if you access a single `float`, the driver migrates the entire 64 KB page that contains it. Thrashing occurs when both CPU and GPU frequently access the same pages within a short time window - leading to continuous page migration and severe performance degradation. The solution is to structure access patterns so that CPU and GPU touch disjoint page ranges, or to synchronize access phases clearly so that only one side is active at a time.

| USM Type | CUDA API | SYCL API | Host R/W | Device R/W | Migration |
| --- | --- | --- | --- | --- | --- |
| Device memory | `cudaMalloc` | `sycl::malloc_device` | No | Yes | None |
| Host memory | `cudaMallocHost` | `sycl::malloc_host` | Yes | Via PCIe | None |
| Shared/Managed | `cudaMallocManaged` | `sycl::malloc_shared` | Yes | Yes | On-demand / hint |

Here's the core mechanic of page migration and how prefetching changes it:

```cpp
Page Migration Flow (cudaMallocManaged):

 CPU accesses page -> Page fault on GPU  -> Driver migrates page to CPU
 GPU accesses page -> Page fault on CPU  -> Driver migrates page to GPU

 With prefetch:
 cudaMemPrefetchAsync(ptr, size, device) -> Bulk migration, no faults
                                          -> ~10x lower first-access latency
```

---

## Self-Assessment

### Q1: Demonstrate CUDA managed memory with RAII, and show the impact of prefetch and memory advise

This example wraps managed memory in a `unique_ptr` with a custom deleter so that the allocation is automatically freed when it goes out of scope. It then runs the same SAXPY kernel three times with different memory strategies so you can compare the wall-clock impact of each approach.

The first run shows the cold cost of page-fault-driven migration. The second run shows what prefetching buys you. The third run introduces `cudaMemAdviseSetReadMostly`, which tells the driver that `x` is read-only from the GPU's perspective - on systems with NVLink, this can cause the driver to replicate the pages on both CPU and GPU, eliminating migration for reads entirely.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <memory>
#include <chrono>

#define CUDA_CHECK(call) do { \
    cudaError_t e = (call); \
    if (e != cudaSuccess) { \
        fprintf(stderr, "CUDA error: %s at %s:%d\n", \
                cudaGetErrorString(e), __FILE__, __LINE__); \
        exit(1); } \
    } while(0)

// RAII wrapper for managed memory
template <typename T>
struct ManagedDeleter {
    void operator()(T* p) const { cudaFree(p); }
};

template <typename T>
using ManagedPtr = std::unique_ptr<T[], ManagedDeleter<T>>;

template <typename T>
ManagedPtr<T> make_managed(size_t n) {
    T* ptr;
    CUDA_CHECK(cudaMallocManaged(&ptr, n * sizeof(T)));
    return ManagedPtr<T>(ptr);
}

__global__ void saxpy(float* y, const float* x, float a, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) y[i] = a * x[i] + y[i];
}

double run_saxpy(float* x, float* y, int n, const char* label) {
    auto t0 = std::chrono::high_resolution_clock::now();
    saxpy<<<(n + 255) / 256, 256>>>(y, x, 2.0f, n);
    CUDA_CHECK(cudaDeviceSynchronize());
    auto t1 = std::chrono::high_resolution_clock::now();
    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("%-35s %8.3f ms\n", label, ms);
    return ms;
}

int main() {
    constexpr int N = 1 << 24;
    int dev = 0;
    CUDA_CHECK(cudaSetDevice(dev));

    auto x = make_managed<float>(N);
    auto y = make_managed<float>(N);

    // Initialize on CPU (pages reside on CPU after this)
    for (int i = 0; i < N; ++i) {
        x[i] = 1.0f;
        y[i] = 2.0f;
    }

    // Run 1: Cold -- pages migrate on-demand via page faults
    run_saxpy(x.get(), y.get(), N, "No prefetch (page faults):");

    // Re-init on CPU
    for (int i = 0; i < N; ++i) { x[i] = 1.0f; y[i] = 2.0f; }

    // Run 2: Prefetch to GPU before kernel
    CUDA_CHECK(cudaMemPrefetchAsync(x.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaMemPrefetchAsync(y.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaDeviceSynchronize());
    run_saxpy(x.get(), y.get(), N, "With prefetch:");

    // Run 3: Advise read-mostly for x (replicated on both)
    for (int i = 0; i < N; ++i) { x[i] = 1.0f; y[i] = 2.0f; }
    CUDA_CHECK(cudaMemAdvise(x.get(), N * sizeof(float),
                              cudaMemAdviseSetReadMostly, dev));
    CUDA_CHECK(cudaMemPrefetchAsync(x.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaMemPrefetchAsync(y.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaDeviceSynchronize());
    run_saxpy(x.get(), y.get(), N, "ReadMostly(x) + prefetch:");

    // Verify on CPU (pages migrate back automatically)
    printf("y[0] = %f (expected 4.0)\n", y[0]);

    return 0;
}
```

Notice that accessing `y[0]` on the CPU after the kernel runs triggers another page migration - the pages are on the GPU, and the CPU access faults them back. If you care about that latency, prefetch back to the CPU with `cudaMemPrefetchAsync(ptr, size, cudaCpuDeviceId)` before reading.

### Q2: Show USM thrashing and how to eliminate it with phased access patterns

This is the most important performance lesson for USM. The "thrashing" pattern - alternating GPU writes and CPU reads in a tight loop - triggers continuous page migration that can be 10x or more slower than the same computation with separated phases.

The fix is conceptually simple: do all the GPU work first, synchronize once, then do all the CPU work. This means the pages migrate to the GPU, all GPU iterations run without any migration overhead, then the pages migrate to the CPU once for all CPU iterations. One migration instead of twenty.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <chrono>

#define CUDA_CHECK(x) do { cudaError_t e=(x); \
    if(e!=cudaSuccess){printf("CUDA err %d\n",e);exit(1);}}while(0)

__global__ void gpu_write(float* data, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) data[i] = static_cast<float>(i);
}

void cpu_read(const float* data, int n) {
    volatile float sum = 0;
    for (int i = 0; i < n; i += 1024) sum += data[i];
}

int main() {
    constexpr int N = 1 << 20;
    int dev = 0;
    float* data;
    CUDA_CHECK(cudaMallocManaged(&data, N * sizeof(float)));

    // BAD: Interleaved CPU/GPU access causes thrashing
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int iter = 0; iter < 10; ++iter) {
        gpu_write<<<(N+255)/256, 256>>>(data, N);
        CUDA_CHECK(cudaDeviceSynchronize());
        cpu_read(data, N);  // Forces page migration GPU->CPU
        // Next iteration: GPU->CPU migration again
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    double thrash_ms = std::chrono::duration<double,
                       std::milli>(t1 - t0).count();

    // GOOD: Phased access -- batch GPU work, then batch CPU
    t0 = std::chrono::high_resolution_clock::now();

    // Phase 1: All GPU work
    for (int iter = 0; iter < 10; ++iter) {
        gpu_write<<<(N+255)/256, 256>>>(data, N);
    }
    CUDA_CHECK(cudaDeviceSynchronize());

    // Phase 2: Prefetch to CPU, then all CPU reads
    CUDA_CHECK(cudaMemPrefetchAsync(data, N * sizeof(float),
                                     cudaCpuDeviceId));
    CUDA_CHECK(cudaDeviceSynchronize());
    for (int iter = 0; iter < 10; ++iter) {
        cpu_read(data, N);
    }

    t1 = std::chrono::high_resolution_clock::now();
    double phased_ms = std::chrono::duration<double,
                       std::milli>(t1 - t0).count();

    printf("Thrashing pattern:  %8.2f ms\n", thrash_ms);
    printf("Phased pattern:     %8.2f ms\n", phased_ms);
    printf("Speedup:            %.1fx\n", thrash_ms / phased_ms);

    CUDA_CHECK(cudaFree(data));
    return 0;
}
```

If you ever profile a USM application and see unexpectedly high kernel times with low compute throughput, page migration thrashing is one of the first things to check. The profiler output from `nsys` will show page fault events if they're happening.

### Q3: Implement the same algorithm using SYCL USM types and compare malloc_shared vs malloc_device + explicit copy

SYCL offers the same three allocation strategies as CUDA USM. This example benchmarks all three - shared (automatic migration), device (explicit copy), and host (zero-copy, CPU-resident) - against each other so you can understand the trade-offs in concrete numbers.

The key insight is that `malloc_shared` cold performance is typically worse than `malloc_device` + explicit copy because the migration happens at page granularity with fault overhead. But `malloc_shared` with prefetching narrows that gap considerably, and it makes your code much simpler.

<!-- compile: needs third-party library header `sycl/sycl.hpp` -->
```cpp
#include <sycl/sycl.hpp>
#include <iostream>
#include <chrono>

template <typename F>
double bench(F&& f) {
    auto t0 = std::chrono::high_resolution_clock::now();
    f();
    auto t1 = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(t1 - t0).count();
}

int main() {
    constexpr size_t N = 1 << 22;
    sycl::queue q{sycl::gpu_selector_v};

    // Approach 1: malloc_shared (automatic migration)
    float* shared = sycl::malloc_shared<float>(N, q);
    for (size_t i = 0; i < N; ++i) shared[i] = 1.0f;

    double t_shared_cold = bench([&] {
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            shared[i] *= 2.0f;
        }).wait();
    });

    // Reset and prefetch
    for (size_t i = 0; i < N; ++i) shared[i] = 1.0f;
    q.prefetch(shared, N * sizeof(float)).wait();

    double t_shared_warm = bench([&] {
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            shared[i] *= 2.0f;
        }).wait();
    });

    // Approach 2: malloc_device + explicit copy
    float* dev = sycl::malloc_device<float>(N, q);
    std::vector<float> host(N, 1.0f);

    double t_explicit = bench([&] {
        q.memcpy(dev, host.data(), N * sizeof(float)).wait();
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            dev[i] *= 2.0f;
        }).wait();
        q.memcpy(host.data(), dev, N * sizeof(float)).wait();
    });

    // Approach 3: malloc_host (zero-copy, CPU-resident)
    float* h_usm = sycl::malloc_host<float>(N, q);
    for (size_t i = 0; i < N; ++i) h_usm[i] = 1.0f;

    double t_host_usm = bench([&] {
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            h_usm[i] *= 2.0f;
        }).wait();
    });

    printf("%-35s %8.2f ms\n", "malloc_shared (cold):", t_shared_cold);
    printf("%-35s %8.2f ms\n", "malloc_shared (prefetched):",t_shared_warm);
    printf("%-35s %8.2f ms\n", "malloc_device + copy:", t_explicit);
    printf("%-35s %8.2f ms\n", "malloc_host (zero-copy):", t_host_usm);

    /*
     When to use each:
     ┌────────────────────┬──────────────────────────────┐
     │ USM type           │ Best for                     │
     ├────────────────────┼──────────────────────────────┤
     │ malloc_shared      │ Prototyping, complex structs │
     │ malloc_device      │ Max performance, bulk data   │
     │ malloc_host        │ Small, frequently read by GPU│
     └────────────────────┴──────────────────────────────┘
    */

    sycl::free(shared, q);
    sycl::free(dev, q);
    sycl::free(h_usm, q);
    return 0;
}
```

`malloc_host` (zero-copy) is worth understanding: the data stays in CPU memory and the GPU accesses it over PCIe on every access. For large arrays with low reuse, this can actually beat `malloc_shared` because there's no migration overhead - but for kernels that touch the same data many times, the PCIe latency on every access will dominate.

---

## Notes

- Managed/shared memory migration granularity is 64 KB on most NVIDIA GPUs - even touching one float migrates the entire 64 KB page containing it, so access patterns matter enormously.
- `cudaMemAdviseSetPreferredLocation` pins pages to a specific processor, reducing migration but potentially causing remote access latency when the non-preferred side accesses the data.
- `cudaMemAdviseSetAccessedBy` creates direct mappings to avoid migration entirely - pages stay in place but are accessed remotely via NVLink or PCIe. Use this when a processor needs occasional access and migration overhead would be worse than remote access.
- SYCL `malloc_shared` portability varies - Intel GPUs support true shared virtual memory, while the NVIDIA backend maps to CUDA managed memory with the same page-migration behavior.
- For pointer-rich data structures (trees, graphs), USM is often the only viable option - explicitly marshaling a pointer graph across address spaces is error-prone and impractical.
- Over-subscription (allocating more managed memory than GPU VRAM) works on modern CUDA but triggers eviction and migration - profile with `nsys` to detect this, as it looks like mysteriously slow kernels with no obvious compute bottleneck.
