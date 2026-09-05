# Understand TCP Nagle algorithm, delayed ACK, and when to disable them

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://en.wikipedia.org/wiki/Nagle%27s_algorithm>  

---

## Topic Overview

The Nagle algorithm and delayed ACK are two completely independent TCP optimizations that were each designed with good intentions - but when they meet inside a request-response protocol, they interact in the worst possible way and can add 40-200 ms of latency per request. Understanding why this deadlock happens, and knowing the three clean ways to escape it, is one of those things that separates people who diagnose mysterious latency spikes from people who spend days staring at Wireshark traces.

Here is the deadlock visualized as a timeline. Read it left-to-right and you will immediately see why neither side makes progress:

```cpp
Nagle + Delayed ACK deadlock:

  Client (Nagle ON):                    Server (Delayed ACK):
  write(header, 20B)                    recv(20B)
  -> sent immediately (nothing pending) -> start delayed ACK timer (40ms)
  write(payload, 100B)
  -> HELD by Nagle!                     -> nothing to ACK yet...
     (waiting for ACK of header)           (waiting to piggyback ACK on response)
  ...40-200ms pass...                   ...timer expires...
                                        -> send standalone ACK
  -> ACK received, send payload         -> finally receives payload

  Total added latency: 40-200ms per request!
```

The reason this trips people up is that both behaviors look perfectly sensible on their own. Nagle says: "don't send tiny packets, wait until you have something worth sending." Delayed ACK says: "don't waste a packet just on an ACK, wait briefly in case there's a response to piggyback it onto." But put them together with a write-write-read pattern and you get a classic deadlock: the client is waiting for an ACK that the server is deliberately withholding, and the server is waiting for more data that the client is deliberately holding back.

The table below shows which combinations kill latency and which escape the trap:

| Scenario | Nagle | Delayed ACK | Latency |
| --- | --- | --- | --- |
| write-write-read | ON | ON | +40-200ms |
| write-write-read | OFF (TCP_NODELAY) | ON | ~0ms |
| writev (both at once) | ON | ON | ~0ms |
| write-write-read | ON | OFF (TCP_QUICKACK) | ~0ms |

---

## Self-Assessment

### Q1: TCP_NODELAY and latency measurement

The code below shows how to set up the measurement pattern. The key is the two consecutive small writes followed by a read - that is the exact sequence that triggers the deadlock. Notice what happens to the per-request latency depending on whether `TCP_NODELAY` is set:

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
// Requires client+server - showing the setup pattern
#include <iostream>
#include <cstring>
#include <chrono>
#include <sys/socket.h>
#include <netinet/tcp.h>
#include <netinet/in.h>
#include <unistd.h>

// Server that echoes back responses
void run_echo_server(int port) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);
    bind(server_fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));
    listen(server_fd, 1);

    int client = accept(server_fd, nullptr, nullptr);
    char buf[1024];
    for (;;) {
        ssize_t n = read(client, buf, sizeof(buf));
        if (n <= 0) break;
        write(client, buf, n);  // echo
    }
    close(client);
    close(server_fd);
}

void measure_rpc_latency(int sock, bool nodelay) {
    if (nodelay) {
        int flag = 1;
        setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
    }

    const char* header = "REQ:";
    const char* payload = "Hello, server!";
    char response[1024];

    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < 100; ++i) {
        // Two small writes (triggers Nagle + delayed ACK interaction)
        write(sock, header, 4);
        write(sock, payload, 14);
        read(sock, response, sizeof(response));
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

    std::cout << (nodelay ? "TCP_NODELAY ON " : "TCP_NODELAY OFF")
              << ": avg " << us.count() / 100 << " us/request\n";
    // TCP_NODELAY OFF: ~40,000 us/request (40ms Nagle + delayed ACK!)
    // TCP_NODELAY ON:  ~100 us/request (immediate send)
}

int main() {
    std::cout << "Nagle + delayed ACK interaction:\n";
    std::cout << "  Two consecutive small writes + read pattern\n";
    std::cout << "  Without TCP_NODELAY: ~40ms per round-trip\n";
    std::cout << "  With TCP_NODELAY: <1ms per round-trip\n";
    std::cout << "\nThe 40ms comes from Linux delayed ACK timer (40ms)\n";
    std::cout << "Some systems use 200ms (RFC 1122 allows up to 500ms)\n";
}
```

The 400x latency difference between `TCP_NODELAY ON` and `TCP_NODELAY OFF` for simple RPC calls is not an exaggeration - it is what you actually measure on Linux. The timer is real, and it fires on every single request.

### Q2: Nagle + delayed ACK 200ms spike

To really understand the deadlock, walk through each step:

1. **Nagle's algorithm** (RFC 896, client-side): "If there is unacknowledged data AND the new data is smaller than MSS, buffer it until the ACK arrives."

2. **Delayed ACK** (RFC 1122, server-side): "Don't send an ACK immediately. Wait up to 200ms (40ms on Linux) hoping to piggyback the ACK on a response packet."

3. **The deadlock**: Client sends a small write (header). Server receives it, starts delayed ACK timer. Client sends second small write (payload) - but Nagle holds it because the first write's ACK hasn't arrived. Server is waiting to send ACK. Client is waiting for ACK to release payload. Nobody moves until the delayed ACK timer expires.

Here is the same sequence as a timeline so the causality is obvious:

```cpp
Timeline (Nagle ON, Delayed ACK ON):

t=0.0ms  Client -> [header 20B]        -> Server
                                           Server starts delayed ACK timer
t=0.0ms  Client: write(payload 100B)
         Nagle: "20B unacknowledged + 100B < MSS(1460)" -> HOLD
         Client waits for ACK...
...
t=40ms   Server: delayed ACK timer fires -> [ACK] -> Client
t=40ms   Client: ACK received -> Nagle releases payload
t=40ms   Client -> [payload 100B]      -> Server
t=40ms   Server processes, sends response
Total: 40ms added latency (or 200ms on some OSes!)
```

**Three fixes:**

1. `TCP_NODELAY` (client): disables Nagle entirely.
2. `TCP_QUICKACK` (server/Linux): disables delayed ACK.
3. `writev()` (client): sends header + payload in one syscall. Nagle sees a full send and doesn't buffer.

### Q3: writev / MSG_MORE to batch writes without disabling Nagle

If you can't or don't want to disable Nagle (perhaps because you're doing bulk data transfer elsewhere on the same socket), you can still escape the deadlock by making the kernel see your header and payload as a single logical write. `writev`, `MSG_MORE`, and `TCP_CORK` are three ways to do exactly that:

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <sys/uio.h>
#include <netinet/tcp.h>
#include <netinet/in.h>
#include <unistd.h>

// Method 1: writev() - gather multiple buffers into ONE TCP send
void send_request_writev(int sock) {
    const char* header = "REQ:";
    const char* payload = "Hello, server!";

    struct iovec iov[2];
    iov[0].iov_base = const_cast<char*>(header);
    iov[0].iov_len  = 4;
    iov[1].iov_base = const_cast<char*>(payload);
    iov[1].iov_len  = 14;

    writev(sock, iov, 2);  // ONE syscall, ONE TCP segment
    // Nagle sees a single "large" write -> sends immediately
    // No Nagle + delayed ACK interaction!
}

// Method 2: MSG_MORE - tell kernel "more data coming, don't send yet"
void send_request_msg_more(int sock) {
    const char* header = "REQ:";
    const char* payload = "Hello, server!";

    send(sock, header, 4, MSG_MORE);  // kernel: buffer this, more coming
    send(sock, payload, 14, 0);       // kernel: no more, send everything
    // Both are sent in a single TCP segment!
    // Similar to TCP_CORK but per-send, not per-socket
}

// Method 3: TCP_CORK bracket (for async/streaming writes)
void send_request_cork(int sock) {
    int flag = 1;
    setsockopt(sock, IPPROTO_TCP, TCP_CORK, &flag, sizeof(flag));

    const char* header = "REQ:";
    const char* payload = "Hello, server!";
    write(sock, header, 4);     // buffered by cork
    write(sock, payload, 14);   // buffered by cork

    flag = 0;
    setsockopt(sock, IPPROTO_TCP, TCP_CORK, &flag, sizeof(flag));  // flush!
}

int main() {
    std::cout << "Methods to batch writes WITHOUT TCP_NODELAY:\n\n";
    std::cout << "1. writev() / sendmsg():\n";
    std::cout << "   - Best: one syscall, one TCP segment\n";
    std::cout << "   - Requires all data ready at once\n";
    std::cout << "\n2. MSG_MORE flag:\n";
    std::cout << "   - Per-send hint: 'more data coming'\n";
    std::cout << "   - Flexible: works with async writes\n";
    std::cout << "   - Linux-specific\n";
    std::cout << "\n3. TCP_CORK:\n";
    std::cout << "   - Socket-level: cork/uncork bracket\n";
    std::cout << "   - Good for HTTP: cork, write headers+body, uncork\n";
    std::cout << "   - Linux-specific\n";
    std::cout << "\nAll three avoid the Nagle+delayed ACK deadlock\n";
    std::cout << "while keeping Nagle's benefits for other traffic.\n";
}
```

Notice that `writev` is the cleanest option when all your data is available at once - it combines the scatter-gather into a single syscall, so Nagle never gets a chance to hold anything back. `MSG_MORE` is useful when you generate the pieces sequentially but still want them batched. `TCP_CORK` is the right choice when you're building a response incrementally (like an HTTP server writing status, headers, then body separately).

---

## Notes

- Complementary to #644 (TCP_NODELAY/CORK/buffer sizing).
- The 40ms number is Linux-specific; Windows uses 200ms, macOS uses 75ms.
- gRPC, Redis, and most RPC frameworks set TCP_NODELAY by default.
- MSG_MORE is Linux-specific; use `writev()` for portability.
- `TCP_QUICKACK` is non-sticky on Linux: must be re-set after each recv().
