# Implement a custom sender type satisfying the sender concept

**Category:** Std Execution and Senders Receivers  
**Standard:** C++20  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

To satisfy the `sender` concept, a type must do two things: advertise which completion channels it can produce, and support `connect()` with any compatible receiver. The compiler uses these pieces to type-check entire pipelines at compile time - if your downstream `then` expects an `int` but the sender emits a `std::string`, you get a hard error before anything runs.

Here is the minimal contract:

```cpp
Requirements for sender concept:

1. sender_concept tag type alias     -> identifies as a sender
2. completion_signatures type alias   -> what channels it can complete on
3. connect(receiver) -> operation_state
   - operation_state must have start()
   - start() eventually calls one of:

     set_value(recv, args...) | set_error(recv, err) | set_stopped(recv)
```

The `completion_signatures` type is how you declare what your sender can emit. Think of it as a compile-time function signature for your async operation:

| Completion Signature | Meaning |
| --- | --- |
| `set_value_t(int, std::string)` | Success: sends int and string |
| `set_error_t(std::exception_ptr)` | Error: sends exception |
| `set_error_t(std::error_code)` | Error: sends error_code |
| `set_stopped_t()` | Cancellation |

---

## Self-Assessment

### Q1: Async file descriptor read sender

This is a realistic example of a custom sender that wraps a platform I/O operation. The key insight is that `start()` submits the actual work and then signals the receiver when it is done. In production you would use io_uring or epoll here; the example uses blocking `read()` to keep the focus on the sender protocol.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// async_read_sender.cpp - a sender that reads from a file descriptor
// Simplified example using POSIX read(); real implementation would use io_uring/epoll
#include <stdexec/execution.hpp>
#include <iostream>
#include <vector>
#include <cstring>
#include <unistd.h>    // POSIX read()
#include <fcntl.h>
#include <system_error>

// Custom sender: async_read(fd, max_bytes)
struct async_read_sender {
    using sender_concept = stdexec::sender_t;

    // Can succeed with bytes read (vector<char>),
    // or fail with an exception_ptr:
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(std::vector<char>),
        stdexec::set_error_t(std::exception_ptr)
    >;

    int fd;
    std::size_t max_bytes;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;

        int fd;
        std::size_t max_bytes;
        Receiver receiver;

        void start() noexcept {
            // In production: submit to io_uring and resume on completion.
            // Simplified: blocking read (for demonstration).
            try {
                std::vector<char> buffer(max_bytes);
                ssize_t n = ::read(fd, buffer.data(), max_bytes);
                if (n < 0) {
                    throw std::system_error(
                        errno, std::system_category(), "read failed");
                }
                buffer.resize(static_cast<std::size_t>(n));
                stdexec::set_value(std::move(receiver), std::move(buffer));
            } catch (...) {
                stdexec::set_error(std::move(receiver),
                                   std::current_exception());
            }
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{fd, max_bytes, std::move(recv)};
    }
};

auto async_read(int fd, std::size_t max_bytes) {
    return async_read_sender{fd, max_bytes};
}

int main() {
    // Read from stdin (fd 0) - pipe some data:
    // echo "hello" | ./async_read_sender
    auto pipeline = async_read(STDIN_FILENO, 1024)
        | stdexec::then([](std::vector<char> data) {
            std::string s(data.begin(), data.end());
            std::cout << "Read " << data.size() << " bytes: " << s;
            return data.size();
        })
        | stdexec::upon_error([](std::exception_ptr ep) -> std::size_t {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cerr << "Error: " << e.what() << '\n';
            }
            return 0;
        });

    auto [bytes] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Total bytes: " << bytes << '\n';
}
```

### Q2: Implement `completion_signatures` for a custom sender

The `completion_signatures` declaration is the part most people under-think when writing their first custom sender. It is worth getting right because the compiler uses it to validate your entire pipeline at compile time. The examples below show common patterns - from the trivial single-value case all the way to signatures that depend on the receiver's environment.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <system_error>
#include <string>
#include <variant>

// Example 1: Simple sender - only value channel
struct simple_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int)   // emits a single int
    >;
    // ...
};

// Example 2: Sender with error handling
struct fallible_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(std::string),           // success: string
        stdexec::set_error_t(std::exception_ptr),    // error: exception
        stdexec::set_error_t(std::error_code)        // error: error_code
    >;
    // ...
};

// Example 3: Cancellable sender with multiple value types
struct complex_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int),                   // can emit int
        stdexec::set_value_t(double, std::string),   // OR double + string
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_stopped_t()                     // supports cancellation
    >;
    // ...
};

// Example 4: Dependent completion signatures (computed from environment)
struct dependent_sender {
    using sender_concept = stdexec::sender_t;

    // When signatures depend on the receiver's environment:
    template <typename Env>
    auto get_completion_signatures(Env&&) const
        -> stdexec::completion_signatures<
            stdexec::set_value_t(int),
            stdexec::set_error_t(std::exception_ptr)
        > {
        return {};
    }
    // ...
};

int main() {
    // Verify our types satisfy the sender concept:
    static_assert(stdexec::sender<simple_sender>);
    static_assert(stdexec::sender<fallible_sender>);
    static_assert(stdexec::sender<complex_sender>);
}
```

Notice that a sender can emit more than one *shape* of value (like `complex_sender` with both `int` and `double + string`). The framework handles the resulting variant at the pipeline level, and downstream adaptors need to be able to handle all of the declared shapes.

### Q3: How `connect()` binds a sender to a receiver

The best way to understand the `connect()` protocol is to watch it happen step by step. The diagram below shows the call sequence, and the code below that instruments a custom sender so you can see each step print at runtime.

```cpp
connect() protocol:

  Sender + Receiver ──-> connect() ──-> Operation State
                                           |
                                       start()
                                           |
                              ┌────────────┼────────────┐
                              v            v            v
                      set_value(recv)  set_error(recv) set_stopped(recv)
                        (success)       (failure)     (cancelled)
```

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

// Step-by-step: what sync_wait does internally
struct my_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int)
    >;
    int value;

    template <typename Receiver>
    struct op {
        using operation_state_concept = stdexec::operation_state_t;
        int value;
        Receiver recv;
        void start() noexcept {
            std::cout << "start() called, delivering " << value << '\n';
            stdexec::set_value(std::move(recv), value);
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        std::cout << "connect() called\n";
        return op<Receiver>{value, std::move(recv)};
    }
};

int main() {
    // What happens inside sync_wait(my_sender{42}):
    // 1. sync_wait creates an internal receiver
    // 2. connect(sender, internal_receiver) -> op_state
    //    (our connect() prints "connect() called")
    // 3. start(op_state)
    //    (our start() prints and calls set_value)
    // 4. Internal receiver stores the value
    // 5. sync_wait returns the value
    auto [result] = stdexec::sync_wait(my_sender{42}).value();
    std::cout << "Got: " << result << '\n';
}
// Output:
// connect() called
// start() called, delivering 42
// Got: 42
```

Watching the output in order makes the three-phase protocol concrete: connect, start, signal.

**Summary:**

- `connect(sender, receiver)` produces an `operation_state` - no work starts yet.
- `start(op_state)` kicks off execution.
- The operation state owns all resources needed for the async operation.
- When done, it signals the receiver via exactly one of: `set_value`, `set_error`, or `set_stopped`.

---

## Notes

- `completion_signatures` enables compile-time type checking of sender pipelines.
- If signatures depend on the receiver's environment, use the `get_completion_signatures(env)` overload.
- `stdexec::sender_of<S, int>` checks if sender S can produce an int value.
- Custom senders are the integration point for platform-specific async (io_uring, IOCP).
- Always call exactly one completion function - calling zero or more than one is UB.
