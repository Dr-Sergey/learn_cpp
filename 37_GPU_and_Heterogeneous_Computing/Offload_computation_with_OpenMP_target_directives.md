# Offload Computation with OpenMP Target Directives

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** https://www.openmp.org/specifications/  

---

## Topic Overview

OpenMP target offloading extends the familiar OpenMP pragma-based parallelism model to accelerators (GPUs, FPGAs). Starting with OpenMP 4.0 and significantly enhanced in 5.x, `#pragma omp target` directives mark regions for device execution, while `map` clauses control data movement. The `teams distribute parallel for` construct maps to GPU execution hierarchies: teams become thread blocks, threads become CUDA threads - enabling GPU-scale parallelism with minimal code changes compared to CUDA or SYCL.

The key advantage of OpenMP target offloading is incremental adoption: existing OpenMP-parallel CPU code can be offloaded to GPUs by adding `target` and `map` directives without rewriting algorithms. The compiler generates device code transparently. This is appealing if you have a working CPU OpenMP codebase and want to try GPU acceleration without committing to CUDA.

That said, performance portability requires understanding how OpenMP constructs map to GPU execution. The reason this trips people up is that `#pragma omp parallel for` means "use CPU threads," but adding `target teams distribute` in front of it changes the meaning entirely: `teams` creates independent thread groups (blocks), `distribute` partitions iterations across teams, and `parallel for` parallelizes within each team. The `simd` clause can further request vectorization within each thread.

Compiler support varies significantly. If your target table feels like a lot of caveats, the practical summary is: use Clang/LLVM for broadest GPU support, nvc++ for best NVIDIA performance, and don't expect MSVC or mixed-vendor combinations to work:

| Compiler      | NVIDIA GPU | AMD GPU  | Intel GPU | Status (2025)      |
| --- | --- | --- | --- | --- |
| Clang/LLVM 17+| Yes        | Yes      | Partial   | Best overall        |
| GCC 14+       | Yes (nvptx)| Yes (gcn)| No        | Good, improving     |
| Intel icpx    | No         | No       | Yes       | Intel GPUs only     |
| NVIDIA nvc++  | Yes        | No       | No        | Excellent NVIDIA    |
| MSVC          | No         | No       | No        | Not supported       |

The diagram below shows how the three-level hierarchy maps onto GPU hardware. Each level corresponds to a different layer of the `#pragma omp target teams distribute parallel for` construct:

```cpp
OpenMP Target Execution Mapping to GPU:

  #pragma omp target teams distribute parallel for
  for (int i = 0; i < N; i++) { ... }

  +--------------- GPU Grid ------------------+
  |  Team 0 (Block)   Team 1 (Block)    ...   |
  |  +--------------+ +--------------+        |
  |  | Thread 0..255| | Thread 0..255|        |
  |  | i=0..255     | | i=256..511   |        |
  |  +--------------+ +--------------+        |
  +--------------------------------------------+

  teams         -> GPU thread blocks (SMs)
  distribute    -> Distribute loop iterations across teams
  parallel for  -> Parallelize within each team
```

---

## Self-Assessment

### Q1: Offload a SAXPY computation to GPU using OpenMP target and compare different map clause strategies

There are three common ways to manage data movement: per-kernel `map`, structured `target data` regions, and unstructured enter/exit data. Each has a different cost profile. This example shows all three on the same SAXPY computation so you can compare them directly.

<!-- compile: needs OpenMP (compile with -fopenmp) -->
```cpp
#include <cstdio>
#include <cstdlib>
#include <chrono>
#include <vector>

void saxpy_map_tofrom(float* y, const float* x, float a, int n) {
    // Full map: copy x to device, y to device and back
    #pragma omp target teams distribute parallel for \
        map(to: x[0:n], a, n) map(tofrom: y[0:n])
    for (int i = 0; i < n; ++i) {
        y[i] = a * x[i] + y[i];
    }
}

void saxpy_data_region(float* y, const float* x, float a, int n) {
    // Persistent data region — data stays on device across kernels
    #pragma omp target data map(to: x[0:n]) map(tofrom: y[0:n])
    {
        // Multiple kernels reuse device data without re-transfer
        #pragma omp target teams distribute parallel for
        for (int i = 0; i < n; ++i) {
            y[i] = a * x[i] + y[i];
        }

        // Second operation on same data — no re-transfer
        #pragma omp target teams distribute parallel for
        for (int i = 0; i < n; ++i) {
            y[i] *= 0.5f;
        }
    }
    // Data mapped back on exit from target data region
}

void saxpy_unstructured(float* y, const float* x, float a, int n) {
    // Unstructured data mapping — fine-grained control
    #pragma omp target enter data map(to: x[0:n], y[0:n])

    #pragma omp target teams distribute parallel for
    for (int i = 0; i < n; ++i) {
        y[i] = a * x[i] + y[i];
    }

    // Update specific sections without full re-transfer
    #pragma omp target update from(y[0:n])

    #pragma omp target exit data map(delete: x[0:n], y[0:n])
}

int main() {
    constexpr int N = 1 << 22;
    std::vector<float> x(N, 1.0f), y(N, 2.0f);

    printf("Devices available: %d\n", omp_get_num_devices());
    printf("Default device: %d\n", omp_get_default_device());

    auto bench = [](const char* label, auto fn) {
        auto t0 = std::chrono::high_resolution_clock::now();
        fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        double ms = std::chrono::duration<double,
                    std::milli>(t1 - t0).count();
        printf("%-30s %8.2f ms\n", label, ms);
    };

    bench("map(tofrom) per kernel:", [&] {
        for (int i = 0; i < N; ++i) y[i] = 2.0f;
        saxpy_map_tofrom(y.data(), x.data(), 2.0f, N);
    });

    bench("target data region:", [&] {
        for (int i = 0; i < N; ++i) y[i] = 2.0f;
        saxpy_data_region(y.data(), x.data(), 2.0f, N);
    });

    bench("unstructured mapping:", [&] {
        for (int i = 0; i < N; ++i) y[i] = 2.0f;
        saxpy_unstructured(y.data(), x.data(), 2.0f, N);
    });

    printf("y[0] = %f (expected ~2.0 after *0.5)\n", y[0]);
    return 0;
}

// Compile (Clang):  clang++ -fopenmp -fopenmp-targets=nvptx64 -O2 saxpy.cpp
// Compile (GCC):    g++ -fopenmp -foffload=nvptx-none -O2 saxpy.cpp
// Compile (nvc++):  nvc++ -mp=gpu -gpu=cc80 -O2 saxpy.cpp
```

The `target data` version is the most efficient of the three when you have multiple kernels operating on the same data: transfers happen once on entry and once on exit, not before and after every kernel. The unstructured version with `enter data`/`exit data` is useful when the allocation and deallocation of device data need to span function boundaries.

### Q2: Use teams, distribute, parallel for, and simd to control GPU execution hierarchy and tune performance

The compiler can choose execution parameters automatically, but explicitly controlling `num_teams` and `num_threads` often gives better results - especially when you know your problem size and target hardware. This example shows progressively more explicit control, from "let the compiler decide" to "fully specified":

```cpp
#include <omp.h>
#include <cstdio>
#include <cmath>
#include <chrono>

// Level 1: Basic offload (compiler chooses everything)
void stencil_basic(float* out, const float* in, int n) {
    #pragma omp target teams distribute parallel for \
        map(to: in[0:n]) map(from: out[0:n])
    for (int i = 1; i < n - 1; ++i) {
        out[i] = 0.25f * (in[i-1] + 2.0f * in[i] + in[i+1]);
    }
}

// Level 2: Explicit team/thread control
void stencil_tuned(float* out, const float* in, int n) {
    #pragma omp target teams num_teams(512) \
        distribute parallel for num_threads(256) \
        map(to: in[0:n]) map(from: out[0:n])
    for (int i = 1; i < n - 1; ++i) {
        out[i] = 0.25f * (in[i-1] + 2.0f * in[i] + in[i+1]);
    }
}

// Level 3: Collapsed 2D loop with simd
void stencil_2d(float* out, const float* in, int nx, int ny) {
    #pragma omp target teams distribute parallel for collapse(2) \
        map(to: in[0:nx*ny]) map(from: out[0:nx*ny])
    for (int i = 1; i < nx - 1; ++i) {
        for (int j = 1; j < ny - 1; ++j) {
            int idx = i * ny + j;
            out[idx] = 0.25f * (
                in[(i-1)*ny + j] + in[(i+1)*ny + j] +
                in[i*ny + (j-1)] + in[i*ny + (j+1)]
            );
        }
    }
}

// Level 4: Reduction on GPU
double gpu_dot(const float* a, const float* b, int n) {
    double result = 0.0;
    #pragma omp target teams distribute parallel for \
        reduction(+:result) map(to: a[0:n], b[0:n])
    for (int i = 0; i < n; ++i) {
        result += static_cast<double>(a[i]) * b[i];
    }
    return result;
}

int main() {
    constexpr int N = 1 << 22;
    auto* in  = new float[N];
    auto* out = new float[N];
    for (int i = 0; i < N; ++i) in[i] = sinf(static_cast<float>(i));

    auto bench = [](const char* lbl, auto fn) {
        auto t0 = std::chrono::high_resolution_clock::now();
        fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        printf("%-25s %8.2f ms\n", lbl,
               std::chrono::duration<double, std::milli>(t1-t0).count());
    };

    bench("Basic offload:", [&]{ stencil_basic(out, in, N); });
    bench("Tuned teams/threads:", [&]{ stencil_tuned(out, in, N); });

    constexpr int NX = 2048, NY = 2048;
    auto* in2  = new float[NX * NY]();
    auto* out2 = new float[NX * NY]();
    bench("2D collapsed:", [&]{ stencil_2d(out2, in2, NX, NY); });

    float* a = new float[N];
    float* b = new float[N];
    for (int i = 0; i < N; ++i) { a[i] = 1.0f; b[i] = 2.0f; }
    double dot = gpu_dot(a, b, N);
    printf("Dot product: %f (expected %f)\n", dot, 2.0 * N);

    delete[] in; delete[] out;
    delete[] in2; delete[] out2;
    delete[] a; delete[] b;
    return 0;
}
```

The `collapse(2)` on the 2D stencil is important - without it, only the outer loop over `i` is distributed across teams, and each team then runs the inner `j` loop sequentially within its threads. With `collapse(2)`, both dimensions are flattened into a single iteration space and partitioned across the full GPU grid.

### Q3: Implement asynchronous offloading with nowait, depend, and task-based offloading for pipeline parallelism

`nowait` lets a `#pragma omp target` region return immediately, allowing the CPU thread to keep working while the GPU computes. The `depend` clause then enforces ordering between async operations - H2D must finish before the compute starts, and compute must finish before D2H can read the results. The combination gives you a software pipeline where multiple chunks overlap in flight:

```cpp
#include <omp.h>
#include <cstdio>
#include <vector>
#include <cmath>

void process_chunk(float* data, int n) {
    // Heavy computation on GPU
    #pragma omp target teams distribute parallel for map(tofrom: data[0:n])
    for (int i = 0; i < n; ++i) {
        float v = data[i];
        for (int j = 0; j < 50; ++j)
            v = sinf(v) + 0.001f;
        data[i] = v;
    }
}

// Pipeline: overlap H2D, compute, and D2H across chunks
void pipeline_async(float* data, int n, int nchunks) {
    int chunk_size = n / nchunks;

    // Allocate on device first
    #pragma omp target enter data map(alloc: data[0:n])

    for (int c = 0; c < nchunks; ++c) {
        int offset = c * chunk_size;

        // Async H2D for this chunk
        #pragma omp target update to(data[offset:chunk_size]) nowait \
            depend(out: data[offset])

        // Async compute — depends on H2D completion
        #pragma omp target teams distribute parallel for \
            nowait depend(inout: data[offset])
        for (int i = offset; i < offset + chunk_size; ++i) {
            float v = data[i];
            for (int j = 0; j < 50; ++j)
                v = sinf(v) + 0.001f;
            data[i] = v;
        }

        // Async D2H — depends on compute completion
        #pragma omp target update from(data[offset:chunk_size]) nowait \
            depend(in: data[offset])
    }

    // Wait for all async operations
    #pragma omp taskwait

    #pragma omp target exit data map(delete: data[0:n])
}

// Task-based heterogeneous execution
void task_offload(float* gpu_data, float* cpu_data, int n) {
    #pragma omp parallel
    #pragma omp single
    {
        // Task 1: GPU computation
        #pragma omp task depend(out: gpu_data[0])
        {
            #pragma omp target teams distribute parallel for \
                map(tofrom: gpu_data[0:n])
            for (int i = 0; i < n; ++i)
                gpu_data[i] = sqrtf(gpu_data[i]);
        }

        // Task 2: CPU computation (runs concurrently with GPU)
        #pragma omp task depend(out: cpu_data[0])
        {
            #pragma omp parallel for
            for (int i = 0; i < n; ++i)
                cpu_data[i] = logf(cpu_data[i] + 1.0f);
        }

        // Task 3: Merge results (depends on both)
        #pragma omp task depend(in: gpu_data[0], cpu_data[0])
        {
            for (int i = 0; i < n; ++i)
                gpu_data[i] += cpu_data[i];
            printf("Merged: gpu_data[0] = %f\n", gpu_data[0]);
        }
    }
}

int main() {
    constexpr int N = 1 << 20;

    // Pipeline test
    std::vector<float> data(N, 1.0f);
    double t0 = omp_get_wtime();
    pipeline_async(data.data(), N, 4);
    double t1 = omp_get_wtime();
    printf("Pipeline async: %.2f ms\n", (t1 - t0) * 1000);

    // Task offload test
    std::vector<float> gpu_d(N, 4.0f), cpu_d(N, 1.0f);
    t0 = omp_get_wtime();
    task_offload(gpu_d.data(), cpu_d.data(), N);
    t1 = omp_get_wtime();
    printf("Task offload: %.2f ms\n", (t1 - t0) * 1000);

    return 0;
}

/*
 Pipeline Execution Timeline:
   Chunk 0: [H2D0][Compute0][D2H0]
   Chunk 1:       [H2D1][Compute1][D2H1]
   Chunk 2:             [H2D2][Compute2][D2H2]
   Chunk 3:                   [H2D3][Compute3][D2H3]

 depend clauses enforce ordering WITHIN a chunk,
 nowait allows overlap BETWEEN chunks.
*/
```

The `task_offload` function at the bottom is worth studying separately. It shows how you can use OpenMP's task dependency system to run GPU and CPU work truly in parallel - task 1 dispatches to the GPU and returns, task 2 runs on CPU threads, and task 3 only starts when both finish. This is heterogeneous parallel programming with no explicit thread management.

---

## Notes

- `OMP_TARGET_OFFLOAD=MANDATORY|DISABLED|DEFAULT` controls offloading behavior - use `MANDATORY` in production to catch missing device errors early rather than silently falling back to CPU.
- `collapse(N)` is essential for 2D/3D loops - without it, only the outer loop is distributed, leaving inner loops sequential on each thread.
- `map(to:)` = H2D only, `map(from:)` = D2H only, `map(tofrom:)` = both, `map(alloc:)` = device alloc without transfer.
- `#pragma omp target data` creates a persistent device allocation - use it to eliminate redundant transfers across multiple kernels.
- Reduction on GPU (`reduction(+:var)`) is fully supported in OpenMP 5.0 - the compiler generates an efficient tree reduction.
- Always compile with `-fopenmp-targets=nvptx64` (Clang) or `-foffload=nvptx-none` (GCC) - without explicit target, OpenMP falls back to CPU silently.
- Profile with `LIBOMPTARGET_INFO=1` environment variable to see data transfer and kernel launch details.
