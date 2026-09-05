# Optimize Host-Device Data Transfer Patterns

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17, C++20  
**Reference:** https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/  

---

## Topic Overview

Data transfer between host (CPU) and device (GPU) is often the dominant bottleneck in GPU-accelerated applications. PCIe 4.0 delivers ~25 GB/s theoretical bandwidth (per direction), but real-world throughput depends critically on memory allocation type, transfer granularity, and overlap strategy. You need to master three core transfer optimization techniques: pinned memory allocation, asynchronous transfers with stream overlap, and whole-transfer elimination via Unified Shared Memory.

Pinned (page-locked) memory bypasses the OS paging system, enabling DMA transfers directly between host RAM and GPU memory. Pageable memory requires an extra copy through a pinned staging buffer internally, reducing effective bandwidth by up to 50%. CUDA provides `cudaMallocHost` / `cudaHostAlloc`; SYCL offers `sycl::malloc_host`. The tradeoff is that pinned memory is a limited system resource - over-allocation can degrade overall system performance by reducing available pageable memory, so don't just pin everything.

Overlapping compute and transfer requires partitioning data into chunks and using multiple streams (CUDA) or queues (SYCL) so that chunk N's transfer occurs concurrently with chunk N-1's kernel execution. This pipelining can approach full utilization of both PCIe bus and GPU compute simultaneously - which is the best you can do short of eliminating transfers entirely.

The table below shows typical bandwidth numbers for each strategy on PCIe 4.0 hardware. Notice that the bandwidth numbers for pinned and async are the same - the advantage of async isn't more bandwidth, it's overlap:

| Transfer Strategy        | Bandwidth (typical) | Latency     | Complexity |
| --- | --- | --- | --- |
| Pageable `memcpy`        | 6-12 GB/s           | Synchronous | Low        |
| Pinned `memcpy`          | 12-25 GB/s          | Synchronous | Low        |
| Pinned + async stream    | 12-25 GB/s          | Overlapped  | Medium     |
| USM (managed memory)     | Varies (on-demand)  | Page faults | Low        |
| USM + prefetch           | 12-25 GB/s          | Controlled  | Medium     |
| Zero-copy (mapped)       | PCIe on access      | Per-access  | Low        |

Here is what the four-chunk overlap pipeline looks like in time. Without overlap, transfers and kernel execution serialize into a long chain. With overlap across two streams, transfers for the next chunk happen while the current chunk's kernel runs:

```cpp
Pipeline: 4-chunk overlap on 2 streams

Time ----------------------------------------------------------------->
Stream 0: [H2D0][Kernel0][D2H0]      [H2D2][Kernel2][D2H2]
Stream 1:       [H2D1][Kernel1][D2H1]       [H2D3][Kernel3][D2H3]

Without overlap:  [H2D][Kernel][D2H][H2D][Kernel][D2H]...
                  ------------ 3x longer ------------------
```

---

## Self-Assessment

### Q1: Benchmark pinned vs pageable host memory transfer rates and demonstrate the bandwidth difference

Let's put real numbers on the difference. This benchmark runs 20 iterations of a 256 MB transfer for each memory type and reports bandwidth in GB/s. The warm-up transfer before the timed loop ensures the CUDA context is initialized and the first-call overhead doesn't skew the results.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <chrono>
#include <vector>

#define CUDA_CHECK(call)                                         \
    do { cudaError_t e = (call);                                 \
         if (e != cudaSuccess) {                                 \
             fprintf(stderr, "CUDA %s:%d: %s\n",                \
                     __FILE__, __LINE__, cudaGetErrorString(e)); \
             exit(1); }                                          \
    } while (0)

void benchmark_transfer(const char* label, float* host_ptr,
                         float* dev_ptr, size_t bytes) {
    // Warm up
    CUDA_CHECK(cudaMemcpy(dev_ptr, host_ptr, bytes,
                          cudaMemcpyHostToDevice));
    CUDA_CHECK(cudaDeviceSynchronize());

    constexpr int ITERS = 20;
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERS; ++i) {
        CUDA_CHECK(cudaMemcpy(dev_ptr, host_ptr, bytes,
                              cudaMemcpyHostToDevice));
    }
    CUDA_CHECK(cudaDeviceSynchronize());
    auto t1 = std::chrono::high_resolution_clock::now();

    double sec = std::chrono::duration<double>(t1 - t0).count();
    double gb = (double)bytes * ITERS / (1024.0 * 1024.0 * 1024.0);
    printf("%-25s %7.2f GB/s  (%6.2f ms per transfer)\n",
           label, gb / sec, (sec / ITERS) * 1000.0);
}

int main() {
    constexpr size_t N = 1 << 24;  // 64M floats = 256 MB
    size_t bytes = N * sizeof(float);

    float* d_buf;
    CUDA_CHECK(cudaMalloc(&d_buf, bytes));

    // Pageable host memory
    std::vector<float> pageable(N, 1.0f);
    benchmark_transfer("Pageable H2D:", pageable.data(), d_buf, bytes);

    // Pinned host memory
    float* pinned;
    CUDA_CHECK(cudaMallocHost(&pinned, bytes));
    for (size_t i = 0; i < N; ++i) pinned[i] = 1.0f;
    benchmark_transfer("Pinned H2D:", pinned, d_buf, bytes);

    // Write-combined pinned (optimal for H2D, bad for CPU reads)
    float* wc_pinned;
    CUDA_CHECK(cudaHostAlloc(&wc_pinned, bytes,
                              cudaHostAllocWriteCombined));
    for (size_t i = 0; i < N; ++i) wc_pinned[i] = 1.0f;
    benchmark_transfer("Write-Combined H2D:", wc_pinned, d_buf, bytes);

    CUDA_CHECK(cudaFreeHost(pinned));
    CUDA_CHECK(cudaFreeHost(wc_pinned));
    CUDA_CHECK(cudaFree(d_buf));
    return 0;
}

/*
 Typical results (PCIe 4.0 x16):
 Pageable H2D:             10.50 GB/s  ( 23.24 ms per transfer)
 Pinned H2D:               24.10 GB/s  ( 10.12 ms per transfer)
 Write-Combined H2D:       25.20 GB/s  (  9.68 ms per transfer)
*/
```

Write-combined memory squeezes out a little extra H2D throughput by bypassing CPU cache for writes, allowing larger burst transactions to the PCIe bus. The catch is that CPU reads from write-combined memory are extremely slow - use it only for buffers that the CPU writes once and never reads back.

### Q2: Implement a multi-stream pipeline that overlaps H2D transfer, kernel execution, and D2H transfer across chunks

This is the core overlap pattern you'll use in any throughput-sensitive GPU pipeline. The key requirement that catches people out: async transfers only work with pinned memory. If you pass pageable memory to `cudaMemcpyAsync`, CUDA silently makes it synchronous, and your overlap disappears without any error or warning.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <vector>
#include <cstdio>

#define CUDA_CHECK(x) do { cudaError_t e=(x); \
    if(e!=cudaSuccess){printf("err %s:%d\n",__FILE__,__LINE__);exit(1);}}\
    while(0)

__global__ void heavy_kernel(float* out, const float* in, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        float v = in[i];
        for (int j = 0; j < 100; ++j) v = sinf(v) + 0.001f;
        out[i] = v;
    }
}

int main() {
    constexpr int N = 1 << 22;       // 4M floats
    constexpr int NSTREAMS = 4;
    constexpr int CHUNK = N / NSTREAMS;
    size_t chunk_bytes = CHUNK * sizeof(float);

    // Pinned host memory (required for async)
    float *h_in, *h_out;
    CUDA_CHECK(cudaMallocHost(&h_in, N * sizeof(float)));
    CUDA_CHECK(cudaMallocHost(&h_out, N * sizeof(float)));
    for (int i = 0; i < N; ++i) h_in[i] = static_cast<float>(i);

    float *d_in, *d_out;
    CUDA_CHECK(cudaMalloc(&d_in, N * sizeof(float)));
    CUDA_CHECK(cudaMalloc(&d_out, N * sizeof(float)));

    cudaStream_t streams[NSTREAMS];
    for (int i = 0; i < NSTREAMS; ++i)
        CUDA_CHECK(cudaStreamCreate(&streams[i]));

    cudaEvent_t start, stop;
    CUDA_CHECK(cudaEventCreate(&start));
    CUDA_CHECK(cudaEventCreate(&stop));

    // === Overlapped execution ===
    CUDA_CHECK(cudaEventRecord(start));
    for (int s = 0; s < NSTREAMS; ++s) {
        int off = s * CHUNK;
        cudaMemcpyAsync(d_in + off, h_in + off, chunk_bytes,
                        cudaMemcpyHostToDevice, streams[s]);
        heavy_kernel<<<(CHUNK+255)/256, 256, 0, streams[s]>>>(
            d_out + off, d_in + off, CHUNK);
        cudaMemcpyAsync(h_out + off, d_out + off, chunk_bytes,
                        cudaMemcpyDeviceToHost, streams[s]);
    }
    CUDA_CHECK(cudaEventRecord(stop));
    CUDA_CHECK(cudaDeviceSynchronize());

    float overlap_ms;
    CUDA_CHECK(cudaEventElapsedTime(&overlap_ms, start, stop));

    // === Sequential execution for comparison ===
    CUDA_CHECK(cudaEventRecord(start));
    cudaMemcpy(d_in, h_in, N*sizeof(float), cudaMemcpyHostToDevice);
    heavy_kernel<<<(N+255)/256, 256>>>(d_out, d_in, N);
    cudaMemcpy(h_out, d_out, N*sizeof(float), cudaMemcpyDeviceToHost);
    CUDA_CHECK(cudaEventRecord(stop));
    CUDA_CHECK(cudaDeviceSynchronize());

    float seq_ms;
    CUDA_CHECK(cudaEventElapsedTime(&seq_ms, start, stop));

    printf("Sequential: %.2f ms\n", seq_ms);
    printf("Overlapped: %.2f ms (%.1fx speedup)\n",
           overlap_ms, seq_ms / overlap_ms);

    for (int i = 0; i < NSTREAMS; ++i)
        CUDA_CHECK(cudaStreamDestroy(streams[i]));
    CUDA_CHECK(cudaEventDestroy(start));
    CUDA_CHECK(cudaEventDestroy(stop));
    CUDA_CHECK(cudaFreeHost(h_in));
    CUDA_CHECK(cudaFreeHost(h_out));
    CUDA_CHECK(cudaFree(d_in));
    CUDA_CHECK(cudaFree(d_out));
    return 0;
}
```

The speedup you get depends on the ratio of transfer time to compute time for each chunk. If compute dominates, transfers hide almost completely and you approach the compute-only limit. If transfers dominate, you need more chunks or larger chunks. Profile with `nsys` to see the actual timeline and find the sweet spot.

### Q3: Compare USM (managed memory) with explicit transfers in SYCL and show prefetch impact

SYCL's Unified Shared Memory has three flavors: device memory (explicit copies required), host memory (device accesses it over PCIe on every touch), and shared memory (migrates on demand). This example compares all three transfer strategies on the same operation so you can see the practical impact of each choice.

<!-- compile: needs third-party library header `sycl/sycl.hpp` -->
```cpp
#include <sycl/sycl.hpp>
#include <iostream>
#include <vector>
#include <chrono>

template <typename F>
double measure_ms(F&& fn) {
    auto t0 = std::chrono::high_resolution_clock::now();
    fn();
    auto t1 = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(t1 - t0).count();
}

void kernel_op(sycl::queue& q, float* data, size_t n) {
    q.parallel_for(sycl::range<1>(n), [=](sycl::id<1> i) {
        data[i] *= 2.0f;
    }).wait();
}

int main() {
    constexpr size_t N = 1 << 22;
    sycl::queue q{sycl::gpu_selector_v};
    auto dev = q.get_device();

    // === Method 1: Explicit buffer + copy ===
    std::vector<float> host_data(N, 1.0f);
    float* d_explicit = sycl::malloc_device<float>(N, q);

    double t_explicit = measure_ms([&] {
        q.memcpy(d_explicit, host_data.data(), N * sizeof(float)).wait();
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            d_explicit[i] *= 2.0f;
        }).wait();
        q.memcpy(host_data.data(), d_explicit, N * sizeof(float)).wait();
    });

    // === Method 2: USM shared (managed) — no prefetch ===
    float* shared = sycl::malloc_shared<float>(N, q);
    for (size_t i = 0; i < N; ++i) shared[i] = 1.0f;

    double t_shared = measure_ms([&] {
        kernel_op(q, shared, N);
    });

    // === Method 3: USM shared + prefetch ===
    for (size_t i = 0; i < N; ++i) shared[i] = 1.0f;
    q.prefetch(shared, N * sizeof(float)).wait();

    double t_prefetch = measure_ms([&] {
        kernel_op(q, shared, N);
    });

    printf("%-30s %8.2f ms\n", "Explicit copy:", t_explicit);
    printf("%-30s %8.2f ms\n", "USM shared (no prefetch):", t_shared);
    printf("%-30s %8.2f ms\n", "USM shared + prefetch:", t_prefetch);

    sycl::free(d_explicit, q);
    sycl::free(shared, q);
    return 0;
}

/*
 Key decision matrix:
 USM Type            | Host access? | Device acc? | Migration
 malloc_device       | No           | Yes         | Explicit
 malloc_host         | Yes          | Via PCIe    | None
 malloc_shared       | Yes          | Yes         | On-demand
*/
```

The decision matrix at the bottom is the practical takeaway. Use `malloc_device` when you control all transfers explicitly and want maximum throughput. Use `malloc_shared` with `prefetch` when you want the convenience of unified addressing but still care about performance. Use `malloc_host` only for small buffers where the device needs occasional access - PCIe-on-every-access is fine for a few elements but terrible for large working sets.

---

## Notes

- Pinned memory delivers ~2x bandwidth over pageable on PCIe 4.0, but limit allocation to what's needed - it reduces the OS-available pageable pool for the whole system.
- Async transfers require pinned memory - `cudaMemcpyAsync` with pageable memory silently becomes synchronous, with no error.
- Optimal chunk count for overlap depends on PCIe/compute ratio; profile with `nsys` to find the sweet spot (typically 2-8 chunks).
- Write-combined pinned memory (`cudaHostAllocWriteCombined`) maximizes H2D throughput but has abysmal CPU read performance - never read back from it on the host.
- SYCL `prefetch()` is a hint - implementations may ignore it; always profile to verify the benefit.
- For repeated small transfers, consider persistent mapped memory (`cudaHostGetDevicePointer`) for zero-copy access.
