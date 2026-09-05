# Understand schedulers and execution contexts in std::execution

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

In P2300, a **scheduler** is a lightweight handle to an **execution context** - the place where work actually runs. The execution context owns real resources (threads, a GPU, an event loop); the scheduler is just a cheap token you pass around to say "I want work to run on this context." Calling `schedule(sched)` produces a sender that, when started, completes on that context.

| Concept | Description | Example |
| --- | --- | --- |
| Execution context | Underlying resource (threads, GPU, event loop) | Thread pool, io_context |
| Scheduler | Handle/factory for senders on a context | `pool.get_scheduler()` |
| `schedule(sched)` | Returns a sender that completes on the context | Start point for pipelines |
| `starts_on(sched, sndr)` | Run sender on a specific scheduler | Replace context |
| `continues_on(sched)` | Transfer subsequent work to another scheduler | Mid-pipeline context switch |

The relationship between the two is worth drawing out clearly:

```cpp
Scheduler vs Execution Context:

┌───────────────────────────────────────┐
│ Execution Context (thread pool)     │
│ [Thread 1] [Thread 2] [Thread 3]    │
└───────────────────┬───────────────────┘
                    |
            ┌───────┴──────┐
            │  Scheduler    │  <- lightweight handle
            │  schedule()   │  <- produces senders
            └──────────────┘
```

---

## Self-Assessment

### Q1: Difference between a scheduler and a thread pool

This is the distinction that catches people off guard. The thread pool is the expensive thing - it owns threads and work queues and has a lifetime you must manage. The scheduler is just a copyable, cheaply movable handle that tells the framework where to send work. You can freely copy a scheduler and pass it around without spawning any threads.

| Thread Pool | Scheduler |
| --- | --- |
| Owns threads (heavy resource) | Lightweight handle (copyable, cheaply movable) |
| Manages work queues | Doesn't own resources |
| Has a lifetime | Doesn't control lifetime |
| One pool can have multiple schedulers | One scheduler refers to one context |
| `pool.get_scheduler()` creates one | `schedule(sched)` creates senders |

The code below shows both in action. Notice how `sched` is created from the pool and then used to build a pipeline - it is just a value:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    // Execution context: owns 4 threads
    exec::static_thread_pool pool(4);

    // Scheduler: lightweight handle to the pool
    auto sched = pool.get_scheduler();
    // sched is cheap to copy, doesn't own threads

    // schedule() produces a sender that will complete on one of the pool's threads:
    auto work = stdexec::schedule(sched)
        | stdexec::then([]() {
            std::cout << "Running on pool thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        });

    auto [result] = stdexec::sync_wait(std::move(work)).value();
    std::cout << "Result: " << result << '\n';
}
// Output:
// Running on pool thread: 140234567890 (some pool thread ID)
// Result: 42
```

### Q2: Use `schedule()` to start work on a context

`schedule(sched)` is your entry point. It returns a sender that does nothing by itself - it just positions the next `then` to run on the target context. The pipeline example below also shows `starts_on`, which is a more explicit alternative for routing an existing sender to a specific scheduler.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <string>

int main() {
    exec::static_thread_pool pool(2);
    auto sched = pool.get_scheduler();

    // schedule(sched) returns a sender that:
    // 1. When started, enqueues work on the pool
    // 2. Completes (set_value()) on a pool thread
    auto pipeline =
        stdexec::schedule(sched)         // switch to pool
        | stdexec::then([]() {
            auto tid = std::this_thread::get_id();
            std::cout << "Step 1 on thread: " << tid << '\n';
            return 42;
        })
        | stdexec::then([](int x) {
            auto tid = std::this_thread::get_id();
            std::cout << "Step 2 on thread: " << tid << '\n';
            return std::to_string(x);
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';

    // starts_on alternative (more explicit):
    auto work = stdexec::just(100)
        | stdexec::then([](int x) { return x * 2; });
    auto on_pool = stdexec::starts_on(sched, std::move(work));
    auto [val] = stdexec::sync_wait(std::move(on_pool)).value();
    std::cout << "starts_on result: " << val << '\n';  // 200
}
```

### Q3: Transfer work mid-pipeline with `continues_on`

`continues_on` is the adaptor you reach for when you want to run different stages of a pipeline on different contexts - for example, CPU-heavy work on a compute pool and file writes on an I/O pool. It inserts a context switch in the middle of the pipeline without breaking the composition.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool compute_pool(4);  // CPU-heavy work
    exec::static_thread_pool io_pool(2);       // I/O work

    auto compute_sched = compute_pool.get_scheduler();
    auto io_sched = io_pool.get_scheduler();

    auto pipeline =
        stdexec::schedule(compute_sched)    // start on compute pool
        | stdexec::then([]() {
            std::cout << "Compute on: "
                      << std::this_thread::get_id() << '\n';
            return 42;  // heavy computation
        })
        | stdexec::continues_on(io_sched)   // TRANSFER to I/O pool
        | stdexec::then([](int result) {
            std::cout << "I/O on: "
                      << std::this_thread::get_id() << '\n';
            // Write result to file/network on I/O pool
            return result;
        })
        | stdexec::continues_on(compute_sched)  // back to compute
        | stdexec::then([](int result) {
            std::cout << "Back on compute: "
                      << std::this_thread::get_id() << '\n';
            return result * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << result << '\n';  // 84
}
// Output (thread IDs will vary):
// Compute on: 140001
// I/O on: 140050
// Back on compute: 140002
// Final: 84
```

The entire pipeline is assembled at compile time. The context switches happen at runtime when the `continues_on` senders complete and re-enqueue work on the new context:

```cpp
Pipeline context transitions:

  schedule(compute)  then(heavy_work)  continues_on(io)  then(save)
  [compute pool]     [compute pool]    [io pool]         [io pool]
       |                  |                |                 |
       └─────────────────┴────────────────┴────────────────┘
       all connected at compile time, context switches at runtime
```

---

## Notes

- `schedule(sched)` is the entry point for all scheduler-based work.
- `starts_on(sched, sender)` is equivalent to `schedule(sched) | let_value([s=sndr]{ return s; })`.
- `continues_on(sched)` was previously called `transfer()` in earlier P2300 drafts.
- Schedulers are equality-comparable: two schedulers from the same pool compare equal.
- The `run_loop` scheduler runs work inline on the calling thread (useful for testing).
