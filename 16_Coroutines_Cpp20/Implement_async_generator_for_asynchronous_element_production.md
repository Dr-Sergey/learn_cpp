# Implement Async Generator for Asynchronous Element Production

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** [cppreference — Coroutines](https://en.cppreference.com/w/cpp/language/coroutines)  

---

## Topic Overview

A **synchronous generator** (`co_yield` + pull iteration) suspends only to emit values. An **async generator** combines `co_yield` with `co_await`: it can **await asynchronous operations** (I/O, timers, network reads) between yields. The consumer of an async generator also uses `co_await` to pull each value, because the next value may not be available until an async operation completes.

The key difference is the **iterator advancement**:

| Generator Type | `operator++()` | Value availability |
| --- | --- | --- |
| Sync `Generator<T>` | Calls `handle.resume()`, returns immediately | Always immediate (CPU-bound) |
| `AsyncGenerator<T>` | Returns an awaitable; consumer `co_await`s it | May be delayed (I/O-bound) |

Internally, the async generator's promise holds a **continuation handle** - the consumer's coroutine that is waiting for the next value. When the generator `co_yield`s a value, its `yield_value` awaiter transfers control back to the consumer via symmetric transfer. When the generator `co_await`s an async operation, both the generator *and* its consumer are suspended; the async callback eventually resumes the generator, which then yields to the consumer.

The reason this is trickier than a sync generator is that there are now two coroutines interleaving - the generator and the consumer - and they take turns being suspended. Each `co_await gen.next()` in the consumer hands control to the generator. The generator runs until it either `co_yield`s a value (handing control back to the consumer via symmetric transfer) or `co_await`s an I/O operation (suspending both until I/O completes).

```cpp
Consumer                   AsyncGenerator              Async I/O
   │                            │                          │
   ├── co_await next() ────>    │                          │
   │   (suspended)              ├── running body ──>       │
   │                            ├── co_await read() ──>    │
   │                            │   (suspended)            ├── completes
   │                            │   <- resume ────────────┘
   │                            ├── co_yield value ──>     │
   │   <- symmetric transfer ──┘                          │
   ├── process value            │                          │
```

The `AsyncGenerator<T>` pattern is vital for streaming data: reading database rows, consuming message queues, paginating API results - any scenario where producing the next element involves waiting.

---

## Self-Assessment

### Q1: Implement a minimal `AsyncGenerator<T>` with a consumer that `co_await`s each value

This implementation is the heart of the pattern. The most important design decision is symmetric transfer in both `yield_value` and `final_suspend`: when the generator yields a value, it immediately transfers to the consumer; when it finishes, it also transfers to the consumer so the consumer can observe the end-of-stream (a `nullopt` from `await_resume`):

```cpp
#include <coroutine>
#include <cstdio>
#include <exception>
#include <optional>
#include <utility>

template <typename T>
class AsyncGenerator {
public:
    struct promise_type {
        std::optional<T> current_value;
        std::coroutine_handle<> consumer_handle;  // who is waiting for us
        std::exception_ptr eptr;

        AsyncGenerator get_return_object() {
            return AsyncGenerator{handle_type::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }

        // At final suspend, transfer to consumer so it sees "done"
        auto final_suspend() noexcept {
            struct TransferToConsumer {
                std::coroutine_handle<> consumer;
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<>) noexcept {
                    return consumer;  // symmetric transfer
                }
                void await_resume() noexcept {}
            };
            return TransferToConsumer{consumer_handle};
        }

        // co_yield stores value and transfers to consumer
        auto yield_value(T value) {
            current_value.emplace(std::move(value));
            struct YieldAwaiter {
                std::coroutine_handle<> consumer;
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<>) noexcept {
                    return consumer;  // symmetric transfer to consumer
                }
                void await_resume() noexcept {}
            };
            return YieldAwaiter{consumer_handle};
        }

        void return_void() noexcept { current_value.reset(); }
        void unhandled_exception() { eptr = std::current_exception(); }
    };

    using handle_type = std::coroutine_handle<promise_type>;

    // Awaitable: co_await gen.next() in consumer
    struct NextAwaiter {
        handle_type gen_handle;

        bool await_ready() noexcept { return false; }

        std::coroutine_handle<> await_suspend(
            std::coroutine_handle<> consumer) noexcept {
            gen_handle.promise().consumer_handle = consumer;
            return gen_handle;  // symmetric transfer to generator
        }

        std::optional<T> await_resume() {
            auto& p = gen_handle.promise();
            if (p.eptr) std::rethrow_exception(p.eptr);
            return std::move(p.current_value);
        }
    };

    NextAwaiter next() { return {handle_}; }

    explicit AsyncGenerator(handle_type h) : handle_(h) {}
    ~AsyncGenerator() { if (handle_) handle_.destroy(); }
    AsyncGenerator(AsyncGenerator&& o) noexcept
        : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    handle_type handle_;
};

// --- Simulated async I/O ---
struct SimulatedRead {
    int value;
    bool await_ready() noexcept { return true; }  // completes immediately for demo
    void await_suspend(std::coroutine_handle<>) noexcept {}
    int await_resume() noexcept { return value; }
};

// Async generator: yields values with async work between them
AsyncGenerator<int> async_iota(int start, int count) {
    for (int i = 0; i < count; ++i) {
        int val = co_await SimulatedRead{start + i};  // async between yields
        co_yield val * 10;
    }
}

// --- Simple synchronous driver (for demonstration) ---
struct DriverTask {
    struct promise_type {
        DriverTask get_return_object() { return {}; }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };
};

DriverTask run() {
    auto gen = async_iota(1, 5);
    while (auto val = co_await gen.next()) {
        std::printf("got: %d\n", *val);
    }
    std::printf("done\n");
}

int main() {
    run();
    // Output: got: 10  got: 20  got: 30  got: 40  got: 50  done
}
```

---

### Q2: Show the difference between sync and async generators when an asynchronous delay is involved

The fundamental limitation of a sync generator is that it cannot `co_await`. As soon as you add an async wait between yields, a sync generator becomes ill-formed. This is why the async generator exists as a separate concept:

```cpp
#include <coroutine>
#include <chrono>
#include <cstdio>
#include <optional>
#include <utility>
#include <thread>

// AsyncGenerator<T> from Q1 (reused, definition omitted for brevity)

// Simulated timer awaitable
struct TimerAwaitable {
    std::chrono::milliseconds duration;

    bool await_ready() noexcept { return duration.count() <= 0; }

    void await_suspend(std::coroutine_handle<> h) noexcept {
        // In production: register with event loop
        // For demo: launch a thread (do NOT do this in real code)
        std::thread([h, d = duration] {
            std::this_thread::sleep_for(d);
            h.resume();
        }).detach();
    }

    void await_resume() noexcept {}
};

// Async generator that waits between yields
AsyncGenerator<int> timed_sequence(int count,
                                    std::chrono::milliseconds interval) {
    for (int i = 1; i <= count; ++i) {
        co_await TimerAwaitable{interval};  // async wait
        co_yield i;
    }
}

// Compare: a sync generator CANNOT do this
// Generator<int> bad_sync(int count) {
//     co_await TimerAwaitable{...};  // ERROR: sync generator cannot co_await
//     co_yield count;
// }

// The critical difference table:
//
// | Feature              | Sync Generator      | Async Generator          |
// |----------------------|---------------------|--------------------------|
// | co_yield             | Yes                 | Yes                      |
// | co_await             | No (ill-formed)     | Yes                      |
// | Iterator advance     | Synchronous         | Returns awaitable        |
// | Caller context       | Regular function    | Must be a coroutine      |
// | Suspension between   | Only at yield       | At yield AND at co_await |
// |   points             |                     |                          |
```

---

### Q3: Implement an async generator that reads "pages" from a paginated source, yielding individual items — a real-world streaming pattern

This is the pattern you'd use to consume a database cursor, a paginated REST API, or a Kafka topic. The generator handles all the batching and refetching logic internally. The consumer just sees a flat stream of items and uses `co_await gen.next()` for each one - completely unaware of how the data is batched:

```cpp
#include <coroutine>
#include <cstdio>
#include <optional>
#include <string>
#include <utility>
#include <vector>

// AsyncGenerator<T> from Q1 (reused)

// Simulated paginated API
struct Page {
    std::vector<std::string> items;
    bool has_next;
};

struct FetchPageAwaitable {
    int page_num;
    Page result;

    bool await_ready() noexcept { return false; }

    void await_suspend(std::coroutine_handle<> h) noexcept {
        // Simulate async fetch — in production this registers with I/O loop
        if (page_num == 0)
            result = {{"alpha", "bravo", "charlie"}, true};
        else if (page_num == 1)
            result = {{"delta", "echo"}, true};
        else
            result = {{}, false};
        h.resume();  // "completion callback"
    }

    Page await_resume() noexcept { return std::move(result); }
};

// Async generator: fetch pages, yield individual items
AsyncGenerator<std::string> paginated_stream() {
    int page_num = 0;
    bool has_more = true;

    while (has_more) {
        Page page = co_await FetchPageAwaitable{page_num++};
        has_more = page.has_next;

        for (auto& item : page.items) {
            co_yield std::move(item);
        }
    }
}

// Consumer
DriverTask consume() {
    auto stream = paginated_stream();
    int count = 0;
    while (auto item = co_await stream.next()) {
        std::printf("[%d] %s\n", ++count, item->c_str());
    }
    std::printf("Total items: %d\n", count);
}

int main() {
    consume();
    // Output:
    // [1] alpha
    // [2] bravo
    // [3] charlie
    // [4] delta
    // [5] echo
    // Total items: 5
}
```

Here is the architecture for the paginated stream case, showing how the three participants (consumer, generator, and the fetch awaitable) interleave:

```cpp
consume()           paginated_stream()          FetchPageAwaitable
    │                      │                           │
    ├─ co_await next() ──> │                           │
    │                      ├─ co_await fetch(0) ──>    │
    │                      │                     <─────┤ {alpha,bravo,charlie}
    │                      ├─ co_yield "alpha" ──>     │
    │  <- "alpha" ────────┘                           │
    ├─ co_await next() ──> │                           │
    │                      ├─ co_yield "bravo" ──>     │
    │  <- "bravo" ────────┘                           │
    ...
    │                      ├─ co_await fetch(1) ──>    │
    │                      │                     <─────┤ {delta,echo}
    ...
```

The consumer sees a flat stream of strings. The pagination logic, I/O waits, and batching are completely encapsulated inside the async generator.

---

## Notes

- **Symmetric transfer** (`await_suspend` returning `coroutine_handle<>`) is essential for async generators to avoid stack overflow when chaining suspensions. Without it, each resume nests another stack frame.
- An async generator's `final_suspend` must transfer to the consumer - otherwise the consumer remains permanently suspended, causing a deadlock/leak.
- Unlike `std::generator` (C++23), there is no standard async generator yet. P2502 covers sync generators; async generators remain a library-level concern.
- In real systems, the `co_await` inside the generator integrates with an **event loop** (Boost.Asio, libuv, io_uring). The awaitables register callbacks that resume the coroutine when I/O completes.
- Memory model: each async generator is one coroutine frame. The consumer is a separate frame. At most **two frames** are alive per producer-consumer pair - O(1) memory regardless of stream length.
- Error propagation: if the generator's body throws, `unhandled_exception()` captures it, and the consumer's `co_await gen.next()` rethrows when `await_resume()` calls `rethrow_exception`.
