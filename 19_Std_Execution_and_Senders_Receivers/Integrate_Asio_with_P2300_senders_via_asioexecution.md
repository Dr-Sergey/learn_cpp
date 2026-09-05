# Integrate Asio with P2300 senders via asio::execution

**Category:** Std Execution and Senders Receivers  
**Standard:** C++20  
**Reference:** <https://think-async.com/Asio/>  

---

## Topic Overview

Boost.Asio and P2300 (`std::execution`) are complementary: Asio provides mature, battle-tested async I/O, while P2300 provides composable async pipelines. They have different mental models for the same underlying idea, which means bridging them takes a small amount of adapter code - but once that adapter exists, you can use Asio's `io_context` as a P2300 scheduler and wrap Asio async operations as first-class senders.

| Concept | Asio | P2300 |
| --- | --- | --- |
| Where work runs | `io_context` / `strand` | scheduler / execution context |
| Async operation | `async_read`, `async_write` | sender |
| Callback model | Completion handler | receiver |
| Composition | `async_compose` | `then`, `when_all`, pipe `\|` |

---

## Self-Assessment

### Q1: Wrap Asio `async_read_some` into a P2300 sender

The key insight here is that an Asio completion handler and a P2300 receiver carry the same information. The completion handler gets `(error_code, result)` - you just need to map that to the right completion signal. `operation_aborted` maps to `set_stopped`, any other error maps to `set_error`, and success maps to `set_value`.

```cpp
// asio_sender.cpp - wrapping Asio async_read into a sender
#include <stdexec/execution.hpp>
#include <boost/asio.hpp>
#include <iostream>
#include <vector>
#include <system_error>

namespace asio = boost::asio;
using tcp = asio::ip::tcp;

// Sender that wraps async_read_some:
struct asio_read_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(std::size_t),             // bytes read
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_stopped_t()
    >;

    tcp::socket& socket;
    asio::mutable_buffer buffer;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;
        tcp::socket& socket;
        asio::mutable_buffer buffer;
        Receiver receiver;

        void start() noexcept {
            // Bridge: Asio completion handler -> P2300 receiver
            socket.async_read_some(
                buffer,
                [this](std::error_code ec, std::size_t bytes_read) {
                    if (ec == asio::error::operation_aborted) {
                        stdexec::set_stopped(std::move(receiver));
                    } else if (ec) {
                        stdexec::set_error(std::move(receiver),
                            std::make_exception_ptr(
                                std::system_error(ec)));
                    } else {
                        stdexec::set_value(std::move(receiver),
                                          bytes_read);
                    }
                });
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{socket, buffer, std::move(recv)};
    }
};

auto async_read_some(tcp::socket& sock, asio::mutable_buffer buf) {
    return asio_read_sender{sock, buf};
}

// Usage in a sender pipeline:
void example(tcp::socket& sock) {
    std::vector<char> buf(1024);
    auto pipeline = async_read_some(sock, asio::buffer(buf))
        | stdexec::then([&buf](std::size_t n) {
            std::string data(buf.data(), n);
            std::cout << "Read: " << data << '\n';
            return n;
        })
        | stdexec::upon_error([](std::exception_ptr ep) -> std::size_t {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cerr << "Error: " << e.what() << '\n';
            }
            return 0;
        });
    // Drive with io_context::run() instead of sync_wait
}
```

Once the wrapper is written, the pipeline composition above looks exactly like any other P2300 pipeline. The bridge code is confined to the `start()` lambda.

### Q2: Asio `io_context` as a P2300 scheduler

Making an `io_context` behave as a P2300 scheduler lets you use `starts_on` and `continues_on` to route work through Asio's event loop, just like you would route work through a thread pool. The adapter uses `asio::post` to enqueue work, and signals `set_value()` when the posted function actually runs.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// asio_scheduler.cpp - adapt io_context to P2300 scheduler concept
#include <stdexec/execution.hpp>
#include <boost/asio.hpp>
#include <iostream>
#include <thread>

namespace asio = boost::asio;

// Adapter: wraps io_context as a P2300 scheduler
struct asio_scheduler {
    asio::io_context& ctx;

    // schedule() returns a sender that completes on the io_context
    struct schedule_sender {
        using sender_concept = stdexec::sender_t;
        using completion_signatures = stdexec::completion_signatures<
            stdexec::set_value_t()
        >;
        asio::io_context& ctx;

        template <typename Receiver>
        struct op {
            using operation_state_concept = stdexec::operation_state_t;
            asio::io_context& ctx;
            Receiver recv;

            void start() noexcept {
                // Post work to the io_context:
                asio::post(ctx, [this]() {
                    stdexec::set_value(std::move(recv));
                });
            }
        };

        template <stdexec::receiver Receiver>
        auto connect(Receiver recv) const noexcept {
            return op<Receiver>{ctx, std::move(recv)};
        }
    };

    auto schedule() const noexcept {
        return schedule_sender{ctx};
    }

    bool operator==(const asio_scheduler&) const = default;
};

int main() {
    asio::io_context io;
    asio_scheduler sched{io};

    // Use our Asio scheduler in a P2300 pipeline:
    auto work = sched.schedule()
        | stdexec::then([]() {
            std::cout << "Running on io_context thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        });

    // Run io_context in another thread to demonstrate scheduling:
    std::thread io_thread([&io]() { io.run(); });

    // Note: sync_wait won't work here since io_context drives the work.
    // In practice, integrate with io.run() event loop.
    io_thread.join();
}
```

### Q3: Compatibility between Asio executors and P2300 schedulers

The two models are close but not identical. The table below maps the most common Asio patterns to their P2300 equivalents. The biggest mismatch is the event loop: Asio uses `io_context::run()` to drive work, while P2300 uses `sync_wait()`. In practice this means you typically need to pick one as the "outer" driver and bridge the other into it.

| Asio Executor Model | P2300 Scheduler Model |
| --- | --- |
| `executor.post(fn)` | `schedule(sched) \| then(fn)` |
| `executor.dispatch(fn)` | `schedule(sched) \| then(fn)` (inline if on context) |
| `strand<executor>` | No direct equivalent (use mutex or serializing scheduler) |
| `io_context::run()` | `sync_wait()` or event loop on main thread |
| `any_io_executor` | Type-erased scheduler |

Here is the bridging strategy at a glance:

```cpp
Asio world:                         P2300 world:
┌──────────────────┐              ┌──────────────────┐
│  io_context      │─── adapt ──->│ P2300 scheduler   │
│  async_read()    │─── wrap  ──->│ sender            │
│  completion hdlr │─── map   ──->│ receiver signals  │
└──────────────────┘              └──────────────────┘

Key mapping:
  ec == operation_aborted  ->  set_stopped()
  ec (other error)         ->  set_error(exception_ptr)
  success                  ->  set_value(result)
```

---

## Notes

- Asio's `deferred` completion token returns sender-like objects natively.
- Asio 1.30+ has experimental support for P2300-compatible senders via `asio::experimental::use_sender`.
- The main challenge is event loop ownership: Asio uses `io_context::run()`, P2300 uses `sync_wait()`.
- Strands don't map directly to P2300 - but you can create a serializing sender adapter.
- Consider `asio::co_spawn` for coroutine-based bridging as an alternative.
