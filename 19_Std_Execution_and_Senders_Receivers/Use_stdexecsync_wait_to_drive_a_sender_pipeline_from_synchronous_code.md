# Use stdexec::sync_wait to drive a sender pipeline from synchronous code

**Category:** Std Execution and Senders Receivers  
**Standard:** C++17  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

Sender pipelines are lazy - nothing happens until something drives them. In a fully async program that something might be a coroutine scheduler or an event loop. But if you're writing a `main()` function, a test, or any synchronous context, you need a way to say "run this pipeline and give me the result right now." That's what `sync_wait` is for. It's the **bridge** between the lazy, async sender world and normal synchronous code.

| Behavior | Detail |
| --- | --- |
| Blocks? | Yes - blocks current thread |
| Return type | `std::optional<std::tuple<Values...>>` |
| On value | Returns `optional(tuple(values...))` |
| On error | Rethrows the exception |
| On stopped | Returns `std::nullopt` |
| Execution context | Internal `run_loop` (unless sender specifies another) |

Here's a simple picture of what happens at the boundary:

```cpp
sync_wait flow:

  Synchronous code          Async sender world
  +--------------+      +------------------+
  | auto result =  | --->| connect(sender,  |
  |  sync_wait(s)  |      |   receiver)     |
  |              |      | start(op_state) |
  | // blocked   |      | ... work ...    |
  |              | <---| set_value(42)   |
  | // unblocked |      +------------------+
  | result = 42  |
  +--------------+
```

`sync_wait` calls `connect`, then `start`, then sits and waits until the receiver signals completion. When the result arrives, `sync_wait` unblocks and hands you the value wrapped in an `optional<tuple<...>>`.

---

## Self-Assessment

### Q1: `sync_wait` blocks until the pipeline completes

Let's see the blocking behavior in concrete terms. The pipeline runs on the thread pool, but the calling thread waits - and you can see that by observing the thread IDs:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <chrono>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto pipeline = stdexec::schedule(sched)
        | stdexec::then([]() {
            std::cout << "Working on thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        })
        | stdexec::then([](int x) {
            return x * 2;
        });

    std::cout << "Before sync_wait (main thread: "
              << std::this_thread::get_id() << ")\n";

    // sync_wait blocks here until the pipeline completes:
    auto result = stdexec::sync_wait(std::move(pipeline));

    std::cout << "After sync_wait\n";

    // Unpack the result:
    auto [value] = result.value();
    std::cout << "Result: " << value << '\n';  // 84
}
// Output:
// Before sync_wait (main thread: 140000001)
// Working on thread: 140000050
// After sync_wait
// Result: 84
```

The thread IDs show that the computation ran on a pool thread while main was blocked. Execution of the lines after `sync_wait` only resumed once the pool thread delivered the result.

### Q2: Return type is `optional<tuple<...>>` - `nullopt` means stopped

The return type feels a bit verbose at first, but it covers all three completion channels cleanly. A value arrives as `optional(tuple(values...))`, an error is rethrown as an exception, and cancellation produces `nullopt`. Here's each case in one place:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <optional>
#include <tuple>
#include <string>

int main() {
    // Case 1: Value channel -> optional(tuple(values...))
    {
        auto s = stdexec::just(42, std::string("hello"));
        auto result = stdexec::sync_wait(std::move(s));
        // result type: std::optional<std::tuple<int, std::string>>

        if (result) {
            auto [num, str] = *result;
            std::cout << num << ", " << str << '\n';  // 42, hello
        }
    }

    // Case 2: Error channel -> throws exception
    {
        auto s = stdexec::just_error(
            std::make_exception_ptr(std::runtime_error("oops")));
        try {
            stdexec::sync_wait(std::move(s));
        } catch (const std::runtime_error& e) {
            std::cout << "Caught: " << e.what() << '\n';  // Caught: oops
        }
    }

    // Case 3: Stopped channel -> nullopt
    {
        auto s = stdexec::just_stopped();
        auto result = stdexec::sync_wait(std::move(s));
        // result type: std::optional<std::tuple<>>

        if (!result) {
            std::cout << "Stopped (nullopt)\n";
        }
    }

    // Case 4: Void sender -> optional<tuple<>>
    {
        auto s = stdexec::just();
        auto result = stdexec::sync_wait(std::move(s));
        if (result) {
            std::cout << "Void sender completed\n";
        }
    }
}
// Output:
// 42, hello
// Caught: oops
// Stopped (nullopt)
// Void sender completed
```

The reason errors are rethrown rather than returned is that `sync_wait` is the boundary where the async world meets synchronous exception-handling idioms. If you want to inspect the error without catching an exception, you need `let_error` inside the pipeline before the `sync_wait` call.

### Q3: `sync_wait` is the bridge between async and sync

| Aspect | Without sync_wait | With sync_wait |
| --- | --- | --- |
| Pipeline state | Lazy (no work done) | Drives execution |
| Thread | No thread runs it | Internal run_loop or pool |
| Result | Inaccessible | Returned as optional<tuple> |
| Use case | Composition | Entry/exit point |

The most important rule about `sync_wait` is where *not* to use it. Calling it from inside a sender callback can deadlock - if the callback is running on the same thread pool that the inner pipeline needs to make progress, you've created a situation where the outer thread is blocked waiting for the inner pipeline, but the inner pipeline can't run because the thread is blocked.

```cpp
// sync_wait is typically used only at the "edge" of your program:

int main() {
    // Build a complex pipeline (no work happens yet):
    auto pipeline = build_complex_pipeline();

    // sync_wait is the ONLY place where work actually executes:
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();

    // DO NOT use sync_wait inside a sender pipeline!
    // BAD: just(42) | then([](int x) {
    //          sync_wait(another_sender);  // DEADLOCK risk!
    //      });
    // Use let_value instead for nested async work.

    return result;
}
```

**Key rules:**

- Use `sync_wait` only at program boundaries (`main`, test functions, thread entry points).
- Never call `sync_wait` from within a sender callback - it can deadlock.
- For nested async work, use `let_value` to return a sender instead.
- `sync_wait` provides its own `run_loop` execution context unless the sender specifies one via `starts_on`.

---

## Notes

- `sync_wait` is the only way to extract a value from a sender pipeline.
- It creates a `run_loop`, connects a receiver to the sender, and runs the loop until completion.
- `sync_wait_with_variant` handles senders with multiple completion signatures.
- In the standard (`<execution>`), it's `std::this_thread::sync_wait`.
- Calling `sync_wait` from a thread pool thread can deadlock if the pipeline needs that same thread.
