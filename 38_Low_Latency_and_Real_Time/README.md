# Low Latency & Real-Time C++

Techniques for ultra-low latency and real-time C++ systems: lock-free data structures, allocation avoidance, deterministic execution, kernel bypass networking, CPU pinning, NUMA, cache management, and tail latency measurement.

**Topics:** 16

## Contents

- [Avoid Memory Allocation in Hot Paths](Avoid_memory_allocation_in_hot_paths.md)
- [Choose Between Busy-Spin and Sleep-Based Waiting](Choose_between_busy-spin_and_sleep-based_waiting.md)
- [Configure Real-Time Scheduling (SCHED_FIFO) for C++ Threads](Configure_real-time_scheduling_SCHED_FIFO_for_C++_threads.md)
- [Design Lock-Free and Wait-Free Data Structures](Design_lock-free_and_wait-free_data_structures.md)
- [Eliminate Virtual Dispatch in Hot Paths with Compile-Time Dispatch](Eliminate_virtual_dispatch_in_hot_paths_with_compile-time_dispatch.md)
- [Implement adaptive spinning strategies: spin-then-yield-then-sleep](Implement_adaptive_spinning_strategies_spin-then-yield-then-sleep.md)
- [Implement shared-memory IPC for inter-process low-latency communication](Implement_shared-memory_IPC_for_inter-process_low-latency_communication.md)
- [Know the impact of system calls on latency and how to minimize them](Know_the_impact_of_system_calls_on_latency_and_how_to_minimize_them.md)
- [Measure Tail Latency (p99/p99.9) Accurately](Measure_tail_latency_p99_p999_accurately.md)
- [Pin Threads to CPU Cores and Use NUMA-Aware Allocation](Pin_threads_to_CPU_cores_and_use_NUMA-aware_allocation.md)
- [Use Cache Warming and Prefetching in Latency-Critical Code](Use_cache_warming_and_prefetching_in_latency_critical_code.md)
- [Use io_uring for low-latency IO without syscall overhead](Use_io_uring_for_low-latency_IO_without_syscall_overhead.md)
- [Use isolcpus and nohz_full kernel parameters for latency isolation](Use_isolcpus_and_nohz_full_kernel_parameters_for_latency_isolation.md)
- [Use Kernel Bypass Networking (DPDK) for Ultra-Low Latency](Use_kernel_bypass_networking_DPDK_for_ultra-low_latency.md)
- [Use mlock and Huge Pages to Prevent Page Faults](Use_mlock_and_huge_pages_to_prevent_page_faults.md)
- [Write Deterministic Execution Paths for Real-Time Code](Write_deterministic_execution_paths_for_real-time_code.md)

## Notes

- Lock-free data structures avoid mutex overhead, but are extremely hard to implement correctly.
- Memory pools and arena allocators provide O(1) deterministic allocation, which is essential for real-time work.
- Avoid allocations, exceptions, and system calls in latency-critical paths - each one is a potential stall.
- CPU pinning (`sched_setaffinity`, `SetThreadAffinityMask`) ensures threads don't migrate between cores and trash their caches.
- Busy-waiting (spin loops) reduces wake-up latency compared to sleeping on a mutex, at the cost of a dedicated CPU core.
- Huge pages (2MB/1GB) reduce TLB misses in memory-intensive workloads, sometimes dramatically.
- Kernel bypass (DPDK, RDMA) eliminates syscall overhead for network-intensive applications.
- `CLOCK_MONOTONIC` / `QueryPerformanceCounter` provide high-resolution timing without NTP jumps.
- Pre-fault memory (`mlock` + page touch) prevents page faults from appearing in the hot path.
- Profile with hardware counters (cache misses, branch mispredictions), not just wall-clock time - the counters tell you *why* you're slow.
