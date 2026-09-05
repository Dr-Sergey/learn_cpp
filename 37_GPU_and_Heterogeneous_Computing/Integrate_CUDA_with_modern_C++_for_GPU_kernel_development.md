# Integrate CUDA with Modern C++ for GPU Kernel Development

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17, C++20  
**Reference:** https://docs.nvidia.com/cuda/cuda-c-programming-guide/  

---

## Topic Overview

CUDA remains the dominant GPU programming model for NVIDIA hardware, and modern CUDA (12.x) embraces C++17 and partial C++20 support in device code. This means you can leverage `constexpr`, structured bindings, `std::optional`, `if constexpr`, and class template argument deduction (CTAD) directly inside `__device__` and `__global__` functions, producing cleaner and safer kernel code than the C-style CUDA of earlier eras. If you've been avoiding these features in kernel code because "CUDA doesn't support C++17", that excuse is now outdated.

Unified Memory (`cudaMallocManaged`) simplifies the programming model by providing a single pointer space accessible from both host and device, with the CUDA runtime handling page migration. However, expert-level usage demands understanding of prefetching (`cudaMemPrefetchAsync`), memory advise hints (`cudaMemAdvise`), and the performance implications of on-demand page faulting versus explicit transfers. The short version: unified memory is great for prototyping; use prefetching before you ship.

Error handling in CUDA has traditionally relied on checking `cudaError_t` return codes - a pattern that is fragile and verbose. Modern C++ RAII wrappers around CUDA resources (streams, events, device memory) ensure deterministic cleanup and can throw exceptions on error, integrating naturally with C++ exception-based control flow on the host side. The shift from raw error codes to RAII is one of the most impactful things you can do to make CUDA code maintainable.

The table below shows how far things have come. If you're writing new code today, target the right column:

| Feature               | CUDA C (Legacy)           | Modern CUDA C++ (12.x)            |
| --- | --- | --- |
| Device lambdas        | Not supported             | `__device__` lambdas, generic     |
| constexpr in device   | Limited                   | Full C++17 constexpr              |
| Structured bindings   | No                        | Yes (C++17)                       |
| RAII for resources    | Manual `cudaFree`         | Custom deleters with unique_ptr   |
| Error handling        | Raw error codes           | RAII + exceptions on host         |
| Unified Memory        | Basic `cudaMallocManaged` | + Prefetch, MemAdvise, overcommit |

---

## Self-Assessment

### Q1: Write a CUDA kernel using C++17 features (constexpr, if constexpr) and wrap device memory in an RAII class

This example shows two things at once: a proper RAII device buffer that throws on allocation failure (so you can't accidentally use a null pointer), and a templated kernel that uses `if constexpr` to select between operations at compile time rather than branching at runtime. Notice that `constexpr` works inside device code just as you'd expect.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <stdexcept>
#include <memory>
#include <type_traits>

// RAII wrapper for CUDA device memory
template <typename T>
class CudaDeviceBuffer {
    T* ptr_ = nullptr;
    size_t count_ = 0;

public:
    explicit CudaDeviceBuffer(size_t n) : count_(n) {
        cudaError_t err = cudaMalloc(&ptr_, n * sizeof(T));
        if (err != cudaSuccess)
            throw std::runtime_error(cudaGetErrorString(err));
    }

    ~CudaDeviceBuffer() {
        if (ptr_) cudaFree(ptr_);
    }

    // Non-copyable, movable
    CudaDeviceBuffer(const CudaDeviceBuffer&) = delete;
    CudaDeviceBuffer& operator=(const CudaDeviceBuffer&) = delete;

    CudaDeviceBuffer(CudaDeviceBuffer&& o) noexcept
        : ptr_(o.ptr_), count_(o.count_) { o.ptr_ = nullptr; }

    CudaDeviceBuffer& operator=(CudaDeviceBuffer&& o) noexcept {
        if (this != &o) {
            if (ptr_) cudaFree(ptr_);
            ptr_ = o.ptr_; count_ = o.count_;
            o.ptr_ = nullptr;
        }
        return *this;
    }

    T* get() noexcept { return ptr_; }
    size_t size() const noexcept { return count_; }

    void copyFromHost(const T* src) {
        cudaMemcpy(ptr_, src, count_ * sizeof(T), cudaMemcpyHostToDevice);
    }
    void copyToHost(T* dst) const {
        cudaMemcpy(dst, ptr_, count_ * sizeof(T), cudaMemcpyDeviceToHost);
    }
};

// Kernel using if constexpr for compile-time dispatch
enum class Op { Add, Mul, Fma };

template <Op op>
__global__ void elementwise(float* out, const float* a,
                            const float* b, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx >= n) return;

    if constexpr (op == Op::Add) {
        out[idx] = a[idx] + b[idx];
    } else if constexpr (op == Op::Mul) {
        out[idx] = a[idx] * b[idx];
    } else if constexpr (op == Op::Fma) {
        constexpr float alpha = 2.0f;  // constexpr in device code
        out[idx] = fmaf(alpha, a[idx], b[idx]);
    }
}

int main() {
    constexpr int N = 1 << 20;
    std::vector<float> ha(N, 1.0f), hb(N, 2.0f), hc(N);

    CudaDeviceBuffer<float> da(N), db(N), dc(N);
    da.copyFromHost(ha.data());
    db.copyFromHost(hb.data());

    int threads = 256;
    int blocks = (N + threads - 1) / threads;
    elementwise<Op::Fma><<<blocks, threads>>>(dc.get(), da.get(), db.get(), N);

    dc.copyToHost(hc.data());
    printf("result[0] = %f\n", hc[0]);  // 2*1+2 = 4
    return 0;
}
```

Because `da`, `db`, and `dc` are stack objects, the device memory is automatically freed when `main` returns - even if an exception is thrown. The `if constexpr` in the kernel generates zero branch instructions at runtime; the compiler emits only the path you selected at instantiation time.

### Q2: Implement CUDA Unified Memory with prefetching and memory advise hints, and compare latency

The performance difference between cold unified memory and prefetched unified memory can be dramatic - often a 10-15x gap. This benchmark makes that concrete. The reason the cold case is slow is that every page fault triggers a CPU-GPU round trip to migrate that page; prefetching batches all migrations before the kernel starts.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <chrono>

#define CUDA_CHECK(call)                                            \
    do {                                                            \
        cudaError_t err = (call);                                   \
        if (err != cudaSuccess) {                                   \
            fprintf(stderr, "CUDA error at %s:%d: %s\n",           \
                    __FILE__, __LINE__, cudaGetErrorString(err));   \
            std::exit(EXIT_FAILURE);                                \
        }                                                           \
    } while (0)

__global__ void scale_kernel(float* data, float factor, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) data[i] *= factor;
}

void benchmark(const char* label, float* data, int n, int device) {
    auto t0 = std::chrono::high_resolution_clock::now();
    int threads = 256, blocks = (n + threads - 1) / threads;
    scale_kernel<<<blocks, threads>>>(data, 2.0f, n);
    CUDA_CHECK(cudaDeviceSynchronize());
    auto t1 = std::chrono::high_resolution_clock::now();
    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("%-30s %8.3f ms\n", label, ms);
}

int main() {
    constexpr int N = 1 << 24;  // 64M floats
    int device = 0;
    CUDA_CHECK(cudaSetDevice(device));

    float* data;
    CUDA_CHECK(cudaMallocManaged(&data, N * sizeof(float)));

    // Initialize on host
    for (int i = 0; i < N; ++i) data[i] = 1.0f;

    // Run 1: No prefetch — on-demand page faulting
    benchmark("No prefetch (cold)", data, N, device);

    // Reset
    for (int i = 0; i < N; ++i) data[i] = 1.0f;

    // Run 2: Prefetch to device before kernel
    CUDA_CHECK(cudaMemPrefetchAsync(data, N * sizeof(float), device));
    CUDA_CHECK(cudaDeviceSynchronize());
    benchmark("With prefetch", data, N, device);

    // Run 3: Use memory advise for read-mostly
    CUDA_CHECK(cudaMemAdvise(data, N * sizeof(float),
                             cudaMemAdviseSetReadMostly, device));
    CUDA_CHECK(cudaMemPrefetchAsync(data, N * sizeof(float), device));
    CUDA_CHECK(cudaDeviceSynchronize());
    benchmark("ReadMostly + prefetch", data, N, device);

    CUDA_CHECK(cudaFree(data));
    return 0;
}

/*
 Typical output (A100):
 No prefetch (cold)              12.450 ms
 With prefetch                    0.820 ms
 ReadMostly + prefetch            0.790 ms
*/
```

The `cudaMemAdvise(ReadMostly)` hint tells the runtime it can create read-only replicas of the data on the device, which helps if multiple devices or the CPU also need to read the same data. For this single-GPU case the improvement over plain prefetch is marginal, but it's good habit to set the hint correctly.

### Q3: Build an RAII-based CUDA stream manager and overlap kernel execution with async memory transfers

This is the pattern you'll use for production GPU code: pinned host memory plus multiple streams, with each stream handling H2D transfer, kernel, and D2H transfer for its chunk independently. The streams run concurrently, so the GPU's copy engine and compute engine can work at the same time on different chunks.

<!-- compile: needs third-party library header `cuda_runtime.h` -->
```cpp
#include <cuda_runtime.h>
#include <vector>
#include <cstdio>
#include <stdexcept>

// RAII CUDA stream
class CudaStream {
    cudaStream_t s_;
public:
    CudaStream() { cudaStreamCreate(&s_); }
    ~CudaStream() { cudaStreamDestroy(s_); }
    CudaStream(const CudaStream&) = delete;
    CudaStream& operator=(const CudaStream&) = delete;
    operator cudaStream_t() const { return s_; }
    void sync() const { cudaStreamSynchronize(s_); }
};

// RAII CUDA event
class CudaEvent {
    cudaEvent_t e_;
public:
    CudaEvent() { cudaEventCreate(&e_); }
    ~CudaEvent() { cudaEventDestroy(e_); }
    CudaEvent(const CudaEvent&) = delete;
    operator cudaEvent_t() const { return e_; }
    void record(cudaStream_t s) { cudaEventRecord(e_, s); }
    static float elapsed(const CudaEvent& start, const CudaEvent& stop) {
        float ms;
        cudaEventElapsedTime(&ms, start.e_, stop.e_);
        return ms;
    }
};

__global__ void process(float* out, const float* in, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) out[i] = sinf(in[i]) * cosf(in[i]);
}

int main() {
    constexpr int N = 1 << 22;
    constexpr int CHUNKS = 4;
    constexpr int CHUNK = N / CHUNKS;

    // Pinned host memory for async transfers
    float *h_in, *h_out;
    cudaMallocHost(&h_in, N * sizeof(float));
    cudaMallocHost(&h_out, N * sizeof(float));
    for (int i = 0; i < N; ++i) h_in[i] = static_cast<float>(i);

    float *d_in, *d_out;
    cudaMalloc(&d_in, N * sizeof(float));
    cudaMalloc(&d_out, N * sizeof(float));

    // Create streams for overlapping
    std::vector<CudaStream> streams(CHUNKS);

    CudaEvent start, stop;
    start.record(nullptr);

    for (int c = 0; c < CHUNKS; ++c) {
        int offset = c * CHUNK;
        cudaStream_t s = streams[c];

        // Async H2D for this chunk
        cudaMemcpyAsync(d_in + offset, h_in + offset,
                        CHUNK * sizeof(float),
                        cudaMemcpyHostToDevice, s);

        // Kernel on this chunk
        int threads = 256, blocks = (CHUNK + threads - 1) / threads;
        process<<<blocks, threads, 0, s>>>(
            d_out + offset, d_in + offset, CHUNK);

        // Async D2H for this chunk
        cudaMemcpyAsync(h_out + offset, d_out + offset,
                        CHUNK * sizeof(float),
                        cudaMemcpyDeviceToHost, s);
    }

    stop.record(nullptr);
    cudaDeviceSynchronize();
    printf("Overlapped: %.3f ms\n", CudaEvent::elapsed(start, stop));

    cudaFree(d_in); cudaFree(d_out);
    cudaFreeHost(h_in); cudaFreeHost(h_out);
    return 0;
}

/*
 Timeline (4 chunks overlapped):
   Stream 0: [H2D][Kernel][D2H]
   Stream 1:      [H2D][Kernel][D2H]
   Stream 2:           [H2D][Kernel][D2H]
   Stream 3:                [H2D][Kernel][D2H]
*/
```

The critical requirement for async transfers is pinned host memory - `cudaMemcpyAsync` with pageable memory silently falls back to synchronous behavior, which defeats the entire purpose. The RAII `CudaStream` and `CudaEvent` wrappers mean you don't need to track `cudaStreamDestroy` calls manually - they clean up when the objects go out of scope.

---

## Notes

- CUDA 12.x supports C++17 fully and C++20 partially in device code; check `__CUDACC_VER_MAJOR__` for feature gates.
- Always use pinned memory (`cudaMallocHost`) for async transfers - pageable memory silently falls back to synchronous copy.
- RAII wrappers for streams, events, and device memory eliminate entire classes of resource leak bugs.
- `cudaMemPrefetchAsync` can reduce first-touch latency by 10-15x compared to on-demand page faulting.
- Compile with `nvcc -std=c++17 --expt-relaxed-constexpr` for maximum C++ feature availability in device code.
- Profile with `nsys` (Nsight Systems) to verify overlap correctness - visual timeline reveals serialization bugs instantly.
