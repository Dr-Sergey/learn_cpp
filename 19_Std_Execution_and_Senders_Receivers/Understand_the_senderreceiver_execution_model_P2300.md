# Understand the sender/receiver execution model (P2300)

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

P2300 is the proposal that defines how async work is modeled in modern C++. The central insight is a clean separation into three roles. If you have ever felt that async code in C++ was messy - threads scattered everywhere, futures that do not chain, callbacks you cannot cancel - P2300 addresses each of those problems with a unified design.

| Role | Responsibility | Analogy |
| --- | --- | --- |
| **Sender** | Describes work to be done | A recipe (not cooked yet) |
| **Receiver** | Handles the result/error/cancellation | A callback with 3 channels |
| **Operation State** | Running work binding sender+receiver | A cooking session |

The protocol that connects them is deliberately minimal. You call `connect` to bind a sender to a receiver, which produces an operation state. You call `start` on the operation state to begin execution. When the work finishes, exactly one of three completion signals fires:

```cpp
P2300 protocol:

  Sender + Receiver ────connect()────-> Operation State
                                          │
                                      start()
                                          │
                                 (work happens)
                                          │
                     ┌────────────┼────────────┐
                     ▼            ▼            ▼
             set_value(recv)  set_error(recv)  set_stopped(recv)
```

The three completion channels correspond to success, failure, and cancellation. Every sender must declare up front which channels it can use via `completion_signatures`, so the compiler can verify the entire pipeline at compile time before a single line executes.

---

## Self-Assessment

### Q1: Three roles in P2300

Let's walk through each role so the mental model is concrete.

**Sender** (describes work):

- A sender is a lazy description of async work. It holds the parameters and logic but does NOT execute.
- Examples: `just(42)`, `schedule(sched) | then(f)`, custom I/O senders.
- Composable via pipe operator: `just(42) | then(f) | then(g)`.

**Receiver** (handles completion):

- A receiver is the "continuation" that processes the result. It has three methods:
  - `set_value(args...)` - success
  - `set_error(err)` - failure
  - `set_stopped()` - cancellation
- Also carries an **environment** (stop_token, scheduler, allocator).

**Operation State** (running work):

- Created by `connect(sender, receiver)`.
- `start(op_state)` begins execution.
- Must remain alive until one completion signal fires.
- Non-movable after `start()`.

Here is the smallest possible custom sender that shows all three roles in one place. It is deliberately trivial so you can focus on the structure rather than the work:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

// Minimal demonstration of the three roles:
struct my_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int)
    >;
    int value;

    template <typename Receiver>  // <-- Receiver role
    struct op_state {             // <-- Operation State role
        using operation_state_concept = stdexec::operation_state_t;
        int value;
        Receiver recv;
        void start() noexcept {
            stdexec::set_value(std::move(recv), value);
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return op_state<Receiver>{value, std::move(recv)};
    }
};  // <-- Sender role

int main() {
    auto [result] = stdexec::sync_wait(my_sender{42}).value();
    std::cout << result << '\n';  // 42
}
```

`my_sender` is the sender. `op_state` is the operation state. The `Receiver` template parameter represents the receiver role - in this case `sync_wait` supplies its own internal receiver that captures the result and hands it back to you.

### Q2: Separation of what/where/result

One of the most useful properties of P2300 is that you can write a computation once and run it anywhere. The computation does not know or care whether it runs on a thread pool, a GPU, or inline. That decision belongs to the scheduler:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // WHAT to do (sender — describes computation):
    auto computation = stdexec::just(21)
        | stdexec::then([](int x) { return x * 2; });

    // WHERE to run it (scheduler — execution context):
    auto on_pool = stdexec::starts_on(sched, std::move(computation));

    // WHAT to do with result (receiver — handled by sync_wait):
    auto [result] = stdexec::sync_wait(std::move(on_pool)).value();
    std::cout << "Result: " << result << '\n';  // 42

    // Key insight: the SAME computation can run on DIFFERENT schedulers
    // without changing the computation code!
}
```

This separation is not just a design nicety. It means you can test your computation logic using the inline scheduler (no threads, fully deterministic) and then deploy it with a thread pool or GPU scheduler without touching the computation code.

```cpp
Separation of concerns:

  WHAT to do          WHERE to run       WHAT to do with result
  (sender)            (scheduler)        (receiver/consumer)
  ┌─────────────┐    ┌───────────┐    ┌──────────────┐
  │ just(21)      │    │ thread pool │    │ sync_wait()    │
  │ | then(f)     │    │ GPU         │    │ | then(print)  │
  │ | then(g)     │    │ io_context  │    │ | store(var)   │
  └─────────────┘    └───────────┘    └──────────────┘
  Independent!       Independent!       Independent!
```

### Q3: P2300 vs `std::async` vs raw threads

Here is how P2300 compares to the older approaches you may already be familiar with. The differences are not cosmetic - each row represents a class of bugs that P2300 eliminates by design:

| Aspect | Raw threads | `std::async` | P2300 senders |
| --- | --- | --- | --- |
| Composition | Manual join/detach | Limited (no chaining) | Pipe `\|` operator |
| Error handling | `std::terminate` | Exception from `get()` | Error channel |
| Cancellation | Manual signals | None built-in | Stop tokens |
| Scheduler control | Thread per task | Unspecified | Explicit schedulers |
| Resource management | Manual | Future owns result | Operation state RAII |
| Structured lifetime | No (detach is UB-prone) | Blocking `get()` | `async_scope` |

The code comparison shows the progression from the most manual approach to the most structured:

```cpp
// Raw threads: manual, error-prone
std::thread t([] { return heavy_work(); });
t.join(); // must not forget!

// std::async: simple but limited
auto f = std::async(std::launch::async, heavy_work);
int result = f.get(); // blocks, no chaining

// P2300: composable, structured, schedulable
auto pipeline = stdexec::schedule(sched)
    | stdexec::then(heavy_work)
    | stdexec::then(process_result)
    | stdexec::upon_error(handle_error);
auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
```

The P2300 version is longer on the page, but each piece has a clear role and the compiler can check the whole thing end to end.

---

## Notes

- P2300 is the biggest addition to C++ async since coroutines landed in C++20, and it complements coroutines rather than competing with them.
- The reference implementation is `stdexec` from NVIDIA, which is open source under the MIT license and available at github.com/NVIDIA/stdexec.
- The key design goals are zero allocations, zero type erasure, and zero-overhead abstraction - pipelines that compile down to the same code as hand-written async state machines.
- `sync_wait` is the only blocking primitive in the model; everything else is fully lazy and non-blocking.
- A useful mental model: think of senders as `std::ranges` for async work - lazy, composable, value-semantic, and verifiable at compile time.
