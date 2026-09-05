# Use std::execution::sync_wait to block until a sender completes

**Category:** Std Execution and Senders Receivers  
**Standard:** C++23  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

Sender pipelines are lazy and composable, but eventually you need to actually run one and get the result back into synchronous code - the entry point of your program, a test, or a thread entry function. That is exactly what `sync_wait` is for.

`sync_wait` drives a sender pipeline from synchronous code. It blocks the current thread until the sender completes via any channel (value, error, or stopped). The return type is always `std::optional<std::tuple<Values...>>`, which lets it cleanly represent all three outcomes:

| Channel | `sync_wait` behavior |
| --- | --- |
| `set_value(vals...)` | Returns `optional<tuple<vals...>>` |
| `set_error(exception_ptr)` | Rethrows the exception |
| `set_stopped()` | Returns `nullopt` |

---

## Self-Assessment

### Q1: Basic `sync_wait` usage - extract results

Let's start with the simplest case. You build a pipeline, call `sync_wait`, and then pull the result out of the returned optional/tuple. Structured bindings make this feel quite natural:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    // Build pipeline:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) { return x + 1; });

    // sync_wait blocks and returns the result:
    auto result = stdexec::sync_wait(std::move(pipeline));
    // type: std::optional<std::tuple<int>>

    // Extract with structured bindings:
    auto [value] = result.value();
    std::cout << "Result: " << value << '\n';  // 43

    // One-liner pattern:
    auto [v] = stdexec::sync_wait(
        stdexec::just(10, 20)
        | stdexec::then([](int a, int b) { return a + b; })
    ).value();
    std::cout << "Sum: " << v << '\n';  // 30

    // Void sender:
    auto r = stdexec::sync_wait(stdexec::just());
    if (r.has_value()) {
        std::cout << "Void sender completed\n";
    }
}
```

The `one-liner pattern` - calling `.value()` directly on the result of `sync_wait` - is the most common idiom in practice when you know the pipeline cannot be cancelled.

### Q2: `sync_wait` as the escape hatch from async to sync

The reason this trips people up is the temptation to call `sync_wait` from *inside* a pipeline. Do not do that. `sync_wait` is a boundary crossing tool - it lives at the edge between the async world and the synchronous world. It belongs at program entry points, not inside callbacks.

```cpp
Async world (lazy, composable)     Sync world (blocking, sequential)
  +----------------------+     +----------------------+
  | just(42)               |     | int main() {         |
  | | then(f)              |     |                      |
  | | let_value(g)         |     |   auto [v] =         |
  | | continues_on(sched)  | --- |     sync_wait(pipe)  |
  | | then(h)              |     |       .value();      |
  +----------------------+     |   use(v);            |
          lazy pipeline            | }                    |
                                   +----------------------+
                                        sync_wait bridges
```

The code below shows what to do and what to avoid:

```cpp
// GOOD: at program boundaries
int main() {
    auto [r] = stdexec::sync_wait(build_pipeline()).value();
    return r;
}

void test_case() {
    auto [r] = stdexec::sync_wait(build_pipeline()).value();
    ASSERT_EQ(r, 42);
}

// BAD: inside a sender callback (deadlock risk!)
auto bad = stdexec::just(42)
    | stdexec::then([](int x) {
        // NEVER do this:
        auto [r] = stdexec::sync_wait(other_pipeline()).value();  // DEADLOCK!
        return r;
    });
// Use let_value instead for nested async work.
```

The reason the `// BAD:` example deadlocks is that `sync_wait` needs to own an execution context (an internal `run_loop`) to drive the pipeline. If you are already inside a pipeline running on such a context, calling `sync_wait` again can cause it to wait for itself.

### Q3: Error and cancellation handling

This example goes through all three channels. The three `{}` blocks are self-contained so you can see each case cleanly, followed by a practical pattern that handles all of them together:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <optional>
#include <string>

int main() {
    // Case 1: Value -> optional(tuple(value))
    {
        auto result = stdexec::sync_wait(stdexec::just(42));
        if (result) {
            auto [v] = *result;
            std::cout << "Value: " << v << '\n';  // 42
        }
    }

    // Case 2: Error -> rethrows exception
    {
        auto err_sender = stdexec::just(42)
            | stdexec::then([](int) -> int {
                throw std::runtime_error("computation failed");
            });

        try {
            stdexec::sync_wait(std::move(err_sender));
        } catch (const std::runtime_error& e) {
            std::cout << "Error: " << e.what() << '\n';
        }
    }

    // Case 3: Stopped -> nullopt
    {
        auto stopped = stdexec::just_stopped();
        auto result = stdexec::sync_wait(std::move(stopped));
        if (!result) {
            std::cout << "Cancelled (nullopt)\n";
        }
    }

    // Practical pattern with error handling:
    auto safe_pipeline = stdexec::just(42)
        | stdexec::then([](int x) -> int {
            if (x < 0) throw std::invalid_argument("negative");
            return x * 2;
        })
        | stdexec::upon_error([](std::exception_ptr) {
            return 0;  // default on error
        });

    auto [safe] = stdexec::sync_wait(std::move(safe_pipeline)).value();
    std::cout << "Safe: " << safe << '\n';  // 84
}
// Output:
// Value: 42
// Error: computation failed
// Cancelled (nullopt)
// Safe: 84
```

The `upon_error` at the end of the safe pipeline converts the error channel back into a value, which means `sync_wait` can never throw from it - a useful pattern when you want to guarantee a result.

---

## Notes

- `sync_wait` creates an internal `run_loop` to drive execution.
- Formally in the standard: `std::this_thread::sync_wait`.
- `sync_wait_with_variant` handles senders with multiple completion signatures.
- Never call from within a sender pipeline - use `let_value` for nested async.
- The return type is always `optional<tuple<Values...>>`.
