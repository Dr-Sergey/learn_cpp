# Implement shared-memory IPC for inter-process low-latency communication

**Category:** Low Latency and Real Time  
**Standard:** C++17, C++20  
**Reference:** <https://man7.org/linux/man-pages/man7/shm_overview.7.html>  

---

## Topic Overview

### Why Shared Memory

Traditional IPC mechanisms add latency through syscalls and data copies. Every time you use a socket or pipe, your data has to travel from your process's memory into the kernel, then back out into the other process's memory. That's two copies and at least two syscall transitions per message.

| IPC Method | Latency | Data copies | Syscalls per message |
| --- | --- | --- | --- |
| TCP loopback | ~10-50 µs | 2+ (user->kernel->user) | 2+ (send/recv) |
| Unix domain socket | ~2-10 µs | 2 (user->kernel->user) | 2 (send/recv) |
| Pipe | ~2-5 µs | 2 (user->kernel->user) | 2 (write/read) |
| **Shared memory** | **~50-200 ns** | **0** (same physical memory) | **0** (after setup) |

Shared memory is 10-100x faster because processes read/write the same physical memory pages directly. The kernel maps the same physical pages into both processes' virtual address spaces. After the initial `mmap` call (which is setup work, not hot path), the processes communicate at the speed of memory accesses - no kernel transitions required.

### POSIX Shared Memory Setup

The RAII wrapper handles the setup and teardown of the shared memory region. The `owner_` flag distinguishes between the process that created the segment (which should clean it up) and processes that merely opened it.

```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>
#include <stdexcept>
#include <string>

class SharedMemoryRegion {
    void* addr_ = MAP_FAILED;
    size_t size_;
    std::string name_;
    bool owner_;  // Creator vs. opener

public:
    // Create or open shared memory
    SharedMemoryRegion(const std::string& name, size_t size, bool create)
        : size_(size), name_(name), owner_(create) {

        int flags = O_RDWR;
        if (create) flags |= O_CREAT | O_EXCL;

        int fd = shm_open(name.c_str(), flags, 0666);
        if (fd < 0) {
            throw std::runtime_error("shm_open failed: " + name);
        }

        if (create) {
            ftruncate(fd, size);
        }

        addr_ = mmap(nullptr, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
        close(fd);  // Can close fd after mmap

        if (addr_ == MAP_FAILED) {
            throw std::runtime_error("mmap failed");
        }

        if (create) {
            std::memset(addr_, 0, size);  // Zero-initialize
        }

        // Lock pages to prevent swapping
        mlock(addr_, size);
    }

    ~SharedMemoryRegion() {
        if (addr_ != MAP_FAILED) {
            munmap(addr_, size_);
        }
        if (owner_) {
            shm_unlink(name_.c_str());
        }
    }

    SharedMemoryRegion(const SharedMemoryRegion&) = delete;
    SharedMemoryRegion& operator=(const SharedMemoryRegion&) = delete;

    void* data() { return addr_; }
    const void* data() const { return addr_; }
    size_t size() const { return size_; }

    template<typename T>
    T* as() { return static_cast<T*>(addr_); }
};
```

The `close(fd)` after `mmap` is intentional - once the mapping is established, the file descriptor is no longer needed and holding it open would waste a system resource.

### Lock-Free SPSC Queue in Shared Memory

The queue structure lives entirely in shared memory. There are important constraints here: it can't contain virtual functions (vtable pointers would be invalid in the other process's address space), can't contain `std::string` or `std::vector` (which contain heap pointers), and must be trivially copyable so it makes sense as raw bytes.

```cpp
#include <atomic>
#include <cstdint>
#include <new>

// This struct lives ENTIRELY in shared memory
// Must be trivially copyable, no pointers, no virtual functions
template<typename T, size_t Capacity>
struct SharedSPSCQueue {
    static_assert(std::is_trivially_copyable_v<T>,
                  "Shared memory types must be trivially copyable");
    static_assert((Capacity & (Capacity - 1)) == 0,
                  "Capacity must be power of 2");

    alignas(64) std::atomic<uint64_t> head{0};  // Written by consumer
    alignas(64) std::atomic<uint64_t> tail{0};  // Written by producer
    alignas(64) T buffer[Capacity];

    bool push(const T& item) {
        uint64_t t = tail.load(std::memory_order_relaxed);
        uint64_t next = t + 1;

        if (next - head.load(std::memory_order_acquire) >= Capacity) {
            return false;  // Full
        }

        buffer[t & (Capacity - 1)] = item;
        tail.store(next, std::memory_order_release);
        return true;
    }

    bool pop(T& item) {
        uint64_t h = head.load(std::memory_order_relaxed);

        if (h >= tail.load(std::memory_order_acquire)) {
            return false;  // Empty
        }

        item = buffer[h & (Capacity - 1)];
        head.store(h + 1, std::memory_order_release);
        return true;
    }
};
```

The `Capacity & (Capacity - 1) == 0` constraint enforces a power-of-two size, enabling the modulo operation to be replaced with a bitwise AND (`t & (Capacity - 1)`). That's a single instruction versus a division, which matters in a hot path.

### Producer Process

The producer creates the shared memory segment, constructs the queue in it using placement new, and then writes into it continuously.

```cpp
struct MarketData {
    uint64_t timestamp;
    uint32_t instrument_id;
    double bid;
    double ask;
    uint32_t bid_size;
    uint32_t ask_size;
};

using Queue = SharedSPSCQueue<MarketData, 65536>;  // 64K entries

int main() {
    // Create shared memory (producer is the owner)
    SharedMemoryRegion shm("/market_data_feed", sizeof(Queue), /*create=*/true);

    // Construct the queue in shared memory using placement new
    auto* queue = new (shm.data()) Queue{};

    // Feed loop
    while (true) {
        MarketData md = receive_from_exchange();
        md.timestamp = __rdtsc();  // Hardware timestamp

        while (!queue->push(md)) {
            // Queue full — consumer is too slow
            // In production: drop oldest or signal backpressure
        }
    }
}
```

The `__rdtsc()` timestamp is a hardware cycle counter. It's extremely cheap (~5ns) and gives you precise end-to-end latency measurements when the consumer reads it on the other side.

### Consumer Process

The consumer opens the existing shared memory segment (not creating it), casts the raw memory to the queue type, and polls for new messages.

```cpp
int main() {
    // Open existing shared memory (consumer is not the owner)
    SharedMemoryRegion shm("/market_data_feed", sizeof(Queue), /*create=*/false);

    auto* queue = static_cast<Queue*>(shm.data());

    MarketData md;
    while (true) {
        if (queue->pop(md)) {
            uint64_t now = __rdtsc();
            uint64_t latency_cycles = now - md.timestamp;
            // Typical: 100-500 cycles (~30-150 ns at 3 GHz)

            process_market_data(md);
        } else {
            _mm_pause();  // Spin wait — tunable
        }
    }
}
```

The `_mm_pause()` in the idle branch is important - without it, the consumer would hammer the cache line containing `tail`, generating constant write traffic that slows down the producer.

### Windows Shared Memory

The Windows API for shared memory uses `CreateFileMapping` / `MapViewOfFile`. The semantics are the same as POSIX - after the mapping is established, both processes access the same physical memory directly.

```cpp
#ifdef _WIN32
#include <windows.h>

class SharedMemoryRegion {
    HANDLE mapping_ = nullptr;
    void* addr_ = nullptr;
    size_t size_;

public:
    SharedMemoryRegion(const std::string& name, size_t size, bool create)
        : size_(size) {

        std::wstring wname(name.begin(), name.end());

        if (create) {
            mapping_ = CreateFileMappingW(INVALID_HANDLE_VALUE, nullptr,
                                           PAGE_READWRITE,
                                           static_cast<DWORD>(size >> 32),
                                           static_cast<DWORD>(size),
                                           wname.c_str());
        } else {
            mapping_ = OpenFileMappingW(FILE_MAP_ALL_ACCESS, FALSE, wname.c_str());
        }

        if (!mapping_) {
            throw std::runtime_error("File mapping failed");
        }

        addr_ = MapViewOfFile(mapping_, FILE_MAP_ALL_ACCESS, 0, 0, size);
        if (!addr_) {
            CloseHandle(mapping_);
            throw std::runtime_error("MapViewOfFile failed");
        }

        // Lock pages
        VirtualLock(addr_, size);
    }

    ~SharedMemoryRegion() {
        if (addr_) UnmapViewOfFile(addr_);
        if (mapping_) CloseHandle(mapping_);
    }

    void* data() { return addr_; }
};
#endif
```

### Safety Considerations

Shared memory between processes is more fragile than in-process communication. When a process crashes, the shared memory persists and can be left in an inconsistent state. Here's how to detect and handle that.

```cpp
// 1. Version your shared memory layout
struct SharedHeader {
    uint32_t magic = 0xDEADBEEF;   // Identify valid shared memory
    uint32_t version = 1;           // Detect incompatible layouts
    uint32_t producer_pid = 0;      // Track producer process
    uint32_t consumer_pid = 0;      // Track consumer process
    std::atomic<bool> active{false}; // Liveness check
};

// 2. Validate on open
void validate_shared_memory(const SharedHeader* header) {
    if (header->magic != 0xDEADBEEF) {
        throw std::runtime_error("Invalid shared memory (wrong magic)");
    }
    if (header->version != 1) {
        throw std::runtime_error("Incompatible shared memory version");
    }
}

// 3. Handle process crashes
// If producer dies, the shared memory persists (shm_unlink not called)
// Clean up stale segments: check if producer PID is alive
bool is_process_alive(pid_t pid) {
    return kill(pid, 0) == 0 || errno == EPERM;
}
```

The magic number and version fields are cheap insurance. They let a consumer detect a stale segment from a previous run (wrong magic) or a layout-incompatible producer (wrong version) before trying to interpret garbage as valid queue data.

---

## Self-Assessment

### Q1: Why must types in shared memory be trivially copyable

Shared memory is raw bytes mapped into both processes' address spaces. Non-trivially-copyable types (with virtual functions, `std::string`, `std::vector`, smart pointers) contain **pointers** that are valid only in the originating process's address space. Virtual function tables point to code at addresses specific to one process. The other process would dereference invalid pointers, causing crashes or undefined behavior. Only trivially copyable types (POD-like, no pointers) are safe.

### Q2: Why use `std::atomic` in shared memory instead of plain variables

Multiple processes access the same memory concurrently - this is the same as multi-threaded access. Without atomics, there are data races (undefined behavior). `std::atomic` provides the necessary memory ordering guarantees (acquire/release) that ensure one process sees the data written by another in the correct order. The C++ standard guarantees that `std::atomic` works across processes sharing the same memory.

### Q3: What happens if the producer crashes

The shared memory segment persists (it's reference-counted by the kernel). The consumer will stop seeing new data (tail stops advancing) and can detect this via a heartbeat or liveness flag. The stale segment must be cleaned up manually (`shm_unlink`). Best practices: (1) use a heartbeat timestamp that the producer updates periodically, (2) check if the producer PID is alive, (3) implement a watchdog that cleans up stale segments.

---

## Notes

- Use `MAP_HUGETLB` for 2MB hugepages - reduces TLB misses for large shared regions
- `memfd_create` (Linux 3.17+) creates anonymous shared memory without filesystem artifacts
- Shared memory works across containers if you mount the same `/dev/shm`
- Always lock shared memory pages with `mlock()` to prevent swapping under memory pressure
- Consider using Boost.Interprocess for a cross-platform shared memory abstraction
