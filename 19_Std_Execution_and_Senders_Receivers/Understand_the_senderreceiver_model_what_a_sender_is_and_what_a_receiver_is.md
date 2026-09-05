# Understand the sender/receiver model: what a sender is and what a receiver is

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

The sender/receiver model splits async work into two cleanly separated halves. A **sender** is the description side - it knows what to compute but has not started yet. A **receiver** is the consumption side - it knows what to do with the result once the work is done. The two are kept apart until you deliberately connect them.

```cpp
Sender (describes):                Receiver (consumes):
┌─────────────────────┐          ┌───────────────────────┐
│ What to compute       │          │ set_value(args...)     │
│ Completion signatures │─connect─>│ set_error(err)         │
│ No execution yet      │          │ set_stopped()          │
└─────────────────────┘          │ Environment (stop_tok) │
                                   └───────────────────────┘
```

The reason this split matters is that it lets you compose, inspect, and redirect the description before a single byte of work happens. You can build a pipeline, attach it to different schedulers, analyze its completion signatures at compile time, and only run it when you are ready. None of that is possible when work is eager.

---

## Self-Assessment

### Q1: Sender = description, Receiver = callback with 3 channels

Let's nail down what each role actually means in code.

**Sender:**

- A sender is a **value type** that describes async work.
- It does NOT start execution on construction.
- It advertises what it can produce via `completion_signatures`.
- It can be composed: `sender1 | then(f) | then(g)` builds a new sender.

**Receiver:**

- A receiver is the **callback** that processes the result.
- It has three methods (one must be called exactly once):
  - `set_value(recv, args...)` - success path
  - `set_error(recv, err)` - error path
  - `set_stopped(recv)` - cancellation path
- It also provides an **environment** via `get_env(recv)` containing:
  - Stop token for cancellation
  - Current scheduler
  - Allocator

Here is a minimal example. The computation is pure description until `sync_wait` runs it. Notice that `make_computation()` returns without doing any work - the lambda inside `then` never runs at that point:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

// Sender: describes work
auto make_computation() {
    return stdexec::just(10, 20)          // sender: emit (10, 20)
        | stdexec::then([](int a, int b) { // sender: transform
            return a + b;
        });
    // Returns a sender. No work done yet!
}

int main() {
    auto sender = make_computation();
    // sender is just a description — zero side effects

    // sync_wait creates a receiver internally:
    //   set_value(recv, 30)  -> stores 30
    //   set_error(recv, err) -> stores exception
    //   set_stopped(recv)    -> returns nullopt
    auto result = stdexec::sync_wait(std::move(sender));
    if (result) {
        auto [val] = *result;
        std::cout << val << '\n';  // 30
    }
}
```

`sync_wait` is the bridge that makes the description execute. It creates its own receiver internally, connects it to your sender, calls `start`, and blocks until one of the three completion methods fires.

### Q2: Senders are lazy - no work until connected+started

Laziness is one of those properties that is easy to state but easy to forget when you are writing code. This example makes it concrete with a global counter that only increments when the sender actually runs:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

int side_effect_counter = 0;

int main() {
    // Creating a sender does NOT execute anything:
    auto sender = stdexec::just(42)
        | stdexec::then([](int x) {
            ++side_effect_counter;  // NOT called yet!
            std::cout << "Executing!\n";
            return x * 2;
        });

    std::cout << "Sender created. Counter: "
              << side_effect_counter << '\n';  // 0

    // Can create multiple senders without executing:
    auto sender2 = stdexec::just(100)
        | stdexec::then([](int x) { return x - 1; });

    std::cout << "Two senders, still nothing executed.\n";

    // Execution happens only when connected to receiver and started:
    auto [r1] = stdexec::sync_wait(std::move(sender)).value();
    std::cout << "After sync_wait. Counter: "
              << side_effect_counter << '\n';  // 1
    std::cout << "Result: " << r1 << '\n';     // 84
}
// Output:
// Sender created. Counter: 0
// Two senders, still nothing executed.
// Executing!
// After sync_wait. Counter: 1
// Result: 84
```

**Key insight:** This laziness enables three important things:

- The scheduler to decide WHERE work runs, because the work has not been dispatched yet.
- The compiler to optimize the entire pipeline at once, because the whole graph is visible as a value.
- Multiple pipelines to be composed before any execution, enabling patterns like `when_all` where all branches are assembled first and then started together.

### Q3: Sender/receiver vs future vs callbacks vs coroutines

Here is how the four major async models in C++ compare. The table is dense but each row tells a story:

| Model | Eager/Lazy | Composable | Cancellation | Error Handling | Allocation |
| --- | --- | --- | --- | --- | --- |
| **Callbacks** | Eager | No (callback hell) | Manual | Manual | None |
| **std::future** | Eager | No (.then() missing) | None | Exception | Heap |
| **Coroutines** | Lazy | Sequential (co_await) | Manual (stop_token) | co_await + try/catch | Coroutine frame |
| **Senders** | Lazy | Yes (pipe `\|`) | Built-in (stopped channel) | Typed error channel | Stack (op_state) |

The code examples below show the same logical operation written in each style, so you can feel the ergonomic difference:

```cpp
// Callback style:
async_read(fd, buffer, [](int bytes, error_code ec) {
    if (ec) { handle_error(ec); return; }
    async_write(out, buffer, [](int, error_code ec2) {
        if (ec2) { handle_error(ec2); return; }
        // callback hell...
    });
});

// Future style:
auto f = async_read(fd, buffer);
auto result = f.get();  // blocks, can't chain

// Coroutine style:
task<void> do_io() {
    auto bytes = co_await async_read(fd, buffer);
    co_await async_write(out, buffer);
}

// Sender style:
auto pipeline = async_read_sender(fd, buffer)
    | then([](auto bytes, auto buf) { return buf; })
    | let_value([&](auto buf) { return async_write_sender(out, buf); })
    | upon_error([](auto err) { log(err); });
sync_wait(std::move(pipeline));
```

The coroutine style is the most readable for purely sequential logic, and senders win for parallel fan-out and explicit scheduler control. The reason this trips people up is that coroutines and senders are not competitors - they are designed to interoperate, and many real programs use both. Use coroutines where sequential flow reads naturally, and reach for senders when you need `when_all`, custom schedulers, or explicit cancellation wiring.

---

## Notes

- Senders unify all async patterns under one model: I/O, timers, thread pools, GPU dispatch - all expressed as composable lazy descriptions.
- The three-channel model (value/error/stopped) is inspired by Reactive Extensions (Rx), which proved the pattern at scale in the functional programming world.
- Receivers are normally created by library code (`sync_wait`, `then`, `when_all`) rather than by you directly - you write senders and the adaptors handle the receiver plumbing.
- `completion_signatures` enable zero-cost type checking at compile time: the compiler knows exactly what types can emerge from each channel before any work starts.
- Sender/receiver is more powerful than coroutines alone when it comes to parallel fan-out and fine-grained scheduler control, but the two models compose well together so you do not have to choose one exclusively.
