# Use async_scope for structured concurrency and lifetime management

**Category:** Std Execution and Senders Receivers  
**Standard:** Not version-specific  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3149r5.html>  

---

## Topic Overview

`async_scope` is the P2300 answer to a very specific problem: how do you do fire-and-forget work without creating a lifetime disaster? Detached threads feel convenient until something goes wrong - and "something goes wrong" usually means a task outlives the locals it captured, or an exception terminates the process, or a task runs forever with no way to cancel it. `async_scope` solves all three by tracking every spawned task and giving you a clean way to wait for them all.

Here is a quick reference for the operations you will use:

| Operation | Purpose |
| --- | --- |
| `scope.spawn(sender)` | Launch fire-and-forget task |
| `scope.spawn_future(sender)` | Launch task, get a sender for its result |
| `scope.on_empty()` | Returns sender that completes when all tasks finish |
| `scope.request_stop()` | Cancel all spawned tasks |

---

## Self-Assessment

### Q1: Spawn fire-and-forget work and join before scope exits

The pattern is: create the scope, spawn as many tasks as you need, then call `on_empty()` and wait. The scope internally counts in-flight tasks, and `on_empty()` produces a sender that does not complete until that count reaches zero. Here is a full example with a mutex-protected accumulator:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <thread>
#include <mutex>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    std::mutex mtx;
    int total = 0;

    // Spawn fire-and-forget tasks:
    for (int i = 0; i < 20; ++i) {
        scope.spawn(
            stdexec::schedule(sched)
            | stdexec::then([i, &total, &mtx]() {
                std::lock_guard lk(mtx);
                total += i;
            })
        );
    }

    // Join: wait for ALL spawned tasks to complete:
    stdexec::sync_wait(scope.on_empty());

    // Safe to read total — all tasks done:
    std::cout << "Total: " << total << '\n';  // 190 (0+1+...+19)
}
```

The read of `total` after `on_empty()` returns needs no additional synchronization. The scope's completion guarantee establishes a happens-before relationship: every write inside the spawned tasks is visible to code that runs after `on_empty()`.

### Q2: Why unstructured spawn (detach) causes bugs

It is easy to dismiss "detach is dangerous" as a stylistic preference, but the bugs it causes are real and reproducible. Here are three distinct failure modes, each in its own function:

```cpp
#include <thread>
#include <iostream>
#include <vector>

// Bug 1: Lifetime — dangling reference
void lifetime_bug() {
    std::vector<int> data = {1, 2, 3};
    std::thread t([&data]() {
        // data might be destroyed!
        for (int x : data) std::cout << x;  // UB!
    });
    t.detach();
}  // data destroyed, thread still running

// Bug 2: Cancellation — no way to stop detached thread
void cancellation_bug() {
    std::thread t([]() {
        while (true) {
            // Infinite loop — no way to cancel!
            heavy_work();
        }
    });
    t.detach();
    // Can't cancel t. It runs until process exits.
}

// Bug 3: Exception safety — unhandled exception in detached thread
void exception_bug() {
    std::thread t([]() {
        throw std::runtime_error("oops");
        // -> std::terminate! No way to catch.
    });
    t.detach();
}

// async_scope solves ALL three:
// 1. Lifetime: on_empty() guarantees all tasks complete before locals die
// 2. Cancellation: request_stop() cancels all spawned tasks via stop_token
// 3. Exceptions: errors propagate via error channel, not terminate
```

The reason these bugs are insidious is that they often do not surface in tests. A fast machine with little contention will frequently let the detached thread finish before the stack frame is destroyed - until it does not, and the failure is non-deterministic and hard to reproduce.

### Q3: `async_scope` propagates cancellation on destruction

You can cancel all outstanding work by calling `request_stop()` on the scope. This does not terminate tasks immediately - cancellation is cooperative. It sets the stop token that each spawned task can check. Tasks that are well-behaved (that is, they check the stop token periodically) will complete with `set_stopped`. Tasks that are already running to completion will finish normally. Either way, `on_empty()` will eventually complete:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <atomic>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();
    std::atomic<int> completed{0};

    {
        exec::async_scope scope;

        // Spawn long-running tasks:
        for (int i = 0; i < 10; ++i) {
            scope.spawn(
                stdexec::schedule(sched)
                | stdexec::then([&completed, i]() {
                    // Simulate work that checks for cancellation:
                    // In real code, check get_stop_token().stop_requested()
                    completed.fetch_add(1);
                })
            );
        }

        // Request cancellation of all spawned work:
        scope.request_stop();

        // on_empty() waits for tasks that already started to finish,
        // but cancelled tasks complete with set_stopped():
        stdexec::sync_wait(scope.on_empty());

        // Some tasks may have completed before cancellation:
        std::cout << "Completed: " << completed.load() << '\n';
    }
    // scope destroyed safely — all work is done or cancelled
}

// Cancellation flow:
// 1. scope.request_stop() sets the stop source
// 2. Each spawned sender gets a stop_token from the scope
// 3. stop_requested() returns true
// 4. Well-behaved senders call set_stopped(receiver)
// 5. scope tracks all completions (value, error, or stopped)
// 6. on_empty() completes when count reaches zero
```

The comment block at the bottom lays out the full cancellation flow. The key word in step 4 is "well-behaved" - the scope cannot force a task to stop, it can only signal the intent. It is the task's responsibility to check and honour that signal.

---

## Notes

- `async_scope` is proposed in P3149 (counting_scope) for C++26; the current implementation is `exec::async_scope` in the `stdexec` library.
- `spawn()` is fire-and-forget and discards the task's result. If you need the result, use `spawn_future()` which returns a sender you can connect to later.
- Always call `on_empty()` before the scope's locals go out of scope, or let the scope destructor handle cleanup - but relying on the destructor is less explicit and harder to reason about.
- `request_stop()` is cooperative: senders must actively check their stop token to actually cancel. A sender that ignores the stop token will run to completion regardless.
- Think of `async_scope` as a `WaitGroup` (Go) or `TaskGroup` (Swift) for C++ - the concept is well-established in other languages with structured concurrency, and the C++ version follows the same mental model.
