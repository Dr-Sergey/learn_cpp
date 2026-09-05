# Design inter-process communication (IPC) architecture in C++

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**IPC** lets separate processes exchange data and coordinate. You reach for it when you need process isolation - a crash in one process should not take down another - or when you need to spread work across processes with different privilege levels. Chrome's process-per-tab model, daemon/client communication, and any distributed system all rely on some form of IPC.

The choice of mechanism depends on three factors: how low your latency requirement is, how much data you need to move, and which platforms you need to support. Shared memory is the fastest option but requires explicit synchronization. Unix domain sockets are slower but much simpler to use correctly. Named pipes are the most portable. Understanding those tradeoffs is what lets you pick the right tool for each situation.

### IPC Mechanisms Comparison

| Mechanism | Latency | Throughput | Platforms | Complexity |
| --- | --- | --- | --- | --- |
| **Shared Memory** | ~100ns | Highest (memcpy) | POSIX, Win32 | High (sync needed) |
| **Unix Domain Socket** | ~1µs | Very high | POSIX | Medium |
| **Named Pipe (FIFO)** | ~1µs | High | All | Low |
| **TCP Socket** | ~10µs+ | Network-bound | All | Medium |
| **Message Queues** | ~1µs | High | POSIX | Medium |
| **gRPC** | ~100µs | Network-bound | All | High (generated) |
| **Memory-mapped file** | ~100ns | Highest | All | Medium |

---

## Self-Assessment

### Q1: Implement shared memory IPC with synchronization

**Answer:**

Shared memory is the fastest IPC mechanism because there is no kernel copy - both processes map the same physical pages. The challenge is synchronization: you need to ensure the reader does not see a partial write. The ring buffer below uses a lock-free SPSC (single-producer, single-consumer) design with carefully chosen memory orderings. The `write_pos` and `read_pos` atomics are cache-line aligned to prevent false sharing.

```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>
#include <atomic>
#include <iostream>

// === Shared memory ring buffer for IPC ===
struct SharedBuffer {
    static constexpr size_t CAPACITY = 4096;

    // Lock-free SPSC (single producer, single consumer)
    alignas(64) std::atomic<size_t> write_pos{0};
    alignas(64) std::atomic<size_t> read_pos{0};
    char data[CAPACITY];

    bool write(const void* src, size_t len) {
        size_t w = write_pos.load(std::memory_order_relaxed);
        size_t r = read_pos.load(std::memory_order_acquire);
        size_t available = CAPACITY - (w - r);
        if (len + sizeof(size_t) > available) return false;  // Full

        size_t offset = w % CAPACITY;
        // Write length prefix + data
        std::memcpy(data + offset, &len, sizeof(size_t));
        std::memcpy(data + offset + sizeof(size_t), src, len);
        write_pos.store(w + sizeof(size_t) + len,
                        std::memory_order_release);
        return true;
    }

    bool read(void* dst, size_t& len) {
        size_t w = write_pos.load(std::memory_order_acquire);
        size_t r = read_pos.load(std::memory_order_relaxed);
        if (w == r) return false;  // Empty

        size_t offset = r % CAPACITY;
        std::memcpy(&len, data + offset, sizeof(size_t));
        std::memcpy(dst, data + offset + sizeof(size_t), len);
        read_pos.store(r + sizeof(size_t) + len,
                       std::memory_order_release);
        return true;
    }
};

// === Producer process ===
void producer() {
    int fd = shm_open("/my_ipc", O_CREAT | O_RDWR, 0666);
    ftruncate(fd, sizeof(SharedBuffer));
    auto* buf = static_cast<SharedBuffer*>(
        mmap(nullptr, sizeof(SharedBuffer),
             PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0));
    new (buf) SharedBuffer{};  // Placement new

    std::string msg = "Hello from producer";
    buf->write(msg.data(), msg.size());
    // ...
    munmap(buf, sizeof(SharedBuffer));
    close(fd);
}

// === Consumer process ===
void consumer() {
    int fd = shm_open("/my_ipc", O_RDWR, 0666);
    auto* buf = static_cast<SharedBuffer*>(
        mmap(nullptr, sizeof(SharedBuffer),
             PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0));

    char data[256];
    size_t len;
    if (buf->read(data, len)) {
        std::cout << std::string(data, len) << "\n";
    }
    munmap(buf, sizeof(SharedBuffer));
    close(fd);
    shm_unlink("/my_ipc");
}
```

The placement new (`new (buf) SharedBuffer{}`) on the producer side is important. The shared memory region is raw memory - there are no C++ objects in it until you construct one. Without placement new, the atomics in `SharedBuffer` would have indeterminate initial values, which is undefined behavior. The consumer does not call placement new because the producer already initialized the object.

### Q2: Implement Unix domain socket IPC with message framing

**Answer:**

Sockets are stream-based, which means `recv()` can return fewer bytes than you sent - it does not preserve message boundaries. You must add your own framing. The standard approach is a length prefix: send a 4-byte integer giving the payload size, then the payload. The receiver reads the length first, then reads exactly that many bytes.

```cpp
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <cstring>
#include <string>

// === Message format: [4-byte length][payload] ===
class SocketIPC {
public:
    static bool send_message(int fd, const std::string& msg) {
        uint32_t len = static_cast<uint32_t>(msg.size());
        // Send length prefix
        if (::send(fd, &len, sizeof(len), 0) != sizeof(len))
            return false;
        // Send payload
        size_t sent = 0;
        while (sent < msg.size()) {
            auto n = ::send(fd, msg.data() + sent,
                           msg.size() - sent, 0);
            if (n <= 0) return false;
            sent += n;
        }
        return true;
    }

    static std::string recv_message(int fd) {
        uint32_t len;
        if (::recv(fd, &len, sizeof(len), MSG_WAITALL) != sizeof(len))
            return {};

        std::string msg(len, '\0');
        size_t received = 0;
        while (received < len) {
            auto n = ::recv(fd, msg.data() + received,
                           len - received, 0);
            if (n <= 0) return {};
            received += n;
        }
        return msg;
    }
};

// === Server ===
void server() {
    int srv = socket(AF_UNIX, SOCK_STREAM, 0);
    sockaddr_un addr{};
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, "/tmp/my_ipc.sock", sizeof(addr.sun_path) - 1);
    unlink(addr.sun_path);
    bind(srv, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));
    listen(srv, 5);

    int client = accept(srv, nullptr, nullptr);
    auto msg = SocketIPC::recv_message(client);
    std::cout << "Received: " << msg << "\n";
    SocketIPC::send_message(client, "ACK: " + msg);

    close(client);
    close(srv);
    unlink(addr.sun_path);
}

// === Client ===
void client() {
    int fd = socket(AF_UNIX, SOCK_STREAM, 0);
    sockaddr_un addr{};
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, "/tmp/my_ipc.sock", sizeof(addr.sun_path) - 1);
    connect(fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));

    SocketIPC::send_message(fd, "Hello from client");
    auto reply = SocketIPC::recv_message(fd);
    std::cout << "Reply: " << reply << "\n";
    close(fd);
}
```

The `unlink(addr.sun_path)` before `bind()` removes any leftover socket file from a previous run. Without that, `bind()` will fail with "address already in use" if the process crashed without cleaning up. The `unlink` at the end of `server()` is the clean shutdown path.

### Q3: Design a portable IPC abstraction layer

**Answer:**

Different platforms and performance requirements call for different transports, but you do not want your application code to care about which one it is using. The transport abstraction below gives you a single interface for sending and receiving raw bytes, and a `IPCChannel` layer on top that handles message framing. Swapping from Unix sockets to shared memory is then a matter of providing a different `ITransport` implementation.

```cpp
// === IPC Transport abstraction ===
class ITransport {
public:
    virtual ~ITransport() = default;
    virtual bool connect(const std::string& endpoint) = 0;
    virtual bool listen(const std::string& endpoint) = 0;
    virtual bool send(const void* data, size_t len) = 0;
    virtual int recv(void* data, size_t max_len) = 0;
    virtual void close() = 0;
};

// === Message layer on top of transport ===
class IPCChannel {
public:
    explicit IPCChannel(std::unique_ptr<ITransport> transport)
        : transport_(std::move(transport)) {}

    bool send(const std::string& msg) {
        uint32_t len = static_cast<uint32_t>(msg.size());
        if (!transport_->send(&len, sizeof(len))) return false;
        return transport_->send(msg.data(), msg.size());
    }

    std::optional<std::string> receive() {
        uint32_t len;
        if (transport_->recv(&len, sizeof(len)) != sizeof(len))
            return std::nullopt;
        std::string buf(len, '\0');
        if (transport_->recv(buf.data(), len) != static_cast<int>(len))
            return std::nullopt;
        return buf;
    }

private:
    std::unique_ptr<ITransport> transport_;
};

// Concrete transports:
// class TcpTransport : public ITransport { ... };
// class UnixSocketTransport : public ITransport { ... };
// class SharedMemoryTransport : public ITransport { ... };
// class NamedPipeTransport : public ITransport { ... };

// Factory:
std::unique_ptr<ITransport> create_transport(const std::string& type) {
    if (type == "tcp") return std::make_unique<TcpTransport>();
    if (type == "unix") return std::make_unique<UnixSocketTransport>();
    if (type == "shm")  return std::make_unique<SharedMemoryTransport>();
    throw std::runtime_error("Unknown transport: " + type);
}
```

The factory function lets you select the transport from configuration at startup. In production on Linux you might use `"unix"` for local IPC and `"tcp"` for cross-machine communication. In tests you might use a `"shm"` transport backed by in-memory buffers that do not involve the kernel at all. The `IPCChannel` code and all the code above it never changes.

---

## Notes

- **Shared memory** is the fastest option but requires careful synchronization using atomics or semaphores.
- **Unix domain sockets** give a good balance of speed, reliability, and ease of use on Linux and macOS.
- **Named pipes** are the simplest cross-platform IPC choice for straightforward unidirectional workflows.
- Always use **message framing** with a length prefix over stream sockets - `recv()` does not guarantee that a single call returns a complete message.
- Clean up IPC resources at shutdown: call `shm_unlink` for shared memory and `unlink` for socket files.
- For structured data passed over IPC, use protobuf or FlatBuffers rather than raw struct `memcpy` - alignment and endianness are platform-dependent.
- On Linux, `eventfd` and `signalfd` are useful for lightweight IPC notification when you need to signal readiness without transferring data.
