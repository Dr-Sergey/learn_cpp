# Understand TCP flow control, Nagle's algorithm, and socket tuning

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://man7.org/linux/man-pages/man7/tcp.7.html>  

---

## Topic Overview

TCP flow control and socket tuning are the dials that determine whether your application uses the network efficiently or leaves performance on the table. This topic covers the three most impactful options: `TCP_NODELAY` for eliminating latency on request-response protocols, `SO_SNDBUF`/`SO_RCVBUF` for maximizing throughput on fat-pipe links, and `TCP_CORK` for batching writes into full-sized segments. These complement the Nagle + delayed ACK interaction covered in #554.

Here is a quick conceptual summary before diving into the code:

```cpp
Nagle's algorithm (default ON):
  Wait to send small data until:
    a) Previous ACK received, OR
    b) Enough data to fill MSS (~1460 bytes)

  Good for: bulk transfers (fewer small packets)
  Bad for:  RPC/interactive (adds latency)

TCP_NODELAY: disables Nagle -> send immediately
TCP_CORK:    hold ALL data until uncorked -> send one large segment
SO_SNDBUF/SO_RCVBUF: kernel buffer sizes -> affect throughput
```

If the table feels like a lot at first glance, the rule of thumb is: use `TCP_NODELAY` when every millisecond of latency matters, use `TCP_CORK` when you're building a multi-part response and want it sent as one burst, and only touch the buffer sizes if you're pushing data across a WAN.

| Socket option | Effect | Use case |
| --- | --- | --- |
| TCP_NODELAY | Disable Nagle, send immediately | RPC, gaming, trading |
| TCP_CORK | Hold data until uncorked | HTTP response headers + body |
| SO_SNDBUF | Send buffer size | High-throughput, WAN links |
| SO_RCVBUF | Receive buffer size | High-throughput, WAN links |
| TCP_QUICKACK | Disable delayed ACK | Low-latency responses |

---

## Self-Assessment

### Q1: Disable Nagle with TCP_NODELAY for low-latency RPC

Setting `TCP_NODELAY` is a one-liner, but it is worth understanding exactly what it changes. Without it, two consecutive small writes will trigger the Nagle + delayed ACK interaction that adds 40ms per round-trip. With it, every write goes out immediately, regardless of whether there is unacknowledged data in flight.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
// Linux/POSIX example
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <netinet/tcp.h>
#include <netinet/in.h>
#include <unistd.h>

void set_tcp_nodelay(int sock) {
    int flag = 1;
    int result = setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
    if (result < 0) perror("TCP_NODELAY");
}

// Why TCP_NODELAY matters for RPC:
//
// Without TCP_NODELAY (Nagle ON):
//   write(sock, header, 8);    // sent immediately (no pending data)
//   write(sock, payload, 100); // HELD! waiting for ACK of header
//   // ... 40ms delay (Nagle + delayed ACK interaction) ...
//   // payload finally sent when ACK arrives
//
// With TCP_NODELAY:
//   write(sock, header, 8);    // sent immediately
//   write(sock, payload, 100); // sent immediately
//   // Total: ~0.1ms (just network RTT)

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    // Enable TCP_NODELAY BEFORE connecting (or after accept)
    set_tcp_nodelay(sock);

    // Verify it's set
    int val = 0;
    socklen_t len = sizeof(val);
    getsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &val, &len);
    std::cout << "TCP_NODELAY: " << (val ? "ON" : "OFF") << '\n';  // ON

    std::cout << "\nWhen to use TCP_NODELAY:\n";
    std::cout << "  - RPC protocols (gRPC, Thrift)\n";
    std::cout << "  - Game servers (player updates)\n";
    std::cout << "  - Financial trading (order submission)\n";
    std::cout << "  - Interactive protocols (SSH, Telnet)\n";
    std::cout << "\nWhen NOT to use TCP_NODELAY:\n";
    std::cout << "  - Bulk file transfer (Nagle helps coalesce)\n";
    std::cout << "  - If you can batch writes yourself (writev)\n";

    close(sock);
}
```

The key nuance here is timing: set `TCP_NODELAY` on the server side right after `accept()`, or on the client side right after `socket()` - before any data flows. It takes effect immediately, so you can also set it mid-connection if you suddenly switch from bulk to interactive mode, though that is an unusual pattern.

### Q2: SO_SNDBUF and SO_RCVBUF for high-throughput

The reason buffer sizes matter for throughput comes down to the **bandwidth-delay product (BDP)**: the amount of data that can be "in flight" on the network at any moment equals bandwidth multiplied by round-trip time. If your TCP receive buffer is smaller than the BDP, the sender must stop and wait for ACKs before the buffer even fills - you're leaving bandwidth unused. On a LAN this rarely matters, but on a WAN with a 10ms or 100ms RTT it can cut your throughput dramatically.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

// Buffer size calculation for WAN links:
//   BDP = Bandwidth * Delay (Bandwidth-Delay Product)
//   1 Gbps * 10ms RTT = 1,250,000 bytes = ~1.2 MB
//   Default SO_RCVBUF: ~128 KB (Linux) -> bottleneck!
//   Need SO_RCVBUF >= BDP for full utilization

void configure_buffers(int sock) {
    // Set send buffer to 4 MB
    int sndbuf = 4 * 1024 * 1024;
    setsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));

    // Set receive buffer to 4 MB
    int rcvbuf = 4 * 1024 * 1024;
    setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));

    // Note: Linux doubles the value (for bookkeeping overhead)
    // So setting 4MB will show 8MB when reading back
    socklen_t len = sizeof(sndbuf);
    getsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sndbuf, &len);
    getsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, &len);
    std::cout << "Actual SO_SNDBUF: " << sndbuf / 1024 << " KB\n";
    std::cout << "Actual SO_RCVBUF: " << rcvbuf / 1024 << " KB\n";
}

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    // Check defaults
    int sndbuf, rcvbuf;
    socklen_t len = sizeof(sndbuf);
    getsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sndbuf, &len);
    getsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, &len);
    std::cout << "Default SO_SNDBUF: " << sndbuf / 1024 << " KB\n";
    std::cout << "Default SO_RCVBUF: " << rcvbuf / 1024 << " KB\n";

    // Set larger buffers
    configure_buffers(sock);

    std::cout << "\nBDP examples:\n";
    std::cout << "  LAN (1Gbps, 0.1ms):  ~12.5 KB (default is fine)\n";
    std::cout << "  WAN (1Gbps, 10ms):   ~1.2 MB (need larger buffer)\n";
    std::cout << "  WAN (1Gbps, 100ms):  ~12.5 MB (need much larger)\n";
    std::cout << "\nLinux auto-tuning: net.ipv4.tcp_rmem = min default max\n";
    std::cout << "  Usually: 4096 131072 6291456 (4KB / 128KB / 6MB)\n";

    close(sock);
}
```

Notice that Linux quietly doubles whatever value you request. If you ask for 4 MB, `getsockopt` will report 8 MB - the kernel uses the extra half for its own bookkeeping. This is expected and normal. Also worth noting: Linux has auto-tuning (via `tcp_wmem`/`tcp_rmem`) that works well for most cases; you only need to override it manually for very high-bandwidth WAN links.

### Q3: TCP_CORK batches writes into full segments

`TCP_CORK` is the opposite of `TCP_NODELAY`. Instead of sending each write immediately, it holds everything back until you "uncork" the socket, then flushes the accumulated data as one burst of full-sized TCP segments. This is ideal for HTTP-style responses where you have a fixed structure (status line, headers, body) that you want to go out together.

<!-- compile: needs POSIX header `sys/socket.h` -->
```cpp
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <netinet/tcp.h>
#include <unistd.h>

// TCP_CORK: buffer ALL writes until uncorked, then send as one burst.
// Like corking a bottle: nothing comes out until you remove the cork.
//
// Without TCP_CORK:
//   write(header)  -> TCP segment 1 (small)
//   write(body)    -> TCP segment 2
//   write(trailer) -> TCP segment 3 (small)
//   = 3 packets, 2 are tiny (inefficient)
//
// With TCP_CORK:
//   cork(sock)
//   write(header)  -> buffered
//   write(body)    -> buffered
//   write(trailer) -> buffered
//   uncork(sock)   -> ALL sent as 1 or 2 full TCP segments
//   = fewer packets, better throughput

void send_http_response(int sock) {
    // Cork the socket (hold all data)
    int flag = 1;
    setsockopt(sock, IPPROTO_TCP, TCP_CORK, &flag, sizeof(flag));

    // Write multiple parts (all buffered)
    const char* status = "HTTP/1.1 200 OK\r\n";
    const char* headers = "Content-Type: text/html\r\nContent-Length: 13\r\n\r\n";
    const char* body = "Hello, World!";

    write(sock, status, strlen(status));    // buffered
    write(sock, headers, strlen(headers));  // buffered
    write(sock, body, strlen(body));        // buffered

    // Uncork: send everything as one burst
    flag = 0;
    setsockopt(sock, IPPROTO_TCP, TCP_CORK, &flag, sizeof(flag));
    // All data now sent in minimum number of TCP segments
}

int main() {
    std::cout << "TCP_CORK vs TCP_NODELAY vs writev:\n\n";
    std::cout << "TCP_NODELAY:\n";
    std::cout << "  - Sends each write() immediately\n";
    std::cout << "  - Good for: RPC (latency-sensitive)\n";
    std::cout << "  - Bad for: many small writes (many tiny packets)\n";
    std::cout << "\nTCP_CORK:\n";
    std::cout << "  - Buffers all writes until uncorked\n";
    std::cout << "  - Good for: HTTP responses (header + body)\n";
    std::cout << "  - Requires cork/uncork bracketing\n";
    std::cout << "\nwritev (scatter-gather):\n";
    std::cout << "  - Sends multiple buffers in one syscall\n";
    std::cout << "  - Best of both worlds: one packet, no cork/uncork\n";
    std::cout << "  - Preferred when all data is ready\n";
    std::cout << "\nBest practice: use writev when possible, TCP_CORK when";
    std::cout << " data arrives asynchronously (e.g., streaming body).\n";
}
```

The golden rule here is: reach for `writev` when all your data is ready at once, because it handles the batching in a single syscall with no socket state to manage. Use `TCP_CORK` when the pieces arrive at different times and you need the buffering to span multiple calls. Never mix `TCP_NODELAY` and `TCP_CORK` on the same socket - they conflict and Linux will reject one of them.

---

## Notes

- Complementary to #554 (Nagle + delayed ACK 200ms spike).
- TCP_NODELAY and TCP_CORK are mutually exclusive on Linux.
- `SO_SNDBUF`/`SO_RCVBUF` should be set BEFORE `connect()`/`listen()` for best effect.
- Linux auto-tuning (tcp_wmem/tcp_rmem) usually does a good job; only override for WAN.
- Use `ss -tnm` to inspect actual buffer usage on Linux.
