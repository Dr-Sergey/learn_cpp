# Implement zero-copy I/O patterns with scatter-gather (iovec / readv/writev)

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://man7.org/linux/man-pages/man2/readv.2.html>  

---

## Topic Overview

When you send a network message that has a header and a payload, the naive approach is to copy both into a single contiguous buffer and then call `send`. That copy is unnecessary work. Scatter-gather I/O lets you point the kernel directly at multiple disjoint buffers and send them as a single logical unit, with no intermediate copy required.

The interface is built around `struct iovec`, which is just a `(pointer, length)` pair. You build an array of these - one per buffer - and pass the array to `readv` or `writev`. The kernel handles the rest.

```cpp
Traditional send (2 syscalls + 1 memcpy):
  memcpy(buf, header, 8);        // copy header into contiguous buffer
  memcpy(buf+8, payload, 1000);  // copy payload
  send(fd, buf, 1008);           // one syscall

Scatter-gather writev (1 syscall, zero copy):
  iov[0] = { header, 8 };   // point directly at header
  iov[1] = { payload, 1000}; // point directly at payload
  writev(fd, iov, 2);        // one syscall, no intermediate buffer!
```

| Approach | Syscalls | Copies | Best for |
| --- | --- | --- | --- |
| Multiple send() calls | N | 0 | Simple cases |
| memcpy + single send() | 1 | N-1 | Small messages |
| writev() | 1 | 0 | Protocol framing |
| sendmsg() | 1 | 0 | + ancillary data |

---

## Self-Assessment

### Q1: Use readv() to scatter data into multiple buffers

This example writes a length-prefixed "protocol message" to a temporary file and then reads it back using `readv` - splitting the 4-byte header and the 12-byte payload into separate buffers in a single syscall. In real networking code, you would call `readv` on the socket FD instead of a file FD, but the mechanics are identical.

<!-- compile: needs POSIX header `unistd.h` -->
```cpp
// Linux/POSIX example
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <sys/uio.h>
#include <fcntl.h>

int main() {
    // Create a test file with known content
    int fd = open("/tmp/scatter_test.bin", O_CREAT | O_RDWR | O_TRUNC, 0644);

    // Write a "protocol message": 4-byte header + 12-byte payload
    const char data[] = "\x00\x00\x00\x0CHello World!";  // header=12, payload="Hello World!"
    write(fd, data, 16);
    lseek(fd, 0, SEEK_SET);

    // readv: scatter into separate buffers (header and payload)
    char header[4];
    char payload[12];

    struct iovec iov[2];
    iov[0].iov_base = header;       // first 4 bytes -> header buffer
    iov[0].iov_len  = sizeof(header);
    iov[1].iov_base = payload;      // next 12 bytes -> payload buffer
    iov[1].iov_len  = sizeof(payload);

    ssize_t n = readv(fd, iov, 2);  // ONE syscall reads into BOTH buffers
    std::cout << "readv returned: " << n << " bytes\n";  // 16

    // Parse header (big-endian length)
    uint32_t len = (static_cast<uint8_t>(header[0]) << 24)
                 | (static_cast<uint8_t>(header[1]) << 16)
                 | (static_cast<uint8_t>(header[2]) << 8)
                 | (static_cast<uint8_t>(header[3]));
    std::cout << "Header (length): " << len << '\n';  // 12
    std::cout << "Payload: " << std::string(payload, len) << '\n';  // Hello World!

    close(fd);
    unlink("/tmp/scatter_test.bin");
}
// Output:
//   readv returned: 16 bytes
//   Header (length): 12
//   Payload: Hello World!
```

### Q2: Use writev() to gather disjoint regions into a single send

The `send_framed` function here is the workhorse pattern you will use any time you have a fixed-size header and a variable-length payload. It encodes the length header into a stack buffer, builds a two-element `iovec`, and sends both in one `writev` call. The while loop handles partial writes - `writev` on a socket is not guaranteed to write everything at once, so you adjust the `iovec` pointers and lengths and retry.

<!-- compile: needs POSIX header `unistd.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <cstdint>
#include <unistd.h>
#include <sys/uio.h>
#include <sys/socket.h>
#include <netinet/in.h>

// Send a length-prefixed message without copying header+payload together
void send_framed(int fd, const void* payload, uint32_t len) {
    // Encode header (4 bytes, big-endian)
    uint8_t header[4] = {
        static_cast<uint8_t>((len >> 24) & 0xFF),
        static_cast<uint8_t>((len >> 16) & 0xFF),
        static_cast<uint8_t>((len >>  8) & 0xFF),
        static_cast<uint8_t>((len >>  0) & 0xFF),
    };

    // Gather write: header + payload in ONE syscall, ZERO copies
    struct iovec iov[2];
    iov[0].iov_base = header;
    iov[0].iov_len  = 4;
    iov[1].iov_base = const_cast<void*>(payload);  // writev takes non-const
    iov[1].iov_len  = len;

    ssize_t total = 0;
    while (total < static_cast<ssize_t>(4 + len)) {
        ssize_t n = writev(fd, iov, 2);
        if (n < 0) { perror("writev"); return; }
        total += n;
        // Handle partial writes by adjusting iov
        size_t written = n;
        for (int i = 0; i < 2; ++i) {
            if (written >= iov[i].iov_len) {
                written -= iov[i].iov_len;
                iov[i].iov_len = 0;
            } else {
                iov[i].iov_base = static_cast<char*>(iov[i].iov_base) + written;
                iov[i].iov_len -= written;
                break;
            }
        }
    }
}

int main() {
    // Demo using pipe instead of network socket
    int pipefd[2];
    pipe(pipefd);

    const char* msg = "Hello scatter-gather!";
    send_framed(pipefd[1], msg, strlen(msg));

    // Read back
    char buf[128];
    ssize_t n = read(pipefd[0], buf, sizeof(buf));
    std::cout << "Read " << n << " bytes (4 header + "
              << strlen(msg) << " payload)\n";

    close(pipefd[0]);
    close(pipefd[1]);
}
// Output: Read 24 bytes (4 header + 20 payload)
```

### Q3: How scatter-gather reduces memcpy in protocol parsing

Let's make the benefit explicit with a side-by-side comparison. Without scatter-gather, parsing a protocol message involves reading everything into an intermediate buffer and then copying the pieces out. With scatter-gather, the data lands directly in its destination - the kernel does the splitting for you.

**Without scatter-gather:**

```cpp
Recv buffer:  [header1][payload1][header2][payload2]
                  |
                  v  Parse header, then memcpy payload to separate buffer
  Header buf: [header1]
  Payload buf: [payload1]     <- COPY!
```

**With scatter-gather (readv):**

```cpp
Recv via readv:
  iov[0] -> [header1]    direct into header buffer (no copy)
  iov[1] -> [payload1]   direct into payload buffer (no copy)
```

**Protocol parser benefits:**

1. **No intermediate buffer**: readv places header and payload directly into separate typed buffers.
2. **Fewer syscalls**: one `readv()` vs two `read()` calls.
3. **No memcpy**: data goes directly from kernel to destination buffers.
4. **Better cache locality**: each buffer is sized and aligned for its purpose.

Here is the same idea with concrete struct types. Notice how the scatter-gather version fills the struct fields in a single syscall with zero copying:

```cpp
// Example: HTTP-like protocol parser
struct ProtocolMessage {
    uint32_t header[4];     // 16-byte header (type, flags, seqno, length)
    char payload[4096];     // payload buffer
};

// Traditional (2 copies):
//   char tmp[4112];
//   read(fd, tmp, 4112);
//   memcpy(msg.header, tmp, 16);       // copy 1
//   memcpy(msg.payload, tmp + 16, n);  // copy 2

// Scatter-gather (0 copies):
//   iov[0] = { msg.header,  16 };
//   iov[1] = { msg.payload, 4096 };
//   readv(fd, iov, 2);  // data lands directly in struct fields!
```

For sending, the same benefit applies: you can prepend a header to an existing payload buffer without copying the payload into a new contiguous buffer. This is critical for high-throughput servers (100K+ msgs/sec) where memcpy becomes a measurable bottleneck.

---

## Notes

- Complementary to #553 (scatter-gather basics + sendfile).
- `writev()` may do partial writes - always loop and adjust iov pointers.
- Maximum iov count: `sysconf(_SC_IOV_MAX)` (typically 1024 on Linux).
- Windows equivalent: `WSASend()`/`WSARecv()` with `WSABUF` arrays.
- For network sockets, `sendmsg()`/`recvmsg()` extend scatter-gather with ancillary data (SCM_RIGHTS, etc.).
