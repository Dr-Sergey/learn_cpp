# Implement a custom sender type with the sender concept

**Category:** Std Execution and Senders Receivers  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

This topic builds a practical `timer_sender` that completes after a delay. A timer is a good teaching example because it involves all three completion channels (value, error, and stopped) and requires checking for cancellation at meaningful points in the operation. It demonstrates the full sender protocol in a setting that feels realistic.

| Concept | Role |
| --- | --- |
| `sender_concept` tag | Marks the type as a sender |
| `completion_signatures` | Advertises value/error/stopped channels |
| `connect(receiver)` | Binds sender to a receiver, returns `operation_state` |
| `start()` | Begins actual execution |
| `set_value` / `set_error` / `set_stopped` | Signals completion to the receiver |

---

## Self-Assessment

### Q1: Write a `timer_sender` that completes after a delay

The trick with a timer sender is that you need to check for cancellation *before* sleeping and *after* waking up - the stop token might have been requested in either window. Also notice that `start()` is marked `noexcept`, so all exceptions must be caught internally and routed through the error channel.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// timer_sender.cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <chrono>
#include <thread>

template <typename Duration>
struct timer_sender {
    using sender_concept = stdexec::sender_t;

    // Completes with void on success, or exception_ptr on error
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(),                       // success: no value
        stdexec::set_error_t(std::exception_ptr),     // error
        stdexec::set_stopped_t()                      // cancellation
    >;

    Duration delay;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;
        Duration delay;
        Receiver receiver;

        void start() noexcept {
            try {
                // Check for cancellation before sleeping:
                auto token = stdexec::get_stop_token(
                    stdexec::get_env(receiver));
                if (token.stop_requested()) {
                    stdexec::set_stopped(std::move(receiver));
                    return;
                }

                // Sleep for the specified duration:
                std::this_thread::sleep_for(delay);

                // Check cancellation again after waking:
                if (token.stop_requested()) {
                    stdexec::set_stopped(std::move(receiver));
                    return;
                }

                // Signal success (void):
                stdexec::set_value(std::move(receiver));
            } catch (...) {
                stdexec::set_error(std::move(receiver),
                                   std::current_exception());
            }
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{delay, std::move(recv)};
    }
};

// Factory:
template <typename Rep, typename Period>
auto async_sleep(std::chrono::duration<Rep, Period> d) {
    return timer_sender<std::chrono::duration<Rep, Period>>{d};
}

int main() {
    using namespace std::chrono_literals;

    auto pipeline = async_sleep(100ms)
        | stdexec::then([]() {
            std::cout << "Timer fired!\n";
            return 42;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';
}
// Output:
// Timer fired!
// Result: 42
```

### Q2: Implement `completion_signatures` for channel advertising

`completion_signatures` is the compile-time contract that lets the framework type-check your whole pipeline before anything runs. The example below shows both the static typedef form and the dynamic `get_completion_signatures(env)` form - the latter is useful when you need the signatures to vary depending on what execution context you are running on.

```cpp
#include <stdexec/execution.hpp>
#include <string>
#include <system_error>

// Completion signatures tell the compiler what a sender can emit.
// This enables compile-time type checking of entire pipelines.

// Example: a network read sender
struct net_read_sender {
    using sender_concept = stdexec::sender_t;

    // We advertise three possible completions:
    using completion_signatures = stdexec::completion_signatures<
        // Success: delivers bytes read + buffer
        stdexec::set_value_t(std::size_t, std::string),
        // Error: delivers an exception or error_code
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_error_t(std::error_code),
        // Cancellation: operation was cancelled
        stdexec::set_stopped_t()
    >;

    // ... connect/operation_state ...
};

// The compiler uses these signatures to verify pipeline types:
// net_read_sender{}
//   | then([](size_t n, std::string buf) { ... })  // OK: matches set_value_t(size_t, string)
//   | upon_error([](auto err) { ... })              // OK: handles errors
//   | upon_stopped([] { ... });                      // OK: handles cancellation

// COMPILE ERROR if types don't match:
// net_read_sender{}
//   | then([](int x) { ... })  // ERROR: no set_value_t(int)!

// Dependent signatures (vary based on receiver environment):
struct env_dependent_sender {
    using sender_concept = stdexec::sender_t;

    template <typename Env>
    auto get_completion_signatures(Env&& env) const {
        // Can inspect the environment to decide signatures:
        return stdexec::completion_signatures<
            stdexec::set_value_t(int),
            stdexec::set_error_t(std::exception_ptr)
        >{};
    }
};
```

Notice the commented-out error case - the compiler catches the type mismatch at pipeline assembly time, not at runtime. This is one of the key benefits of declaring signatures explicitly.

### Q3: The `connect()` CPO and resulting `operation_state`

The `debug_sender` below adds print statements to `connect()` and `start()` so you can see the exact sequence of events when `sync_wait` drives a pipeline. This kind of instrumentation is genuinely useful when you are debugging a custom sender for the first time.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// connect() is the bridge between description (sender) and execution (operation_state)

#include <stdexec/execution.hpp>
#include <iostream>

struct debug_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int)
    >;
    int value;

    template <typename Receiver>
    struct debug_op {
        using operation_state_concept = stdexec::operation_state_t;
        int value;
        Receiver recv;

        // start() is called ONCE after connect():
        void start() noexcept {
            std::cout << "[debug_op] start() called\n";
            std::cout << "[debug_op] delivering value: " << value << '\n';
            stdexec::set_value(std::move(recv), value);
            std::cout << "[debug_op] receiver signaled\n";
        }

        // operation_state is non-movable after start():
        debug_op(debug_op&&) = default;
        debug_op& operator=(debug_op&&) = delete;
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        std::cout << "[debug_sender] connect() called\n";
        return debug_op<Receiver>{value, std::move(recv)};
    }
};

int main() {
    auto [result] = stdexec::sync_wait(
        debug_sender{42} | stdexec::then([](int x) {
            std::cout << "[then] got " << x << '\n';
            return x;
        })
    ).value();
    std::cout << "Final: " << result << '\n';
}
// Output:
// [debug_sender] connect() called
// [debug_op] start() called
// [debug_op] delivering value: 42
// [debug_op] receiver signaled
// [then] got 42
// Final: 42
```

The output confirms the three-phase ordering: connect first, start second, then the completion signal flows downstream. The `[then] got 42` line shows the value reaching the next stage in the pipeline.

---

## Notes

- `completion_signatures` is the key type-level contract between senders and receivers.
- The `sender` concept checks for either a `completion_signatures` typedef or a `get_completion_signatures` method.
- `connect()` is a CPO (customization point object) - it calls your member `connect` via `tag_invoke` in stdexec.
- Operation states are typically stored in variant-like structures when composed.
- Real timer senders would use platform timers (timerfd, WaitableTimer) instead of sleep.
