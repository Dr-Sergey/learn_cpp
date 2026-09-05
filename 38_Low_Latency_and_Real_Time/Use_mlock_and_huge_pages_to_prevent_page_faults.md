# Use mlock and Huge Pages to Prevent Page Faults

**Category:** Low Latency and Real Time  
**Standard:** C++17  
**Reference:** [mlock(2)](https://man7.org/linux/man-pages/man2/mlock.2.html), [hugetlbfs](https://www.kernel.org/doc/html/latest/admin-guide/mm/hugetlbpage.html)  

---

## Topic Overview

A **page fault** occurs when a virtual address has no backing physical page. Minor faults (page is in memory but not mapped) cost 1-10µs; major faults (page must be read from disk) cost 1-10ms. In a real-time hot path, even a single minor fault creates an unacceptable latency spike. The OS defaults to **demand paging** - physical pages are allocated only on first access - which means any newly touched memory triggers a fault.

The reason this trips people up is that demand paging is invisible until you measure for it. Your code looks fine in testing, but the first time a thread touches a page it hasn't seen before, the OS handler runs, maps physical memory, and adds microseconds of latency that looks like random noise in your p99 measurements.

**`mlock()`** and **`mlockall()`** pin pages in physical RAM, preventing them from being swapped out. This eliminates major faults but not minor faults on first access - you must also **pre-fault** by writing to every page. `mlockall(MCL_CURRENT | MCL_FUTURE)` locks all current and future mappings, which is the strongest guarantee for real-time code.

**Huge pages** (2MB or 1GB vs the standard 4KB) reduce TLB misses by covering more address space per TLB entry. The TLB has only 64-1024 entries; with 4KB pages, this covers 256KB-4MB of address space total - with 2MB pages, it covers 128MB-2GB. For data-heavy applications, TLB misses alone can add 5-50ns per access. Linux offers **Transparent Huge Pages (THP)**, which the kernel manages automatically, and **explicit hugetlbfs**, which the application controls directly. In real-time code, THP is usually a trap: the kernel's background compaction work to create 2MB-contiguous regions causes latency spikes at unpredictable times.

| Feature | 4KB Pages | 2MB Huge Pages | 1GB Huge Pages |
| --- | --- | --- | --- |
| TLB coverage (512 entries) | 2 MB | 1 GB | 512 GB |
| TLB miss cost | ~10 ns per miss | Same per miss, fewer misses | Fewest misses |
| Page fault cost | 1-10 µs | 1-10 µs (per huge page) | 1-10 µs (per huge page) |
| Fragmentation | Fine-grained | Requires contiguous 2MB | Requires contiguous 1GB |
| Setup | Automatic | `mmap(MAP_HUGETLB)` or THP | Boot-time reservation |

Here is the concrete TLB math for a sequential 64MB access pattern. With 4KB pages you need over 16,000 TLB entries, but the hardware only has 512, so you're thrashing the TLB on every page boundary. With 2MB pages, 32 entries cover the entire region with room to spare:

```cpp
TLB MISS IMPACT:
                                    4KB pages     2MB pages
Access pattern: sequential 64MB
TLB entries needed:                 16,384        32
TLB capacity (512 entries):         2 MB cover    1 GB cover
TLB misses:                         ~15,872       0
Added latency @ 10ns/miss:          ~159 µs       0 µs
```

---

## Self-Assessment

### Q1: Write a memory manager that allocates from 2MB huge pages, pre-faults all pages, and `mlock`s them - with diagnostics showing page fault counts

The key sequence here is: map -> pre-fault -> lock. If you lock first and then pre-fault you still pay the fault cost, but at least the pages won't get swapped. The `prefault()` method forces every page to take its minor fault now, in the cold path, so the hot path never sees one:

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <sys/mman.h>
#include <sys/resource.h>
#include <cerrno>

class HugePageAllocator {
    void* base_ = nullptr;
    std::size_t size_ = 0;
    std::size_t offset_ = 0;
    bool locked_ = false;

public:
    // Allocate from explicit 2MB huge pages
    bool init(std::size_t total_bytes) {
        // Round up to 2MB boundary
        constexpr std::size_t kHugePageSize = 2 * 1024 * 1024;
        size_ = (total_bytes + kHugePageSize - 1) & ~(kHugePageSize - 1);

        base_ = mmap(nullptr, size_,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                     -1, 0);

        if (base_ == MAP_FAILED) {
            // Fallback: try regular pages with madvise(MADV_HUGEPAGE)
            base_ = mmap(nullptr, size_,
                         PROT_READ | PROT_WRITE,
                         MAP_PRIVATE | MAP_ANONYMOUS,
                         -1, 0);
            if (base_ == MAP_FAILED) {
                std::perror("mmap failed");
                return false;
            }
            madvise(base_, size_, MADV_HUGEPAGE);  // request THP
            std::printf("Using transparent huge pages (THP fallback)\n");
        } else {
            std::printf("Using explicit 2MB huge pages\n");
        }

        // Pre-fault: write every page to force physical mapping
        prefault();

        // Lock pages in RAM
        if (mlock(base_, size_) != 0) {
            std::perror("mlock failed (need CAP_IPC_LOCK)");
        } else {
            locked_ = true;
        }

        return true;
    }

    void prefault() {
        volatile char* p = static_cast<volatile char*>(base_);
        for (std::size_t i = 0; i < size_; i += 4096) {
            p[i] = 0;  // trigger minor fault NOW, not during hot path
        }
    }

    // Bump allocator: O(1), no fragmentation
    void* allocate(std::size_t bytes, std::size_t alignment = 64) {
        std::size_t aligned = (offset_ + alignment - 1) & ~(alignment - 1);
        if (aligned + bytes > size_) return nullptr;
        void* ptr = static_cast<char*>(base_) + aligned;
        offset_ = aligned + bytes;
        return ptr;
    }

    void reset() { offset_ = 0; }

    ~HugePageAllocator() {
        if (base_ && base_ != MAP_FAILED) {
            if (locked_) munlock(base_, size_);
            munmap(base_, size_);
        }
    }
};

long get_minor_faults() {
    struct rusage u;
    getrusage(RUSAGE_SELF, &u);
    return u.ru_minflt;
}

int main() {
    HugePageAllocator alloc;
    if (!alloc.init(64 * 1024 * 1024)) return 1;  // 64 MB

    long faults_before = get_minor_faults();

    // Hot path: allocate from pre-faulted huge pages
    double* data = static_cast<double*>(alloc.allocate(8 * 1024 * 1024));
    for (int i = 0; i < 1'000'000; ++i)
        data[i] = i * 1.001;

    long faults_after = get_minor_faults();
    std::printf("Page faults during hot path: %ld\n",
                faults_after - faults_before);
    std::printf("Expected: 0 (all pre-faulted and locked)\n");
}
```

The bump allocator (`allocate`) is just a pointer increment - O(1) and deterministic. No fragmentation, no bookkeeping. It's the right allocator for a pre-allocated working set.

### Q2: Compare TLB miss rates between 4KB pages and 2MB huge pages when scanning a large array, using `perf`-compatible measurement

This benchmark allocates 256MB with regular pages and 256MB with huge pages, then scans both to measure the time difference. The huge page version skips most TLB misses because 32 TLB entries cover the whole array instead of needing 65,536. Run the binary under `perf stat -e dTLB-load-misses` to see the raw miss counts:

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <sys/mman.h>
#include <chrono>

constexpr std::size_t kSize = 256 * 1024 * 1024;  // 256 MB
constexpr int kIterations = 5;

void* alloc_regular(std::size_t size) {
    void* p = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (p == MAP_FAILED) return nullptr;
    memset(p, 0, size);  // pre-fault
    return p;
}

void* alloc_huge(std::size_t size) {
    void* p = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
    if (p == MAP_FAILED) {
        // THP fallback
        p = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
        if (p == MAP_FAILED) return nullptr;
        madvise(p, size, MADV_HUGEPAGE);
    }
    memset(p, 0, size);
    return p;
}

double benchmark_scan(volatile char* data, std::size_t size) {
    auto t0 = std::chrono::high_resolution_clock::now();
    uint64_t sum = 0;
    for (int iter = 0; iter < kIterations; ++iter) {
        for (std::size_t i = 0; i < size; i += 4096) {  // touch each page
            sum += data[i];
        }
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    (void)sum;
    return std::chrono::duration<double, std::milli>(t1 - t0).count();
}

int main() {
    void* regular = alloc_regular(kSize);
    void* huge = alloc_huge(kSize);

    if (!regular || !huge) {
        std::fprintf(stderr, "Allocation failed\n");
        return 1;
    }

    double reg_ms = benchmark_scan(static_cast<volatile char*>(regular), kSize);
    double huge_ms = benchmark_scan(static_cast<volatile char*>(huge), kSize);

    std::printf("┌────────────────┬────────────┬────────────────────┐\n");
    std::printf("│ Page Size      │ Time (ms)  │ Pages Touched      │\n");
    std::printf("├────────────────┼────────────┼────────────────────┤\n");
    std::printf("│ 4 KB (regular) │ %8.2f   │ %zu              │\n",
                reg_ms, kSize / 4096);
    std::printf("│ 2 MB (huge)    │ %8.2f   │ %zu                │\n",
                huge_ms, kSize / (2 * 1024 * 1024));
    std::printf("└────────────────┴────────────┴────────────────────┘\n");
    std::printf("Speedup: %.2fx\n", reg_ms / huge_ms);
    std::printf("\nRun with: perf stat -e dTLB-load-misses ./binary\n");

    munmap(regular, kSize);
    munmap(huge, kSize);
}
```

The speedup tends to be more dramatic on access patterns with poor spatial locality, or when the working set exceeds what the hardware page-walk cache can cover.

### Q3: Implement `mlockall` at program startup with stack pre-growth, and monitor that no faults occur during the main loop

This is the full real-time startup recipe. The tricky part is the stack: `mlockall(MCL_CURRENT)` locks memory that is already mapped, but the stack grows lazily as functions are called. If your real-time loop calls a function with a large stack frame for the first time, it triggers a stack-extension fault. The `prefault_stack()` function forces the OS to map the stack pages right now:

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <sys/mman.h>
#include <sys/resource.h>
#include <signal.h>
#include <unistd.h>
#include <cerrno>

// Fault counter: set by signal handler if any fault occurs (debugging aid)
static volatile sig_atomic_t unexpected_fault = 0;

void sigsegv_handler(int /*sig*/) {
    unexpected_fault = 1;  // should never fire after setup
}

// Pre-grow the stack to avoid stack-extension faults
void prefault_stack() {
    constexpr std::size_t kStackSize = 8 * 1024 * 1024;  // 8 MB
    volatile char dummy[kStackSize];
    memset(const_cast<char*>(dummy), 0, sizeof(dummy));
    (void)dummy;  // ensure not optimized away
}

long get_faults() {
    struct rusage u;
    getrusage(RUSAGE_SELF, &u);
    return u.ru_minflt;
}

struct RTSetup {
    bool success = false;

    RTSetup() {
        // 1. Lock all current and future pages
        if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
            std::fprintf(stderr, "mlockall failed: %s "
                        "(run with CAP_IPC_LOCK)\n", strerror(errno));
            return;
        }

        // 2. Pre-fault the stack
        prefault_stack();

        // 3. Pre-fault the heap (pre-allocate a known working set)
        constexpr std::size_t kHeapSize = 32 * 1024 * 1024;
        void* heap = std::malloc(kHeapSize);
        if (heap) {
            memset(heap, 0, kHeapSize);
            std::free(heap);  // pages remain locked due to MCL_FUTURE
        }

        success = true;
        std::printf("RT setup complete: all pages locked and pre-faulted\n");
    }

    ~RTSetup() {
        munlockall();
    }
};

void realtime_loop(int iterations) {
    long faults_start = get_faults();

    volatile double accumulator = 0;
    for (int i = 0; i < iterations; ++i) {
        accumulator += i * 0.001;  // pure computation, no allocation
    }

    long faults_end = get_faults();
    std::printf("RT loop faults: %ld (target: 0)\n",
                faults_end - faults_start);
}

int main() {
    RTSetup rt;
    if (!rt.success) {
        std::fprintf(stderr, "Failed to set up RT environment\n");
        return 1;
    }

    // Report pre-setup faults (expected: many during init)
    std::printf("Total faults after setup: %ld\n", get_faults());

    // Real-time loop: should have ZERO faults
    realtime_loop(10'000'000);

    return 0;
}
```

The `MCL_FUTURE` flag in `mlockall` is the interesting piece: it means every `mmap` or `malloc` call made after this point will also have its pages locked immediately. The faults still happen (you can't escape demand paging entirely), but they happen during the cold path where you have `memset`-ing them explicitly, not in the hot loop.

---

## Notes

- **`mlockall(MCL_FUTURE)`** is aggressive: every future `mmap`, `malloc`, and library load will be locked. Monitor RSS with `cat /proc/<pid>/status | grep VmLck`.
- **Huge page reservation**: `echo 1024 > /proc/sys/vm/nr_hugepages` must be done at boot or when memory is unfragmented.
- **THP (Transparent Huge Pages)** can cause latency spikes due to **compaction** - the kernel defragments memory to create 2MB contiguous regions. Disable with `echo never > /sys/kernel/mm/transparent_hugepage/enabled` and use explicit hugetlbfs instead.
- `mmap` with `MAP_POPULATE` pre-faults automatically but doesn't guarantee huge pages.
- Monitor with `cat /proc/<pid>/smaps | grep -i huge` to verify huge page usage.
- **`perf stat -e dTLB-load-misses,dTLB-store-misses,page-faults`** is the key command for measuring the impact.
