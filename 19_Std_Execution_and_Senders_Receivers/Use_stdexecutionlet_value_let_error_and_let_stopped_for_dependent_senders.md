# Use std::execution::let_value, let_error, and let_stopped for dependent senders

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

A sender pipeline has three possible completion channels: it can deliver a value, signal an error, or signal cancellation (stopped). The `let_*` family gives you a way to intercept any of those channels and replace the outcome with a new sender. This is how you build async decision trees - logic that runs different async work depending on what happened upstream.

| Adaptor | Triggered by | Callback signature | Returns |
| --- | --- | --- | --- |
| `let_value(f)` | `set_value(vals...)` | `f(vals...)` | sender |
| `let_error(f)` | `set_error(err)` | `f(err)` | sender |
| `let_stopped(f)` | `set_stopped()` | `f()` | sender |

```cpp
Completion channels:

  sender -+- set_value  -> let_value(f)  -> f(vals...) -> sender
         +- set_error  -> let_error(f)  -> f(err)     -> sender
         +- set_stopped -> let_stopped(f) -> f()       -> sender
```

---

## Self-Assessment

### Q1: `let_value` - type-dependent chaining

The key thing `let_value` gives you that `then` cannot: the sender you return can be chosen at runtime based on the incoming value. Different branches, different async work, all expressed in one pipeline:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <string>
#include <variant>

// The power of let_value: the RETURNED SENDER TYPE
// can depend on the runtime value from the predecessor.

auto process_command(int cmd) {
    if (cmd == 1) {
        // Returns a sender<string>:
        return stdexec::just(std::string("processed"));
    } else {
        // Returns the SAME sender type:
        return stdexec::just(std::string("unknown command"));
    }
}

int main() {
    // let_value: next step DEPENDS on the result
    auto pipeline = stdexec::just(1)
        | stdexec::let_value([](int cmd) {
            std::cout << "Command: " << cmd << '\n';
            return process_command(cmd);  // returns a SENDER
        })
        | stdexec::then([](const std::string& result) {
            std::cout << "Result: " << result << '\n';
            return result;
        });

    auto [val] = stdexec::sync_wait(std::move(pipeline)).value();
    // Output:
    // Command: 1
    // Result: processed

    // Nested let_value for multi-step dependent chains:
    auto deep = stdexec::just(10)
        | stdexec::let_value([](int x) {
            return stdexec::just(x * 2)
                | stdexec::let_value([](int y) {
                    return stdexec::just(y + 5);  // nested!
                });
        });
    auto [r] = stdexec::sync_wait(std::move(deep)).value();
    std::cout << "Deep: " << r << '\n';  // 25
}
```

The nested `let_value` at the end is worth studying. Each inner `let_value` creates a sub-pipeline from scratch. This nesting is how you express complex multi-stage async logic without coroutines.

### Q2: `let_value` is monadic bind (flatMap)

If you've worked with functional programming, `let_value` is exactly monadic bind - the operation that lets you chain computations that each return a wrapped value. The reason this trips people up: `then` does a simple transform (`A -> B`), but `let_value` does a flatMap (`A -> Sender<B>`, and the framework unwraps the outer sender). Without that unwrapping step, you'd end up with a `Sender<Sender<B>>` which is useless.

```cpp
Monad parallel:

  Haskell:    m a >>= (\a -> m b)       // bind
  Rust:       future.and_then(|a| ...)
  JavaScript: promise.then(a => ...)
  P2300:      sender | let_value([](a) { return sender; })

Laws:

  1. Left identity:  just(x) | let_value(f)  ===  f(x)
  2. Right identity: s | let_value(just)      ===  s
  3. Associativity:  (s|let_value(f)) | let_value(g)

                     === s | let_value([](x){ return f(x)|let_value(g); })
```

Here's the practical version - what goes wrong with `then` and why `let_value` fixes it:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
// Why "monadic"?
// then(f):       Sender<A> -> (A -> B) -> Sender<B>           // functor map
// let_value(f):  Sender<A> -> (A -> Sender<B>) -> Sender<B>   // monad bind

// The difference: let_value FLATTENS the nested sender.
// Without let_value, then would give Sender<Sender<B>> which is unusable.

#include <stdexec/execution.hpp>
#include <iostream>

auto double_async(int x) {
    return stdexec::just(x * 2);  // returns a sender
}

int main() {
    // then would fail here (can't unwrap a sender-of-sender):
    // auto bad = just(21) | then(double_async);  // type: sender<sender<int>>!

    // let_value flattens:
    auto good = stdexec::just(21)
        | stdexec::let_value([](int x) {
            return double_async(x);  // sender<int>, not sender<sender<int>>
        });
    auto [val] = stdexec::sync_wait(std::move(good)).value();
    std::cout << val << '\n';  // 42
}
```

The commented-out `then(double_async)` line doesn't compile for this reason: `double_async` returns a sender, and `then` would try to wrap that sender in *another* sender, creating a type the pipeline doesn't know how to execute. `let_value` is the correct tool whenever your transformation naturally returns a sender.

### Q3: `let_error` for retry on failure

`let_error` fires only when the upstream signals an error. You can use it to build a retry combinator - each retry attempt is just another `let_error` layered on top of the previous one:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <atomic>

std::atomic<int> attempt_count{0};

auto unreliable_operation() {
    return stdexec::just()
        | stdexec::then([]() -> int {
            int n = ++attempt_count;
            if (n < 3) {
                throw std::runtime_error("Attempt " + std::to_string(n) + " failed");
            }
            return 42;  // success on 3rd try
        });
}

// Build a retry combinator using let_error:
auto with_retry(auto sender, int max_retries) {
    auto result = std::move(sender);
    for (int i = 0; i < max_retries; ++i) {
        result = std::move(result)
            | stdexec::let_error([](std::exception_ptr ep) {
                try { std::rethrow_exception(ep); }
                catch (const std::exception& e) {
                    std::cout << "Retrying after: " << e.what() << '\n';
                }
                return unreliable_operation();  // retry!
            });
    }
    return result;
}

int main() {
    auto pipeline = with_retry(unreliable_operation(), 3);

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Success: " << result << '\n';
    std::cout << "Total attempts: " << attempt_count.load() << '\n';
}
// Output:
// Retrying after: Attempt 1 failed
// Retrying after: Attempt 2 failed
// Success: 42
// Total attempts: 3
```

Once `unreliable_operation()` succeeds on attempt 3, its value flows through the remaining `let_error` nodes without triggering any of them - they only react to errors. This is the "pass-through" property of the `let_*` adaptors: each one only intercepts its own completion channel and lets the others through unchanged.

**`let_stopped` for cancellation recovery:**

```cpp
auto pipeline = some_cancellable_sender()
    | stdexec::let_stopped([]() {
        std::cout << "Operation was cancelled, using fallback...\n";
        return stdexec::just(default_value);  // recover from cancel
    });
```

`let_stopped` gives you the same pattern for the cancellation channel. When the upstream signals stopped (cancelled), you can substitute a fallback sender and continue the pipeline as if the cancellation never happened.

---

## Notes

- `let_value` keeps predecessor values alive for the returned sender's lifetime.
- `let_error` only fires on error; values pass through unchanged.
- `let_stopped` only fires on cancellation; values and errors pass through.
- All three return senders, enabling complex async decision trees.
- For simple transforms (non-sender return), prefer `then`/`upon_error`/`upon_stopped`.
