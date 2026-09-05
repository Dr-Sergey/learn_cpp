# Implement zero-copy I/O with scatter-gather using iovec and readv/writev

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://man7.org/linux/man-pages/man2/readv.2.html>  

---

## Topic Overview

This topic extends scatter-gather basics (#641) to cover protocol framing with iovec and the `sendfile()` syscall for kernel-to-kernel zero-copy transfer.

The central question in high-throughput I/O is: how many times does data get copied as it moves from disk or memory to the network? Every copy burns CPU time and memory bandwidth. The techniques below are ranked from best (fewest copies) to worst, so you can pick the right tool for the job.

```cpp
Zero-copy spectrum (least to most copies):
  sendfile:   file -> kernel -> NIC    (0 user-space copies)
  splice:     pipe -> kernel -> socket (0 user-space copies)
  mmap+write: file -> mapped -> socket (1 copy: page cache -> socket buf)
  read+write: file -> user -> socket   (2 copies)

scatter-gather with iovec:
  struct iovec {
      void  *iov_base;  // buffer start
      size_t iov_len;   // buffer length
  };
  writev(fd, iov, count);  // gather write
  readv(fd, iov, count);   // scatter read
```

The table below summarises each technique along two dimensions that matter most in practice - how many copies happen and how many syscalls you spend:

| Technique | Copies | Syscalls | Use case |
| --- | --- | --- | --- |
| readv/writev | 0 (scatter/gather) | 1 | Protocol framing |
| sendfile | 0 | 1 | Static file serving |
| splice | 0 | 1 | Pipe-to-socket proxy |
| mmap + write | 1 | 1 | Random-access file I/O |

---

## Self-Assessment

### Q1: readv/writev scatter-gather across non-contiguous buffers

The classic use case is a network message with a fixed-size header struct and a variable payload sitting in different memory locations. Without `writev` you would need to either copy them into a single contiguous buffer first (wasting memory and CPU) or issue two separate `write` calls (wasting syscalls). With `writev` you just describe each piece using an `iovec` element and the kernel stitches them together in one shot.

<!-- compile: needs POSIX header `unistd.h` -->
```cpp
// Linux/POSIX only
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <sys/uio.h>
#include <fcntl.h>

int main() {
    // Scenario: reassemble a fragmented network message
    // We have a header struct and payload in different memory locations
    struct Header {
        uint32_t type;
        uint32_t length;
    };

    Header hdr = {1, 13};  // type=1, length=13
    char payload[] = "Hello, World!";
    char trailer[] = "\n";

    // === WRITEV: gather write from 3 disjoint buffers ===
    int fd = open("/tmp/iovec_test.bin", O_CREAT | O_RDWR | O_TRUNC, 0644);

    struct iovec wv[3];
    wv[0].iov_base = &hdr;
    wv[0].iov_len  = sizeof(hdr);       // 8 bytes
    wv[1].iov_base = payload;
    wv[1].iov_len  = strlen(payload);   // 13 bytes
    wv[2].iov_base = trailer;
    wv[2].iov_len  = 1;                 // 1 byte

    ssize_t written = writev(fd, wv, 3);  // ONE syscall, 3 buffers, 0 copies
    std::cout << "writev: " << written << " bytes written\n";  // 22

    // === READV: scatter read into separate buffers ===
    lseek(fd, 0, SEEK_SET);

    Header hdr_read{};
    char pay_read[64]{};
    char trail_read[4]{};

    struct iovec rv[3];
    rv[0].iov_base = &hdr_read;
    rv[0].iov_len  = sizeof(hdr_read);
    rv[1].iov_base = pay_read;
    rv[1].iov_len  = 13;  // we know length from header
    rv[2].iov_base = trail_read;
    rv[2].iov_len  = 1;

    ssize_t nread = readv(fd, rv, 3);
    std::cout << "readv: " << nread << " bytes read\n";
    std::cout << "  type=" << hdr_read.type << " length=" << hdr_read.length << '\n';
    std::cout << "  payload: " << std::string(pay_read, 13) << '\n';

    close(fd);
    unlink("/tmp/iovec_test.bin");
}
// Output:
//   writev: 22 bytes written
//   readv: 22 bytes read
//     type=1 length=13
//     payload: Hello, World!
```

Notice that the read side knows how much payload to expect because it reads the length out of the header first - that is the standard length-prefix framing pattern. The kernel scatters the incoming bytes directly into the right buffers; your header struct is filled with the first 8 bytes, the payload array receives the next 13, and so on.

### Q2: Protocol framing with two-element iovec

Length-prefixed framing is everywhere in network protocols. The sender writes a fixed-size header containing the payload length, then the payload. Without `writev` you would typically concatenate them into a temporary buffer, which is an unnecessary copy. Here the `FramedWriter` uses a two-element iovec so the header and payload travel together in a single syscall.

<!-- compile: needs POSIX header `unistd.h` -->
```cpp
#include <iostream>
#include <cstdint>
#include <cstring>
#include <vector>
#include <unistd.h>
#include <sys/uio.h>

// Protocol: 4-byte big-endian length prefix + payload
// Use iovec to send header + payload without concatenation

class FramedWriter {
public:
    explicit FramedWriter(int fd) : fd_(fd) {}

    // Send a length-prefixed message using writev (zero copy)
    bool send(const void* data, uint32_t len) {
        uint8_t header[4];
        header[0] = (len >> 24) & 0xFF;
        header[1] = (len >> 16) & 0xFF;
        header[2] = (len >>  8) & 0xFF;
        header[3] = (len >>  0) & 0xFF;

        struct iovec iov[2];
        iov[0].iov_base = header;
        iov[0].iov_len  = 4;
        iov[1].iov_base = const_cast<void*>(data);
        iov[1].iov_len  = len;

        // writev is atomic for <= PIPE_BUF (4096) on pipes
        // For sockets, may need to handle partial writes
        ssize_t n = writev(fd_, iov, 2);
        return n == static_cast<ssize_t>(4 + len);
    }

private:
    int fd_;
};

class FramedReader {
public:
    explicit FramedReader(int fd) : fd_(fd) {}

    // Receive a length-prefixed message using readv
    std::vector<uint8_t> recv() {
        // Read header separately (we need length before payload)
        uint8_t header[4];
        if (read(fd_, header, 4) != 4) return {};

        uint32_t len = (static_cast<uint32_t>(header[0]) << 24)
                     | (static_cast<uint32_t>(header[1]) << 16)
                     | (static_cast<uint32_t>(header[2]) << 8)
                     | (static_cast<uint32_t>(header[3]));

        std::vector<uint8_t> payload(len);
        if (read(fd_, payload.data(), len) != static_cast<ssize_t>(len)) return {};
        return payload;
    }

private:
    int fd_;
};

int main() {
    int pipefd[2];
    pipe(pipefd);

    FramedWriter writer(pipefd[1]);
    FramedReader reader(pipefd[0]);

    writer.send("Hello", 5);
    writer.send("World!", 6);

    auto msg1 = reader.recv();
    auto msg2 = reader.recv();
    std::cout << "msg1: " << std::string(msg1.begin(), msg1.end()) << '\n';  // Hello
    std::cout << "msg2: " << std::string(msg2.begin(), msg2.end()) << '\n';  // World!

    close(pipefd[0]);
    close(pipefd[1]);
}
```

The reader has to do its header and payload reads separately because it does not know the payload length until it has parsed the header - that is the one place where a two-step read is unavoidable. Once you have the length, you could use a two-element `readv` to read header-of-next-message and payload together if you want to pipeline reads.

### Q3: sendfile() for kernel-to-kernel zero-copy

`sendfile` is the gold standard for serving static files. The comments in the code walk through the exact data path - the key point is that with a modern DMA-capable NIC, the CPU never has to touch the file data at all. It just tells the kernel "send bytes from this file descriptor to that socket descriptor" and the hardware does the rest.

<!-- compile: needs POSIX header `unistd.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <fcntl.h>
#include <sys/sendfile.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <netinet/in.h>

// sendfile() copies data from one FD to another ENTIRELY in kernel space.
// No data ever enters user space -> true zero-copy.

// Traditional file serving (2 copies, 4 context switches):
//   read(file_fd, buf, n);   // kernel -> user (copy 1)
//   write(sock_fd, buf, n);  // user -> kernel (copy 2)

// sendfile (0 copies, 2 context switches):
//   sendfile(sock_fd, file_fd, &offset, n);  // kernel -> kernel directly!

// With DMA + scatter-gather NIC:
//   sendfile uses DMA from disk -> page cache -> NIC buffer
//   CPU never touches the data at all!

int main() {
    // Create a test file
    int file_fd = open("/tmp/sendfile_test.txt", O_CREAT | O_RDWR | O_TRUNC, 0644);
    const char* content = "This is served via sendfile - zero copy!\n";
    write(file_fd, content, strlen(content));

    struct stat st;
    fstat(file_fd, &st);
    lseek(file_fd, 0, SEEK_SET);

    // In a real server:
    //   sendfile(client_sock, file_fd, &offset, st.st_size);
    // This sends the entire file without it ever entering user memory.

    std::cout << "sendfile() zero-copy data path:\n";
    std::cout << "  1. DMA: disk -> kernel page cache\n";
    std::cout << "  2. Kernel: page cache ref -> socket buffer (no copy!)\n";
    std::cout << "  3. DMA: socket buffer -> NIC\n";
    std::cout << "\nCompared to read()+write():\n";
    std::cout << "  1. DMA: disk -> kernel page cache\n";
    std::cout << "  2. Copy: page cache -> user buffer (COPY!)\n";
    std::cout << "  3. Copy: user buffer -> socket buffer (COPY!)\n";
    std::cout << "  4. DMA: socket buffer -> NIC\n";
    std::cout << "\nFor a 1GB file: saves ~2GB of memory bandwidth!\n";

    close(file_fd);
    unlink("/tmp/sendfile_test.txt");
}
```

The bandwidth saving highlighted at the end is not hypothetical. For a 1 GB file, `read`+`write` moves 1 GB from kernel to user space and then 1 GB back - that is 2 GB of memory bus traffic just for copying. `sendfile` skips both of those trips. At scale, this translates directly to lower CPU utilisation and higher sustainable throughput.

---

## Notes

- Complementary to #641 (scatter-gather patterns + protocol parsing).
- `sendfile()` works only file-to-socket (not socket-to-socket); use `splice()` for pipes.
- `writev()` atomicity: guaranteed atomic for `<= PIPE_BUF` on pipes, not on sockets.
- Windows equivalent: `TransmitFile()` for sendfile, `WSASend()` with `WSABUF` for scatter-gather.
- DPDK and io_uring provide even more zero-copy options (registered buffers).
