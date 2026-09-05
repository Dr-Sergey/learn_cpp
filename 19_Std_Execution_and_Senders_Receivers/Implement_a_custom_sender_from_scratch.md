# Implement a custom sender from scratch

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

A sender is an object that describes asynchronous work. It does not *do* the work - it just describes it. To build one from scratch you need three pieces working together. The anatomy below shows how they relate:

```cpp
Sender anatomy:

┌──────────────────────────────────────────────┐
│ Sender (describes work)                      │
│  - completion_signatures: what it can emit   │
│  - connect(receiver) -> operation_state      │
├──────────────────────────────────────────────┤
│ Operation State (holds live work)            │
│  - start() -> begins execution              │
│  - Stores receiver + any state               │
│  - MUST live until completion signal fires   │
├──────────────────────────────────────────────┤
│ Receiver (consumer)                          │
│  - set_value(args...)  -> success            │
│  - set_error(err)      -> failure            │
│  - set_stopped()       -> cancellation       │
└──────────────────────────────────────────────┘
```

The reason this trips people up is that the sender itself is inert. All it does is hold a description of the work and implement `connect()`. The operation state is what actually *runs* - and it only runs when `start()` is called on it.

---

## Self-Assessment

### Q1: Write a `just_on(scheduler, value)` sender

This example shows two approaches to the same idea: a sender that delivers a value on a specific scheduler. The first version sketches the full hand-rolled anatomy; the second shows how to achieve the same thing by composing existing senders - which is usually what you want in practice.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// just_on_sender.cpp - schedules delivery of a value on a given scheduler
// Using stdexec for concepts and utilities
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

// A sender that schedules a value delivery on a specific scheduler
template <typename Scheduler, typename T>
struct just_on_sender {
    using sender_concept = stdexec::sender_t;

    // Completion signatures: can send T or an exception
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(T),
        stdexec::set_error_t(std::exception_ptr)
    >;

    Scheduler sched;
    T value;

    // Operation state: holds the receiver and orchestrates execution
    template <typename Receiver>
    struct operation_state {
        using operation_state_concept = stdexec::operation_state_t;

        Scheduler sched;
        T value;
        Receiver receiver;

        // Inner operation state for the schedule() sender
        using InnerOp = stdexec::connect_result_t<
            stdexec::schedule_result_t<Scheduler>,
            /* ... simplified ... */
        >;

        void start() noexcept {
            // Schedule work on the target scheduler, then deliver value
            try {
                auto work = stdexec::schedule(sched)
                    | stdexec::then([this]() {
                        stdexec::set_value(std::move(receiver), std::move(value));
                    });
                // In a real implementation, you'd connect + start this inner sender
                auto op = stdexec::connect(std::move(work), /* inner receiver */...);
                stdexec::start(op);
            } catch (...) {
                stdexec::set_error(std::move(receiver), std::current_exception());
            }
        }
    };

    template <typename Receiver>
    auto connect(Receiver recv) const {
        return operation_state<Receiver>{sched, value, std::move(recv)};
    }
};

// Factory function:
template <typename Scheduler, typename T>
auto just_on(Scheduler sched, T value) {
    return just_on_sender<Scheduler, T>{std::move(sched), std::move(value)};
}

// Simpler practical approach using existing building blocks:
template <typename Scheduler, typename T>
auto just_on_simple(Scheduler sched, T value) {
    // Compose from existing senders:
    return stdexec::schedule(sched)
        | stdexec::then([v = std::move(value)]() mutable { return std::move(v); });
}

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto s = just_on_simple(sched, 42)
        | stdexec::then([](int x) {
            std::cout << "Got " << x << " on thread "
                      << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(s)).value();
    std::cout << "Result: " << result << '\n';  // 84
}
```

### Q2: Implement `connect` returning an operation state with `start()`

Here is the minimal version - just a sender that emits a single `int`. It shows the three required pieces at their absolute smallest, which makes it a good template to copy when you need to write a custom sender.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// Minimal custom sender with explicit connect/start
#include <stdexec/execution.hpp>
#include <iostream>
#include <utility>

// A sender that emits a single integer
struct my_just_int {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int)
    >;

    int value;

    // Operation state - created by connect(), started by start()
    template <typename Receiver>
    struct op_state {
        using operation_state_concept = stdexec::operation_state_t;
        int value;
        Receiver recv;

        // start() begins the work and signals the receiver
        void start() noexcept {
            // Deliver the value to the receiver's value channel:
            stdexec::set_value(std::move(recv), value);
        }
    };

    // connect() binds this sender to a receiver
    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return op_state<Receiver>{value, std::move(recv)};
    }
};

int main() {
    // Use our custom sender in a pipeline:
    auto pipeline = my_just_int{42}
        | stdexec::then([](int x) {
            std::cout << "Received: " << x << '\n';
            return x;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';
}
// Output:
// Received: 42
// Result: 42
```

The protocol is always the same three steps, no matter how complex the sender:

1. `connect(sender, receiver)` -> returns `operation_state` object
2. `start(op_state)` -> begins execution
3. When done, op_state calls one of: `set_value(recv, ...)`, `set_error(recv, ...)`, or `set_stopped(recv)`

### Q3: Operation state lifetime rules

This is where custom senders most often go wrong. The operation state holds everything the async operation needs - the receiver, any buffers, any child operation states. If you destroy it before the completion signal fires, you have undefined behavior. The rules are strict:

```cpp
Lifetime diagram:

auto op = connect(sender, receiver);  // op_state created
         |
         v
     start(op);                        // execution begins
         |
         v (work happens asynchronously)
         |
     set_value(recv, result);          // completion signal fires
         |
         v
     op can now be destroyed           // ONLY after completion!
```

**Rules:**

1. The `operation_state` object MUST remain alive from `start()` until a completion signal fires.
2. You may NOT move an operation state after `start()` is called.
3. You may NOT call `start()` more than once.
4. The operation state typically stores: the receiver, intermediate buffers, child operation states.

The "bad" example below shows exactly the kind of mistake that is easy to write and hard to spot. The "good" examples show the two correct patterns - either let `sync_wait` manage lifetime for you, or keep the operation state on the stack and make sure the sender is synchronous:

```cpp
// BAD - dangling operation state:
void bad() {
    auto s = stdexec::just(42) | stdexec::then([](int x) { return x; });
    {
        auto op = stdexec::connect(std::move(s), my_receiver{});
        stdexec::start(op);
    }  // op destroyed here - but completion may not have fired!
    // UNDEFINED BEHAVIOR if the work is asynchronous
}

// GOOD - sync_wait manages lifetime:
void good() {
    auto s = stdexec::just(42) | stdexec::then([](int x) { return x; });
    auto [result] = stdexec::sync_wait(std::move(s)).value();
    // sync_wait keeps op_state alive until completion
}

// GOOD - manual lifetime management:
void manual() {
    auto s = stdexec::just(42);
    auto op = stdexec::connect(std::move(s), my_receiver{});
    stdexec::start(op);
    // op stays alive on the stack; synchronous sender completes before return
}
```

---

## Notes

- In practice, compose from existing senders (`just`, `then`, `schedule`) rather than writing from scratch.
- Custom senders are needed for integrating async I/O (epoll, io_uring, IOCP).
- The `tag_invoke` CPO (customization point object) pattern is used for `connect` in stdexec.
- Operation states are typically non-movable after `start()` - use `std::optional` or heap if needed.
- `stdexec::connect` is the CPO that calls your sender's `connect` method.
