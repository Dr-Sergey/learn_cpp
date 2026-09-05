# Understand zero-copy techniques with scatter-gather I/O (iovec / WSABuf)

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://man7.org/linux/man-pages/man2/readv.2.html>  

---

## Topic Overview

Scatter-gather I/O is one of those features that feels like an obscure API detail until you actually measure the difference, and then you wonder why you were ever copying data manually. The core idea is simple: instead of writing one contiguous buffer per syscall, you hand the kernel an array of (pointer, length) pairs and it handles all of them in a single trip. No temporary buffer, no `memcpy`, no extra allocation. This topic extends #641 and #553 to cover the cross-platform story - POSIX `iovec` on Linux and macOS, and `WSABUF` on Windows.

The two structs look nearly identical but have the field order reversed, which is a classic trap:

```cpp
POSIX (Linux/macOS):              Windows:
  struct iovec {                    typedef struct {
    void  *iov_base;                  ULONG len;      // note: len first!
    size_t iov_len;                   CHAR  *buf;
  };                                } WSABUF;
  writev(fd, iov, count);           WSASend(s, wsabufs, count, ...);
  readv(fd, iov, count);            WSARecv(s, wsabufs, count, ...);
```

The reversed order in `WSABUF` (length before pointer) is a common source of subtle bugs when writing cross-platform code. Always double-check which platform you're targeting.

| Platform | Scatter-gather API | Header |
| --- | --- | --- |
| Linux | readv/writev, sendmsg/recvmsg | `<sys/uio.h>` |
| Windows | WSASend/WSARecv | `<winsock2.h>` |
| macOS/BSD | readv/writev, sendmsg/recvmsg | `<sys/uio.h>` |
| Asio | const_buffer / mutable_buffer sequence | `<asio.hpp>` |

---

## Self-Assessment

### Q1: readv/writev with multiple non-contiguous buffers

The example below demonstrates writing three separate buffers - one on the stack as an integer, one as a char array, and one on the heap via `std::vector` - in a single `writev` call, then reading them back into separate destinations with `readv`. The buffers live at completely different addresses in memory, but the kernel stitches them together seamlessly:

```cpp
// Cross-platform wrapper concept (POSIX implementation)
#include <iostream>
#include <cstdint>
#include <cstring>
#include <vector>

#ifdef _WIN32
#include <winsock2.h>
#pragma comment(lib, "ws2_32.lib")
// Windows scatter-gather
struct IoVec {
    WSABUF buf;
    void set(void* base, size_t len) {
        buf.buf = static_cast<char*>(base);
        buf.len = static_cast<ULONG>(len);
    }
};
#else
#include <sys/uio.h>
#include <unistd.h>
#include <fcntl.h>
// POSIX scatter-gather
struct IoVec {
    struct iovec iov;
    void set(void* base, size_t len) {
        iov.iov_base = base;
        iov.iov_len = len;
    }
};
#endif

int main() {
#ifndef _WIN32
    // POSIX demo: write 3 non-contiguous buffers in one syscall
    uint32_t header = 42;        // 4 bytes (stack)
    char name[] = "Alice";       // 6 bytes (stack, different location)
    std::vector<uint8_t> data = {1, 2, 3, 4};  // 4 bytes (heap)

    struct iovec iov[3];
    iov[0].iov_base = &header;
    iov[0].iov_len  = sizeof(header);
    iov[1].iov_base = name;
    iov[1].iov_len  = 5;  // without null terminator
    iov[2].iov_base = data.data();
    iov[2].iov_len  = data.size();

    int fd = open("/tmp/scatter_demo.bin", O_CREAT | O_RDWR | O_TRUNC, 0644);
    ssize_t written = writev(fd, iov, 3);  // ONE syscall, 3 disjoint buffers
    std::cout << "Written: " << written << " bytes\n";  // 13

    // Read back into different buffers
    lseek(fd, 0, SEEK_SET);
    uint32_t hdr_read = 0;
    char name_read[6] = {};
    uint8_t data_read[4] = {};

    struct iovec rv[3];
    rv[0].iov_base = &hdr_read;  rv[0].iov_len = 4;
    rv[1].iov_base = name_read;  rv[1].iov_len = 5;
    rv[2].iov_base = data_read;  rv[2].iov_len = 4;

    readv(fd, rv, 3);
    std::cout << "Header: " << hdr_read << '\n';  // 42
    std::cout << "Name: " << name_read << '\n';    // Alice
    std::cout << "Data: ";
    for (auto b : data_read) std::cout << (int)b << ' ';  // 1 2 3 4
    std::cout << '\n';

    close(fd);
    unlink("/tmp/scatter_demo.bin");
#else
    std::cout << "Windows: use WSASend/WSARecv with WSABUF arrays\n";
#endif
}
```

The important point here is that the kernel treats the array of iovecs as a single logical stream. The data arrives at the other end (or in the file) in exactly the order you specified, as if you had copied everything into one contiguous buffer first - just without the copy.

### Q2: Scatter-gather avoids header+payload copy

Here is a direct benchmark comparison between the traditional "copy everything into a temporary buffer" approach and the scatter-gather approach. The difference compounds over thousands of iterations because you're eliminating both the allocation overhead and the actual `memcpy` work:

```cpp
#include <iostream>
#include <cstring>
#include <cstdint>
#include <chrono>
#include <vector>

#ifndef _WIN32
#include <sys/uio.h>
#include <unistd.h>
#include <fcntl.h>
#endif

// Traditional approach: copy header + payload into contiguous buffer
void send_traditional(int fd, const uint8_t* header, size_t hlen,
                      const uint8_t* payload, size_t plen) {
    std::vector<uint8_t> buf(hlen + plen);
    std::memcpy(buf.data(), header, hlen);          // COPY 1
    std::memcpy(buf.data() + hlen, payload, plen);  // COPY 2
    write(fd, buf.data(), buf.size());               // syscall
    // 2 copies + 1 allocation + 1 syscall
}

// Scatter-gather: zero copies
void send_scatter(int fd, const uint8_t* header, size_t hlen,
                  const uint8_t* payload, size_t plen) {
#ifndef _WIN32
    struct iovec iov[2];
    iov[0].iov_base = const_cast<uint8_t*>(header);
    iov[0].iov_len  = hlen;
    iov[1].iov_base = const_cast<uint8_t*>(payload);
    iov[1].iov_len  = plen;
    writev(fd, iov, 2);  // ONE syscall, ZERO copies, ZERO allocations
#endif
}

int main() {
#ifndef _WIN32
    int fd = open("/dev/null", O_WRONLY);
    uint8_t header[64] = {};
    std::vector<uint8_t> payload(64 * 1024, 0xAA);  // 64KB payload

    constexpr int ITERS = 100'000;

    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERS; ++i)
        send_traditional(fd, header, 64, payload.data(), payload.size());
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERS; ++i)
        send_scatter(fd, header, 64, payload.data(), payload.size());
    auto t2 = std::chrono::high_resolution_clock::now();

    auto ms1 = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    auto ms2 = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1);
    std::cout << "Traditional (copy): " << ms1.count() << " ms\n";
    std::cout << "Scatter (zero-copy): " << ms2.count() << " ms\n";
    // Scatter is faster: avoids 100K allocations + 200K memcpy calls

    close(fd);
#endif
}
```

At 100K iterations with a 64KB payload, you're eliminating 100K allocations and 200K `memcpy` calls. The scatter version is faster even when writing to `/dev/null` - meaning the CPU time saved on copies is measurable even when the actual I/O cost is zero.

### Q3: Zero-copy HTTP response sender with iovec

This is the pattern you'd use in a real HTTP server. The status line, headers, and body live in separate string objects (or memory-mapped regions), and `writev` sends them all in one syscall without ever creating a combined buffer:

```cpp
#include <iostream>
#include <string>
#include <cstring>

#ifndef _WIN32
#include <sys/uio.h>
#include <sys/socket.h>
#include <unistd.h>

// Send HTTP response using scatter-gather: header and body are separate
// No need to concatenate them into one buffer
void send_http_response(int client_fd,
                        const std::string& status_line,
                        const std::string& headers,
                        const char* body, size_t body_len) {
    // Construct the HTTP response parts
    // Part 1: "HTTP/1.1 200 OK\r\n"
    // Part 2: "Content-Type: text/html\r\n" + other headers + "\r\n"
    // Part 3: body (could be from mmap'd file, no need to copy!)

    struct iovec iov[3];
    iov[0].iov_base = const_cast<char*>(status_line.data());
    iov[0].iov_len  = status_line.size();
    iov[1].iov_base = const_cast<char*>(headers.data());
    iov[1].iov_len  = headers.size();
    iov[2].iov_base = const_cast<char*>(body);
    iov[2].iov_len  = body_len;

    // Send all three parts in ONE syscall
    writev(client_fd, iov, 3);
    // The body could be a memory-mapped file (mmap) -> truly zero-copy!
}
#endif

int main() {
    // Demonstrate the concept
    std::string status = "HTTP/1.1 200 OK\r\n";
    std::string headers = "Content-Type: text/html\r\n"
                          "Content-Length: 45\r\n"
                          "\r\n";
    const char* body = "<html><body>Hello, World!</body></html>\r\n\r\n";

    std::cout << "HTTP response parts (would be sent via writev):\n";
    std::cout << "  Part 1 (" << status.size() << "B): " << status;
    std::cout << "  Part 2 (" << headers.size() << "B): [headers]\n";
    std::cout << "  Part 3 (" << strlen(body) << "B): [body]\n";
    std::cout << "\nTotal: 1 syscall, 0 copies\n";
    std::cout << "\nWindows equivalent:\n";
    std::cout << "  WSABUF bufs[3];\n";
    std::cout << "  bufs[0] = { status.size(), status.data() };\n";
    std::cout << "  bufs[1] = { headers.size(), headers.data() };\n";
    std::cout << "  bufs[2] = { body_len, body };\n";
    std::cout << "  WSASend(client_sock, bufs, 3, &sent, 0, NULL, NULL);\n";
}
```

The comment about `mmap` is worth expanding on: if you serve static files, you can `mmap` the file contents into memory and pass that pointer directly as the body iovec. The kernel then transfers data from the page cache to the network socket without ever copying it through user space. That is the "truly zero-copy" path that high-performance web servers exploit.

---

## Notes

- Complementary to #641 (POSIX scatter-gather) and #553 (sendfile).
- Windows `WSABUF` has `len` before `buf` (opposite order from `iovec`).
- Asio abstracts this: `asio::buffer()` works on both platforms.
- Maximum iov count: Linux `UIO_MAXIOV` = 1024, Windows: no fixed limit.
- For truly zero-copy HTTP: `mmap` the file + `writev` with header iovec = no data copied from disk to user space.
