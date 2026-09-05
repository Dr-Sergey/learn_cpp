# Understand cancellation in P2300 via stop tokens and stop callbacks

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

P2300 provides *structured* cancellation through stop tokens. The idea is cooperative: cancellation is not forced on an operation from the outside - instead, the operation checks for a cancellation request at sensible points and voluntarily signals `set_stopped()` when it is ready to stop. Every receiver carries a stop token in its environment, and senders retrieve it via `get_stop_token(get_env(receiver))`.

Here is the flow from the moment a parent scope requests a stop through to the final signal:

```cpp
Cancellation flow:

  Parent scope          Sender/Operation         Receiver
  request_stop() ──->  stop_token.stop_requested()  |
                        |                            |
                   if true:                          |
                     clean up  ───────────────────-> set_stopped()
                   if false:                         |
                     continue work ───────────────-> set_value(result)
```

There are two ways to react to a stop request:

| Mechanism | Purpose |
| --- | --- |
| `stop_token` | Polling: check `stop_requested()` periodically |
| `stop_callback` | Reactive: register callback invoked when stop is requested |
| `set_stopped()` | Signal the receiver that operation was cancelled |
| `upon_stopped(f)` | Pipeline adaptor to handle cancellation |

Polling is simpler and is the right choice for CPU-bound work that runs in a loop. Stop callbacks are the right choice for true async I/O, where you need to actually cancel an in-flight kernel operation when a stop is requested.

---

## Self-Assessment

### Q1: Poll `get_stop_token()` inside a long-running sender

This example shows the polling pattern. The compute loop checks the stop token every 1000 iterations - frequent enough to respond promptly, infrequent enough not to waste cycles on the check itself.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <vector>
#include <numeric>

// A sender that does work in chunks and checks for cancellation:
struct chunked_compute_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(long long),
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_stopped_t()
    >;

    int iterations;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;
        int iterations;
        Receiver receiver;

        void start() noexcept {
            try {
                // Get the stop token from the receiver's environment:
                auto token = stdexec::get_stop_token(
                    stdexec::get_env(receiver));

                long long result = 0;
                for (int i = 0; i < iterations; ++i) {
                    // Poll for cancellation periodically:
                    if (i % 1000 == 0 && token.stop_requested()) {
                        std::cout << "Cancelled at iteration " << i << '\n';
                        stdexec::set_stopped(std::move(receiver));
                        return;
                    }
                    result += static_cast<long long>(i) * i;  // heavy work
                }

                stdexec::set_value(std::move(receiver), result);
            } catch (...) {
                stdexec::set_error(std::move(receiver),
                                   std::current_exception());
            }
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{iterations, std::move(recv)};
    }
};

int main() {
    // Without cancellation - runs to completion:
    auto pipeline = chunked_compute_sender{1000000}
        | stdexec::then([](long long r) {
            std::cout << "Result: " << r << '\n';
            return r;
        })
        | stdexec::upon_stopped([]() -> long long {
            std::cout << "Was cancelled, returning 0\n";
            return 0;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << result << '\n';
}
```

The `upon_stopped` at the end converts the stopped channel back into a value of the same type, which is the idiomatic way to provide a default result when cancellation happens.

### Q2: Register a `stop_callback` for async I/O cancellation

Stop callbacks are the harder pattern but necessary for real async I/O. The reason is that a sleeping or waiting operation cannot poll - it is blocked. You need a callback that fires on the thread calling `request_stop()` and actually tells the platform to cancel the pending I/O. The important lifecycle rule is: always unregister the callback (call `cb.reset()`) before signaling the receiver, to avoid a race between the callback and the completion handler.

```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <stop_token>

// A sender wrapping an async I/O operation that supports cancellation:
struct async_io_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(std::size_t),
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_stopped_t()
    >;

    int fd;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;
        int fd;
        Receiver receiver;

        using stop_token_t = stdexec::stop_token_of_t<
            stdexec::env_of_t<Receiver>>;

        // stop_callback registered when start() is called:
        std::optional<std::stop_callback<std::function<void()>>> cb;

        void start() noexcept {
            auto token = stdexec::get_stop_token(
                stdexec::get_env(receiver));

            // Already cancelled?
            if (token.stop_requested()) {
                stdexec::set_stopped(std::move(receiver));
                return;
            }

            // Register callback: when stop is requested, cancel the I/O
            cb.emplace(token, [this]() {
                // This fires asynchronously when stop is requested:
                cancel_io(fd);  // platform-specific: cancel pending I/O
                std::cout << "I/O cancelled via stop_callback\n";
            });

            // Submit async I/O (simplified):
            submit_async_read(fd, [this](int result, bool cancelled) {
                cb.reset();  // unregister callback
                if (cancelled) {
                    stdexec::set_stopped(std::move(receiver));
                } else if (result < 0) {
                    stdexec::set_error(std::move(receiver),
                        std::make_exception_ptr(
                            std::runtime_error("I/O error")));
                } else {
                    stdexec::set_value(std::move(receiver),
                                      static_cast<std::size_t>(result));
                }
            });
        }

        // Platform stubs:
        static void cancel_io(int) { /* cancel pending I/O */ }
        static void submit_async_read(int, auto&&) { /* submit I/O */ }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{fd, std::move(recv)};
    }
};

// Key pattern:
// 1. Extract stop_token from receiver's environment
// 2. Register stop_callback that cancels platform-specific operation
// 3. When callback fires, clean up and call set_stopped()
// 4. Always unregister callback before signaling completion
```

### Q3: Three completion channels and receiver methods

Every operation state must signal the receiver via exactly one of three methods, on exactly one code path. Think of it as a single return from a function - you can return once, and the "return value" tells the receiver which channel fired.

| Channel | Receiver Method | Meaning | Pipeline Adaptor |
| --- | --- | --- | --- |
| **Value** | `set_value(recv, args...)` | Success | `then(f)` |
| **Error** | `set_error(recv, err)` | Failure | `upon_error(f)` |
| **Stopped** | `set_stopped(recv)` | Cancellation | `upon_stopped(f)` |

Here is the correct pattern and the UB-inducing mistake side by side:

```cpp
// Exactly ONE completion signal must be called:

// GOOD:
void start() noexcept {
    if (cancelled)
        stdexec::set_stopped(std::move(recv));   // stopped channel
    else if (error)
        stdexec::set_error(std::move(recv), ep); // error channel
    else
        stdexec::set_value(std::move(recv), 42); // value channel
}

// BAD: calling multiple signals = UB
void bad_start() noexcept {
    stdexec::set_value(std::move(recv), 42);
    stdexec::set_stopped(std::move(recv));  // UB: already completed!
}
```

The diagram below shows how channels flow through a composed pipeline. The key thing to notice is that `then` only handles the value channel - errors and stopped signals travel straight through it untouched. Only the matching `upon_*` adaptor handles each non-value channel.

```cpp
Channel flow through a pipeline:

just(42) | then(f) | upon_error(g) | upon_stopped(h)
  |          |            |                |
  | value    | value      |                |
  |          | error  ----+ value          |
  |          | stopped    | stopped  ------+ value -> consumer

Errors pass through then() untouched.
Stopped passes through then() and upon_error() untouched.
```

---

## Notes

- Stop tokens are non-owning; the `stop_source` that controls them lives in the parent scope.
- Polling (`stop_requested()`) is cheap - typically an atomic load.
- Stop callbacks run on the thread that calls `request_stop()` - be careful with locks.
- `in_place_stop_token` is the lightweight, non-type-erased stop token used by stdexec.
- `upon_stopped` converts the stopped channel to a value - useful for providing defaults.
