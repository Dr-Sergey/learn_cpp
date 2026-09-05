# Understand the sender/receiver model and why it replaces futures and callbacks

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2300r10.html>  

---

## Topic Overview

P2300 replaces `std::future` and raw callbacks with a structured model of three primitives. Before looking at each primitive, it is worth understanding why `std::future` needed replacing at all - because the problems are not superficial.

| Primitive | Role | Lifetime |
| --- | --- | --- |
| **Sender** | Describes async work (lazy) | Until connected |
| **Receiver** | Continuation that handles result | Part of operation state |
| **Operation State** | Live work binding sender+receiver | Until completion signal |

Here is the honest list of what `std::future` gets wrong:

- `std::future` requires heap allocation for shared state - the shared state is reference-counted and allocated every time, even for trivial tasks.
- `std::future` has no built-in chaining: there is no `.then()` in standard C++, so you cannot express "when this finishes, do that" without blocking.
- `std::future` has no cancellation - once you have started a `std::async` task, there is no standard way to stop it.
- `std::future` forces eager execution - work starts immediately when you call `std::async`, which means you cannot compose operations before running them.

The sender/receiver model fixes all four of these at the design level, not as add-ons.

---

## Self-Assessment

### Q1: Three async primitives

The best way to see the three primitives working together is to look at what `sync_wait` does internally. You hand it a sender, and it quietly creates a receiver, connects them, starts the operation, and waits. Here is the full picture:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    // 1. SENDER: describes work (lazy)
    auto sender = stdexec::just(42)
        | stdexec::then([](int x) { return x * 2; });
    // Nothing has executed yet!

    // 2. RECEIVER: handles the result
    // (sync_wait creates an internal receiver that stores the result)

    // 3. OPERATION STATE: live work
    // connect(sender, receiver) -> operation_state
    // start(operation_state) -> begins execution
    // On completion: set_value(receiver, 84)

    auto [result] = stdexec::sync_wait(std::move(sender)).value();
    std::cout << result << '\n';  // 84
}

// What sync_wait does internally:
// 1. Creates a receiver (stores result + synchronization)
// 2. auto op = connect(sender, receiver)   // binds them
// 3. start(op)                              // runs the work
// 4. Blocks until receiver is signaled
// 5. Returns the stored result
```

The thing to notice is that `sender` is just a value. After the `| then(...)` line, no computation has happened at all. The transformation is described but not executed. That laziness is not a quirk - it is intentional, and it is what makes composition possible.

### Q2: Why `std::future::then()` is not composable but sender pipelines are

The code below shows the `std::future` approach and its problems, followed by the sender alternative. The problems with futures are not bugs in your code - they are fundamental limitations of the design:

```cpp
// std::future problems:
#include <future>
#include <iostream>

void future_problems() {
    // Problem 1: No chaining (no .then() in standard C++)
    auto f = std::async(std::launch::async, []() { return 42; });
    int result = f.get();  // BLOCKS here
    // Can't do: f.then([](int x) { return x * 2; }).then(print);

    // Problem 2: Eager execution (work starts immediately)
    auto f2 = std::async(std::launch::async, heavy_work);
    // heavy_work is ALREADY running, can't compose first

    // Problem 3: No error channel (exceptions only via get())
    // Problem 4: No cancellation
    // Problem 5: Heap allocation for shared state
}

// Sender pipeline: fully composable, lazy, zero-allocation
#include <stdexec/execution.hpp>

void sender_solution() {
    // Composable chaining with pipe operator:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) { return x * 2; })     // chain 1
        | stdexec::then([](int x) { return x + 10; })     // chain 2
        | stdexec::upon_error([](auto) { return -1; })     // error handling
        | stdexec::upon_stopped([]() { return 0; });        // cancellation
    // NOTHING has executed! Pure description.

    // Execute when ready:
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    // result = 94
}
```

| Feature | `std::future` | Sender Pipeline |
| --- | --- | --- |
| Chaining | No `.then()` in standard | `\| then(f) \| then(g)` |
| Laziness | Eager (runs immediately) | Lazy (runs on sync_wait) |
| Allocation | Heap (shared state) | Stack (operation state) |
| Error handling | Exception via `get()` | Typed error channel |
| Cancellation | None | Stop tokens |
| Parallel fan-out | Manual | `when_all(s1, s2, s3)` |

The allocation difference is worth slowing down on. Every `std::async` call allocates a control block on the heap to hold the shared state between the future and the promise. Sender pipelines store their operation state inline - if you connect a sender to a receiver on the stack, the whole pipeline lives on the stack with zero heap allocation. For hot loops or embedded systems this matters enormously.

### Q3: Separating algorithm from execution context

This is one of the most practical benefits of the model. You write the algorithm once, and the scheduler determines where it runs. The algorithm code does not even need to know whether it is running on a thread pool, a GPU, or the calling thread:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

// The SAME algorithm works on ANY scheduler:
auto make_pipeline(int input) {
    return stdexec::just(input)
        | stdexec::then([](int x) { return x * x; })
        | stdexec::then([](int x) {
            std::cout << "Result: " << x
                      << " on thread " << std::this_thread::get_id() << '\n';
            return x;
        });
}

int main() {
    exec::static_thread_pool pool(4);
    auto pool_sched = pool.get_scheduler();

    // Same pipeline, different execution contexts:

    // 1. Run inline (sync_wait's run_loop scheduler):
    auto [r1] = stdexec::sync_wait(make_pipeline(5)).value();

    // 2. Run on thread pool:
    auto [r2] = stdexec::sync_wait(
        stdexec::starts_on(pool_sched, make_pipeline(5))
    ).value();

    // Algorithm code is IDENTICAL.
    // Only the scheduler changes where it runs.
    std::cout << r1 << " == " << r2 << '\n';  // 25 == 25
}
```

The thread ID in the output will differ between the two runs, but the result will be identical. That is the promise of the model: computation is portable across execution contexts because the "where" is plugged in at the call site, not baked into the algorithm.

---

## Notes

- Senders are to async what ranges are to synchronous sequences: lazy, composable, and value-semantic. If you are comfortable with ranges, the mental model transfers directly.
- The P2300 model is inspired by Facebook's Folly executors and libunifex, both of which explored similar ideas in production before the standard proposal was written.
- `std::future` is not deprecated and will continue to work, but it is now considered the low-level, manually-managed option rather than the recommended approach for new async code.
- Because sender pipelines express the entire computation graph as a value, the compiler can see and optimize the whole graph at once - something impossible with callbacks or futures.
- C++26 is the target for `std::execution` to land in the standard library; until then, `stdexec` from NVIDIA is the reference implementation you can use today.
