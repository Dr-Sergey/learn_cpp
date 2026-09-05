# Use stdexec::let_value and let_error for monadic sender chaining

**Category:** Std Execution and Senders Receivers  
**Standard:** C++20  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

`let_value` and `let_error` are the adaptors you reach for when the next step in your async pipeline needs to launch another async operation - not just transform a value, but kick off new async work whose shape depends on the previous result. The distinction from `then` is critical: `then` maps a value to another value, while `let_value` maps a value to a whole new sender that gets connected and started.

| Adaptor | Input | Callback returns | Purpose |
| --- | --- | --- | --- |
| `then(f)` | value | value (not a sender) | Simple transform |
| `let_value(f)` | value | **sender** | Dependent async step |
| `let_error(f)` | error | **sender** | Error recovery with async work |
| `let_stopped(f)` | stopped | **sender** | Cancellation recovery |

The flow looks like this:

```cpp
then(f):       value -> f(value) -> new_value
let_value(f):  value -> f(value) -> sender -> (execute sender) -> result
```

If you have ever used `flatMap` in a functional language or `>>=` (bind) in Haskell, `let_value` is exactly that operation applied to async senders.

---

## Self-Assessment

### Q1: `let_value` for dependent sender chaining

The classic use case is an async request chain where each step needs the result of the previous step to decide what to fetch next. You cannot use `then` here because each function returns a sender, not a plain value:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <string>

// Simulate async operations that return senders:
auto fetch_user_id(const std::string& token) {
    return stdexec::just(42);  // user_id = 42
}

auto fetch_user_profile(int user_id) {
    return stdexec::just(std::string("Alice"));
}

auto fetch_user_posts(const std::string& username) {
    return stdexec::just(10);  // 10 posts
}

int main() {
    // let_value chains DEPENDENT senders:
    // Each step needs the RESULT of the previous step to decide what to do next.
    auto pipeline = stdexec::just(std::string("auth_token_xyz"))
        | stdexec::let_value([](const std::string& token) {
            // Returns a SENDER (not a value!):
            return fetch_user_id(token);
        })
        | stdexec::let_value([](int user_id) {
            std::cout << "User ID: " << user_id << '\n';
            return fetch_user_profile(user_id);
        })
        | stdexec::let_value([](const std::string& username) {
            std::cout << "Username: " << username << '\n';
            return fetch_user_posts(username);
        })
        | stdexec::then([](int post_count) {
            std::cout << "Posts: " << post_count << '\n';
            return post_count;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << result << '\n';
}
// Output:
// User ID: 42
// Username: Alice
// Posts: 10
// Final: 10
```

**Why not `then`?** `then` expects `f(value)` to return a **value**. But `fetch_user_id` returns a **sender**. If you tried to use `then`, you would get a sender-of-sender, which is not what you want. `let_value` unwraps the returned sender and connects it into the pipeline, so the final result is the value the returned sender produces.

### Q2: `let_error` for error recovery with a fallback sender

`let_error` is the same idea applied to the error channel. When an error arrives, your callback can return a new sender that represents a recovery operation - which could itself be an async operation like fetching from a backup server:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <string>

auto fetch_from_primary() {
    // Simulate primary server failure:
    return stdexec::just_error(
        std::make_exception_ptr(std::runtime_error("primary down")));
}

auto fetch_from_backup() {
    return stdexec::just(std::string("data from backup"));
}

int main() {
    // let_error: receives the error, returns a SENDER for recovery
    auto pipeline = fetch_from_primary()
        | stdexec::let_error([](std::exception_ptr ep) {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cout << "Primary failed: " << e.what()
                          << ", trying backup...\n";
            }
            // Return a fallback SENDER:
            return fetch_from_backup();
        })
        | stdexec::then([](const std::string& data) {
            std::cout << "Got: " << data << '\n';
            return data;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';
}
// Output:
// Primary failed: primary down, trying backup...
// Got: data from backup
// Result: data from backup
```

**`upon_error` vs `let_error`:**

- `upon_error(f)`: `f` returns a **value** (simple transformation - you know the fallback at compile time).
- `let_error(f)`: `f` returns a **sender** (you need to do async work to recover, like calling a backup service).

The pattern here - try primary, fall back to backup asynchronously if it fails - is a real pattern in distributed systems and `let_error` expresses it cleanly without blocking.

### Q3: `let_value` vs coroutine `co_await`

Both `let_value` and coroutines can express sequential async steps. Here they are side by side solving the same problem:

```cpp
// With coroutines (sequential, readable):
exec::task<int> coroutine_version() {
    auto token = co_await get_auth_token();
    auto user_id = co_await fetch_user_id(token);
    auto profile = co_await fetch_user_profile(user_id);
    co_return profile.post_count;
}

// With let_value (pipeline, composable):
auto sender_version() {
    return get_auth_token()
        | stdexec::let_value([](auto token) {
            return fetch_user_id(token);
        })
        | stdexec::let_value([](auto user_id) {
            return fetch_user_profile(user_id);
        })
        | stdexec::then([](auto profile) {
            return profile.post_count;
        });
}
```

| Aspect | `co_await` (Coroutines) | `let_value` (Senders) |
| --- | --- | --- |
| Readability | More natural for sequential | Pipeline-style |
| Parallel fan-out | Manual | `when_all` + `let_value` |
| Scheduler control | Implicit | Explicit (`starts_on`, `continues_on`) |
| Allocation | Coroutine frame (heap) | Operation state (stack possible) |
| Error handling | try/catch + co_await | `let_error` + error channel |
| Cancellation | Manual stop_token | Automatic via stopped channel |

The best approach for complex programs is often to combine both, using each where it shines. Here is a concrete example that does exactly that - the sequential parts are a coroutine, and the parallel fan-out part uses `when_all`:

```cpp
// Best of both: combine them!
exec::task<int> combined() {
    // Sequential part (coroutine):
    auto config = co_await load_config();

    // Fan-out part (senders):
    auto [a, b] = co_await stdexec::when_all(
        fetch_a(config.url_a),
        fetch_b(config.url_b)
    );

    // Sequential again:
    co_return merge(a, b);
}
```

The sequential sections read like ordinary code while the parallel fan-out uses `when_all` to run two requests concurrently. This hybrid style is idiomatic in codebases that use P2300.

---

## Notes

- `let_value` is the sender equivalent of monadic `flatMap` / `>>=` (bind) from functional programming - if that framing helps, the mental model is exact.
- The callback you pass to `let_value` MUST return a sender. Returning a non-sender is a compile error, which is the model catching your mistake at the earliest possible moment.
- `let_value` keeps the incoming value alive for the full lifetime of the sender your callback returns, so it is safe to capture the value by reference inside the returned sender.
- Use `then` for simple synchronous transforms and reserve `let_value` for cases where you genuinely need to return a sender - overusing `let_value` for simple transforms adds unnecessary complexity.
- `let_stopped` is the third member of this family, handling the cancellation channel and allowing cancellation recovery to itself be async work.
