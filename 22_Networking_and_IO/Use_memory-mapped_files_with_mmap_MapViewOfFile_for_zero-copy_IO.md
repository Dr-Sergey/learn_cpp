# Use memory-mapped files with mmap / MapViewOfFile for zero-copy I/O

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://man7.org/linux/man-pages/man2/mmap.2.html>  

---

## Topic Overview

This topic focuses on the page-fault model and cross-platform mmap (MapViewOfFile on Windows) - complementing #642 (basic mmap + madvise + benchmarks) and #552 (high-throughput + persistent KV store).

The "zero-copy" claim is worth understanding concretely. When you call `fread()`, the OS reads data into a kernel buffer, then copies it into stdio's internal buffer, then copies it again into the buffer you provided - two copies, one syscall per fill. When you access an mmap'd region, the kernel maps the same physical page directly into your process's address space. Your pointer touches that page directly. There is nothing to copy.

```cpp
Page-fault driven loading vs fread:

  fread (buffered I/O):           mmap (page-fault I/O):
  +--------+                      +--------+
  | fread()|                      | ptr[i] |  <- memory access
  +--------+                      +--------+
      |                               |
  [syscall: read()]               [page fault if not in RAM]
      |                               |
  [kernel copies to                [kernel maps page from
   stdio buffer]                   page cache to process]
      |                               |
  [memcpy to user buf]            [CPU resumes, data at ptr]
  +--------+                      +--------+
  | 2 copies                      | 0 copies (shared page)
  | 1 syscall per call             | 0 syscalls after fault
  +--------+                      +--------+
```

The API to achieve this is different on Linux and Windows, but the concept is identical. Here is the mapping between the two:

| Aspect | Linux (mmap) | Windows (MapViewOfFile) |
| --- | --- | --- |
| Create mapping | `mmap()` | `CreateFileMapping()` + `MapViewOfFile()` |
| Unmap | `munmap()` | `UnmapViewOfFile()` + `CloseHandle()` |
| Flush to disk | `msync()` | `FlushViewOfFile()` |
| Advise kernel | `madvise()` | `PrefetchVirtualMemory()` (Win8+) |
| Large pages | `MAP_HUGETLB` | `SEC_LARGE_PAGES` |

---

## Self-Assessment

### Q1: Cross-platform memory-mapped file access

This is a single class that compiles and works correctly on both Linux and Windows. The `#ifdef _WIN32` blocks handle the platform difference so the calling code in `main()` stays clean and portable. Pay attention to the Windows side - it requires three separate handles (file handle, mapping handle, and view pointer) that all need to be closed in the right order:

```cpp
#include <iostream>
#include <cstring>

#ifdef _WIN32
  #include <windows.h>
#else
  #include <sys/mman.h>
  #include <sys/stat.h>
  #include <fcntl.h>
  #include <unistd.h>
#endif

class MemoryMappedFile {
public:
    bool open(const char* path, bool read_only = true) {
#ifdef _WIN32
        DWORD access = read_only ? GENERIC_READ : (GENERIC_READ | GENERIC_WRITE);
        DWORD share = FILE_SHARE_READ;
        hFile_ = CreateFileA(path, access, share, nullptr,
                             OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
        if (hFile_ == INVALID_HANDLE_VALUE) return false;

        LARGE_INTEGER sz;
        GetFileSizeEx(hFile_, &sz);
        size_ = static_cast<size_t>(sz.QuadPart);

        DWORD prot = read_only ? PAGE_READONLY : PAGE_READWRITE;
        hMapping_ = CreateFileMappingA(hFile_, nullptr, prot, 0, 0, nullptr);
        if (!hMapping_) { CloseHandle(hFile_); return false; }

        DWORD map_access = read_only ? FILE_MAP_READ : FILE_MAP_ALL_ACCESS;
        data_ = static_cast<char*>(MapViewOfFile(hMapping_, map_access, 0, 0, 0));
        return data_ != nullptr;
#else
        int flags = read_only ? O_RDONLY : O_RDWR;
        fd_ = ::open(path, flags);
        if (fd_ < 0) return false;

        struct stat st;
        fstat(fd_, &st);
        size_ = st.st_size;

        int prot = read_only ? PROT_READ : (PROT_READ | PROT_WRITE);
        data_ = static_cast<char*>(
            mmap(nullptr, size_, prot, MAP_SHARED, fd_, 0));
        return data_ != MAP_FAILED;
#endif
    }

    void close() {
#ifdef _WIN32
        if (data_) { UnmapViewOfFile(data_); data_ = nullptr; }
        if (hMapping_) { CloseHandle(hMapping_); hMapping_ = nullptr; }
        if (hFile_ != INVALID_HANDLE_VALUE) {
            CloseHandle(hFile_); hFile_ = INVALID_HANDLE_VALUE;
        }
#else
        if (data_ && data_ != MAP_FAILED) { munmap(data_, size_); data_ = nullptr; }
        if (fd_ >= 0) { ::close(fd_); fd_ = -1; }
#endif
    }

    ~MemoryMappedFile() { close(); }

    const char* data() const { return data_; }
    size_t size() const { return size_; }

private:
    char* data_ = nullptr;
    size_t size_ = 0;
#ifdef _WIN32
    HANDLE hFile_ = INVALID_HANDLE_VALUE;
    HANDLE hMapping_ = nullptr;
#else
    int fd_ = -1;
#endif
};

int main() {
    MemoryMappedFile mmf;

    // Works on both Linux and Windows
    const char* path =
#ifdef _WIN32
        "C:\\Windows\\System32\\drivers\\etc\\hosts";
#else
        "/etc/hostname";
#endif

    if (!mmf.open(path)) {
        std::cerr << "Failed to map file\n";
        return 1;
    }

    // Access as byte array - zero copy!
    std::cout << "File size: " << mmf.size() << " bytes\n";
    std::cout << "Content: ";
    std::cout.write(mmf.data(), mmf.size());
    std::cout << '\n';
}
```

On Windows, the reason you need both a mapping handle and a view pointer is that the mapping object tracks the kernel-side resource, while the view is the actual address-space projection. You can create multiple views from the same mapping (for example, mapping different byte ranges), which is why they are separate.

### Q2: Page-fault driven loading vs fread

Here is the step-by-step breakdown of what actually happens under each approach. Understanding this is what lets you reason about when mmap is faster and when it is not.

**fread (buffered stdio):**

1. User calls `fread(buf, 1, n, fp)`.
2. If stdio buffer empty: `read()` syscall - kernel copies data to kernel buffer.
3. Kernel copies from kernel buffer to stdio buffer (copy #1).
4. `fread` copies from stdio buffer to user buffer (copy #2).
5. Returns. **2 copies + 1 syscall per buffer refill.**

**mmap (page-fault driven):**

1. User accesses `ptr[offset]`.
2. CPU translates virtual address - page table entry.
3. If page not present: **minor page fault** triggers.
4. Kernel finds page in page cache (or reads from disk: **major fault**).
5. Kernel maps the SAME physical page into process's address space.
6. CPU retries the instruction - data is at `ptr[offset]`. **0 copies!**

```cpp
Performance timeline for 1GB sequential read:

  fread:  [read 8KB][copy][copy][read 8KB][copy][copy]... 131K syscalls
  mmap:   [fault][map 4KB page]....[no fault, page in cache]... ~250K faults
                                                                then 0 overhead

  After warm-up (pages in cache):
    fread: still copies data every call
    mmap:  direct memory access, zero overhead
```

**Key insight:** After the first access, mmap pages stay in the page cache. Subsequent accesses are pure memory reads - no syscalls, no copies, no faults. This is why mmap shines for files you access repeatedly or that are already warm in cache.

### Q3: mmap vs read() for sequential vs random access

The pattern of your access matters a lot. Here is a breakdown of how the two approaches perform across different scenarios:

| Pattern | read() + buffer | mmap |
| --- | --- | --- |
| **Sequential scan** | Good (predictable, prefetch works) | Good (similar with MADV_SEQUENTIAL) |
| **Random access** | Poor (seek + read per access) | Excellent (pointer arithmetic) |
| **Repeated access** | Re-read from disk or hope for cache | Page cache keeps hot pages |
| **Multiple processes** | Each gets own buffer copy | Shared physical pages (saves RAM) |
| **Large file > RAM** | Buffer window slides | Pages evicted/loaded on demand |
| **Small files** | Simple, low overhead | mmap() setup cost dominates |

The random access case is where the gap is most dramatic. With `read()`, jumping to a random offset requires `lseek()` followed by `read()` - two syscalls per access, plus the kernel has to copy the data to your buffer each time. With mmap, it is one pointer dereference, and if the page is already cached it costs essentially nothing:

```cpp
// Random access comparison:
// read(): lseek(fd, random_offset, SEEK_SET); read(fd, buf, 64);
//   -> 2 syscalls per access (lseek + read)
//
// mmap(): ptr[random_offset];
//   -> 0 syscalls (page fault only if page not cached)
//   -> For 1M random accesses on a 4GB file: mmap is 5-10x faster
```

For truly random access on a large file, mmap wins decisively. For a single sequential pass, the two approaches are much more similar - `read()` with a reasonably sized buffer is often comparable, and has the advantage of more predictable latency.

---

## Notes

- Complementary to #642 (basic mmap + benchmarks) and #552 (high-throughput + persistent KV).
- Windows: `MapViewOfFile` handles are reference-counted; must close ALL handles.
- Portable wrapper libraries: Boost.Interprocess (`mapped_region`), mio (header-only).
- mmap is NOT async - page faults block the thread. For true async file I/O, use io_uring.
- 32-bit processes can only mmap ~2-3GB. 64-bit has virtually no limit.
