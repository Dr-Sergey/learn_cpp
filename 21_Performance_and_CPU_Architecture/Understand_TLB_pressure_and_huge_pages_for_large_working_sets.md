# Understand TLB pressure and huge pages for large working sets

**Category:** Performance and CPU Architecture  
**Standard:** C++17  
**Reference:** <https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html>  

---

## Topic Overview

Every memory access your program makes goes through address translation: the CPU maps a virtual address to a physical one by walking a multi-level page table in RAM. That walk can cost dozens to hundreds of cycles. The **Translation Lookaside Buffer (TLB)** is a small hardware cache that stores recently used virtual-to-physical mappings so most accesses can skip the walk entirely.

The problem is that TLBs are tiny. A typical L2 TLB holds about 1536 entries. With the default 4 KB page size, 1536 entries cover only 6 MB of memory. If your program randomly accesses a 1 GB working set, 262,144 pages are needed - far more than the TLB can hold - so almost every random access triggers a page walk.

Huge pages solve this by trading granularity for coverage. A 2 MB page covers 512x more address space per TLB entry, so those same 1536 TLB entries now cover 3 GB. A 1 GB page covers 262,144x more.

```cpp
4KB pages (default):                2MB huge pages:
  Working set: 1 GB                   Working set: 1 GB
  Pages: 262,144                      Pages: 512
  TLB entries: ~1,536                 TLB entries: ~1,536
  Coverage: 6 MB                      Coverage: 3 GB
  -> constant TLB misses!             -> everything fits in TLB!

TLB miss penalty: ~7-100 cycles (page walk through 4 levels)
```

| Page size | TLB entries (typ.) | Coverage | Use case |
| --- | --- | --- | --- |
| 4 KB | 1536 (L1+L2) | ~6 MB | Default, small programs |
| 2 MB | 32 (dedicated) | 64 MB | Large arrays, databases |
| 1 GB | 4 (dedicated) | 4 GB | HPC, in-memory DBs |

---

## Self-Assessment

### Q1: How TLB misses add latency

The page walk that a TLB miss triggers is itself a series of memory accesses - one for each level of the page table. In the best case those page table entries are cached in the CPU's data cache. In the worst case they are in DRAM, and you pay a full DRAM latency at each level.

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <random>
#include <numeric>

// TLB miss triggers a PAGE WALK:
// 1. CPU reads PML4 table entry    (~4 cycles, L1 hit)
// 2. CPU reads PDPT entry           (~4 cycles)
// 3. CPU reads Page Directory entry  (~4 cycles)
// 4. CPU reads Page Table entry      (~4 cycles if cached, ~200 if DRAM)
// Total: 7-100+ cycles per TLB miss
//
// x86-64 TLB hierarchy (Intel Skylake):
// L1 dTLB: 64 entries (4KB), 32 entries (2MB)
// L2 sTLB: 1536 entries (4KB/2MB shared)

int main() {
    // Demonstrate TLB pressure with random access across many pages
    std::vector<size_t> sizes = {
        1ULL << 20,    // 1 MB = 256 pages
        16ULL << 20,   // 16 MB = 4096 pages (exceeds TLB)
        256ULL << 20,  // 256 MB = 65536 pages
        1ULL << 30     // 1 GB = 262144 pages
    };

    for (size_t bytes : sizes) {
        size_t n = bytes / sizeof(int);
        std::vector<int> data(n, 1);

        // Random access pattern: touches many pages
        std::mt19937 rng(42);
        std::vector<size_t> indices(1'000'000);
        for (auto& idx : indices) idx = rng() % n;

        volatile long long sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (size_t idx : indices) {
            sum += data[idx];  // random access -> TLB miss likely
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        double ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count()
                    / 1'000'000.0;

        std::cout << (bytes >> 20) << " MB: " << ns << " ns/access\n";
    }
    // Expected:
    //   1 MB:   ~5 ns   (fits in TLB, L1/L2 dTLB covers 256 pages)
    //  16 MB:  ~15 ns   (exceeds TLB, page walks start)
    // 256 MB:  ~30 ns   (heavy TLB misses + L3 cache misses)
    //   1 GB:  ~80 ns   (constant TLB misses + DRAM access)
}
```

Notice the step from 1 MB to 16 MB: latency jumps from 5 ns to 15 ns. That 3x increase has nothing to do with cache misses - the L3 cache is large enough to hold 16 MB. It is purely TLB pressure. Once you cross the TLB coverage boundary, random accesses start triggering page walks on almost every iteration.

### Q2: Enable huge pages with `madvise`

On Linux you can request huge pages for an existing allocation using `madvise`. The kernel will then promote the underlying 4 KB pages to 2 MB pages when the physical memory is available, transparently reducing the number of TLB entries you need.

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <chrono>
#include <random>
#include <sys/mman.h>  // Linux only

// Allocate with huge page support on Linux:
void* alloc_huge(size_t bytes) {
    // Method 1: mmap + madvise (Transparent Huge Pages)
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (ptr == MAP_FAILED) return nullptr;
    madvise(ptr, bytes, MADV_HUGEPAGE);  // request THP
    return ptr;
}

void* alloc_normal(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (ptr == MAP_FAILED) return nullptr;
    madvise(ptr, bytes, MADV_NOHUGEPAGE);  // force 4KB pages
    return ptr;
}

int main() {
    constexpr size_t SIZE = 256ULL << 20; // 256 MB
    constexpr int ITERS = 10'000'000;

    auto bench = [&](void* mem, const char* label) {
        int* data = static_cast<int*>(mem);
        size_t n = SIZE / sizeof(int);
        std::memset(data, 1, SIZE); // fault all pages

        std::mt19937 rng(42);
        volatile long long sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < ITERS; ++i) {
            sum += data[rng() % n];
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        double ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count()
                    / static_cast<double>(ITERS);
        std::cout << label << ": " << ns << " ns/access\n";
    };

    void* normal = alloc_normal(SIZE);
    void* huge = alloc_huge(SIZE);

    if (normal && huge) {
        bench(normal, "4KB pages  ");
        bench(huge,   "2MB pages  ");
        // Typical (256MB, random access):
        // 4KB pages: ~30 ns/access
        // 2MB pages: ~20 ns/access (33% faster, fewer TLB misses)
        munmap(normal, SIZE);
        munmap(huge, SIZE);
    }

    // Measure with perf:
    //   perf stat -e dTLB-load-misses,dTLB-store-misses ./test
    //   4KB: dTLB-load-misses = 5,000,000
    //   2MB: dTLB-load-misses =    50,000  (100x reduction!)
}
```

The `std::memset` before the benchmark is important: it forces the kernel to actually allocate and fault in the physical pages. Without it the pages might not exist yet and the benchmark would measure page-fault time rather than TLB behavior.

### Q3: THP vs explicit `MAP_HUGETLB`

There are two distinct mechanisms for getting huge pages on Linux, and they have different trade-offs. Transparent Huge Pages (THP) are managed automatically by the kernel; explicit huge pages via `MAP_HUGETLB` require pre-reserved memory but give you a firm guarantee.

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <iostream>
#include <sys/mman.h>
#include <cstring>

// Method 1: Transparent Huge Pages (THP)
// - OS automatically promotes 4KB pages to 2MB when possible
// - No special privileges needed
// - May not always use huge pages (fragmentation)
void* alloc_thp(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    madvise(ptr, bytes, MADV_HUGEPAGE);
    return ptr;
    // Pros: easy, automatic, works with malloc if THP enabled
    // Cons: may silently fall back to 4KB, allocation latency spikes
}

// Method 2: Explicit huge pages (MAP_HUGETLB)
// - Requires preallocated huge page pool:
//   echo 128 > /proc/sys/vm/nr_hugepages  (reserves 128 * 2MB = 256MB)
// - Guaranteed 2MB pages (or allocation fails)
void* alloc_explicit_huge(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
    // Returns MAP_FAILED if huge pages not available
    return ptr;
    // Pros: guaranteed huge pages, no fragmentation issues
    // Cons: needs root/privileges, wastes memory if partially used
}

// Method 3: 1GB huge pages (for very large allocations)
void* alloc_1gb_huge(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB
                     | (30 << MAP_HUGE_SHIFT), -1, 0);
    // 30 = log2(1GB), requires boot-time reservation:
    //   hugepagesz=1G hugepages=4 on kernel cmdline
    return ptr;
}

int main() {
    // Comparison table:
    std::cout << "Feature comparison:\n";
    std::cout << "+------------------+-------------------+-------------------+\n";
    std::cout << "| Feature          | THP               | MAP_HUGETLB       |\n";
    std::cout << "+------------------+-------------------+-------------------+\n";
    std::cout << "| Page size        | 2MB (auto)        | 2MB or 1GB        |\n";
    std::cout << "| Guaranteed       | No (best effort)  | Yes               |\n";
    std::cout << "| Privileges       | None              | CAP_IPC_LOCK      |\n";
    std::cout << "| Fragmentation    | May fail silently  | Pre-reserved      |\n";
    std::cout << "| Allocation speed | Slow (compaction) | Fast (reserved)   |\n";
    std::cout << "| Swap support     | Yes               | No                |\n";
    std::cout << "| Use case         | General purpose   | HPC, databases    |\n";
    std::cout << "+------------------+-------------------+-------------------+\n";

    // For C++ allocators:
    //   Boost.Interprocess provides huge page allocators
    //   jemalloc: MALLOC_CONF="thp:always" enables THP
    //   tcmalloc: set TCMALLOC_MEMFS_MALLOC_PATH=/dev/hugepages
}
```

The practical advice is: start with THP (`MADV_HUGEPAGE`) because it requires no system configuration and degrades gracefully. Switch to `MAP_HUGETLB` only if you need a hard guarantee that huge pages are in use - for example, in a production database where latency variance is unacceptable.

---

## Notes

- TLB miss cost: ~7 cycles (L2 TLB hit) to ~100+ cycles (full page walk to DRAM).
- `perf stat -e dTLB-load-misses` measures TLB miss rate.
- For databases (large hash tables): huge pages can reduce query latency by 10-30%.
- Linux: check THP status with `cat /sys/kernel/mm/transparent_hugepage/enabled`.
- Windows: `VirtualAlloc` with `MEM_LARGE_PAGES` (requires `SeLockMemoryPrivilege`).
