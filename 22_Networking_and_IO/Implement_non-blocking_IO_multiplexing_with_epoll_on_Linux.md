# Implement non-blocking I/O multiplexing with epoll on Linux

**Category:** Networking and IO  
**Standard:** Not version-specific  
**Reference:** <https://man7.org/linux/man-pages/man7/epoll.7.html>  

---

## Topic Overview

`epoll` is Linux's answer to the question: how do you efficiently wait for I/O events across thousands of file descriptors? The older interfaces - `select` and `poll` - scan the entire list of file descriptors on every call. If you are watching 10,000 sockets and only 3 have data, you still pay the cost of checking all 10,000. `epoll` avoids this by using kernel callbacks: the kernel tracks which descriptors have events ready and hands you only those.

```cpp
epoll architecture:
  epoll_create1(0)         -> epollfd
  epoll_ctl(ADD, sockfd)   -> register interest
  epoll_wait(epollfd, ...) -> returns only READY fds

  select/poll: O(N) scan all fds every call
  epoll:       O(1) add/remove, O(ready) per wait
```

| Multiplexer | Add/Remove | Wait cost | Max FDs | Edge-trigger |
| --- | --- | --- | --- | --- |
| select | O(N) | O(N) | 1024 (FD_SETSIZE) | No |
| poll | O(N) | O(N) | Unlimited | No |
| epoll | O(1) | O(ready) | Unlimited | Yes |
| kqueue (BSD) | O(1) | O(ready) | Unlimited | Yes |
| io_uring | O(1) | O(ready) | Unlimited | N/A |

---

## Self-Assessment

### Q1: Create epoll FD and register sockets with EPOLLIN | EPOLLET

This is a complete, runnable echo server using edge-triggered epoll. It accepts new connections and echoes data back to each client. Pay close attention to the inner `for (;;)` loops - with edge-triggered mode, you *must* drain the file descriptor completely on each event, or you risk missing data. That is why both the accept loop and the read loop run until they get `EAGAIN`.

<!-- compile: needs POSIX header `unistd.h` -->
```cpp
// Linux-only example
#include <iostream>
#include <cstring>
#include <unistd.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>

// Make a socket non-blocking (required for edge-triggered epoll)
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    // 1. Create listening socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(8080);
    bind(listen_fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));
    listen(listen_fd, 128);
    set_nonblocking(listen_fd);

    // 2. Create epoll instance
    int epoll_fd = epoll_create1(0);
    if (epoll_fd < 0) { perror("epoll_create1"); return 1; }

    // 3. Register listen socket with edge-triggered mode
    epoll_event ev{};
    ev.events = EPOLLIN | EPOLLET;  // Edge-triggered: only notified on NEW data
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    std::cout << "Listening on :8080 with epoll (edge-triggered)\n";

    // 4. Event loop
    constexpr int MAX_EVENTS = 64;
    epoll_event events[MAX_EVENTS];

    for (;;) {
        int n = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);  // block until ready
        for (int i = 0; i < n; ++i) {
            if (events[i].data.fd == listen_fd) {
                // Accept ALL pending connections (edge-triggered!)
                for (;;) {
                    int client_fd = accept(listen_fd, nullptr, nullptr);
                    if (client_fd < 0) break;  // EAGAIN = no more pending

                    set_nonblocking(client_fd);
                    ev.events = EPOLLIN | EPOLLET;
                    ev.data.fd = client_fd;
                    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
                    std::cout << "Accepted client fd=" << client_fd << '\n';
                }
            } else {
                // Read ALL available data (edge-triggered!)
                char buf[4096];
                for (;;) {
                    ssize_t bytes = read(events[i].data.fd, buf, sizeof(buf));
                    if (bytes <= 0) {
                        if (bytes == 0) {  // client disconnected
                            close(events[i].data.fd);
                        }
                        break;  // EAGAIN or error
                    }
                    write(events[i].data.fd, buf, bytes);  // echo
                }
            }
        }
    }

    close(epoll_fd);
    close(listen_fd);
}
// Test: echo "Hello" | nc localhost 8080
```

### Q2: Level-triggered vs edge-triggered epoll

This is the most important conceptual distinction in epoll programming. Getting it wrong causes data loss that is very hard to debug, because the data is not gone - it is sitting in the kernel buffer, but epoll will never wake you up for it again.

**Level-Triggered (LT, default):**

- `epoll_wait` returns a FD as long as data is available.
- If you read 100 bytes but 500 remain, the FD is returned again on the next `epoll_wait`.
- Safe: hard to lose events.
- Wasteful: may report the same FD repeatedly.

**Edge-Triggered (ET, EPOLLET):**

- `epoll_wait` returns a FD only when NEW data arrives (state transition).
- If you read 100 bytes but 500 remain, the FD is NOT returned again until MORE data arrives.
- Must drain the FD completely in a loop until `EAGAIN`.
- Efficient: fewer `epoll_wait` returns.

```cpp
Data arrives: 1000 bytes

Level-triggered:
  epoll_wait -> fd ready
  read(fd, buf, 500) -> 500 bytes
  epoll_wait -> fd STILL ready (500 bytes remain)   <- reported again!
  read(fd, buf, 500) -> 500 bytes
  epoll_wait -> fd NOT ready (empty)

Edge-triggered:
  epoll_wait -> fd ready
  read(fd, buf, 500) -> 500 bytes
  epoll_wait -> fd NOT reported!                     <- 500 bytes LEFT BEHIND!
  ... data stuck until new data arrives ...
  FIX: read in a loop until EAGAIN:
    while (read(fd, buf, sizeof(buf)) > 0) { ... }
```

The missed-event bug is subtle because everything *looks* fine most of the time - data loss only occurs when the sender happens to fill the send buffer before you drain the receive side. Here is the pattern that causes it and the fix:

```cpp
// BAD: edge-triggered + single read = data loss
events[i].events = EPOLLIN | EPOLLET;
// ...
ssize_t n = read(fd, buf, sizeof(buf));  // reads partial data
// Remaining bytes are NEVER reported by epoll_wait!
// Data is stuck in the kernel buffer until new data arrives.

// GOOD: always drain in a loop
for (;;) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n < 0) {
        if (errno == EAGAIN) break;  // all data consumed
        perror("read"); break;
    }
    if (n == 0) { close(fd); break; }  // EOF
    process(buf, n);
}
```

### Q3: Minimal reactor event loop with epoll_wait

The reactor pattern is the design behind Asio, libevent, and libuv. The idea is simple: instead of a monolithic event loop with a giant switch statement, each file descriptor gets its own handler callback. When epoll says the FD is ready, you look up its handler and call it. New FDs can be added (or removed) dynamically, even from within handlers.

<!-- compile: needs POSIX header `sys/epoll.h` -->
```cpp
// A reactor dispatches events to registered handlers
#include <iostream>
#include <functional>
#include <unordered_map>
#include <sys/epoll.h>
#include <unistd.h>

class Reactor {
public:
    using Handler = std::function<void(uint32_t events)>;

    Reactor() : epfd_(epoll_create1(0)) {}
    ~Reactor() { close(epfd_); }

    // Register a file descriptor with a handler
    void add(int fd, uint32_t events, Handler handler) {
        handlers_[fd] = std::move(handler);
        epoll_event ev{};
        ev.events = events;
        ev.data.fd = fd;
        epoll_ctl(epfd_, EPOLL_CTL_ADD, fd, &ev);
    }

    void remove(int fd) {
        epoll_ctl(epfd_, EPOLL_CTL_DEL, fd, nullptr);
        handlers_.erase(fd);
    }

    // Run the event loop
    void run() {
        constexpr int MAX_EV = 64;
        epoll_event events[MAX_EV];
        running_ = true;

        while (running_) {
            int n = epoll_wait(epfd_, events, MAX_EV, 1000);  // 1s timeout
            for (int i = 0; i < n; ++i) {
                int fd = events[i].data.fd;
                auto it = handlers_.find(fd);
                if (it != handlers_.end()) {
                    it->second(events[i].events);  // dispatch!
                }
            }
        }
    }

    void stop() { running_ = false; }

private:
    int epfd_;
    bool running_ = false;
    std::unordered_map<int, Handler> handlers_;
};

// Usage example (conceptual, needs socket setup):
// int main() {
//     Reactor reactor;
//     int listen_fd = create_listening_socket(8080);
//
//     reactor.add(listen_fd, EPOLLIN | EPOLLET, [&](uint32_t) {
//         int client = accept(listen_fd, nullptr, nullptr);
//         set_nonblocking(client);
//         reactor.add(client, EPOLLIN | EPOLLET, [&reactor, client](uint32_t ev) {
//             char buf[4096];
//             ssize_t n = read(client, buf, sizeof(buf));
//             if (n <= 0) { reactor.remove(client); close(client); return; }
//             write(client, buf, n);  // echo
//         });
//     });
//
//     reactor.run();
// }

int main() {
    std::cout << "Reactor pattern with epoll:\n";
    std::cout << "  1. Create epoll_create1(0)\n";
    std::cout << "  2. Register FDs with handlers via EPOLL_CTL_ADD\n";
    std::cout << "  3. epoll_wait returns only ready FDs\n";
    std::cout << "  4. Dispatch to registered handler callbacks\n";
    std::cout << "  5. Handler can add/remove other FDs (dynamic)\n";
    std::cout << "\nAdvantages over raw epoll loop:\n";
    std::cout << "  - Handler per FD (no giant switch statement)\n";
    std::cout << "  - Easy to compose (timer FDs, signal FDs, socket FDs)\n";
    std::cout << "  - Foundation for Asio, libevent, libuv\n";
}
```

---

## Notes

- Always use `EPOLLET` with non-blocking sockets and drain-to-`EAGAIN` loops.
- `EPOLLONESHOT` forces re-arming after each event (useful for thread-pool dispatch).
- `EPOLLRDHUP` detects peer shutdown without requiring a failed `read()`.
- For Windows: use IOCP (completion ports). For cross-platform: use Asio or libuv.
- epoll handles 100K+ concurrent connections efficiently (C10K problem solved).
