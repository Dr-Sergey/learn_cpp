# Use memory-mapped files with mmap for high-throughput file access

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://man7.org/linux/man-pages/man2/mmap.2.html>  

---

## Topic Overview

This topic focuses on high-throughput mmap patterns including persistent key-value storage with msync - complementing #642 (basic mmap + madvise + benchmarks) and #727 (page-fault model + MapViewOfFile cross-platform).

One pattern that surprises people is using mmap not just for reading but as the storage backend for a persistent data structure. You map a file read-write, write your struct directly through the pointer, and then call `msync()` to flush those changes to disk. The kernel's page cache does the buffering for you, and `msync(MS_SYNC)` guarantees that the data has reached the disk before returning. This is the same basic technique used by some embedded databases and memory-mapped log files.

```cpp
mmap + msync for persistent storage:

  Process memory              Page cache              Disk
  +-----------+              +-----------+          +------+
  | mapped    | ----------> | dirty     | msync()  | file |
  | region    |   write     | pages     | -------> | data |
  | ptr[k]=v  |             |           |          |      |
  +-----------+              +-----------+          +------+

  MS_SYNC:  blocks until pages written to disk (durable)
  MS_ASYNC: schedules writeback, returns immediately
```

The choice of `mmap` flags controls behavior quite a bit. Here is a quick reference for the flags that come up most often in high-throughput scenarios:

| mmap flag | Behavior | Use case |
| --- | --- | --- |
| MAP_SHARED | Changes visible to other processes + written to file | Persistent storage, IPC |
| MAP_PRIVATE | Copy-on-write; changes NOT written to file | Read-only + scratch |
| MAP_POPULATE | Pre-fault all pages at mmap() time | Avoid later page faults |
| MAP_HUGETLB | Use huge pages (2MB/1GB) | Large working sets, TLB misses |
| MAP_LOCKED | Lock pages in RAM (no swap) | Real-time, latency-sensitive |

---

## Self-Assessment

### Q1: Map a large file and read it as a byte array

Notice the `MAP_POPULATE` flag here - it tells the kernel to fault in all pages immediately at mmap time rather than lazily on first access. This trades a predictable up-front cost for eliminating unpredictable page-fault latency later. For a server handling requests, you usually prefer that predictability:

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
// Linux/POSIX
#include <iostream>
#include <cstring>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // Create a test file
    const char* path = "/tmp/mmap_read_test.bin";
    {
        int fd = open(path, O_CREAT | O_WRONLY | O_TRUNC, 0644);
        const char data[] = "Hello, memory-mapped world! This is mmap data.";
        write(fd, data, sizeof(data) - 1);
        close(fd);
    }

    // Open and map
    int fd = open(path, O_RDONLY);
    struct stat st;
    fstat(fd, &st);

    // MAP_POPULATE pre-faults all pages (avoids page faults during access)
    void* addr = mmap(nullptr, st.st_size, PROT_READ,
                      MAP_SHARED | MAP_POPULATE, fd, 0);
    close(fd);  // safe to close after mmap

    if (addr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    // Access as byte array - NO read() syscalls!
    const char* bytes = static_cast<const char*>(addr);

    // Sequential scan
    std::cout << "Content: ";
    std::cout.write(bytes, st.st_size);
    std::cout << '\n';

    // Random access via pointer arithmetic
    std::cout << "byte[0]  = '" << bytes[0] << "'\n";
    std::cout << "byte[7]  = '" << bytes[7] << "'\n";

    // High-throughput: process 4KB chunks
    size_t total = 0;
    for (off_t off = 0; off < st.st_size; off += 4096) {
        size_t chunk = std::min<size_t>(4096, st.st_size - off);
        for (size_t i = 0; i < chunk; ++i)
            total += static_cast<unsigned char>(bytes[off + i]);
    }
    std::cout << "Byte sum: " << total << '\n';

    munmap(addr, st.st_size);
    unlink(path);
}
```

After the mapping is established, both sequential and random access use the same plain pointer arithmetic. The kernel transparently handles loading pages from disk when needed.

### Q2: madvise(MADV_SEQUENTIAL) for prefetch-friendly scans

`madvise()` must be called before you start accessing the data - it is a hint about what you are about to do, not a description of what you already did. Here we compare three hints on a 50 MB file to show how much the kernel's prefetching strategy matters:

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <chrono>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

long long scan_with_hint(const char* path, int advice) {
    int fd = open(path, O_RDONLY);
    struct stat st;
    fstat(fd, &st);

    auto* addr = static_cast<const char*>(
        mmap(nullptr, st.st_size, PROT_READ, MAP_SHARED, fd, 0));
    close(fd);

    // Apply madvise hint BEFORE accessing data
    madvise(const_cast<char*>(addr), st.st_size, advice);

    auto t0 = std::chrono::high_resolution_clock::now();
    volatile long long sum = 0;
    for (off_t i = 0; i < st.st_size; ++i)
        sum += addr[i];
    auto t1 = std::chrono::high_resolution_clock::now();

    munmap(const_cast<char*>(addr), st.st_size);
    return std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
}

int main() {
    const char* path = "/tmp/mmap_advise_test.bin";
    // Create 50MB file
    {
        int fd = open(path, O_CREAT | O_WRONLY | O_TRUNC, 0644);
        char buf[4096];
        memset(buf, 'B', sizeof(buf));
        for (int i = 0; i < 12800; ++i) write(fd, buf, sizeof(buf));
        close(fd);
    }

    // Compare: no hint vs MADV_SEQUENTIAL
    auto t_normal = scan_with_hint(path, MADV_NORMAL);
    auto t_seq    = scan_with_hint(path, MADV_SEQUENTIAL);
    auto t_random = scan_with_hint(path, MADV_RANDOM);

    std::cout << "50 MB sequential scan:\n";
    std::cout << "  MADV_NORMAL:     " << t_normal / 1000 << " ms\n";
    std::cout << "  MADV_SEQUENTIAL: " << t_seq / 1000 << " ms (read-ahead)\n";
    std::cout << "  MADV_RANDOM:     " << t_random / 1000 << " ms (no prefetch)\n";

    // MADV_SEQUENTIAL enables aggressive read-ahead:
    //   kernel pre-loads next pages while current pages are processed
    //   also frees pages behind the read pointer (lower RSS)

    unlink(path);
}
```

`MADV_SEQUENTIAL` is usually the fastest for a linear scan because the kernel can pre-load pages while you are still processing the current ones - effectively hiding the I/O latency. `MADV_RANDOM` turns off read-ahead entirely, which is the right choice when you will be jumping around the file, since prefetching would just waste I/O bandwidth and RAM.

### Q3: Persistent key-value store with mmap + msync

This example shows mmap used as a real storage engine, not just a reading tool. The `MmapKVStore` maps a file, writes key-value entries directly into the mapped memory as C structs, and calls `msync()` to flush. When the process restarts and remaps the same file, the data is still there:

<!-- compile: needs POSIX header `sys/mman.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

// Fixed-size KV entry for mmap storage
struct KVEntry {
    char key[32];
    char value[96];
    bool occupied;
    char pad[3];  // alignment
};

static_assert(sizeof(KVEntry) == 132);

class MmapKVStore {
public:
    static constexpr size_t MAX_ENTRIES = 1024;
    static constexpr size_t FILE_SIZE = MAX_ENTRIES * sizeof(KVEntry);

    explicit MmapKVStore(const char* path) : path_(path) {
        bool exists = (access(path, F_OK) == 0);
        fd_ = open(path, O_CREAT | O_RDWR, 0644);

        if (!exists) {
            ftruncate(fd_, FILE_SIZE);  // extend file to required size
        }

        data_ = static_cast<KVEntry*>(
            mmap(nullptr, FILE_SIZE, PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd_, 0));

        if (!exists) {
            memset(data_, 0, FILE_SIZE);  // zero-initialize
            sync();  // flush to disk
        }
    }

    ~MmapKVStore() {
        if (data_ != MAP_FAILED) {
            msync(data_, FILE_SIZE, MS_SYNC);  // final flush
            munmap(data_, FILE_SIZE);
        }
        if (fd_ >= 0) close(fd_);
    }

    void put(const char* key, const char* value) {
        // Find existing or first free slot
        for (size_t i = 0; i < MAX_ENTRIES; ++i) {
            if (!data_[i].occupied || strcmp(data_[i].key, key) == 0) {
                strncpy(data_[i].key, key, sizeof(data_[i].key) - 1);
                strncpy(data_[i].value, value, sizeof(data_[i].value) - 1);
                data_[i].occupied = true;
                // Sync just this page to disk
                msync(&data_[i], sizeof(KVEntry), MS_ASYNC);
                return;
            }
        }
    }

    const char* get(const char* key) const {
        for (size_t i = 0; i < MAX_ENTRIES; ++i) {
            if (data_[i].occupied && strcmp(data_[i].key, key) == 0)
                return data_[i].value;
        }
        return nullptr;
    }

    void sync() { msync(data_, FILE_SIZE, MS_SYNC); }

private:
    const char* path_;
    int fd_ = -1;
    KVEntry* data_ = nullptr;
};

int main() {
    const char* path = "/tmp/mmap_kv.db";

    {
        MmapKVStore store(path);
        store.put("name", "Modern C++");
        store.put("version", "C++20");
        store.put("topic", "mmap persistent storage");
        std::cout << "Stored 3 entries\n";
    }  // destructor: msync + munmap + close

    // Reopen - data persists!
    {
        MmapKVStore store(path);
        std::cout << "name:    " << store.get("name") << '\n';
        std::cout << "version: " << store.get("version") << '\n';
        std::cout << "topic:   " << store.get("topic") << '\n';
    }

    unlink(path);
}
// Output:
//   Stored 3 entries
//   name:    Modern C++
//   version: C++20
//   topic:   mmap persistent storage
```

Notice that `put()` uses `MS_ASYNC` for individual writes (schedule writeback without waiting) but the destructor uses `MS_SYNC` (wait until everything is on disk). The `ftruncate()` call before the first `mmap()` is critical - you must size the file before mapping it, because accessing memory beyond the file's size causes SIGBUS. This is a common trap when building mmap-based storage.

---

## Notes

- Complementary to #642 (basic mmap + benchmarks) and #727 (page-fault model + cross-platform).
- `msync(MS_SYNC)` guarantees durability; `MS_ASYNC` only schedules writeback.
- For production KV stores: add CRC checksums, write-ahead log, and proper hash table.
- SIGBUS protection: `signal(SIGBUS, handler)` or `sigaction` to handle truncated files.
- ftruncate() must size the file BEFORE mmap; mmap beyond EOF causes SIGBUS.
