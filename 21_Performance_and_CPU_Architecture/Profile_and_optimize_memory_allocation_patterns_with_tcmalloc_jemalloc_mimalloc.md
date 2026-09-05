# Profile and optimize memory allocation patterns with tcmalloc, jemalloc, mimalloc

**Category:** Performance and CPU Architecture  
**Standard:** C++17 (allocator-agnostic)  
**Reference:** <https://google.github.io/tcmalloc/> <https://jemalloc.net/> <https://microsoft.github.io/mimalloc/>  

---

## Topic Overview

### Why the Default Allocator Isn't Always Enough

The default `malloc`/`new` implementation (glibc's ptmalloc2 on Linux, MSVC's allocator on Windows) is a solid general-purpose allocator. For most programs it is perfectly fine. But for high-performance multi-threaded applications it can become a genuine bottleneck, not because of bad code on your part, but because of structural properties of how ptmalloc works:

- **Lock contention** in multi-threaded allocation/deallocation. All threads share an allocator lock, so threads serialise on every `new` and `delete`.
- **Fragmentation** causing poor cache utilization. Over time the heap develops holes that prevent tight packing of live objects.
- **Large working set** from metadata overhead. Each allocation carries bookkeeping that inflates memory usage.

The three alternatives below were designed to fix specific subsets of these problems.

### Allocator Comparison

| Allocator | Developer | Best For | Key Feature |
| --- | --- | --- | --- |
| **tcmalloc** | Google | Multi-threaded servers | Per-thread caches, huge page support |
| **jemalloc** | Meta/FreeBSD | General high-perf | Arena-based, low fragmentation |
| **mimalloc** | Microsoft | Latency-sensitive | Segment-based, free list sharding |
| **ptmalloc2** | glibc | General purpose | Default on Linux |

### Drop-In Replacement

All three can replace the system allocator without changing a single line of your C++ code. `LD_PRELOAD` intercepts the dynamic linker and reroutes all `malloc`/`free` calls at the OS level, so `operator new` and `operator delete` follow along automatically.

```bash
# tcmalloc
LD_PRELOAD=/usr/lib/libtcmalloc.so ./my_app

# jemalloc
LD_PRELOAD=/usr/lib/libjemalloc.so ./my_app

# mimalloc
LD_PRELOAD=/usr/lib/libmimalloc.so ./my_app

# On Windows with mimalloc: link against mimalloc-override.lib
```

This is the fastest way to answer the question "would a different allocator help my program?" - no recompile, no code changes, just a timing comparison.

### CMake Integration

Once you have chosen an allocator for production, link it explicitly rather than relying on `LD_PRELOAD`, which is fragile and environment-dependent.

```cmake
# tcmalloc via find_package or FetchContent
find_package(tcmalloc REQUIRED)
target_link_libraries(myapp PRIVATE tcmalloc::tcmalloc)

# jemalloc
find_package(PkgConfig REQUIRED)
pkg_check_modules(JEMALLOC REQUIRED jemalloc)
target_link_libraries(myapp PRIVATE ${JEMALLOC_LIBRARIES})

# mimalloc
find_package(mimalloc REQUIRED)
target_link_libraries(myapp PRIVATE mimalloc)
```

### Profiling Allocation Patterns

Before swapping allocators, it is worth understanding where your program allocates. Both jemalloc and tcmalloc ship built-in heap profilers that produce call-stack-annotated reports.

**With jemalloc's built-in profiler:**

```bash
# Enable heap profiling
MALLOC_CONF="prof:true,prof_prefix:jeprof" ./my_app

# Analyze with jeprof
jeprof --svg ./my_app jeprof.*.heap > heap_profile.svg
```

**With tcmalloc's heap profiler:**

<!-- compile: needs third-party library header `gperftools/heap-profiler.h` -->
```cpp
#include <gperftools/heap-profiler.h>

int main() {
    HeapProfilerStart("my_profile");

    // ... application code ...
    // Periodic dumps:
    HeapProfilerDump("checkpoint1");

    HeapProfilerStop();
}
```

```bash
# Convert to visualization
pprof --svg ./my_app my_profile.0001.heap > heap.svg
```

**With `perf` for allocation hotspots:**

```bash
perf record -g -e probe:malloc ./my_app
perf report --sort=symbol,dso
```

### tcmalloc: Per-Thread Caching

The key insight behind tcmalloc is that most allocation traffic in a multi-threaded program consists of small objects that a thread allocates and then frees from the same thread. If you keep a per-thread freelist for those sizes, you never need a lock:

```cpp
Thread 1: [local cache] -> [central freelist] -> [page heap]
Thread 2: [local cache] -> [central freelist] -> [page heap]
Thread 3: [local cache] -> [central freelist] -> [page heap]
```

Small allocations (<256KB) are served entirely from the thread-local cache - zero locking.

### jemalloc: Arena-Based Allocation

jemalloc uses multiple independent arenas, each with size-class bins. Threads are pinned to arenas in round-robin fashion, which spreads contention across all cores rather than serializing on a single lock:

```cpp
Arena 0: [bins: 8B, 16B, 32B, ... 14KB] [large: 14KB-4MB] [huge: >4MB]
Arena 1: [bins: 8B, 16B, 32B, ... 14KB] [large: 14KB-4MB] [huge: >4MB]
...
```

Threads are distributed across arenas, reducing contention.

### mimalloc: Free List Sharding

mimalloc shards free lists per page per size class, achieving excellent cache locality. It also exposes optional C APIs for aligned allocation and heap isolation:

```cpp
// mimalloc-specific APIs (optional - also works as drop-in)
#include <mimalloc.h>

void* p = mi_malloc(1024);
mi_free(p);

// Aligned allocation
void* aligned = mi_malloc_aligned(4096, 64);
mi_free(aligned);
```

---

## Self-Assessment

### Q1: How do you benchmark different allocators

The simplest and most reliable experiment is to time your own workload under each allocator with `LD_PRELOAD`. No synthetic benchmark can substitute for this.

```bash
# Run the same workload with different allocators and compare:
time LD_PRELOAD="" ./my_app                          # default
time LD_PRELOAD=/usr/lib/libtcmalloc.so ./my_app     # tcmalloc
time LD_PRELOAD=/usr/lib/libjemalloc.so ./my_app      # jemalloc
time LD_PRELOAD=/usr/lib/libmimalloc.so ./my_app      # mimalloc

# Also measure:
# - RSS (resident set size): ps -o rss
# - Allocation rate: perf stat -e malloc calls
# - Lock contention: perf stat -e context-switches
```

Watch RSS alongside wall time. An allocator that is faster but uses significantly more memory may not be a net win on a memory-constrained machine.

### Q2: When should you prefer jemalloc over tcmalloc

These three questions capture the situations where each one tends to win:

- **jemalloc**: lower fragmentation for long-running services with diverse allocation sizes (databases, web servers).
- **tcmalloc**: better for workloads with many small same-size allocations from many threads (RPC servers, microservices).
- **mimalloc**: best latency characteristics for interactive/real-time applications.

### Q3: How can you profile per-allocation-site memory usage

Both jemalloc and tcmalloc can produce allocation call stacks with byte counts, which lets you answer "which line of my code is responsible for the most heap memory?"

With jemalloc:

```bash
MALLOC_CONF="prof:true,lg_prof_interval:30,prof_prefix:heap" ./my_app
jeprof --text ./my_app heap.*.heap
```

With tcmalloc:

```bash
HEAPPROFILE=/tmp/hp ./my_app
pprof --text --alloc_space ./my_app /tmp/hp.0001.heap
```

Both show allocation call stacks with byte counts, allowing you to identify the hottest allocation sites.

---

## Notes

- Always benchmark with **your** workload - allocator performance varies dramatically by access pattern.
- `LD_PRELOAD` is the easiest way to test - no recompilation needed.
- For C++ specifically, `operator new`/`delete` go through `malloc`/`free`, so replacing `malloc` replaces all standard allocations.
- Consider combining a global allocator (tcmalloc/jemalloc) with local arena allocators (`std::pmr`) for hot paths.
