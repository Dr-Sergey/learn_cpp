# Use std::execution::just to create a sender that emits a value

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

`just(values...)` is the simplest sender you'll write. It completes immediately with whatever values you give it, does no async work at all, and requires no scheduler. It's the entry point of nearly every sender pipeline you'll build - the place where you inject the initial data that the rest of the chain will transform.

| Variant | Channel | Carries |
| --- | --- | --- |
| `just(v1, v2, ...)` | value | Zero or more values |
| `just_error(e)` | error | One error value |
| `just_stopped()` | stopped | Nothing (cancellation signal) |

Under the hood, `just` is about as simple as a sender gets. When you call `connect` on it and then `start` the resulting operation state, it immediately calls `set_value` on your receiver:

```cpp
just(42) internals:

  connect(just(42), receiver)
        |
        v
  operation_state:
    start() { receiver.set_value(42); }  // immediate
```

---

## Self-Assessment

### Q1: Create `just(42)` and connect it to a receiver

Most of the time you'll use `sync_wait` to drive a pipeline, but it's worth seeing the low-level `connect` + `start` API at least once. It shows exactly what `sync_wait` is doing for you under the hood:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

// A minimal receiver for demonstration:
struct my_receiver {
    using receiver_concept = stdexec::receiver_t;

    void set_value(int val) && noexcept {
        std::cout << "Received value: " << val << '\n';
    }
    void set_error(std::exception_ptr) && noexcept {
        std::cout << "Error!\n";
    }
    void set_stopped() && noexcept {
        std::cout << "Stopped!\n";
    }

    auto get_env() const noexcept {
        return stdexec::empty_env{};
    }
};

int main() {
    // Create the simplest sender:
    auto sender = stdexec::just(42);

    // Manual connect + start (low-level API):
    auto op = stdexec::connect(std::move(sender), my_receiver{});
    stdexec::start(op);
    // Output: Received value: 42

    // Normal usage with sync_wait:
    auto [val] = stdexec::sync_wait(stdexec::just(42)).value();
    std::cout << "Got: " << val << '\n';  // 42

    // Multiple values:
    auto [a, b, c] = stdexec::sync_wait(
        stdexec::just(1, 2.0, std::string("three"))
    ).value();
    std::cout << a << ", " << b << ", " << c << '\n';  // 1, 2, three
}
```

The manual `connect` + `start` path shows that `set_value(42)` is called synchronously inside `start()`. There's no thread switch, no suspension - `just` really does complete inline.

### Q2: Chain `just()` with `then()` to transform values

In practice you'll almost never use `just` in isolation. Its purpose is to seed the pipeline with an initial value so that the `then` steps have something to work with:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <string>

int main() {
    // Basic chaining:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) {
            return x * 2;  // 84
        })
        | stdexec::then([](int x) {
            return std::to_string(x);  // "84"
        })
        | stdexec::then([](std::string s) {
            return "Result: " + s;  // "Result: 84"
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << result << '\n';  // Result: 84

    // Multi-value to single-value:
    auto sum = stdexec::just(10, 20, 30)
        | stdexec::then([](int a, int b, int c) {
            return a + b + c;  // then receives ALL values
        });
    auto [total] = stdexec::sync_wait(std::move(sum)).value();
    std::cout << "Total: " << total << '\n';  // 60

    // Void just as trigger:
    auto trigger = stdexec::just()  // no values
        | stdexec::then([]() {
            return 42;  // produces a value from nothing
        });
    auto [v] = stdexec::sync_wait(std::move(trigger)).value();
    std::cout << "Triggered: " << v << '\n';  // 42
}
```

The void `just()` at the end is a useful pattern when you want a pipeline that starts with no input data - for example, a background task that creates its own data. The `then` after it conjures a value from thin air.

### Q3: `just()` as a building block in larger pipelines

Because `just` carries no scheduler dependency, it's the cleanest way to inject constants or configuration values into a pipeline. You can also use it inside `let_value` to create sub-senders, or pair it with `when_all` for simple fan-out:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <string>

// just() is the simplest sender because:
// 1. No async work (completes synchronously)
// 2. No scheduler dependency
// 3. Known value at compile time
// 4. Zero overhead

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Building block: inject initial values into a pipeline:
    auto compute = stdexec::just(std::string("input_data"))
        | stdexec::let_value([sched](const std::string& data) {
            // Use just to create a new sender in let_value:
            return stdexec::starts_on(sched,
                stdexec::just(data.size())
                | stdexec::then([](size_t len) {
                    return len * 2;  // process on pool
                })
            );
        });

    auto [result] = stdexec::sync_wait(std::move(compute)).value();
    std::cout << "Result: " << result << '\n';  // 20

    // Fan-out with when_all + just:
    auto parallel = stdexec::when_all(
        stdexec::just(10) | stdexec::then([](int x) { return x * x; }),
        stdexec::just(20) | stdexec::then([](int x) { return x * x; }),
        stdexec::just(30) | stdexec::then([](int x) { return x * x; })
    );
    auto [a, b, c] = stdexec::sync_wait(std::move(parallel)).value();
    std::cout << a << ", " << b << ", " << c << '\n';  // 100, 400, 900
}
```

The `when_all` example shows three independent pipelines, each seeded by their own `just`, running concurrently. The results arrive together once all three are done. This is the canonical fan-out pattern with senders.

---

## Notes

- `just()` copies its arguments into the sender. Use `std::move` for move-only types.
- `just()` completes synchronously - the receiver's `set_value` is called inside `start()`.
- `just()` is to senders what `return` is to coroutines.
- Use `just()` to inject constants, configuration, or mock data into pipelines.
- Combine with `when_all` for parallel fan-out patterns.
