# GPU & Heterogeneous Computing

C++ for GPU and accelerator programming: CUDA, SYCL/oneAPI, OpenCL, std::execution GPU backends, data transfer optimization, unified memory, GPU-friendly data structures, and offloading with OpenMP.

**Topics:** 13

## Contents

- [Bridge std::execution with GPU Backends](Bridge_stdexecution_with_GPU_backends.md)
- [Design GPU-Friendly Data Structures for Coalesced Access](Design_GPU-friendly_data_structures_for_coalesced_access.md)
- [Implement CPU-GPU task graphs for heterogeneous pipelines](Implement_CPU-GPU_task_graphs_for_heterogeneous_pipelines.md)
- [Integrate CUDA with Modern C++ for GPU Kernel Development](Integrate_CUDA_with_modern_C++_for_GPU_kernel_development.md)
- [Offload Computation with OpenMP Target Directives](Offload_computation_with_OpenMP_target_directives.md)
- [Optimize Host-Device Data Transfer Patterns](Optimize_host-device_data_transfer_patterns.md)
- [Profile GPU kernels with NSight and ROCProfiler](Profile_GPU_kernels_with_NSight_and_ROCProfiler.md)
- [Understand GPU memory hierarchy: registers, shared memory, L1, L2, global](Understand_GPU_memory_hierarchy_registers_shared_memory_L1_L2_global.md)
- [Use CUDA Cooperative Groups for Flexible Thread Synchronization](Use_CUDA_Cooperative_Groups_for_flexible_thread_synchronization.md)
- [Use OpenCL C++ Bindings for Vendor-Neutral GPU Programming](Use_OpenCL_C++_bindings_for_vendor-neutral_GPU_programming.md)
- [Use SYCL and oneAPI for Portable Heterogeneous Computing](Use_SYCL_and_oneAPI_for_portable_heterogeneous_computing.md)
- [Use Unified Shared Memory for Simplified GPU Programming](Use_unified_shared_memory_for_simplified_GPU_programming.md)
- [Use Vulkan Compute for portable GPU computing from C++](Use_Vulkan_Compute_for_portable_GPU_computing_from_Cpp.md)

## Notes

- CUDA extends C++ for GPU programming - kernel functions run on thousands of GPU threads simultaneously.
- SYCL provides standard C++ heterogeneous computing - single-source GPU code without language extensions.
- GPU memory management (device vs host) is the primary complexity - use unified memory for prototyping, but understand its performance cost before shipping.
- Occupancy optimization (balancing threads, registers, shared memory) is key to GPU performance - more threads isn't always better.
- std::execution (C++26) aims to unify CPU and GPU execution under one programming model, letting you swap schedulers rather than rewrite algorithms.
- GPU workloads suit data-parallel operations - matrix math, image processing, simulations. If your problem isn't data-parallel, GPU gains can be minimal.
- Memory coalescing (adjacent threads accessing adjacent memory) is critical for GPU throughput - this is why data layout decisions matter so much on the GPU.
- Shared memory is fast but limited (~48KB per block) - use it for data reuse within thread blocks, and treat it like a programmer-managed cache.
- Vulkan Compute and OpenCL are cross-vendor GPU compute APIs - less C++-friendly than CUDA/SYCL, but vendor-neutral.
- Profile GPU code with NSight (NVIDIA) or RGP (AMD) - GPU bottlenecks differ from CPU code, and intuition from CPU profiling rarely transfers.
