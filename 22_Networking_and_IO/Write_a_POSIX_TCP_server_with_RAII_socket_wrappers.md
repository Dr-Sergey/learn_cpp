# Write a POSIX TCP server with RAII socket wrappers

**Category:** Networking and IO  
**Standard:** C++11  
**Reference:** <https://man7.org/linux/man-pages/man7/socket.7.html>  

---

## Topic Overview

This topic focuses on building a complete TCP echo server using `unique_ptr` with custom deleters for RAII - complementing #726 (basic RAII class + TCP client) and #549 (move-only + socket options + EINTR).

The `unique_ptr` with a custom deleter is an alternative to writing a full Socket class. It is lighter - you do not need to define a class at all - but slightly awkward because `unique_ptr<int, Closer>` stores a pointer to an int, which means a heap allocation for each descriptor. For a server socket or a handful of client sockets that is fine. For a server managing thousands of connections you would prefer a dedicated class. Both approaches are worth knowing.

```cpp
TCP server lifecycle:

  socket()  -> bind()  -> listen()  -> accept() loop
  (create)    (addr)     (backlog)    (wait for clients)
                                          |
                                     +---------+
                                     | client  |  read/write
                                     | socket  |  echo loop
                                     +---------+
                                          |
                                     close(client)

  RAII: unique_ptr<int, SocketCloser> ensures every fd is closed
```

---

## Self-Assessment

### Q1: unique_ptr with custom closer for socket fd

The custom deleter calls `close()` on the descriptor and then frees the heap-allocated int. The output shows move semantics working correctly - after moving, the original pointer is null and only the destination pointer owns the descriptor.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <memory>
#include <sys/socket.h>
#include <unistd.h>

// Custom deleter that calls close()
struct SocketCloser {
    void operator()(int* fd_ptr) const {
        if (fd_ptr && *fd_ptr >= 0) {
            std::cout << "  Closing fd=" << *fd_ptr << '\n';
            ::close(*fd_ptr);
        }
        delete fd_ptr;
    }
};

using UniqueSocket = std::unique_ptr<int, SocketCloser>;

UniqueSocket make_socket(int domain, int type, int proto = 0) {
    int fd = ::socket(domain, type, proto);
    if (fd < 0) return nullptr;
    return UniqueSocket(new int(fd));
}

int main() {
    auto sock = make_socket(AF_INET, SOCK_STREAM);
    if (!sock) {
        std::cerr << "Failed to create socket\n";
        return 1;
    }
    std::cout << "Created socket fd=" << *sock << '\n';

    // Transfer ownership
    auto sock2 = std::move(sock);
    std::cout << "Moved ownership\n";
    std::cout << "sock is null: " << (sock == nullptr) << '\n';
    std::cout << "sock2 fd=" << *sock2 << '\n';
}
// Output:
//   Created socket fd=3
//   Moved ownership
//   sock is null: 1
//   sock2 fd=3
//   Closing fd=3
```

The `Closing fd=3` message appears at scope exit, which demonstrates that the destructor is running exactly when `sock2` goes out of scope - not earlier, not later. That predictable timing is what makes RAII reliable.

### Q2: Echo server: socket, bind, listen, accept loop

Now here is the full working server. Each step of the TCP server lifecycle - create, bind, listen, accept, echo, close - is shown in order. Pay attention to how `wrap_fd` is used for both the server socket and each accepted client socket: every descriptor that enters the program immediately gets an RAII owner.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <memory>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <cerrno>

struct FDCloser {
    void operator()(int* p) const {
        if (p && *p >= 0) ::close(*p);
        delete p;
    }
};
using UniqueFD = std::unique_ptr<int, FDCloser>;

UniqueFD wrap_fd(int fd) {
    return (fd >= 0) ? UniqueFD(new int(fd)) : nullptr;
}

int main() {
    // 1. Create server socket
    auto server = wrap_fd(socket(AF_INET, SOCK_STREAM, 0));
    if (!server) { perror("socket"); return 1; }

    // 2. Set SO_REUSEADDR (avoids "Address already in use")
    int opt = 1;
    setsockopt(*server, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 3. Bind
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    addr.sin_addr.s_addr = INADDR_ANY;
    if (bind(*server, (sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind"); return 1;
    }

    // 4. Listen (backlog = 128)
    if (listen(*server, 128) < 0) {
        perror("listen"); return 1;
    }

    std::cout << "Echo server on port 8080\n";

    // 5. Accept loop
    for (;;) {
        auto client = wrap_fd(accept(*server, nullptr, nullptr));
        if (!client) {
            if (errno == EINTR) continue;  // signal, retry
            perror("accept"); break;
        }

        std::cout << "Client connected (fd=" << *client << ")\n";

        // Echo loop
        char buf[1024];
        ssize_t n;
        while ((n = recv(*client, buf, sizeof(buf), 0)) > 0) {
            // Send back what we received
            ssize_t sent = 0;
            while (sent < n) {
                ssize_t s = send(*client, buf + sent, n - sent, 0);
                if (s < 0) {
                    if (errno == EINTR) continue;
                    perror("send"); break;
                }
                sent += s;
            }
        }

        std::cout << "Client disconnected\n";
        // client's unique_ptr destructor closes the fd automatically!
    }
    // server's unique_ptr destructor closes the listen fd!
}
// Test: echo "hello" | nc localhost 8080
```

The inner send loop (`while (sent < n)`) is important and easy to overlook. `send` can return a short count on a non-blocking socket or under memory pressure - it does not guarantee that all `n` bytes were sent. Looping until `sent == n` handles partial writes correctly.

### Q3: EINTR retry on blocking syscalls

This Q&A isolates the EINTR retry patterns so you can see them cleanly. The key rule: if a syscall returns `-1` and `errno == EINTR`, it is safe to call it again. The I/O did not happen; a signal interrupted the wait. There is one important exception at the end of the listing - `close` on Linux.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <cerrno>
#include <sys/socket.h>
#include <unistd.h>

// EINTR happens when a signal (like SIGCHLD) interrupts a blocking syscall.
// The syscall returns -1 with errno == EINTR.
// Safe to retry immediately.

// Wrapper: accept with EINTR retry
int safe_accept(int server_fd) {
    int client_fd;
    do {
        client_fd = ::accept(server_fd, nullptr, nullptr);
    } while (client_fd < 0 && errno == EINTR);
    return client_fd;
}

// Wrapper: recv with EINTR retry
ssize_t safe_recv(int fd, void* buf, size_t len, int flags = 0) {
    ssize_t n;
    do {
        n = ::recv(fd, buf, len, flags);
    } while (n < 0 && errno == EINTR);
    return n;
}

// Wrapper: send all data with EINTR retry
ssize_t safe_send_all(int fd, const void* data, size_t len) {
    const char* ptr = static_cast<const char*>(data);
    size_t remaining = len;
    while (remaining > 0) {
        ssize_t n = ::send(fd, ptr, remaining, 0);
        if (n < 0) {
            if (errno == EINTR) continue;  // signal, retry
            return -1;  // real error
        }
        ptr += n;
        remaining -= n;
    }
    return len;
}

int main() {
    std::cout << "EINTR retry patterns for blocking syscalls:\n\n";

    std::cout << "accept: retry if client_fd < 0 && errno == EINTR\n";
    std::cout << "recv:   retry if n < 0 && errno == EINTR\n";
    std::cout << "send:   retry partial sends AND EINTR\n";
    std::cout << "connect: EINTR is trickier - check with SO_ERROR via getsockopt\n";
    std::cout << "close:  do NOT retry on Linux (fd already closed even on EINTR)\n";

    std::cout << "\nWhich syscalls auto-restart with SA_RESTART?\n";
    std::cout << "  Auto-restart: read, write, recv, send (most I/O)\n";
    std::cout << "  NOT auto-restart: accept, select, poll, epoll_wait\n";
    std::cout << "  -> Always handle EINTR explicitly for accept/poll!\n";
}
```

The `close` note at the end is genuinely tricky. On Linux, if `close` returns `EINTR`, the file descriptor has already been released by the kernel - retrying `close` would either be a no-op or accidentally close a descriptor that a different thread has since allocated with the same number. Do not retry `close` on Linux.

---

## Notes

- Complementary to #726 (RAII Socket class + TCP client) and #549 (move-only + socket options).
- `unique_ptr<int, Closer>` is a valid RAII pattern but slightly awkward (heap-allocates int). A dedicated Socket class (see #726, #549) is cleaner.
- SO_REUSEADDR is mandatory for server sockets that restart frequently.
- For concurrent clients: fork(), threads, or epoll (see #639, #555).
- close() on Linux: do NOT retry on EINTR (the fd is already released).
