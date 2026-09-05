# Use stdexec::when_all to join multiple senders

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

`when_all` launches multiple senders concurrently and joins their results. The crucial property that makes it different from ad-hoc approaches is **structured concurrency**: the `when_all` sender does NOT complete until every single child sender has finished - or been fully cancelled. There are no escaped tasks.

The diagram below shows the guarantee. Even if C finishes first and there is an error, `when_all` holds the door open and waits for A and B to wrap up before delivering the result:

```cpp
Structured concurrency guarantee:

  when_all(A, B, C)
    |
    +- start A  -------------------------------------------+
    +- start B  --------------------+                      |
    +- start C  -------+            |                      |
                       |            |                      |
                    C done       B done                 A done
                       |            |                      |
                       +------------+----------------------+
                                    |
                             when_all completes
                             (only after ALL done)
```

---

## Self-Assessment

### Q1: Join two independent async computations

This example deliberately uses two different result types - an `int` and a `std::string` - to show that `when_all` handles heterogeneous senders just fine. The tuple that comes back is strongly typed, and structured bindings let you unpack it cleanly:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <string>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto task_a = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "[A] Computing on " << std::this_thread::get_id() << '\n';
            return 100;
        }));

    auto task_b = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "[B] Computing on " << std::this_thread::get_id() << '\n';
            return std::string("hello");
        }));

    // when_all: both run concurrently, results collected:
    auto [num, str] = stdexec::sync_wait(
        stdexec::when_all(std::move(task_a), std::move(task_b))
    ).value();

    std::cout << "Num: " << num << ", Str: " << str << '\n';
}
// Output:
// [A] Computing on 14000001
// [B] Computing on 14000002
// Num: 100, Str: hello
```

Both tasks run on separate threads (different IDs), and the result binds `num` to the `int` and `str` to the `std::string` exactly as declared.

### Q2: Error cancels remaining senders

The important detail here is the `completions` counter. `when_all` waits for all tasks to acknowledge cancellation before delivering the error, but tasks that finish legitimately before they see the stop signal will still increment the counter. The exact count depends on timing - and that is intentional and correct:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <atomic>

std::atomic<int> completions{0};

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto good_task = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            ++completions;
            return 42;
        }));

    auto bad_task = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() -> int {
            throw std::runtime_error("bad task!");
        }));

    auto good_task2 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            ++completions;
            return 99;
        }));

    try {
        stdexec::sync_wait(
            stdexec::when_all(
                std::move(good_task),
                std::move(bad_task),
                std::move(good_task2)
            )
        );
    } catch (const std::runtime_error& e) {
        std::cout << "Error: " << e.what() << '\n';
    }

    // Some good tasks may have completed before cancellation was received.
    // But when_all waits for ALL to finish before propagating the error.
    std::cout << "Completions: " << completions.load() << '\n';
}
// Output:
// Error: bad task!
// Completions: may be 0, 1, or 2 (depends on timing)
```

The reason this trips people up is that they expect "cancelled" to mean "never ran." What it actually means in structured concurrency is "was told to stop and acknowledged the request." A task that already ran before the cancellation arrived is still counted as complete.

### Q3: Structured concurrency - `when_all` never leaks tasks

This is the conceptual heart of the topic. **Structured concurrency** means child lifetimes are bounded by the parent - there is no "fire and forget" that you might forget to wait for. Compare the two patterns:

1. Child lifetimes are bounded by the parent.
2. `when_all` does NOT complete until ALL children finish.
3. Even on error/cancellation, `when_all` waits for cleanup.

```cpp
// SAFE: when_all guarantees all tasks complete before returning
auto pipeline = stdexec::when_all(
    stdexec::starts_on(sched, task_a),
    stdexec::starts_on(sched, task_b),
    stdexec::starts_on(sched, task_c)
);
auto result = stdexec::sync_wait(std::move(pipeline));
// Here, ALL tasks are done. No dangling threads.

// UNSAFE (old style): fire-and-forget with std::async
auto f1 = std::async(task_a);  // might outlive scope!
auto f2 = std::async(task_b);  // might outlive scope!
// If exception thrown before get(), futures destructor blocks.
// No structured lifetime guarantee.
```

The visual comparison below puts it plainly. With `when_all`, you know everything inside the scope has finished when the scope exits. With unstructured async, you have to reason about it yourself - and it is easy to get wrong:

```cpp
Structured vs Unstructured:

Structured (when_all):           Unstructured (async):
+--------------------+     +--------------------+
| scope {             |     | scope {             |
|   when_all(A, B, C) |     |   async(A)          |
|   // ALL finish     |     |   async(B) // may    |
| }                   |     | } // leak if error   |
| // safe: all done   |     | // B might still run!|
+--------------------+     +--------------------+
```

---

## Notes

- `when_all` is the structured concurrency primitive of P2300.
- Results are in **declaration order**, not completion order.
- Use `when_all` + `let_value` for dependent parallel + sequential patterns.
- `when_all_with_variant` handles senders with different signatures.
- `when_all` with zero senders completes immediately with `set_value()`.
