# Wrap POSIX sockets in RAII and use them safely from C++

**Category:** Networking and IO  
**Standard:** C++11  
**Reference:** <https://man7.org/linux/man-pages/man2/socket.2.html>  

---

## Topic Overview

This topic focuses on building an RAII Socket class and using it for a TCP client - complementing #549 (move-only socket + socket options + EINTR) and #638 (TCP server with unique_ptr closer).

The core idea is simple: a file descriptor is a resource, so it needs an owner. The RAII Socket class is that owner. Once a descriptor lives inside a `Socket`, you cannot forget to close it because the destructor does it for you. Early returns, exceptions, and scope exits all trigger the destructor - you get cleanup for free regardless of how the code exits.

```cpp
RAII socket lifecycle:

  Socket sock(AF_INET, SOCK_STREAM);   // constructor: socket()
      |                                 // fd is owned
      v
  sock.connect(endpoint);               // use the socket
  sock.send(data);                      // send data
  auto resp = sock.recv(1024);          // receive data
      |                                 // fd still owned
      v
  } // destructor: close(fd)            // ALWAYS cleaned up!

  Exception thrown at any point -> destructor runs -> fd closed
  Early return at any point -> destructor runs -> fd closed
  No leak possible!
```

---

## Self-Assessment

### Q1: RAII Socket class

Here is a minimal but complete `Socket` class. The critical decisions are: throw in the constructor if the OS call fails (so you never hold an invalid descriptor), delete the copy operations (you cannot meaningfully duplicate a file descriptor), and implement move so the class can transfer ownership into containers and return values.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <stdexcept>
#include <cstring>
#include <string>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

class Socket {
public:
    // Create a new socket
    Socket(int domain, int type, int protocol = 0)
        : fd_(::socket(domain, type, protocol)) {
        if (fd_ < 0)
            throw std::runtime_error(
                std::string("socket(): ") + strerror(errno));
    }

    // Take ownership of existing fd
    explicit Socket(int fd) : fd_(fd) {
        if (fd_ < 0)
            throw std::runtime_error("Invalid fd");
    }

    // Destructor: close the fd
    ~Socket() {
        if (fd_ >= 0) {
            ::close(fd_);  // ignoring error on close is acceptable
            fd_ = -1;
        }
    }

    // Move-only (no copying file descriptors!)
    Socket(const Socket&) = delete;
    Socket& operator=(const Socket&) = delete;

    Socket(Socket&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;  // source no longer owns
    }

    Socket& operator=(Socket&& other) noexcept {
        if (this != &other) {
            if (fd_ >= 0) ::close(fd_);
            fd_ = other.fd_;
            other.fd_ = -1;
        }
        return *this;
    }

    int fd() const { return fd_; }

    // Release ownership (caller responsible for closing)
    int release() {
        int fd = fd_;
        fd_ = -1;
        return fd;
    }

private:
    int fd_;
};

int main() {
    // Demonstrate RAII
    {
        Socket sock(AF_INET, SOCK_STREAM);
        std::cout << "Created socket fd=" << sock.fd() << '\n';

        // Move ownership
        Socket sock2 = std::move(sock);
        std::cout << "Moved to sock2 fd=" << sock2.fd() << '\n';
        std::cout << "Original fd=" << sock.fd() << " (released)\n";
    }
    // Both destructors run; only sock2 had a valid fd -> close() called once
    std::cout << "Sockets destroyed (RAII cleanup)\n";
}
// Output:
//   Created socket fd=3
//   Moved to sock2 fd=3
//   Original fd=-1 (released)
//   Sockets destroyed (RAII cleanup)
```

After the move, `sock.fd()` returns `-1` because the source deliberately sets its descriptor to `-1` in the move constructor. The destructor checks for `-1` before calling `close`, so the moved-from object's destructor is a no-op. That is the standard pattern for RAII move semantics.

### Q2: TCP client using RAII socket

Here is the `Socket` class put to practical use as a TCP client. Notice that the `try`/`catch` in `main` is not doing any resource management - it just prints the error. The resource management happens automatically through RAII regardless of whether the connection succeeds, the send fails, or the receive throws.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <string>
#include <stdexcept>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

class Socket {
public:
    Socket(int domain, int type, int proto = 0)
        : fd_(::socket(domain, type, proto)) {
        if (fd_ < 0) throw std::runtime_error(strerror(errno));
    }
    explicit Socket(int fd) : fd_(fd) {}
    ~Socket() { if (fd_ >= 0) ::close(fd_); }
    Socket(Socket&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
    Socket& operator=(Socket&&) = delete;
    Socket(const Socket&) = delete;
    int fd() const { return fd_; }

    void connect(const char* ip, uint16_t port) {
        sockaddr_in addr{};
        addr.sin_family = AF_INET;
        addr.sin_port = htons(port);
        inet_pton(AF_INET, ip, &addr.sin_addr);
        if (::connect(fd_, (sockaddr*)&addr, sizeof(addr)) < 0)
            throw std::runtime_error(
                std::string("connect: ") + strerror(errno));
    }

    ssize_t send(const void* data, size_t len) {
        ssize_t n = ::send(fd_, data, len, 0);
        if (n < 0) throw std::runtime_error(strerror(errno));
        return n;
    }

    std::string recv(size_t max_len) {
        std::string buf(max_len, '\0');
        ssize_t n = ::recv(fd_, buf.data(), buf.size(), 0);
        if (n < 0) throw std::runtime_error(strerror(errno));
        buf.resize(n);
        return buf;
    }

private:
    int fd_;
};

int main() {
    try {
        Socket client(AF_INET, SOCK_STREAM);
        client.connect("93.184.216.34", 80);  // example.com

        const char* request = "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n";
        client.send(request, strlen(request));

        std::string response = client.recv(4096);
        std::cout << "Response (" << response.size() << " bytes):\n";
        std::cout << response.substr(0, 200) << "...\n";
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << '\n';
    }
    // Socket automatically closed by destructor, even on exception!
}
```

### Q3: RAII prevents fd leaks

This example makes the leak scenario concrete so you can see exactly what goes wrong without RAII and exactly what RAII fixes. The `without_raii` function has a file descriptor that never gets closed if an exception is thrown, which is a resource leak. The `with_raii` version lets the destructor handle it.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <stdexcept>
#include <sys/socket.h>
#include <unistd.h>

class Socket {
public:
    Socket(int domain, int type, int proto = 0)
        : fd_(::socket(domain, type, proto)) {}
    ~Socket() {
        if (fd_ >= 0) {
            std::cout << "  ~Socket: closing fd=" << fd_ << '\n';
            ::close(fd_);
        }
    }
    Socket(Socket&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
    Socket(const Socket&) = delete;
    int fd() const { return fd_; }
private:
    int fd_;
};

void without_raii() {
    // BAD: fd leak on exception!
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    // ... some work ...
    throw std::runtime_error("oops");
    close(fd);  // NEVER REACHED! fd leaked!
}

void with_raii() {
    Socket sock(AF_INET, SOCK_STREAM);
    std::cout << "  Created fd=" << sock.fd() << '\n';
    // ... some work ...
    throw std::runtime_error("oops");
    // Destructor ALWAYS runs -> fd closed!
}

void early_return_demo() {
    Socket sock(AF_INET, SOCK_STREAM);
    std::cout << "  Created fd=" << sock.fd() << '\n';
    if (true) {
        std::cout << "  Early return!\n";
        return;  // destructor runs -> fd closed
    }
    // ... more code ...
}

int main() {
    std::cout << "Without RAII (fd leak):\n";
    try { without_raii(); }
    catch (...) { std::cout << "  Exception caught, but fd leaked!\n"; }

    std::cout << "\nWith RAII (no leak):\n";
    try { with_raii(); }
    catch (...) { std::cout << "  Exception caught, fd was closed!\n"; }

    std::cout << "\nEarly return (no leak):\n";
    early_return_demo();
}
// Output:
//   Without RAII (fd leak):
//     Exception caught, but fd leaked!
//   With RAII (no leak):
//     Created fd=4
//     ~Socket: closing fd=4
//     Exception caught, fd was closed!
//   Early return (no leak):
//     Created fd=4
//     Early return!
//     ~Socket: closing fd=4
```

File descriptor leaks are one of the nastier bugs to track down in long-running servers because they accumulate slowly. The process gradually uses up its file descriptor limit until new connections fail with "too many open files". RAII eliminates this entire class of bug.

---

## Notes

- Complementary to #549 (move-only + socket options + EINTR) and #638 (TCP server).
- Rule of five: if you write a destructor, you need move constructor + move assignment (and delete copy).
- `release()` transfers ownership out of RAII (like `unique_ptr::release()`).
- For production: use Asio which handles all this internally.
