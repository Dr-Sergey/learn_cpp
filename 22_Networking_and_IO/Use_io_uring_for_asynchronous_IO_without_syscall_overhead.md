# Use io_uring for asynchronous I/O without syscall overhead

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://kernel.dk/io_uring.pdf>  

---

## Topic Overview

io_uring is Linux's newest async I/O interface (kernel 5.1+). It uses shared memory ring buffers between user space and kernel, allowing I/O submission and completion without syscalls in the hot path.

The reason this is such a big deal is that traditional async I/O on Linux always required at least one syscall per operation. epoll needs `epoll_wait` to collect events. Even `aio_read` needs `io_submit`. With io_uring, the submission and completion rings live in memory that both user space and the kernel can access directly, so in the best case you write a request into shared memory and read the result out of shared memory - no kernel crossing required.

```cpp
io_uring architecture:
  User space          Kernel
  +--------+        +--------+
  |  SQE   | -----> |  SQ    |  Submission Queue
  | (write) |        | (read) |
  +--------+        +--------+
                         |
                    [kernel does I/O]
                         |
  +--------+        +--------+
  |  CQE   | <----- |  CQ    |  Completion Queue
  | (read) |        | (write) |
  +--------+        +--------+

  Submit: write SQE to shared ring -> io_uring_enter() or SQPOLL
  Complete: read CQE from shared ring (NO syscall needed!)
```

The table below compares the four main I/O models you might choose on Linux. The progression from top to bottom trades simplicity for throughput:

| I/O model | Submit syscall | Complete syscall | Batching |
| --- | --- | --- | --- |
| Blocking read/write | 1 per op | N/A | No |
| epoll + non-blocking | epoll_wait + read | N/A | Per-event |
| io_uring | io_uring_enter (batched) | None (CQ polling) | Yes |
| io_uring + SQPOLL | None (kernel polls SQ) | None (CQ polling) | Yes |

---

## Self-Assessment

### Q1: Set up io_uring and submit a read request

Here is the minimal end-to-end flow. You initialize the ring, open a file, prepare a submission queue entry (SQE), submit it with a single syscall, and then wait for the completion queue entry (CQE). The `user_data` field is your way of tagging a request so you can identify it when the completion arrives.

<!-- compile: needs POSIX header `liburing.h` -->
```cpp
// Requires: liburing (apt install liburing-dev)
// Compile: g++ -std=c++20 -luring uring_read.cpp
#include <iostream>
#include <cstring>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // 1. Initialize io_uring with 32 SQ entries
    struct io_uring ring;
    int ret = io_uring_queue_init(32, &ring, 0);
    if (ret < 0) {
        std::cerr << "io_uring_queue_init: " << strerror(-ret) << '\n';
        return 1;
    }

    // 2. Open a file
    int fd = open("/etc/hostname", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    // 3. Prepare a read SQE (Submission Queue Entry)
    char buf[256] = {};
    struct io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, sizeof(buf) - 1, 0);  // read at offset 0
    sqe->user_data = 42;  // tag for identifying in CQE

    // 4. Submit the request (single syscall for all pending SQEs)
    ret = io_uring_submit(&ring);  // io_uring_enter() under the hood
    std::cout << "Submitted " << ret << " request(s)\n";

    // 5. Wait for completion (read from CQ ring)
    struct io_uring_cqe* cqe;
    ret = io_uring_wait_cqe(&ring, &cqe);  // blocks until CQE available
    if (ret < 0) {
        std::cerr << "wait_cqe: " << strerror(-ret) << '\n';
    } else {
        std::cout << "Completed: user_data=" << cqe->user_data
                  << " result=" << cqe->res << " bytes\n";
        std::cout << "Data: " << buf;
        io_uring_cqe_seen(&ring, cqe);  // mark CQE as consumed
    }

    close(fd);
    io_uring_queue_exit(&ring);
}
// Output:
//   Submitted 1 request(s)
//   Completed: user_data=42 result=12 bytes
//   Data: my-hostname
```

`io_uring_cqe_seen` is easy to forget but essential - it advances the CQ ring's head pointer to tell the kernel you have consumed that slot. If you skip it the ring fills up and subsequent completions are lost.

### Q2: Harvest completions without syscall (CQ polling)

Once you have submitted operations you can poll the completion queue without making any syscall. `io_uring_peek_cqe` reads from shared memory and returns immediately whether or not a CQE is available. This is the zero-syscall hot path that makes io_uring so attractive for high-throughput workloads.

<!-- compile: needs POSIX header `liburing.h` -->
```cpp
#include <iostream>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>

int main() {
    // Setup with SQPOLL (kernel thread polls submission queue)
    struct io_uring ring;
    struct io_uring_params params{};
    params.flags = IORING_SETUP_SQPOLL;  // kernel polls SQ automatically
    params.sq_thread_idle = 2000;         // SQ thread sleeps after 2s idle

    int ret = io_uring_queue_init_params(256, &ring, &params);
    if (ret < 0) {
        // SQPOLL requires root or CAP_SYS_NICE
        std::cerr << "init with SQPOLL: " << strerror(-ret)
                  << " (needs root)\n";
        // Fallback to normal mode
        io_uring_queue_init(256, &ring, 0);
    }

    // Submit multiple reads
    int fd = open("/etc/hostname", O_RDONLY);
    char bufs[4][64] = {};

    for (int i = 0; i < 4; ++i) {
        auto* sqe = io_uring_get_sqe(&ring);
        io_uring_prep_read(sqe, fd, bufs[i], 63, 0);
        sqe->user_data = i;
    }
    io_uring_submit(&ring);

    // Harvest completions by polling CQ (no syscall!)
    // The CQ is in shared memory; we just read from it.
    int completed = 0;
    while (completed < 4) {
        struct io_uring_cqe* cqe;

        // io_uring_peek_cqe: non-blocking check (NO syscall)
        ret = io_uring_peek_cqe(&ring, &cqe);
        if (ret == 0) {
            std::cout << "CQE[" << cqe->user_data << "]: "
                      << cqe->res << " bytes\n";
            io_uring_cqe_seen(&ring, cqe);
            ++completed;
        }
        // In real code: io_uring_wait_cqe_nr for batch waiting
    }

    close(fd);
    io_uring_queue_exit(&ring);

    std::cout << "\nZero-syscall hot path:\n";
    std::cout << "  Submit: write SQE to shared memory ring\n";
    std::cout << "  Complete: read CQE from shared memory ring\n";
    std::cout << "  With SQPOLL: kernel thread polls SQ -> no submit syscall\n";
    std::cout << "  With CQ polling: peek_cqe reads shared mem -> no wait syscall\n";
}
```

The busy-poll loop shown here works correctly but burns CPU spinning. In production you would use `io_uring_wait_cqe_nr` to wait for a batch of completions, which lets the thread sleep until at least N completions are ready. That way you get good latency without wasting CPU cycles.

### Q3: io_uring vs epoll throughput comparison

Here is a comprehensive comparison of the two interfaces across the dimensions that matter most in production:

| Metric | epoll + read() | io_uring | io_uring + SQPOLL |
| --- | --- | --- | --- |
| Syscalls per I/O | 2 (epoll_wait + read) | 1 (io_uring_enter, batched) | 0 (kernel polls) |
| Batch submission | No | Yes (multiple SQEs) | Yes |
| File I/O | Must use thread pool | Native async | Native async |
| Network I/O | Excellent | Good (improving) | Good |
| IOPS (4K random reads) | ~100K | ~400K | ~500K |
| Latency (p99) | ~10µs | ~3µs | ~2µs |
| CPU usage at 100K IOPS | ~30% | ~15% | ~20% (SQ thread) |
| Kernel version | 2.6+ | 5.1+ | 5.11+ |

**When to use io_uring over epoll:**

- High-IOPS file I/O (databases, storage engines)
- When syscall overhead matters (>100K ops/sec)
- Mixed file + network I/O in one event loop
- When batching submissions (N ops per enter())

**When epoll is still fine:**

- Network-only servers (<50K connections)
- When portability matters (io_uring is Linux-only)
- Simple request/response protocols
- When using Asio (abstracts the backend)

The "file I/O must use thread pool" row for epoll is the most commonly misunderstood entry. Regular files on Linux are always considered "ready" by epoll, so you cannot use epoll to wait for disk reads to complete. You would need a separate thread pool to do blocking reads and then signal back to your event loop. io_uring handles this natively - you submit a file read and get a completion event just like any other operation.

---

## Notes

- Complementary to #728 (SQPOLL mode) and #551 (C++ RAII wrapper).
- io_uring supports: read, write, accept, connect, send, recv, close, timeout, and more.
- liburing is the recommended C wrapper (header-only, easy to wrap in C++ RAII).
- Security: SQPOLL mode requires elevated privileges (CAP_SYS_NICE or root).
- Asio 2.0 has experimental io_uring backend (`ASIO_HAS_IO_URING`).
