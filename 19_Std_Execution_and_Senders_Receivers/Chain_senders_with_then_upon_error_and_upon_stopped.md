# Chain senders with then, upon_error, and upon_stopped

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

The P2300 sender/receiver model gives you composable async building blocks. The key thing to internalize right away is that senders are *lazy* - they describe work but execute nothing until something actually drives them (like `sync_wait()`). Until that happens, all you have is a description of a pipeline, not a running computation.

The adaptors below are how you wire that pipeline together. Each one transforms one completion channel into another:

| Adaptor | Purpose | Channel |
| --- | --- | --- |
| `then(f)` | Transform the value channel | value -> value |
| `upon_error(f)` | Handle/recover from errors | error -> value |
| `upon_stopped(f)` | Handle cancellation | stopped -> value |
| `let_value(f)` | Chain dependent senders | value -> sender |
| `let_error(f)` | Chain error-recovery senders | error -> sender |

To make the laziness concrete, here is how a pipeline sits idle until driven:

```cpp
Sender pipeline (lazy):

just(42) | then(double_it) | then(to_string)
   |              |                |
   |              |                └─ returns "84"
   |              └─ returns 84
   └─ emits 42

Nothing executes until sync_wait() or start()!
```

---

## Self-Assessment

### Q1: Build a pipeline with `just | then | then` and explain lazy evaluation

Here is the simplest possible demonstration - a three-stage pipeline where each stage just transforms the value. Pay attention to when the print statements actually fire relative to when the pipeline is built.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// Using stdexec (NVIDIA reference implementation of P2300)
// Install: https://github.com/NVIDIA/stdexec
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <string>

int main() {
    // Pipeline is LAZY: just describes what to emit,
    // then describes transformations - nothing runs yet!
    auto pipeline =
        stdexec::just(21)                           // sender: emits int(21)
        | stdexec::then([](int x) {                 // transform: double it
            std::cout << "Step 1: " << x << " -> " << x * 2 << '\n';
            return x * 2;
        })
        | stdexec::then([](int x) {                 // transform: to string
            std::cout << "Step 2: " << x << " -> \"" << std::to_string(x) << "\"\n";
            return std::to_string(x);
        });

    // At this point, NOTHING has executed.
    // pipeline is just a description of work (a sender).
    std::cout << "Pipeline created (nothing executed yet)\n";

    // sync_wait drives execution: connects + starts + blocks
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // "42"
}
// Output:
// Pipeline created (nothing executed yet)
// Step 1: 21 -> 42
// Step 2: 42 -> "42"
// Result: 42
```

Notice that "Pipeline created" prints *before* the step lambdas fire. That is lazy evaluation in action - the pipeline was assembled at zero cost, and only when `sync_wait()` drove it did anything actually execute.

**Lazy evaluation** means:

- `just(21)` doesn't emit anything - it's a factory.
- Each `then(f)` wraps the previous sender - no function calls happen.
- Only when `sync_wait()` calls `connect()` + `start()` does the pipeline execute.
- This enables the scheduler to decide **where** and **when** work runs.

### Q2: Use `upon_error` to recover from errors

Errors in a sender pipeline travel down a separate *error channel*. A plain `then` ignores that channel entirely - errors just pass through. `upon_error` is the adaptor that intercepts the error channel and converts it back into a value, so the downstream `then` stages can keep running as if nothing went wrong.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <string>
#include <system_error>

// A sender that might fail:
auto fetch_config(bool simulate_fail) {
    if (simulate_fail) {
        // Emit an error via the error channel:
        return stdexec::just_error(
            std::make_exception_ptr(
                std::runtime_error("network timeout")));
    }
    return stdexec::just(std::string("production.json"));
}

int main() {
    // Pipeline with error recovery:
    auto pipeline =
        stdexec::just_error(
            std::make_exception_ptr(std::runtime_error("DB offline")))
        | stdexec::upon_error([](std::exception_ptr ep) -> std::string {
            // upon_error intercepts the error channel
            // and converts it BACK to a value channel:
            try {
                std::rethrow_exception(ep);
            } catch (const std::exception& e) {
                std::cout << "Recovered from: " << e.what() << '\n';
            }
            return "fallback_default";  // resume with fallback value
        })
        | stdexec::then([](std::string config) {
            std::cout << "Using config: " << config << '\n';
            return config;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << result << '\n';
}
// Output:
// Recovered from: DB offline
// Using config: fallback_default
// Final: fallback_default
```

The `then` stage after `upon_error` runs normally with the fallback string because `upon_error` converted the error channel back into a value. From `then`'s perspective, the upstream just produced a `std::string`.

**Key point:** `upon_error` transforms the error channel into a value, so downstream `then` stages continue normally.

### Q3: Handle cancellation with `upon_stopped`

There are three completion channels in P2300: value, error, and *stopped*. The stopped channel fires when an operation is cancelled - for instance, because a timeout expired or the caller requested a stop. `upon_stopped` intercepts that channel and lets you convert it into a value so the rest of the pipeline can keep going.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <string>

int main() {
    // upon_stopped fires when a sender is cancelled
    // (e.g., via stop_token from the environment)
    auto pipeline =
        stdexec::just_stopped()    // simulate cancellation signal
        | stdexec::upon_stopped([]() -> std::string {
            // Convert stopped signal -> value channel
            std::cout << "Operation was cancelled, providing default\n";
            return "cancelled_placeholder";
        })
        | stdexec::then([](std::string val) {
            std::cout << "Continuing with: " << val << '\n';
            return val;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';
}
// Output:
// Operation was cancelled, providing default
// Continuing with: cancelled_placeholder
// Result: cancelled_placeholder
```

Here is the full picture of how all three channels relate to the adaptors. Each `upon_*` intercepts its channel and routes it back through the value channel so the downstream pipeline sees a clean value:

```cpp
Three completion channels in P2300:

               ┌─ set_value(args...)   -> then(f) transforms
Sender ────────┼─ set_error(err)       -> upon_error(f) recovers
               └─ set_stopped()        -> upon_stopped(f) handles cancel

Each upon_* converts its channel BACK to the value channel,
so the rest of the pipeline continues.
```

---

## Notes

- `then` only fires on the **value** channel; errors/stopped pass through untouched.
- `upon_error` and `upon_stopped` convert their channels to values for recovery.
- For dependent sender chaining (returning new senders), use `let_value` / `let_error`.
- The reference implementation is `stdexec` from NVIDIA; the standard library will adopt it in C++26.
- All pipeline composition is at compile time - zero overhead from the chaining itself.
