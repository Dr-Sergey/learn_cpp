# Use std::this_thread::sync_wait to drive a sender pipeline synchronously

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

`std::this_thread::sync_wait` is the formal standard name for the synchronous blocking driver. It is the bridge between the async, lazy world of sender pipelines and the synchronous code that needs to actually use the result. It wraps an async sender pipeline and blocks the calling thread until the sender completes.

The return type encodes all three possible completion channels cleanly:

| Channel | Behavior |
| --- | --- |
| `set_value(vals...)` | Returns `optional<tuple<vals...>>` |
| `set_error(exception_ptr)` | Rethrows the contained exception |
| `set_error(error_code)` | Throws `std::system_error` |
| `set_stopped()` | Returns `std::nullopt` |

---

## Self-Assessment

### Q1: Block the current thread until the pipeline completes

The pipeline in this example runs on a thread pool, but the calling thread (main) blocks at `sync_wait` until the result is ready. Watch the thread IDs - the lambda prints a different ID from the one `main` is running on, which confirms the work really happened elsewhere:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Build an async pipeline:
    auto pipeline = stdexec::schedule(sched)
        | stdexec::then([]() {
            std::cout << "Running on pool thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        })
        | stdexec::then([](int x) { return x * 2; });

    // sync_wait blocks until pipeline completes:
    // (In the standard: std::this_thread::sync_wait)
    auto result = stdexec::sync_wait(std::move(pipeline));
    // type: std::optional<std::tuple<int>>

    if (result) {
        auto [value] = *result;
        std::cout << "Result: " << value << '\n';  // 84
    }

    // Common one-liner pattern:
    auto [v] = stdexec::sync_wait(
        stdexec::just(10) | stdexec::then([](int x) { return x * 3; })
    ).value();
    std::cout << "Quick: " << v << '\n';  // 30
}
```

The one-liner at the end - calling `.value()` directly - is the preferred idiom when you know the pipeline cannot be cancelled and exceptions will propagate naturally.

### Q2: How `sync_wait` bridges async and sync worlds

Understanding what `sync_wait` does internally helps you reason about when it is safe to call and why nesting it causes deadlocks. Here is the step-by-step process it goes through:

```cpp
Sync_wait internals:

  1. Creates an internal run_loop
  2. Creates a special receiver that:
     - On set_value: stores result, stops the run_loop
     - On set_error: stores error, stops the run_loop
     - On set_stopped: sets nullopt, stops the run_loop
  3. Connects the sender to this receiver -> operation_state
  4. Starts the operation_state
  5. Runs the run_loop (blocks calling thread)
  6. When receiver is signaled, run_loop stops
  7. Returns the stored result

  Calling thread:     Async world:
  +----------+       +---------------+
  | sync_wait |--->| connect+start |
  | (blocks)  |       | ... work ...  |
  |           |<---| set_value(42) |
  | returns   |       +---------------+
  | opt(42)   |
  +----------+
```

The key insight is that `sync_wait` provides an execution context (a `run_loop`) for senders that do not specify one. If the sender uses `starts_on(pool_sched, ...)`, the actual work runs on the pool - `sync_wait`'s `run_loop` only handles the final notification when the result comes back. This is why nesting `sync_wait` inside a pipeline is dangerous: you would be trying to drive a nested `run_loop` while the outer one is already waiting.

### Q3: Error and stopped signal propagation

This example walks through all three channels in separate blocks so you can see each case in isolation. The final block shows the fully defensive pattern that handles all three outcomes:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <system_error>

int main() {
    // Error propagation: exception_ptr -> rethrown
    {
        auto sender = stdexec::just(42)
            | stdexec::then([](int) -> int {
                throw std::runtime_error("async failure");
            });

        try {
            stdexec::sync_wait(std::move(sender));
        } catch (const std::runtime_error& e) {
            std::cout << "Exception: " << e.what() << '\n';
        }
    }

    // Error propagation: error_code
    {
        auto sender = stdexec::just_error(
            std::make_error_code(std::errc::connection_refused));

        try {
            stdexec::sync_wait(std::move(sender));
        } catch (const std::system_error& e) {
            std::cout << "System error: " << e.what() << '\n';
        }
    }

    // Stopped signal: returns nullopt
    {
        auto sender = stdexec::just_stopped();
        auto result = stdexec::sync_wait(std::move(sender));

        if (!result.has_value()) {
            std::cout << "Pipeline was cancelled\n";
        }
    }

    // Practical: handle all cases
    {
        auto sender = stdexec::just(42)
            | stdexec::then([](int x) { return x * 2; });

        try {
            auto result = stdexec::sync_wait(std::move(sender));
            if (result) {
                auto [v] = *result;
                std::cout << "Value: " << v << '\n';    // 84
            } else {
                std::cout << "Cancelled\n";
            }
        } catch (const std::exception& e) {
            std::cout << "Error: " << e.what() << '\n';
        }
    }
}
// Output:
// Exception: async failure
// System error: Connection refused
// Pipeline was cancelled
// Value: 84
```

The error_code case is worth calling out separately. When a sender completes with a `std::error_code`, `sync_wait` wraps it in a `std::system_error` and throws that - you do not need to check for `error_code` returns explicitly.

---

## Notes

- In stdexec (NVIDIA impl): `stdexec::sync_wait`. In standard: `std::this_thread::sync_wait`.
- Never call `sync_wait` from within a sender callback - deadlock risk.
- `sync_wait_with_variant` handles senders with multiple completion signatures.
- The internal `run_loop` is lightweight - no threads are created.
- Always use at program boundaries: `main()`, test functions, thread entry points.
