# Wrap POSIX sockets in RAII for safe resource management

**Category:** Networking and IO  
**Standard:** C++11  
**Reference:** <https://man7.org/linux/man-pages/man2/socket.2.html>  

---

## Topic Overview

This topic focuses on move-only semantics, socket options, and EINTR handling in RAII socket wrappers - complementing #726 (basic RAII + TCP client + leak prevention) and #638 (TCP server with unique_ptr).

Move-only ownership is the right model for file descriptors because a descriptor is a unique kernel resource - there is no meaningful way to "copy" it. Deleting the copy operations enforces this in the type system, so the compiler will tell you immediately if you accidentally try to duplicate a socket. The EINTR retry pattern is a separate concern: the OS delivers a signal to your thread, the blocked syscall returns early with `EINTR`, and you need to retry the call because the I/O did not actually complete.

```cpp
Move-only socket ownership:

  Socket a(AF_INET, SOCK_STREAM);   // a owns fd=3
  Socket b = std::move(a);           // b owns fd=3, a.fd_=-1
  // Socket c = b;                   // COMPILE ERROR: deleted copy

  EINTR retry pattern:
  ssize_t n;
  do {
      n = ::recv(fd, buf, len, 0);
  } while (n < 0 && errno == EINTR);  // signal interrupted, retry
```

Here is a reference table for the most useful socket options you will set in practice. These are the knobs that separate a toy socket from a production-quality one:

| Socket option | Level | What it does |
| --- | --- | --- |
| SO_REUSEADDR | SOL_SOCKET | Allow bind to port in TIME_WAIT |
| SO_REUSEPORT | SOL_SOCKET | Multiple sockets bind to same port |
| TCP_NODELAY | IPPROTO_TCP | Disable Nagle (send immediately) |
| SO_KEEPALIVE | SOL_SOCKET | Detect dead connections |
| SO_RCVBUF/SO_SNDBUF | SOL_SOCKET | Set buffer sizes |
| SO_LINGER | SOL_SOCKET | Control close() behavior |

---

## Self-Assessment

### Q1: Move-only RAII Socket class

This design is slightly more sophisticated than the basic version in #726. It uses a private tag type to distinguish the "adopt existing fd" constructor from the "create new socket" constructor, which avoids the ambiguity of having two `int`-taking constructors.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <utility>
#include <cstring>
#include <sys/socket.h>
#include <unistd.h>

class Socket {
public:
    // Create new socket
    Socket(int domain, int type, int protocol = 0)
        : fd_(::socket(domain, type, protocol)) {
        if (fd_ < 0)
            throw std::runtime_error(strerror(errno));
    }

    // Adopt existing fd
    static Socket from_fd(int fd) { return Socket(fd, adopt_tag{}); }

    // Destructor
    ~Socket() { close(); }

    // Move constructor
    Socket(Socket&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;
    }

    // Move assignment
    Socket& operator=(Socket&& other) noexcept {
        if (this != &other) {
            close();           // close current fd
            fd_ = other.fd_;   // take ownership
            other.fd_ = -1;    // release source
        }
        return *this;
    }

    // Delete copy (you cannot duplicate a file descriptor safely)
    Socket(const Socket&) = delete;
    Socket& operator=(const Socket&) = delete;

    int fd() const { return fd_; }
    bool valid() const { return fd_ >= 0; }
    explicit operator bool() const { return valid(); }

private:
    struct adopt_tag {};
    Socket(int fd, adopt_tag) : fd_(fd) {}

    void close() {
        if (fd_ >= 0) {
            ::close(fd_);
            fd_ = -1;
        }
    }

    int fd_ = -1;
};

int main() {
    Socket a(AF_INET, SOCK_STREAM);
    std::cout << "a.fd=" << a.fd() << '\n';

    Socket b = std::move(a);  // move ownership
    std::cout << "After move: a.fd=" << a.fd()
              << " b.fd=" << b.fd() << '\n';

    // Socket c = b;  // ERROR: copy deleted

    b = Socket(AF_INET, SOCK_STREAM);  // move-assign: old fd closed
    std::cout << "After reassign: b.fd=" << b.fd() << '\n';
}
// Output:
//   a.fd=3
//   After move: a.fd=-1 b.fd=3
//   After reassign: b.fd=4
```

The `from_fd` factory is useful when you receive a raw descriptor from a C API - for example, from `accept()` - and want to immediately transfer it into RAII ownership. Calling `Socket::from_fd(raw_fd)` is cleaner and more explicit than constructing with a raw integer.

### Q2: Socket options through the wrapper

The templated `set_option` and `get_option` methods give you a type-safe way to call `setsockopt`/`getsockopt`. Rather than sprinkling `setsockopt` calls all over your connection setup code, each common option gets a named method. Notice that the kernel often silently adjusts the value you set - `SO_RCVBUF` is doubled internally as you can see in the output comment.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <stdexcept>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <unistd.h>

class Socket {
public:
    Socket(int domain, int type, int proto = 0)
        : fd_(::socket(domain, type, proto)) {}
    ~Socket() { if (fd_ >= 0) ::close(fd_); }
    Socket(Socket&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
    Socket(const Socket&) = delete;
    int fd() const { return fd_; }

    // Generic setsockopt wrapper
    template<typename T>
    void set_option(int level, int optname, const T& value) {
        if (setsockopt(fd_, level, optname, &value, sizeof(value)) < 0)
            throw std::runtime_error(
                std::string("setsockopt: ") + strerror(errno));
    }

    template<typename T>
    T get_option(int level, int optname) const {
        T value{};
        socklen_t len = sizeof(value);
        if (getsockopt(fd_, level, optname, &value, &len) < 0)
            throw std::runtime_error(strerror(errno));
        return value;
    }

    // Typed convenience methods
    void set_reuse_addr(bool on = true) {
        int val = on ? 1 : 0;
        set_option(SOL_SOCKET, SO_REUSEADDR, val);
    }

    void set_tcp_nodelay(bool on = true) {
        int val = on ? 1 : 0;
        set_option(IPPROTO_TCP, TCP_NODELAY, val);
    }

    void set_recv_buffer(int size) {
        set_option(SOL_SOCKET, SO_RCVBUF, size);
    }

    void set_keepalive(bool on = true) {
        int val = on ? 1 : 0;
        set_option(SOL_SOCKET, SO_KEEPALIVE, val);
    }

private:
    int fd_;
};

int main() {
    Socket sock(AF_INET, SOCK_STREAM);

    // Set options safely through the wrapper
    sock.set_reuse_addr();      // SO_REUSEADDR=1
    sock.set_tcp_nodelay();     // TCP_NODELAY=1 (disable Nagle)
    sock.set_recv_buffer(65536);  // SO_RCVBUF=64K
    sock.set_keepalive();       // SO_KEEPALIVE=1

    // Verify
    int nodelay = sock.get_option<int>(IPPROTO_TCP, TCP_NODELAY);
    int rcvbuf  = sock.get_option<int>(SOL_SOCKET, SO_RCVBUF);

    std::cout << "TCP_NODELAY: " << nodelay << '\n';
    std::cout << "SO_RCVBUF: "   << rcvbuf << '\n';
}
// Output:
//   TCP_NODELAY: 1
//   SO_RCVBUF: 131072  (kernel doubles the requested size)
```

### Q3: EINTR retry loop

EINTR is one of those POSIX details that bites you the first time you add a signal handler to a server. When a signal arrives while a thread is blocked in `recv` or `accept`, the kernel immediately returns from the syscall with `errno == EINTR`. Your code needs to detect this and retry, because the I/O did not happen - it was just interrupted. The examples below show the three variants you will encounter in practice.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <cerrno>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

class Socket {
public:
    Socket(int domain, int type, int proto = 0)
        : fd_(::socket(domain, type, proto)) {}
    ~Socket() { if (fd_ >= 0) ::close(fd_); }
    Socket(Socket&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
    Socket(const Socket&) = delete;
    int fd() const { return fd_; }

    // Send with EINTR retry
    ssize_t send_all(const void* data, size_t len) {
        const char* ptr = static_cast<const char*>(data);
        size_t remaining = len;

        while (remaining > 0) {
            ssize_t n = ::send(fd_, ptr, remaining, 0);
            if (n < 0) {
                if (errno == EINTR) continue;  // signal interrupted, retry!
                return -1;  // real error
            }
            ptr += n;
            remaining -= n;
        }
        return len;
    }

    // Recv with EINTR retry
    ssize_t recv_some(void* buf, size_t len) {
        ssize_t n;
        do {
            n = ::recv(fd_, buf, len, 0);
        } while (n < 0 && errno == EINTR);  // retry on signal
        return n;
    }

    // Accept with EINTR retry
    Socket accept() {
        int client_fd;
        do {
            client_fd = ::accept(fd_, nullptr, nullptr);
        } while (client_fd < 0 && errno == EINTR);

        if (client_fd < 0)
            throw std::runtime_error(strerror(errno));
        return Socket(client_fd);
    }

private:
    Socket(int fd) : fd_(fd) {}
    int fd_;
};

int main() {
    std::cout << "EINTR handling patterns:\n";
    std::cout << "  recv: do { n = recv(); } while (n<0 && errno==EINTR);\n";
    std::cout << "  send: if (n<0 && errno==EINTR) continue;\n";
    std::cout << "  accept: do { fd = accept(); } while (fd<0 && errno==EINTR);\n";
    std::cout << "\nWhy EINTR happens:\n";
    std::cout << "  A signal (SIGCHLD, SIGALRM, etc.) interrupts a blocking syscall.\n";
    std::cout << "  The syscall returns -1 with errno=EINTR.\n";
    std::cout << "  Solution: retry the syscall (it's safe).\n";
    std::cout << "  SA_RESTART flag makes some syscalls auto-restart,\n";
    std::cout << "  but NOT all (accept, select, poll don't auto-restart).\n";
}
```

The reason `accept` needs explicit EINTR handling even with `SA_RESTART` is that `SA_RESTART` does not cover all syscalls. The POSIX standard leaves it implementation-defined for some, and Linux does not auto-restart `accept`, `poll`, or `select`. Always handle EINTR explicitly for those.

---

## Notes

- Complementary to #726 (basic RAII + TCP client) and #638 (TCP server with unique_ptr).
- Always handle EINTR in production code, especially on servers with signal handlers.
- `SO_REUSEADDR` is essential for servers: without it, bind() fails for ~60s after restart (TIME_WAIT).
- TCP_NODELAY is critical for interactive protocols (SSH, HTTP/2); harmful for bulk transfers.
- Consider `SO_LINGER` with timeout=0 for immediate close (sends RST instead of FIN).
