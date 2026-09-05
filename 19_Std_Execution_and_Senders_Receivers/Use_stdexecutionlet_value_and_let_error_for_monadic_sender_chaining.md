# Use std::execution::let_value and let_error for monadic sender chaining

**Category:** Std Execution and Senders Receivers  
**Standard:** C++20  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

`then` is great when you know the transformation at the time you write the pipeline. But sometimes the *next step* of the pipeline depends on the *result* of the previous step - not just the value passed through, but which sender you use next. That's what `let_value` and `let_error` are for. They enable **monadic** sender chaining: the callback you provide doesn't return a value, it returns a whole new sender.

```cpp
then(f):       value --> f(value) --> direct_result
let_value(f):  value --> f(value) --> sender --> (execute) --> result
let_error(f):  error --> f(error) --> sender --> (execute) --> recovery
```

The callback returns a **sender**, not a value. This allows dynamic async decisions at each step - you can choose completely different pipelines based on what arrived, including pipelines that run on different schedulers or do further async I/O.

---

## Self-Assessment

### Q1: `let_value` - create a new sender based on the previous result

Here's a realistic example: a two-step lookup where each step depends on the result of the previous one. Each lookup function returns a sender - and that's exactly what `let_value` expects:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <string>

// Simulate a database lookup that returns a sender:
auto lookup_user(int user_id) {
    // In reality, this would do async I/O:
    return stdexec::just(std::string("User_" + std::to_string(user_id)));
}

auto lookup_permissions(const std::string& username) {
    return stdexec::just(username == "User_42" ? 0xFF : 0x00);
}

int main() {
    // let_value: each step depends on the previous result
    // and returns a NEW SENDER (not just a value)
    auto pipeline = stdexec::just(42)
        | stdexec::let_value([](int user_id) {
            std::cout << "Looking up user " << user_id << '\n';
            return lookup_user(user_id);  // returns a sender!
        })
        | stdexec::let_value([](const std::string& username) {
            std::cout << "Looking up permissions for " << username << '\n';
            return lookup_permissions(username);  // returns a sender!
        })
        | stdexec::then([](int perms) {
            std::cout << "Permissions: 0x"
                      << std::hex << perms << '\n';
            return perms;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
}
// Output:
// Looking up user 42
// Looking up permissions for User_42
// Permissions: 0xff
```

Notice how the username from the first `let_value` is passed directly into the second `let_value` callback. The framework keeps it alive for the entire duration of the inner sender's execution. This is one of the things `let_value` manages that `then` simply cannot: the predecessor's value stays in scope until the returned sender completes.

### Q2: `let_error` - recover from an error with a fallback sender

`let_error` is the error-channel counterpart. When the upstream sender signals an error, `let_error` gives you a chance to replace it with a recovery sender. This is the pattern for fallback logic and retry strategies:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <string>

auto fetch_primary() {
    return stdexec::just_error(
        std::make_exception_ptr(std::runtime_error("Connection refused")));
}

auto fetch_cached() {
    return stdexec::just(std::string("cached_data"));
}

int main() {
    // let_error: on failure, substitute a recovery sender
    auto pipeline = fetch_primary()
        | stdexec::let_error([](std::exception_ptr ep) {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cout << "Primary failed: " << e.what() << '\n';
            }
            return fetch_cached();  // returns a recovery SENDER
        })
        | stdexec::then([](const std::string& data) {
            std::cout << "Got: " << data << '\n';
            return data;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    // Output:
    // Primary failed: Connection refused
    // Got: cached_data

    // Chained retries:
    auto with_retries = fetch_primary()
        | stdexec::let_error([](auto) { return fetch_primary(); })  // retry 1
        | stdexec::let_error([](auto) { return fetch_primary(); })  // retry 2
        | stdexec::let_error([](auto) { return fetch_cached(); });  // fallback

    stdexec::sync_wait(std::move(with_retries));
}
```

The chained retry pattern at the bottom is particularly useful. Each `let_error` in the chain only fires if the upstream produced an error - if a retry succeeds, its value flows through the remaining `let_error` nodes untouched (since those only react to errors, not values).

### Q3: `let_value`/`let_error` vs coroutine `co_await`

If you've used coroutines, you'll notice that `let_value` chains read similarly to a sequence of `co_await` calls. Both express "run this async step, then when it finishes, run the next one." The tradeoffs are real though:

```cpp
// Coroutine approach (if using exec::task):
exec::task<std::string> coro_pipeline(int user_id) {
    auto username = co_await lookup_user(user_id);
    auto perms    = co_await lookup_permissions(username);
    co_return std::to_string(perms);
}

// Sender approach:
auto sender_pipeline(int user_id) {
    return stdexec::just(user_id)
        | stdexec::let_value([](int id) {
            return lookup_user(id);
        })
        | stdexec::let_value([](const std::string& name) {
            return lookup_permissions(name);
        })
        | stdexec::then([](int perms) {
            return std::to_string(perms);
        });
}
```

| Aspect | `co_await` (Coroutines) | `let_value` (Senders) |
| --- | --- | --- |
| Syntax | Sequential, natural | Pipeline composition |
| Type deduction | At suspend points | At compile time |
| Error handling | try/catch | `let_error` / error channel |
| Parallel work | Awkward | `when_all` composable |
| Cancellation | Manual | Automatic via stopped |
| Heap allocation | Coroutine frame | Stack-possible (HALO optimization) |
| Scheduler control | Implicit | Explicit |

**When to use which:**

- **Coroutines**: Long sequential chains, readability priority
- **let_value**: Dynamic sender creation, scheduler manipulation, parallel composition
- **Both together**: Use coroutines for the sequential backbone, senders for fan-out

---

## Notes

- `let_value` keeps the predecessor's result alive for the lifetime of the returned sender.
- `let_value` is the sender equivalent of monadic `bind` / `flatMap` / `>>=`.
- `let_error` only fires on the error channel; value channel passes through.
- `let_stopped` handles the cancellation channel similarly.
- Unlike `then`, the callback MUST return a sender, not a plain value.
