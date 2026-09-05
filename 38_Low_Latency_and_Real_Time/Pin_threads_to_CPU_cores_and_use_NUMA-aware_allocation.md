# Pin Threads to CPU Cores and Use NUMA-Aware Allocation

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20 (with POSIX / Win32 extensions)  
**Reference:** [Linux `sched_setaffinity(2)`](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html), [NUMA API](https://man7.org/linux/man-pages/man3/numa.3.html)  

---

## Topic Overview

Modern CPUs are multi-core, multi-socket machines with Non-Uniform Memory Access (NUMA) topology: each socket has local DRAM that it can access in roughly 80ns, while accessing remote DRAM costs 140-200ns. The OS scheduler freely migrates threads between cores, which causes cache invalidation, cross-NUMA memory access, and latency spikes. **CPU pinning** (affinity) locks a thread to a specific core, eliminating migration overhead.

On Linux, `pthread_setaffinity_np` sets which cores a thread may run on via a `cpu_set_t` bitmask. On Windows, `SetThreadAffinityMask` provides the same functionality. For complete isolation, `isolcpus=` in the kernel command line removes cores from the general scheduler, reserving them exclusively for pinned threads with zero interference from kernel tasks and other processes.

NUMA-aware allocation ensures that memory is allocated on the same NUMA node as the core that will access it. `libnuma` provides `numa_alloc_onnode` for explicit placement. Without NUMA awareness, a thread pinned to socket 0 may access memory that the OS allocated on socket 1's DRAM, adding 60-100ns per cache miss - devastating for latency-critical paths. The reason this trips people up is that affinity alone is not enough: you also have to tell the allocator *where* to put the memory, not just tell the scheduler where to run your thread.

| Technique | Linux API | Windows API | Effect |
| --- | --- | --- | --- |
| Pin thread to core | `pthread_setaffinity_np` | `SetThreadAffinityMask` | No migration, warm cache |
| Isolate cores | `isolcpus=` boot param | Processor group affinity | Zero interference |
| NUMA-local alloc | `numa_alloc_onnode` | `VirtualAllocExNuma` | Local DRAM access |
| Query topology | `numa_node_of_cpu` | `GetNumaProcessorNode` | Map core -> NUMA node |
| Interleave alloc | `numa_alloc_interleaved` | N/A | Balance bandwidth |

Here is a diagram showing what the two-socket situation looks like. The key takeaway is that if Thread A runs on Socket 0 but its buffer was allocated on Socket 1's DRAM, every cache miss crosses the QPI/UPI interconnect and pays the full remote latency penalty:

```cpp
NUMA TOPOLOGY (2-socket system):
┌──────────────────────┐     QPI/UPI     ┌──────────────────────┐
│  Socket 0            │◄──────────────►│  Socket 1            │
│  Cores 0-7           │   ~140ns       │  Cores 8-15          │
│  Local DRAM (~80ns)  │                │  Local DRAM (~80ns)  │
│  L3 Cache            │                │  L3 Cache            │
└──────────────────────┘                └──────────────────────┘
    Thread A pinned here                    Thread B pinned here
    Memory allocated here                   Memory allocated here
```

---

## Self-Assessment

### Q1: Write a cross-platform `ThreadPinner` class that pins a `std::thread` to a specific core, with NUMA-aware memory allocation for the thread's working set

The interesting thing here is not just the affinity call - it's that the buffer is allocated on the correct NUMA node *before* the thread starts, so the thread never touches remote memory:

```cpp
#ifdef __linux__
#include <pthread.h>
#include <sched.h>
#include <numa.h>
#include <numaif.h>
#elif _WIN32
#include <windows.h>
#endif

#include <thread>
#include <cstdio>
#include <cstdlib>
#include <cassert>
#include <stdexcept>
#include <memory>

class ThreadPinner {
public:
    static void pin_this_thread(int core_id) {
#ifdef __linux__
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        CPU_SET(core_id, &cpuset);
        int rc = pthread_setaffinity_np(pthread_self(),
                                         sizeof(cpu_set_t), &cpuset);
        if (rc != 0) throw std::runtime_error("Failed to set affinity");
#elif _WIN32
        DWORD_PTR mask = 1ULL << core_id;
        if (!SetThreadAffinityMask(GetCurrentThread(), mask))
            throw std::runtime_error("SetThreadAffinityMask failed");
#endif
    }

    static int get_numa_node(int core_id) {
#ifdef __linux__
        return numa_node_of_cpu(core_id);
#elif _WIN32
        UCHAR node;
        GetNumaProcessorNode(static_cast<UCHAR>(core_id), &node);
        return static_cast<int>(node);
#endif
    }

    // Allocate memory on the NUMA node local to the given core
    static void* alloc_on_core_node(std::size_t bytes, int core_id) {
#ifdef __linux__
        int node = numa_node_of_cpu(core_id);
        void* p = numa_alloc_onnode(bytes, node);
        if (!p) throw std::bad_alloc();
        return p;
#elif _WIN32
        UCHAR node;
        GetNumaProcessorNode(static_cast<UCHAR>(core_id), &node);
        void* p = VirtualAllocExNuma(GetCurrentProcess(), nullptr, bytes,
                       MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE, node);
        if (!p) throw std::bad_alloc();
        return p;
#endif
    }

    static void free_numa(void* ptr, [[maybe_unused]] std::size_t bytes) {
#ifdef __linux__
        numa_free(ptr, bytes);
#elif _WIN32
        VirtualFree(ptr, 0, MEM_RELEASE);
#endif
    }
};

int main() {
    constexpr int TARGET_CORE = 2;
    constexpr std::size_t BUF_SIZE = 1024 * 1024;  // 1 MB

    // Allocate buffer on the NUMA node of core 2
    void* buf = ThreadPinner::alloc_on_core_node(BUF_SIZE, TARGET_CORE);

    std::thread worker([buf, BUF_SIZE] {
        ThreadPinner::pin_this_thread(TARGET_CORE);
        std::printf("Thread pinned to core %d (NUMA node %d)\n",
                    TARGET_CORE, ThreadPinner::get_numa_node(TARGET_CORE));

        // Work on NUMA-local buffer - all accesses are local
        volatile char* p = static_cast<volatile char*>(buf);
        for (std::size_t i = 0; i < BUF_SIZE; ++i)
            p[i] = static_cast<char>(i);

        std::printf("Processed %zu bytes on local NUMA node\n", BUF_SIZE);
    });

    worker.join();
    ThreadPinner::free_numa(buf, BUF_SIZE);
}
```

Notice that both the affinity pin and the buffer allocation target the same core. The thread writes 1MB and every byte stays in local DRAM.

### Q2: Query the system's NUMA topology at startup and assign each worker thread to a core with its local memory, printing the topology map

This is how you discover the topology programmatically and then lay out your workers so each one starts life on its home node. In production you'd do this once at startup and cache the result:

```cpp
#ifdef __linux__
#include <numa.h>
#include <sched.h>
#include <pthread.h>
#endif

#include <thread>
#include <vector>
#include <map>
#include <cstdio>
#include <cassert>
#include <functional>

struct CoreInfo {
    int core_id;
    int numa_node;
};

std::vector<CoreInfo> discover_topology() {
    std::vector<CoreInfo> cores;
#ifdef __linux__
    if (numa_available() < 0) {
        std::fprintf(stderr, "NUMA not available\n");
        return cores;
    }
    int max_cpus = numa_num_configured_cpus();
    for (int c = 0; c < max_cpus; ++c) {
        int node = numa_node_of_cpu(c);
        if (node >= 0)
            cores.push_back({c, node});
    }
#endif
    return cores;
}

void print_topology(const std::vector<CoreInfo>& cores) {
    std::map<int, std::vector<int>> node_to_cores;
    for (auto& ci : cores)
        node_to_cores[ci.numa_node].push_back(ci.core_id);

    std::printf("┌─────────────────────────────────────┐\n");
    std::printf("│        NUMA Topology                │\n");
    std::printf("├──────────┬──────────────────────────┤\n");
    std::printf("│ Node     │ Cores                    │\n");
    std::printf("├──────────┼──────────────────────────┤\n");
    for (auto& [node, cvec] : node_to_cores) {
        std::printf("│ Node %-3d │ ", node);
        for (int c : cvec) std::printf("%d ", c);
        std::printf("\n");
    }
    std::printf("└──────────┴──────────────────────────┘\n");
}

void worker_function(int core_id, int numa_node) {
#ifdef __linux__
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
#endif
    std::printf("Worker on core %d (NUMA node %d) running\n",
                core_id, numa_node);
    // Real-time work here...
}

int main() {
    auto topology = discover_topology();
    print_topology(topology);

    // Assign one worker per core on NUMA node 0
    std::vector<std::thread> workers;
    for (auto& ci : topology) {
        if (ci.numa_node == 0) {
            workers.emplace_back(worker_function, ci.core_id, ci.numa_node);
        }
    }
    for (auto& w : workers) w.join();
}
```

The `discover_topology` function queries libnuma to build the full core-to-node map, then `main` filters for node 0 and launches one worker per core there. You'd replicate this pattern per node on a larger machine.

### Q3: Benchmark local vs remote NUMA memory access latency to demonstrate the cost of incorrect placement

This benchmark pins the main thread to core 0 (node 0), allocates one buffer on node 0 and another on node 1, then measures how long it takes to touch each. The numbers will show the remote penalty directly. If you've never run this before, the result is usually a genuine surprise:

<!-- compile: needs POSIX header `sched.h` -->
```cpp
#ifdef __linux__
#include <numa.h>
#endif
#include <chrono>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <thread>
#include <sched.h>
#include <pthread.h>

constexpr std::size_t kSize = 64 * 1024 * 1024;  // 64 MB
constexpr int kIterations = 10;

double benchmark_access(volatile char* buf, std::size_t size) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int iter = 0; iter < kIterations; ++iter) {
        for (std::size_t i = 0; i < size; i += 64) {  // cache line stride
            buf[i] = static_cast<char>(buf[i] + 1);
        }
    }
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
#ifdef __linux__
    if (numa_available() < 0 || numa_max_node() < 1) {
        std::printf("Need >= 2 NUMA nodes for this benchmark\n");
        return 1;
    }

    // Pin to core 0 (node 0)
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(0, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);

    // Local allocation (node 0)
    void* local_buf = numa_alloc_onnode(kSize, 0);
    memset(local_buf, 0, kSize);

    // Remote allocation (node 1)
    void* remote_buf = numa_alloc_onnode(kSize, 1);
    memset(remote_buf, 0, kSize);

    double local_ms  = benchmark_access(static_cast<volatile char*>(local_buf), kSize);
    double remote_ms = benchmark_access(static_cast<volatile char*>(remote_buf), kSize);

    std::printf("┌──────────┬────────────┬──────────────┐\n");
    std::printf("│ Location │ Time (ms)  │ Slowdown     │\n");
    std::printf("├──────────┼────────────┼──────────────┤\n");
    std::printf("│ Local    │ %8.2f   │ 1.00x        │\n", local_ms);
    std::printf("│ Remote   │ %8.2f   │ %.2fx        │\n",
                remote_ms, remote_ms / local_ms);
    std::printf("└──────────┴────────────┴──────────────┘\n");

    numa_free(local_buf, kSize);
    numa_free(remote_buf, kSize);
#endif
}
```

The slowdown column is the honest answer to "does NUMA placement matter?" - typical results are 1.5-2x slower for the remote buffer on a bandwidth-bound loop, and worse for pointer-chasing patterns.

---

## Notes

- **`isolcpus=2,3,4,5`** on the kernel command line is essential for true isolation - affinity alone doesn't prevent the scheduler from placing other tasks on your core.
- Use `taskset -c 2 ./my_app` to pin an entire process; use `pthread_setaffinity_np` for per-thread pinning.
- **Hyperthreading** (SMT) siblings share execution units; pin latency-critical threads to physical cores only, and disable HT on the sibling core.
- **`numactl --membind=0 --cpunodebind=0 ./app`** is a quick way to test NUMA-local execution without code changes.
- Check placement with `numastat -p <pid>` - look for `other_node` hits indicating remote access.
- Remote NUMA access is typically 1.5-2x slower for bandwidth-bound workloads.
