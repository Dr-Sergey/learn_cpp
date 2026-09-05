# Understand stop tokens and cancellation in sender pipelines

**Category:** Std Execution and Senders Receivers  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

Cancellation in the sender/receiver model works through a mechanism called stop tokens. A stop token is a small, cheap-to-copy object that answers one question: "has cancellation been requested?" Every receiver carries an environment, and that environment is where the stop token lives. When the framework wires your pipeline together, it automatically threads the stop token from the outermost consumer all the way down to each individual sender stage - you do not have to pass it manually.

There are two ways a sender can react to cancellation: it can poll the token at convenient checkpoints, or it can register a callback that fires the moment a cancellation request arrives. Once a sender decides to honour the cancellation, it calls `set_stopped` on its receiver instead of `set_value` or `set_error`. That signal then propagates outward through the rest of the pipeline.

Here is a quick reference for the key APIs you will encounter:

| API | Purpose |
| --- | --- |
| `get_stop_token(get_env(recv))` | Extract stop token from receiver |
| `token.stop_requested()` | Poll: was cancellation requested? |
| `stop_callback(token, fn)` | Register callback for stop request |
| `set_stopped(recv)` | Signal cancellation to receiver |
| `stopped_as_optional` | Convert stopped channel to `std::nullopt` |
| `stopped_as_error(err)` | Convert stopped channel to an error |

---

## Self-Assessment

### Q1: Thread a stop_token through a sender pipeline

The first thing to understand is that you do not thread the stop token through explicitly - the pipeline infrastructure does it for you. What you do is reach into the receiver's environment to read the token, then use it to decide whether to keep working or bail out. Here is a custom sender that does exactly that:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <stop_token>

// The stop_token is part of the receiver's ENVIRONMENT.
// Each adaptor (then, when_all, etc.) propagates it automatically.

struct my_cancellable_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int),
        stdexec::set_stopped_t()
    >;
    int work_items;

    template <typename Receiver>
    struct op {
        using operation_state_concept = stdexec::operation_state_t;
        int work_items;
        Receiver recv;

        void start() noexcept {
            // Step 1: Extract stop token from receiver's environment:
            auto env = stdexec::get_env(recv);
            auto token = stdexec::get_stop_token(env);

            // Step 2: Use it for cooperative cancellation:
            int result = 0;
            for (int i = 0; i < work_items; ++i) {
                if (token.stop_requested()) {
                    std::cout << "Cancelled at iteration " << i << '\n';
                    stdexec::set_stopped(std::move(recv));
                    return;
                }
                result += i;
            }
            stdexec::set_value(std::move(recv), result);
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return op<Receiver>{work_items, std::move(recv)};
    }
};

int main() {
    // Pipeline: stop_token flows through each stage automatically
    auto pipeline = my_cancellable_sender{1000}
        | stdexec::then([](int x) {
            // This stage also gets the stop_token in its environment
            std::cout << "Sum: " << x << '\n';
            return x;
        })
        | stdexec::upon_stopped([]() -> int {
            std::cout << "Was cancelled\n";
            return -1;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';
}
// Output: Sum: 499500  Result: 499500
```

Notice the two-step pattern inside `start()`: first pull the environment out of the receiver, then pull the stop token out of the environment. Checking `stop_requested()` at the top of each iteration is the polling approach - it is simple and predictable. When cancellation has not been requested (which is the common case), the work completes normally and `set_value` fires. If the token is tripped, we call `set_stopped` instead and return immediately. The `upon_stopped` adaptor at the end of the pipeline catches that signal and produces a fallback value so `sync_wait` always gets something.

### Q2: Bulk operation with cooperative cancellation

For work that fans out across many items, the same principle applies. Here `bulk` runs a function once per index. In production you would add a token check inside the body; the comments below show where that guard would go:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <vector>
#include <atomic>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    std::vector<double> data(10'000'000, 1.0);

    // bulk() runs a function for each index in a range.
    // Each invocation can check the stop_token:
    auto pipeline =
        stdexec::schedule(sched)
        | stdexec::bulk(data.size(),
            [&data](std::size_t i) {
                // In a real scenario, check stop_token periodically:
                // if (token.stop_requested()) return;
                data[i] = data[i] * 2.0 + 1.0;
            })
        | stdexec::then([&data]() {
            // Verify some results:
            std::cout << "data[0] = " << data[0] << '\n';    // 3.0
            std::cout << "data[999] = " << data[999] << '\n'; // 3.0
        });

    stdexec::sync_wait(std::move(pipeline));
}
// Output:
// data[0] = 3
// data[999] = 3
```

When you need cancellation inside a `bulk` body but you cannot access the token directly, wrapping the work in a `let_value` that builds an inner sender lets you recover the token from the inner receiver's environment:

```cpp
// Pattern: check stop_token every N iterations
auto pipeline = stdexec::schedule(sched)
    | stdexec::let_value([&data]() {
        return stdexec::just()
            | stdexec::then([&data]() {
                for (std::size_t i = 0; i < data.size(); ++i) {
                    // Inside a sender, access the stop_token from the receiver:
                    // if (i % 10000 == 0 && stop_requested) break;
                    data[i] *= 2.0;
                }
            });
    });
```

### Q3: `stopped_as_optional` - convert cancellation to `std::nullopt`

Sometimes the caller does not want cancellation to be a hard stop. It just wants to know "did I get a value or not?" That is exactly what `stopped_as_optional` is for - it absorbs the stopped channel and emits an `optional` instead:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <optional>

int main() {
    // A sender that might be stopped:
    auto maybe_cancelled = stdexec::just_stopped()  // simulates cancellation
        | stdexec::upon_stopped([]() { return 42; });
    // upon_stopped converts stopped -> value

    // Alternative: stopped_as_optional
    // Converts: set_value(x) -> optional(x)
    //           set_stopped() -> nullopt
    auto pipeline = stdexec::just_stopped()
        | stdexec::stopped_as_optional();

    auto result = stdexec::sync_wait(std::move(pipeline)).value();
    // result is std::tuple<std::optional<void>> or similar

    // Practical usage:
    auto compute_or_cancel = stdexec::just(42)
        | stdexec::then([](int x) { return x * 2; })
        | stdexec::stopped_as_optional();

    // If pipeline completes normally: optional(84)
    // If pipeline is cancelled: nullopt
}

// stopped_as_optional is useful when cancellation is expected
// and you want to handle it as a "no result" rather than an error.
//
// Comparison:
//   upon_stopped([] { return default; })  -- provides fallback value
//   stopped_as_optional()                 -- wraps in optional
//   stopped_as_error(err)                 -- converts to error channel
```

The key design decision is which channel you want cancellation to arrive on. If you want it to look like a missing value, use `stopped_as_optional`. If you want it to look like a failure you can recover from with `let_error`, use `stopped_as_error`. If you just want a hardcoded fallback value, `upon_stopped` is the simplest choice.

---

## Notes

- Stop tokens propagate automatically through all P2300 adaptors (`then`, `when_all`, `bulk`, etc.), so you never need to forward them by hand.
- `stopped_as_optional` is particularly handy inside `when_all` where one branch might cancel without wanting to abort the whole fan-out.
- Stop callbacks run on the thread that calls `request_stop()`, so keep them short and non-blocking.
- `in_place_stop_token` (stdexec) is the default token type; `std::stop_token` from C++20 can also be used with the same pattern.
- Never silently discard the stopped channel - if a sender can produce `set_stopped`, callers downstream must be ready to handle it, otherwise the unhandled cancellation will propagate outward and may surprise you.
