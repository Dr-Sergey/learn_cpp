# Use SYCL and oneAPI for Portable Heterogeneous Computing

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++20, C++17  
**Reference:** <https://registry.khronos.org/SYCL/specs/sycl-2020/html/sycl-2020.html>  

---

## Topic Overview

SYCL is a Khronos open standard that enables single-source heterogeneous programming in pure C++. Unlike CUDA, SYCL is vendor-neutral - targeting Intel, NVIDIA, AMD, and FPGA backends through implementations such as Intel DPC++ (oneAPI), AdaptiveCpp (formerly hipSYCL), and ComputeCpp. Kernels are expressed as C++ lambdas or function objects submitted to a `sycl::queue`, making the programming model natural for modern C++ developers.

The "single-source" part is worth emphasizing: unlike OpenCL where your kernel is a separate string or file, SYCL lets you write both your host code and your device code in the same C++ source file. The compiler splits it at build time and generates the right code for each target. This makes it much easier to share type definitions, constants, and utility functions between host and device.

The SYCL 2020 specification introduces Unified Shared Memory (USM) as an alternative to the buffer/accessor model, group algorithms for collective operations, and sub-groups for warp/wavefront-level programming. These features bring SYCL close to CUDA in expressiveness while maintaining portability. The oneAPI ecosystem layers on top of SYCL with domain libraries (oneMKL, oneDNN, oneTBB) for a full heterogeneous platform.

| Aspect | CUDA | SYCL 2020 |
| --- | --- | --- |
| Vendor lock-in | NVIDIA only | Portable (Intel, NVIDIA, AMD, FPGA) |
| Kernel syntax | `__global__` functions | C++ lambdas/functors |
| Memory model | Explicit + Unified | Buffer/Accessor + USM |
| Compiler | nvcc | DPC++, AdaptiveCpp, ComputeCpp |
| Sub-group / warp | Warp intrinsics | `sycl::sub_group` API |
| Collective operations | Cooperative Groups | Group algorithms (`reduce`, `scan`) |
| Ecosystem libraries | cuBLAS, cuDNN, Thrust | oneMKL, oneDNN, oneTBB |

Here's the layered architecture showing how a single SYCL application reaches different hardware backends:

```cpp
┌──────────────────────────────────────┐
│           SYCL Application           │
│  (Single-source C++17 with lambdas)  │
├──────────────────────────────────────┤
│         SYCL Runtime / API           │
├─────────┬──────────┬─────────────────┤
│  OpenCL │  Level0  │  CUDA backend   │
│ backend │ backend  │  (via plugins)  │
├─────────┼──────────┼─────────────────┤
│  Any GPU │ Intel GPU│  NVIDIA GPU     │
└─────────┴──────────┴─────────────────┘
```

---

## Self-Assessment

### Q1: Write a SYCL 2020 program using buffer/accessor model to perform a vector addition on any available device

The buffer/accessor model is SYCL's original programming interface. You wrap your host data in `sycl::buffer` objects, then request access to them inside a kernel submission. The key idea is that the runtime tracks data dependencies automatically - if two kernels both use the same buffer, the runtime schedules them in the correct order without you having to manage that explicitly.

Notice the async exception handler passed to `sycl::queue`. Device-side errors in SYCL are asynchronous - if you don't provide a handler, they get silently dropped.

<!-- compile: needs third-party library header `sycl/sycl.hpp` -->
```cpp
#include <sycl/sycl.hpp>
#include <iostream>
#include <vector>
#include <cassert>

int main() {
    constexpr size_t N = 1024;
    std::vector<float> a(N, 1.0f), b(N, 2.0f), c(N, 0.0f);

    // Create a queue targeting the default device (GPU if available)
    sycl::queue q{sycl::default_selector_v,
                  [](sycl::exception_list elist) {
                      for (auto& e : elist) {
                          try { std::rethrow_exception(e); }
                          catch (const sycl::exception& se) {
                              std::cerr << "SYCL async error: "
                                        << se.what() << '\n';
                          }
                      }
                  }};

    std::cout << "Device: "
              << q.get_device().get_info<sycl::info::device::name>()
              << '\n';

    {
        // Create buffers wrapping host data
        sycl::buffer<float> buf_a(a.data(), sycl::range<1>(N));
        sycl::buffer<float> buf_b(b.data(), sycl::range<1>(N));
        sycl::buffer<float> buf_c(c.data(), sycl::range<1>(N));

        q.submit([&](sycl::handler& h) {
            // Accessors define data dependencies -- runtime builds DAG
            auto acc_a = buf_a.get_access<sycl::access::mode::read>(h);
            auto acc_b = buf_b.get_access<sycl::access::mode::read>(h);
            auto acc_c = buf_c.get_access<sycl::access::mode::write>(h);

            h.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
                acc_c[i] = acc_a[i] + acc_b[i];
            });
        });
        // Buffer destruction triggers implicit sync + data writeback
    }

    assert(c[0] == 3.0f);
    std::cout << "c[0] = " << c[0] << " (expected 3.0)\n";
    return 0;
}

// Compile: icpx -fsycl -fsycl-targets=nvptx64-nvidia-cuda vec_add.cpp
//     or: icpx -fsycl vec_add.cpp    (Intel GPU / CPU fallback)
```

The closing brace of the inner scope is doing real work here: when the buffers go out of scope, SYCL synchronizes and writes the results back to the host vectors. This is implicit and easy to miss when reading the code - if you move that scope boundary, you change when the data is available.

### Q2: Implement a matrix transpose using SYCL nd_range, local memory, and sub-group operations

This example demonstrates `nd_range` (the SYCL equivalent of CUDA's grid/block launch) and `local_accessor` (shared memory). The tiling technique is the same as in CUDA - load a tile from global into local memory with coalesced reads, then write it transposed with coalesced writes. Without the tile, the writes would be strided and much slower.

<!-- compile: needs third-party library header `sycl/sycl.hpp` -->
```cpp
#include <sycl/sycl.hpp>
#include <vector>
#include <iostream>

constexpr int TILE = 16;

void transpose(sycl::queue& q, const float* in, float* out,
               int rows, int cols) {
    sycl::range<2> global(rows, cols);
    sycl::range<2> local(TILE, TILE);

    q.submit([&](sycl::handler& h) {
        // Local (shared) memory tile -- avoids uncoalesced global writes
        sycl::local_accessor<float, 2> tile({TILE, TILE}, h);

        h.parallel_for(
            sycl::nd_range<2>(global, local),
            [=](sycl::nd_item<2> item) {
                int gr = item.get_global_id(0);
                int gc = item.get_global_id(1);
                int lr = item.get_local_id(0);
                int lc = item.get_local_id(1);

                // Load into local memory (coalesced read)
                tile[lr][lc] = in[gr * cols + gc];
                item.barrier(sycl::access::fence_space::local_space);

                // Compute transposed global coordinates
                int new_row = item.get_group(1) * TILE + lr;
                int new_col = item.get_group(0) * TILE + lc;

                // Write from local memory (coalesced write after transpose)
                out[new_row * rows + new_col] = tile[lc][lr];
            });
    }).wait();
}

int main() {
    constexpr int R = 1024, C = 1024;
    sycl::queue q{sycl::gpu_selector_v};
    std::cout << "Device: "
              << q.get_device().get_info<sycl::info::device::name>()
              << '\n';

    float* in  = sycl::malloc_shared<float>(R * C, q);
    float* out = sycl::malloc_shared<float>(R * C, q);

    for (int i = 0; i < R * C; ++i) in[i] = static_cast<float>(i);

    transpose(q, in, out, R, C);

    // Verify: out[j * R + i] should equal in[i * C + j]
    bool ok = true;
    for (int i = 0; i < R && ok; ++i)
        for (int j = 0; j < C && ok; ++j)
            if (out[j * R + i] != in[i * C + j]) ok = false;

    std::cout << "Transpose: " << (ok ? "PASS" : "FAIL") << '\n';

    sycl::free(in, q);
    sycl::free(out, q);
    return 0;
}
```

The `item.barrier(...)` is the SYCL equivalent of `__syncthreads()`. All work-items in the work-group must reach it before any of them continue past it - this ensures every thread has finished writing its element into the local tile before any thread reads from it.

### Q3: Compare SYCL group algorithms (reduce, scan) with manual implementations and show sub-group usage

SYCL 2020 introduced built-in group algorithms that express collective operations at multiple granularities. The key insight is that `reduce_over_group` on a sub-group maps directly to warp shuffle instructions on NVIDIA hardware and wavefront operations on AMD - without any shared memory, and without you writing any architecture-specific intrinsics.

This is the SYCL equivalent of CUDA Cooperative Groups' `cg::reduce()`. If you write it this way, the same source code generates efficient shuffle instructions on NVIDIA and efficient wavefront operations on AMD.

<!-- compile: needs third-party library header `sycl/sycl.hpp` -->
```cpp
#include <sycl/sycl.hpp>
#include <iostream>
#include <vector>
#include <numeric>

// Reduction using SYCL 2020 group algorithms
float sycl_reduce(sycl::queue& q, const float* data, size_t n) {
    float* result = sycl::malloc_shared<float>(1, q);
    *result = 0.0f;

    constexpr int WG = 256;
    size_t global = ((n + WG - 1) / WG) * WG;  // round up

    q.submit([&](sycl::handler& h) {
        h.parallel_for(
            sycl::nd_range<1>(global, WG),
            [=](sycl::nd_item<1> item) {
                size_t gid = item.get_global_id(0);
                float val = (gid < n) ? data[gid] : 0.0f;

                // Sub-group level reduction (warp-level, no barrier)
                auto sg = item.get_sub_group();
                float sg_sum = sycl::reduce_over_group(
                    sg, val, sycl::plus<float>());

                // Work-group level reduction
                float wg_sum = sycl::reduce_over_group(
                    item.get_group(), val, sycl::plus<float>());

                // One thread per work-group atomically accumulates
                if (item.get_local_id(0) == 0) {
                    sycl::atomic_ref<float,
                        sycl::memory_order::relaxed,
                        sycl::memory_scope::device,
                        sycl::access::address_space::global_space>
                            ref(*result);
                    ref.fetch_add(wg_sum);
                }
            });
    }).wait();

    float ret = *result;
    sycl::free(result, q);
    return ret;
}

int main() {
    constexpr size_t N = 1 << 20;
    sycl::queue q{sycl::gpu_selector_v};

    float* data = sycl::malloc_shared<float>(N, q);
    for (size_t i = 0; i < N; ++i) data[i] = 1.0f;

    float gpu_sum = sycl_reduce(q, data, N);
    float cpu_sum = static_cast<float>(N);  // all ones

    std::cout << "GPU reduce: " << gpu_sum << '\n'
              << "Expected:   " << cpu_sum << '\n'
              << "Match: " << (gpu_sum == cpu_sum ? "YES" : "NO") << '\n';

    // Query sub-group sizes supported
    auto sg_sizes = q.get_device()
        .get_info<sycl::info::device::sub_group_sizes>();
    std::cout << "Supported sub-group sizes:";
    for (auto s : sg_sizes) std::cout << ' ' << s;
    std::cout << '\n';

    sycl::free(data, q);
    return 0;
}

/*
 Sub-group mapping:
 ┌────────────────────────────────────────────┐
 │              Work-group (256 items)         │
 │  ┌──────────┐ ┌──────────┐     ┌────────┐  │
 │  │ Sub-grp 0│ │ Sub-grp 1│ ... │ SG n-1 │  │
 │  │ (32 items)│ │ (32 items)│    │(32 items)│ │
 │  └──────────┘ └──────────┘     └────────┘  │
 └────────────────────────────────────────────┘
 Sub-group = warp (NVIDIA) / wavefront (AMD)
*/
```

The sub-group size query at the end is a practical necessity. Sub-groups are 32 items on NVIDIA, 64 on most AMD GPUs, and may be smaller on Intel integrated graphics. If your algorithm assumes a fixed sub-group size, you need to query and verify at runtime.

---

## Notes

- SYCL 2020 buffer/accessor model builds an implicit DAG - the runtime schedules data movement and kernel ordering automatically, which is convenient but can make performance harder to reason about.
- USM (`malloc_shared`, `malloc_device`, `malloc_host`) provides a CUDA-like explicit pointer model for developers who prefer direct control over data placement.
- Sub-groups map to hardware warps (NVIDIA, 32) or wavefronts (AMD, 64) - use `sycl::sub_group` for vendor-portable warp-level programming without writing platform-specific intrinsics.
- DPC++ (Intel) supports NVIDIA GPUs via the CUDA backend plugin: compile with `-fsycl-targets=nvptx64-nvidia-cuda`.
- Always attach an async exception handler to `sycl::queue` - uncaught device errors are silently lost otherwise, and debugging GPU errors without this is extremely painful.
- Group algorithms (`reduce_over_group`, `inclusive_scan_over_group`) replace manual shared-memory reductions with single function calls that the compiler lowers to optimal hardware instructions.
