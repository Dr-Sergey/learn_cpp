# Implement CPU-GPU task graphs for heterogeneous pipelines

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17, C++20  
**Reference:** <https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#cuda-graphs>  

---

## Topic Overview

### The Problem: Launch Overhead

Individual kernel launches have overhead (~5-20 µs each on CUDA). For pipelines with many small kernels, this launch overhead dominates execution time - the GPU spends more time waiting for the next command than actually computing. The contrast looks like this:

```cpp
Traditional approach (serial launches):
CPU: [launch A][wait][launch B][wait][launch C][wait][launch D]
GPU: [idle][A][idle][B][idle][C][idle][D]
     <- wasted time: kernel launch overhead + synchronization

Task graph approach:
CPU: [define graph][launch graph]
GPU: [A -> B -> C -> D]  <- minimal idle time, pipelined
```

With a task graph, you define the entire workflow once, hand it to the driver as a unit, and then fire it off with a single launch command. The driver knows all the dependencies upfront and can optimize scheduling internally.

### CUDA Graphs - Hardware-Accelerated Task Graphs

CUDA's stream capture API lets you record a sequence of operations - including kernels, memory copies, and synchronization - by running them normally on a stream while capture mode is active. The result is a graph object that can be launched repeatedly with very low overhead. Here is the pattern:

```cpp
#include <cuda_runtime.h>

void build_and_run_graph() {
    cudaGraph_t graph;
    cudaGraphExec_t instance;

    // Capture a stream into a graph
    cudaStream_t stream;
    cudaStreamCreate(&stream);

    cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);
    {
        // These calls are recorded, NOT executed
        kernel_preprocess<<<grid1, block>>>(d_input, d_temp, n);
        kernel_compute<<<grid2, block>>>(d_temp, d_output, n);
        cudaMemcpyAsync(d_feedback, d_output, size,
                        cudaMemcpyDeviceToDevice, stream);
        kernel_postprocess<<<grid3, block>>>(d_output, d_result, n);
    }
    cudaStreamEndCapture(stream, &graph);

    // Instantiate — validates and optimizes the graph
    cudaGraphInstantiate(&instance, graph, nullptr, nullptr, 0);

    // Execute — much lower overhead than individual launches
    for (int frame = 0; frame < 1000; ++frame) {
        cudaGraphLaunch(instance, stream);
    }
    cudaStreamSynchronize(stream);

    // Cleanup
    cudaGraphExecDestroy(instance);
    cudaGraphDestroy(graph);
    cudaStreamDestroy(stream);
}
```

The instantiation step is where the driver does its optimization work - it validates the graph structure and may reorder or merge operations. You pay that cost once, then amortize it across all subsequent launches.

### Explicit Graph Construction with Dependencies

Stream capture is convenient but limited to linear or simple fork-join patterns. When you need more complex dependency structures - like two independent branches that both depend on a shared predecessor - explicit construction gives you full control. You add nodes and then declare which nodes must complete before each node starts:

```cpp
void build_explicit_graph() {
    cudaGraph_t graph;
    cudaGraphCreate(&graph, 0);

    // Define nodes
    cudaGraphNode_t nodeA, nodeB, nodeC, nodeD;

    // Node A: H2D transfer
    cudaMemcpy3DParms memcpy_params = {};
    // ... fill params ...
    cudaGraphAddMemcpyNode(&nodeA, graph, nullptr, 0, &memcpy_params);

    // Node B: Kernel (depends on A)
    cudaKernelNodeParams kernel_params = {};
    kernel_params.func = (void*)process_kernel;
    kernel_params.gridDim = dim3(grid_size);
    kernel_params.blockDim = dim3(block_size);
    void* args[] = {&d_data, &n};
    kernel_params.kernelParams = args;

    cudaGraphNode_t deps_B[] = {nodeA};
    cudaGraphAddKernelNode(&nodeB, graph, deps_B, 1, &kernel_params);

    // Node C: Another kernel (depends on A, independent of B)
    cudaGraphNode_t deps_C[] = {nodeA};
    cudaGraphAddKernelNode(&nodeC, graph, deps_C, 1, &kernel_params2);

    // Node D: Reduction (depends on B AND C)
    cudaGraphNode_t deps_D[] = {nodeB, nodeC};
    cudaGraphAddKernelNode(&nodeD, graph, deps_D, 2, &reduce_params);

    // Graph structure:
    //      A
    //     / \
    //    B   C    <- B and C run in parallel
    //     \ /
    //      D

    cudaGraphExec_t instance;
    cudaGraphInstantiate(&instance, graph, nullptr, nullptr, 0);
    cudaGraphLaunch(instance, stream);
}
```

Notice that B and C can execute in parallel because neither depends on the other - only both depend on A. The driver handles that overlap automatically once you've declared the dependency edges.

### C++ Task Graph with CPU+GPU Nodes

CUDA graphs only cover GPU-side operations. For heterogeneous pipelines that mix CPU work (file I/O, decompression, analysis) with GPU compute, a framework like **Taskflow** lets you express both kinds of tasks in a single dependency graph. This is where you get true CPU/GPU overlap without managing threads and streams manually:

```cpp
#include <taskflow/taskflow.hpp>
#include <taskflow/cuda/cudaflow.hpp>

void heterogeneous_pipeline() {
    tf::Executor executor;
    tf::Taskflow taskflow;

    // CPU task: load and preprocess data
    auto load = taskflow.emplace([&]() {
        load_data_from_disk(host_buffer);
    }).name("CPU: Load Data");

    // GPU task: process on GPU
    auto gpu_process = taskflow.emplace([&](tf::cudaFlowCapturer& cap) {
        auto h2d = cap.memcpy(d_input, host_buffer, size);
        auto kernel = cap.on([&](cudaStream_t stream) {
            process<<<grid, block, 0, stream>>>(d_input, d_output, n);
        });
        auto d2h = cap.memcpy(host_result, d_output, result_size);

        h2d.precede(kernel);
        kernel.precede(d2h);
    }).name("GPU: Process");

    // CPU task: post-process results
    auto analyze = taskflow.emplace([&]() {
        analyze_results(host_result);
    }).name("CPU: Analyze");

    // Dependencies: load -> gpu_process -> analyze
    load.precede(gpu_process);
    gpu_process.precede(analyze);

    executor.run(taskflow).wait();
}
```

The `precede` calls build the dependency graph. Taskflow's executor then runs tasks as soon as their dependencies are satisfied, on whatever CPU threads are available, while the GPU tasks are dispatched to their streams.

### Pipelined Multi-Frame Processing

Double-buffering (or the more general "pipeline with N lines") is a classic technique for keeping both CPU and GPU busy simultaneously. While the GPU processes frame N, the CPU loads frame N+1 into the other buffer. Taskflow's pipeline API makes this pattern clean:

```cpp
void pipelined_processing() {
    tf::Executor executor;
    tf::Taskflow taskflow;

    // Pipeline: 3 stages, 2 lines (double-buffered)
    tf::Pipeline pipeline(/*num_lines=*/2,
        // Stage 1: CPU — load frame
        tf::Pipe(tf::PipeType::SERIAL, [&](tf::Pipeflow& pf) {
            if (pf.token() >= total_frames) {
                pf.stop();
                return;
            }
            load_frame(pf.token(), buffers[pf.line()]);
        }),

        // Stage 2: GPU — process frame
        tf::Pipe(tf::PipeType::PARALLEL, [&](tf::Pipeflow& pf) {
            gpu_process(buffers[pf.line()], results[pf.line()]);
        }),

        // Stage 3: CPU — save results
        tf::Pipe(tf::PipeType::SERIAL, [&](tf::Pipeflow& pf) {
            save_results(pf.token(), results[pf.line()]);
        })
    );

    taskflow.composed_of(pipeline);
    executor.run(taskflow).wait();

    // Timeline:
    // Line 0: [Load F0][Process F0][Save F0]         [Load F2][Process F2][Save F2]
    // Line 1:          [Load F1]   [Process F1][Save F1]         [Load F3] ...
    //                  <- overlapped execution ->
}
```

The key insight here is that CPU and GPU are separate hardware units that can genuinely work at the same time. A pipeline fills both processors' idle time. Without double-buffering, the GPU sits idle while the CPU loads the next frame, and the CPU sits idle while the GPU computes. With it, you get near-full utilization of both.

### Graph Update Without Rebuild

One of the practical advantages of CUDA graphs is that you can update kernel parameters between launches without destroying and rebuilding the graph. This is useful when you have a fixed computation structure but varying input data (different pointers, different constants each frame):

```cpp
// Update kernel parameters without destroying/recreating the graph
void update_graph_iteration(cudaGraphExec_t instance,
                            cudaGraphNode_t kernel_node,
                            float new_param) {
    cudaKernelNodeParams params;
    cudaGraphExecKernelNodeSetParams(instance, kernel_node, &params);
    // Updated parameters take effect on next launch — no rebuild overhead
}
```

---

## Self-Assessment

### Q1: Why are CUDA Graphs faster than individual kernel launches

Individual kernel launches require CPU-GPU communication each time (~5-20 µs per launch). CUDA Graphs capture the entire workflow (kernels, copies, dependencies) once, then replay it with a single launch command. The driver optimizes the graph internally, overlapping independent operations and minimizing CPU-GPU round trips. For pipelines with many small kernels, this can improve throughput by 2-5x.

### Q2: When should you use stream capture vs explicit graph construction

**Stream capture** (`cudaStreamBeginCapture`) is simpler - you write normal CUDA code and it records the operations. Good for straightforward linear pipelines. **Explicit construction** (`cudaGraphAddKernelNode`) gives more control - you can create complex DAGs with branches and joins that are hard to express with stream ordering. Use explicit construction when you need fork-join parallelism or conditional nodes.

### Q3: How does a heterogeneous pipeline overlap CPU and GPU work

Using double-buffering or multi-line pipelines: while the GPU processes frame N, the CPU loads frame N+1 and saves frame N-1. A task graph framework (like Taskflow) manages dependencies automatically. The key insight: CPU and GPU are separate hardware that can work simultaneously - the pipeline fills both processors' idle time.

---

## Notes

- CUDA Graphs have an instantiation cost - amortize it over many launches.
- Use `cudaGraphExecUpdate` to modify a graph without full re-instantiation.
- Conditional nodes (CUDA 12+) allow dynamic graph behavior.
- Taskflow is header-only and supports CUDA, SYCL, and CPU tasks.
- Profile with `nsys` to verify that graph launches have lower overhead than individual launches.
