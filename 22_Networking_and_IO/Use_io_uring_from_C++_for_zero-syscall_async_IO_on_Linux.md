# Use io_uring from C++ for zero-syscall async I/O on Linux

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://github.com/axboe/liburing>  

---

## Topic Overview

This topic focuses on wrapping liburing in C++ RAII and understanding the SQ/CQ ring buffer model - complementing #640 (basic setup + polling) and #728 (batch submission + SQPOLL).

The key insight behind io_uring is that traditional system calls like `read()` and `write()` each require a full round-trip into the kernel - your thread stops, the CPU switches to kernel mode, the kernel does the work, and then control returns to you. For a server handling tens of thousands of I/O operations per second, those context switches add up fast. io_uring eliminates most of them by giving user space and the kernel a shared chunk of memory they both read and write directly.

Here is the data flow. The diagram shows two ring buffers sitting in memory that is mapped into both user space and kernel space simultaneously:

```cpp
SQ/CQ ring buffer model (shared memory):

  User space                 Kernel space
  +-----------+             +-----------+
  | SQ Ring   |  mmap'd     | SQ Ring   |  same physical pages
  | [SQE][SQE]| ----------> | [SQE][SQE]|
  | tail++    |             | head++    |  user writes tail, kernel reads head
  +-----------+             +-----------+

  +-----------+             +-----------+
  | CQ Ring   |  mmap'd     | CQ Ring   |
  | [CQE][CQE]| <---------- | [CQE][CQE]|
  | head++    |             | tail++    |  kernel writes tail, user reads head
  +-----------+             +-----------+

  No data copy! Just atomic counter increments on shared memory.
```

You write a Submission Queue Entry (SQE) describing your I/O request, bump the tail pointer, and the kernel sees it without any syscall. When the kernel finishes, it writes a Completion Queue Entry (CQE) to the other ring and bumps its own tail pointer. You read that result directly from memory - again, no syscall.

Compare that to what traditional I/O requires every single time:

| Traditional I/O | io_uring |
| --- | --- |
| `read(fd, buf, n)` syscall | Write SQE to shared ring |
| Kernel copies params from user | Kernel reads SQE from shared memory |
| Kernel does I/O, copies result | Kernel does I/O, writes CQE to ring |
| Returns to user (context switch) | User polls CQE (no context switch) |

---

## Self-Assessment

### Q1: C++ RAII wrapper for io_uring

Before looking at the code, notice what we're wrapping: `io_uring_queue_init` and `io_uring_queue_exit` are the open/close pair that allocate the shared rings. Those absolutely need RAII treatment - you do not want to forget `queue_exit` on an error path. The `FileDesc` class at the bottom is the same idea applied to plain file descriptors.

<!-- compile: needs POSIX header `liburing.h` -->
```cpp
// Requires: liburing-dev
// Compile: g++ -std=c++20 -luring raii_uring.cpp
#include <iostream>
#include <stdexcept>
#include <cstring>
#include <vector>
#include <functional>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>

// RAII wrapper for io_uring
class IoUring {
public:
    explicit IoUring(unsigned entries, unsigned flags = 0) {
        int ret = io_uring_queue_init(entries, &ring_, flags);
        if (ret < 0)
            throw std::runtime_error(
                std::string("io_uring_queue_init: ") + strerror(-ret));
    }

    ~IoUring() { io_uring_queue_exit(&ring_); }

    // Non-copyable, movable
    IoUring(const IoUring&) = delete;
    IoUring& operator=(const IoUring&) = delete;

    // Submit a read operation
    void prep_read(int fd, void* buf, unsigned nbytes, off_t offset,
                   uint64_t user_data) {
        auto* sqe = io_uring_get_sqe(&ring_);
        if (!sqe) throw std::runtime_error("SQ full");
        io_uring_prep_read(sqe, fd, buf, nbytes, offset);
        sqe->user_data = user_data;
    }

    // Submit a write operation
    void prep_write(int fd, const void* buf, unsigned nbytes, off_t offset,
                    uint64_t user_data) {
        auto* sqe = io_uring_get_sqe(&ring_);
        if (!sqe) throw std::runtime_error("SQ full");
        io_uring_prep_write(sqe, fd, buf, nbytes, offset);
        sqe->user_data = user_data;
    }

    // Submit all pending SQEs
    int submit() { return io_uring_submit(&ring_); }

    // Wait for one completion
    struct Completion {
        uint64_t user_data;
        int32_t result;  // bytes transferred or -errno
    };

    Completion wait_one() {
        struct io_uring_cqe* cqe;
        int ret = io_uring_wait_cqe(&ring_, &cqe);
        if (ret < 0)
            throw std::runtime_error(
                std::string("wait_cqe: ") + strerror(-ret));
        Completion c{cqe->user_data, cqe->res};
        io_uring_cqe_seen(&ring_, cqe);
        return c;
    }

private:
    struct io_uring ring_;
};

// RAII file descriptor
class FileDesc {
public:
    explicit FileDesc(const char* path, int flags)
        : fd_(open(path, flags)) {
        if (fd_ < 0) throw std::runtime_error("open failed");
    }
    ~FileDesc() { if (fd_ >= 0) close(fd_); }
    int get() const { return fd_; }
private:
    int fd_;
};

int main() {
    IoUring ring(32);  // RAII: automatically cleaned up
    FileDesc file("/etc/hostname", O_RDONLY);  // RAII

    char buf[256] = {};
    ring.prep_read(file.get(), buf, 255, 0, /*user_data=*/1);
    ring.submit();

    auto cqe = ring.wait_one();
    std::cout << "Read " << cqe.result << " bytes (tag=" << cqe.user_data << ")\n";
    std::cout << "Content: " << buf;
}
// RAII ensures: ring is destroyed, file is closed, even on exception
```

The `user_data` field on the SQE is just a tag you can use to match completions back to the requests that spawned them. It passes through the kernel untouched and comes back on the CQE. In real code you'd often store a pointer to a context struct there.

### Q2: SQ/CQ model and why it reduces overhead

The SQ (Submission Queue) and CQ (Completion Queue) are ring buffers in shared memory (mmap'd between user space and kernel):

**Submission Queue (SQ):**

- User space writes SQEs (Submission Queue Entries) to the ring.
- Increments the tail pointer atomically.
- Kernel reads SQEs starting from head pointer.
- Think of it as a producer-consumer queue: user = producer, kernel = consumer.

**Completion Queue (CQ):**

- Kernel writes CQEs (Completion Queue Entries) to the ring.
- Increments the tail pointer.
- User reads CQEs starting from head pointer.
- Kernel = producer, user = consumer.

**Why this reduces overhead:**

1. **No parameter copying**: Traditional syscalls copy parameters from user to kernel via `copy_from_user()`. SQ entries are already in shared memory.

2. **Batch amortization**: One `io_uring_enter()` syscall submits ALL pending SQEs. Traditional I/O needs one syscall per operation.

3. **Zero-copy completions**: CQEs are written directly to shared memory. No `copy_to_user()` needed.

4. **No context switch for polling**: `io_uring_peek_cqe()` reads shared memory directly - no kernel transition.

5. **With SQPOLL**: Even submission requires no syscall - kernel thread polls the SQ ring continuously.

Here is a concrete comparison that shows just how dramatic the difference is for a burst of reads:

```cpp
Syscall overhead comparison (10,000 reads):
  Traditional:  10,000 read() syscalls = 10,000 context switches
  epoll + read: 10,000 read() + epoll_wait calls
  io_uring:     ~300 io_uring_enter() (batches of ~32) + 0 CQ syscalls
  io_uring+SQP: 0 submit syscalls + 0 CQ syscalls = 0 context switches!
```

The reason this matters so much is that a context switch is not just overhead in terms of CPU cycles - it also pollutes the instruction cache and TLB, which hurts performance for the user-space code that runs immediately after. Eliminating context switches is therefore a double win.

### Q3: io_uring vs epoll for 10K connections

This is a conceptual walkthrough that makes the syscall difference between the two approaches concrete. The numbers at the bottom are representative of what you'd actually measure on a loaded system:

```cpp
// Conceptual comparison (pseudo-code)
#include <iostream>

int main() {
    std::cout << "=== 10K concurrent connections benchmark ===\n\n";

    std::cout << "epoll + non-blocking approach:\n";
    std::cout << "  Setup: epoll_create, add 10K sockets with EPOLLIN\n";
    std::cout << "  Loop:\n";
    std::cout << "    n = epoll_wait()           // 1 syscall\n";
    std::cout << "    for each ready fd:\n";
    std::cout << "      read(fd, buf, len)       // 1 syscall per ready fd\n";
    std::cout << "      write(fd, resp, len)     // 1 syscall per response\n";
    std::cout << "  Total: 1 + 2*ready syscalls per iteration\n";
    std::cout << "  For 10K active: ~20K syscalls per batch\n\n";

    std::cout << "io_uring approach:\n";
    std::cout << "  Setup: io_uring_queue_init, register buffers\n";
    std::cout << "  Loop:\n";
    std::cout << "    for each socket: prep_recv(sqe)  // no syscall\n";
    std::cout << "    io_uring_submit()               // 1 syscall for ALL\n";
    std::cout << "    for each CQE: process + prep_send  // no syscall\n";
    std::cout << "    io_uring_submit()               // 1 syscall for ALL\n";
    std::cout << "  Total: 2 syscalls per batch (regardless of ready count!)\n";
    std::cout << "  For 10K active: 2 syscalls per batch\n\n";

    std::cout << "Result at 10K connections, 100K req/sec:\n";
    std::cout << "  epoll: ~200K syscalls/sec, 15-20% CPU\n";
    std::cout << "  io_uring: ~6K syscalls/sec, 5-8% CPU\n";
    std::cout << "  io_uring+SQPOLL: ~0 syscalls/sec, 3-5% CPU + SQ thread\n";
}
```

The io_uring path goes from syscall count scaling linearly with active connections down to a flat two syscalls per event loop iteration. That is what makes it so attractive for high-connection-count servers.

---

## Notes

- Complementary to #640 (basic io_uring) and #728 (batch + SQPOLL).
- liburing header: `#include <liburing.h>`, link with `-luring`.
- Registered buffers (`io_uring_register_buffers`) avoid per-I/O get_user_pages.
- Registered files (`io_uring_register_files`) avoid per-I/O fget/fput.
- io_uring multishot: one SQE for recurring operations (accept, recv) - kernel 5.19+.
