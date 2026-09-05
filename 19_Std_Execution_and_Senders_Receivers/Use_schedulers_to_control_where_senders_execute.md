# Use schedulers to control where senders execute

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

Schedulers are how the sender/receiver model separates "what to compute" from "where to compute it." A scheduler is a lightweight, copyable handle to an execution context - a thread pool, a GPU dispatch queue, an I/O event loop, or anything else that can run work. You obtain a scheduler from the context and use it to direct senders onto that context without changing the sender's logic at all.

Here is the core API:

| API | Purpose |
| --- | --- |
| `schedule(sched)` | Create a sender that completes on the scheduler |
| `starts_on(sched, sender)` | Ensure sender starts on given scheduler |
| `continues_on(sched)` | Transfer continuation to a different scheduler |

---

## Self-Assessment

### Q1: Use `continues_on` (transfer) to move execution onto a scheduler

`continues_on` is useful when you want to begin work on the current thread and then hand off to a different context. The pipeline below starts on the calling thread, prints the thread ID, then transfers to the thread pool and prints again. The result value flows through the transfer unchanged:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto pool_sched = pool.get_scheduler();

    // Start with a value on the current thread:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Before transfer, thread: "
                      << std::this_thread::get_id() << '\n';
            return x;
        })
        // Transfer execution to the thread pool:
        | stdexec::continues_on(pool_sched)
        | stdexec::then([](int x) {
            std::cout << "After transfer, thread: "
                      << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 84
}
// Output:
// Before transfer, thread: 140000001 (main)
// After transfer, thread: 140000050 (pool)
// Result: 84
```

The thread IDs show exactly where each `then` ran. Everything before `continues_on` ran on the main thread; everything after ran on a pool thread. The value `42` was carried across the transfer automatically.

### Q2: `schedule()` creates a sender on a specific scheduler

`schedule(sched)` is the most direct way to start execution on a scheduler. It produces a sender that emits no value - it just "enters" the scheduler's context. Every `then` after it will run on that context unless you explicitly transfer away:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // schedule(sched) produces a sender with no value.
    // It just "enters" the scheduler's context:
    auto work = stdexec::schedule(sched)
        | stdexec::then([]() {
            // This runs on a pool thread:
            std::cout << "On pool: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        })
        | stdexec::then([](int x) {
            // Still on pool thread (unless transferred):
            return x * 2;
        });

    auto [val] = stdexec::sync_wait(std::move(work)).value();
    std::cout << "Result: " << val << '\n';  // 84

    // Multiple schedulers:
    exec::static_thread_pool io_pool(1);
    auto io_sched = io_pool.get_scheduler();

    // Start on compute pool, transfer to I/O pool:
    auto pipeline = stdexec::schedule(sched)
        | stdexec::then([]() { return heavy_computation(); })
        | stdexec::continues_on(io_sched)
        | stdexec::then([](auto result) { write_to_disk(result); });
}
```

The two-scheduler pattern at the bottom - compute on a CPU pool, then transfer to a single-threaded I/O pool to write the result - is a common real-world pattern. It keeps CPU-bound work off the I/O thread and I/O-bound work off the CPU pool.

### Q3: Scheduler (where) vs Sender (what)

The table below captures the design distinction. Schedulers are handles, not containers - they are cheap to copy and do not own the execution context themselves. Senders are lazy descriptions that can be moved through a pipeline:

| Concept | Scheduler (Where) | Sender (What) |
| --- | --- | --- |
| Represents | Execution context (threads, GPU) | Work to be done |
| Owns resources | No (lightweight handle) | No (lazy description) |
| Examples | Thread pool scheduler, io_context sched | `just(42)`, `then(f)` |
| Copyable | Yes (cheap) | Movable (owns state) |
| Lifetime tied to | Execution context | Pipeline composition |

The diagram below shows how `starts_on` wires the two together:

```cpp
Separation:
                    Sender (WHAT)
                    ┌─────────────────┐
                    │ just(42)        │
                    │ | then(double)  │
                    │ | then(print)   │
                    └────────┬────────┘
                             │
                    starts_on(sched, sender)
                             │
                    Scheduler (WHERE)
                    ┌────────┴────────┐
                    │ thread_pool     │
                    │ GPU context     │
                    │ io_context      │
                    └─────────────────┘
```

Because the scheduler is plugged in at the call site rather than baked into the sender, the same sender description can run anywhere. Here is that portability shown in three lines:

```cpp
auto work = stdexec::just(42) | stdexec::then([](int x) { return x * 2; });

// Run on thread pool:
auto [r1] = stdexec::sync_wait(stdexec::starts_on(pool_sched, work)).value();

// Run on GPU:
auto [r2] = stdexec::sync_wait(stdexec::starts_on(gpu_sched, work)).value();

// Run inline:
auto [r3] = stdexec::sync_wait(work).value();
```

All three produce the same result (`84`), but they run in completely different execution contexts. The `work` sender is not modified between uses - it is just connected to different schedulers.

---

## Notes

- `continues_on` was previously called `transfer` in earlier P2300 drafts, so you may see both names in older code and documentation.
- `starts_on(sched, s)` is semantically equivalent to `schedule(sched) | let_value([s]{ return s; })` - it schedules an entry point and then runs the sender from there.
- A scheduler is equality-comparable: two schedulers obtained from the same pool compare equal (`pool.get_scheduler() == pool.get_scheduler()` is `true`).
- You can write custom schedulers for specialized contexts - FPGAs, RDMA networks, or any domain-specific dispatch mechanism - by conforming to the scheduler concept.
- The separation of what/where is what makes P2300 senders genuinely portable across hardware and execution models, not just a syntactic improvement over raw threads.
