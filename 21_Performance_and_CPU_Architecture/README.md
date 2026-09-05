# Performance & CPU Architecture

Cache optimization, SIMD, branch prediction, memory layout, and performance measurement.

**Topics:** 39

## Contents

- [Annotate hot and cold functions with [[gnu::hot]] and [[gnu::cold]]](Annotate_hot_and_cold_functions_with_gnuhot_and_gnucold.md)
- [Apply struct packing and padding to minimize memory footprint](Apply_struct_packing_and_padding_to_minimize_memory_footprint.md)
- [Apply whole-program optimization and devirtualization with LTO](Apply_whole-program_optimization_and_devirtualization_with_LTO.md)
- [Profile and optimize memory allocation patterns with tcmalloc, jemalloc, mimalloc](Profile_and_optimize_memory_allocation_patterns_with_tcmalloc_jemalloc_mimalloc.md)
- [Understand AoSoA (Array of Structures of Arrays) for SIMD-friendly layout](Understand_AoSoA_Array_of_Structures_of_Arrays_for_SIMD-friendly_layout.md)
- [Understand branch-free programming techniques for hot paths](Understand_branch-free_programming_techniques_for_hot_paths.md)
- [Understand cache-oblivious algorithms and recursive tiling](Understand_cache-oblivious_algorithms_and_recursive_tiling.md)
- [Understand cache-oblivious algorithms and recursive tiling (Part 2: Transpose)](Understand_cache-oblivious_algorithms_and_recursive_tiling_2.md)
- [Understand CPU branch prediction and how code structure affects it](Understand_CPU_branch_prediction_and_how_code_structure_affects_it.md)
- [Understand CPU branch prediction and how code structure affects it (Part 2: Pipeline Cost)](Understand_CPU_branch_prediction_and_how_code_structure_affects_it_2.md)
- [Understand CPU branch prediction and write branch-predictor-friendly code](Understand_CPU_branch_prediction_and_write_branch-predictor-friendly_code.md)
- [Understand devirtualization and how to help the compiler eliminate virtual calls](Understand_devirtualization_and_how_to_help_the_compiler_eliminate_virtual_calls.md)
- [Understand false sharing and cache line bouncing in multi-threaded code](Understand_false_sharing_and_cache_line_bouncing_in_multi-threaded_code.md)
- [Understand instruction-level parallelism (ILP) and out-of-order execution](Understand_instruction-level_parallelism_ILP_and_out-of-order_execution.md)
- [Understand loop unrolling and control it with pragmas and attributes](Understand_loop_unrolling_and_control_it_with_pragmas_and_attributes.md)
- [Understand loop vectorization and how to write vectorization-friendly code](Understand_loop_vectorization_and_how_to_write_vectorization-friendly_code.md)
- [Understand memory bandwidth limits and arithmetic intensity (roofline model)](Understand_memory_bandwidth_limits_and_arithmetic_intensity_roofline_model.md)
- [Understand micro-architectural bottlenecks: port pressure, ROB size, store buffers](Understand_micro-architectural_bottlenecks_port_pressure_ROB_size_store_buffers.md)
- [Understand the cost model of virtual dispatch in tight loops](Understand_the_cost_model_of_virtual_dispatch_in_tight_loops.md)
- [Understand TLB pressure and huge pages for large working sets](Understand_TLB_pressure_and_huge_pages_for_large_working_sets.md)
- [Use __builtin_prefetch and std::prefetch (C++26) to hide memory latency](Use___builtin_prefetch_and_stdprefetch_C++26_to_hide_memory_latency.md)
- [Use __builtin_prefetch to hide memory latency in hot loops](Use___builtin_prefetch_to_hide_memory_latency_in_hot_loops.md)
- [Use cache-oblivious algorithms and tiling for memory-bound code](Use_cache-oblivious_algorithms_and_tiling_for_memory-bound_code.md)
- [Use compile-time CPU feature detection with __builtin_cpu_supports](Use_compile-time_CPU_feature_detection_with___builtin_cpu_supports.md)
- [Use hot/cold function splitting to improve instruction cache density](Use_hotcold_function_splitting_to_improve_instruction_cache_density.md)
- [Use hot/cold function splitting with [[gnu::hot]] and [[gnu::cold]]](Use_hotcold_function_splitting_with_gnuhot_and_gnucold.md)
- [Use Intel VTune and AMD uProf for micro-architectural profiling](Use_Intel_VTune_and_AMD_uProf_for_micro-architectural_profiling.md)
- [Use loop vectorization hints and verify SIMD code generation](Use_loop_vectorization_hints_and_verify_SIMD_code_generation.md)
- [Use memory prefetching with __builtin_prefetch to hide memory latency](Use_memory_prefetching_with___builtin_prefetch_to_hide_memory_latency.md)
- [Use non-temporal stores for write-only large buffers to bypass cache](Use_non-temporal_stores_for_write-only_large_buffers_to_bypass_cache.md)
- [Use perf annotate and flamegraphs for source-level hotspot analysis](Use_perf_annotate_and_flamegraphs_for_source-level_hotspot_analysis.md)
- [Use perf annotate and flamegraphs to understand CPU time distribution](Use_perf_annotate_and_flamegraphs_to_understand_CPU_time_distribution.md)
- [Use SIMD intrinsics (SSE/AVX) for explicit vectorisation](Use_SIMD_intrinsics_SSEAVX_for_explicit_vectorisation.md)
- [Use SIMD intrinsics for explicit vectorization with SSE/AVX](Use_SIMD_intrinsics_for_explicit_vectorization_with_SSEAVX.md)
- [Use std::experimental::simd (C++ SIMD library) for portable vectorization](Use_stdexperimentalsimd_C++_SIMD_library_for_portable_vectorization.md)
- [Use std::experimental::simd (Parallelism TS 2) for portable SIMD](Use_stdexperimentalsimd_Parallelism_TS_2_for_portable_SIMD.md)
- [Use the Highway library for portable SIMD without platform ifdefs](Use_the_Highway_library_for_portable_SIMD_without_platform_ifdefs.md)
- [Write branchless code using conditional moves and arithmetic tricks](Write_branchless_code_using_conditional_moves_and_arithmetic_tricks.md)
- [Write SIMD-vectorisable loops and verify auto-vectorisation](Write_SIMD-vectorisable_loops_and_verify_auto-vectorisation.md)

## Notes

- Cache locality dominates performance - `std::vector` often beats `std::list` even for insertions.
- Branch misprediction costs ~15 cycles - sort data or use branchless techniques in hot paths.
- SIMD intrinsics (SSE, AVX, NEON) can deliver 4-16x throughput for data-parallel operations.
- `[[likely]]`/`[[unlikely]]` (C++20) hint the compiler about branch prediction.
- False sharing occurs when threads write to variables on the same cache line - align to 64 bytes.
- Profile before optimizing - use `perf`, VTune, or Tracy to find actual bottlenecks.
- Link-Time Optimization (LTO) enables cross-translation-unit inlining and dead code elimination.
- Small Buffer Optimization (SBO) in `std::string`, `std::function` avoids heap allocation for small objects.
- Prefetch instructions (`__builtin_prefetch`) hint the CPU to load data before it is needed.
- Memory alignment (`alignas`) matters for SIMD operations and can impact cache efficiency.
