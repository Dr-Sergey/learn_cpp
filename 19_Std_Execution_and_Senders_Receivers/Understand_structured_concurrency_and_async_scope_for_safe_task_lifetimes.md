# Understand structured concurrency and async_scope for safe task lifetimes

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

Structured concurrency is the idea that a child operation must always finish before its parent scope does. It is the async equivalent of the rule you already know for local variables: a local variable lives until the end of its scope, so anything that refers to it must also stop before the scope exits. Structured concurrency enforces the same discipline for async work.

`async_scope` is the mechanism that enforces this. It keeps an internal count of all spawned tasks and lets you wait on a sender that completes only when that count drops to zero. The contrast with `thread::detach` is sharp:

```cpp
Unstructured (dangerous):        Structured (safe):

std::thread t(work);             async_scope scope;
t.detach();  // fire-and-forget   scope.spawn(work);
// t might outlive main!          scope.join();  // blocks until work done
// accessing destroyed locals!    // all spawned tasks guaranteed complete
```

With `detach` you are essentially saying "I don't care when this finishes" - and the runtime takes you at your word, even if the task is still holding a reference to a local variable that has already been destroyed. `async_scope` removes that gamble entirely.

---

## Self-Assessment

### Q1: Spawn fire-and-forget tasks with `async_scope`

Here you can see the basic lifecycle: spawn tasks inside the scope, then call `on_empty()` to get a sender you can wait on. The key line is `stdexec::sync_wait(scope.on_empty())` - nothing after that line runs until every spawned task has called one of `set_value`, `set_error`, or `set_stopped` on its receiver:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <thread>
#include <vector>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    exec::async_scope scope;
    std::vector<int> results(10, 0);

    // Spawn multiple fire-and-forget tasks:
    for (int i = 0; i < 10; ++i) {
        auto work = stdexec::schedule(sched)
            | stdexec::then([i, &results]() {
                results[i] = i * i;
                std::cout << "Task " << i << " on thread "
                          << std::this_thread::get_id() << '\n';
            });
        // spawn_future returns immediately:
        scope.spawn(std::move(work));
    }

    // CRITICAL: join blocks until ALL spawned tasks complete:
    stdexec::sync_wait(scope.on_empty());

    // Safe to access results — all tasks are done:
    for (int i = 0; i < 10; ++i)
        std::cout << "results[" << i << "] = " << results[i] << '\n';
}
// Output (order varies):
// Task 3 on thread 140001
// Task 0 on thread 140002
// ...
// results[0] = 0
// results[1] = 1
// results[9] = 81
```

The task order in the output varies because the thread pool runs tasks concurrently, but the final reads of `results` are safe because `on_empty()` acts as a full barrier - every write is complete before any read starts.

### Q2: `async_scope` prevents dangling async operations

The reason dangling references are such a common async bug is that the programmer has to remember to synchronize manually. Structured concurrency makes the synchronization a structural property of the code, not something that can be forgotten. In this example, `local_data` is a plain string on the stack. Both spawned tasks capture it by reference. The only thing that makes this safe is the `on_empty()` call before the function returns:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <string>

void process_request(exec::static_thread_pool& pool) {
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    // Local data that must not be accessed after function returns:
    std::string local_data = "important data";

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&local_data]() {
            // Access local_data safely — scope guarantees
            // this completes before function returns:
            std::cout << "Processing: " << local_data << '\n';
        })
    );

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&local_data]() {
            std::cout << "Also using: " << local_data << '\n';
        })
    );

    // scope's destructor or on_empty() ensures ALL spawned tasks complete
    // BEFORE local_data goes out of scope:
    stdexec::sync_wait(scope.on_empty());

    // Now local_data can safely be destroyed.
    // Without async_scope, tasks might still reference local_data!
}

int main() {
    exec::static_thread_pool pool(4);
    process_request(pool);
    std::cout << "All done\n";
}
```

Remove the `on_empty()` call and you have a classic dangling-reference bug. With it, the compiler cannot even give you a warning - the bug is invisible to static analysis. `async_scope` makes the correct pattern the easy pattern.

### Q3: `async_scope` vs `thread::detach` - why detach is unsafe

The table below summarizes the differences. The bottom line is that every advantage `detach` appears to offer (simplicity, fire-and-forget convenience) comes with a hidden cost that only shows up when things go wrong:

| Aspect | `std::thread::detach()` | `async_scope` |
| --- | --- | --- |
| Lifetime tracking | None | Tracks all spawned tasks |
| Join guarantee | No | Yes (`on_empty()`) |
| Access to locals | Dangling references! | Safe - scope joins first |
| Cancellation | Manual (error-prone) | Structured (stop tokens) |
| Exception safety | Exceptions in detached threads = `std::terminate` | Errors propagated via error channel |
| Resource cleanup | Hope for the best | Deterministic |

Here is the concrete code showing the difference:

```cpp
// DANGEROUS: thread::detach()
void bad_example() {
    std::string data = "hello";

    std::thread t([&data]() {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << data << '\n';  // data is DESTROYED!
    });
    t.detach();  // fire and forget

}  // data destroyed here, t still running -> UNDEFINED BEHAVIOR

// SAFE: async_scope
void good_example(exec::static_thread_pool& pool) {
    auto sched = pool.get_scheduler();
    exec::async_scope scope;
    std::string data = "hello";

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&data]() {
            std::cout << data << '\n';  // SAFE: scope joins first
        })
    );

    stdexec::sync_wait(scope.on_empty());
    // data destroyed AFTER task completes
}
```

The ASCII timeline below makes the problem with `detach` visual. The function returns and destroys its locals at the bottom of its frame, but the detached thread keeps running and trying to read memory that no longer belongs to it:

```cpp
Lifetime comparison:

detach():                      async_scope:
┌─ function ─────────────┐     ┌─ function ──────────────────┐
│ create data             │     │ create data                    │
│ thread.detach()         │     │ scope.spawn(work)              │
│ return (data destroyed) │     │ scope.on_empty() // wait       │
└────────────────────────┘     │ return (data destroyed safely) │
  thread still running! UB!     └───────────────────────────────┘
```

---

## Notes

- `async_scope::spawn()` is the only way to do fire-and-forget work safely inside the sender/receiver model - it takes ownership of the sender and manages the operation state for you.
- `on_empty()` returns a sender that completes when all spawned work has finished. You can wait on it as many times as you need - it is idempotent.
- `request_stop()` on the scope sends a cooperative cancellation signal to all spawned tasks. Tasks that check their stop token will complete with `set_stopped`; others will run to completion normally.
- `async_scope` is proposed in P3149 for C++26 standardization; the current implementation lives in the `stdexec` library.
- Never use `thread::detach()` in production code where any local data is involved - prefer `std::jthread` for simple one-shot tasks or `async_scope` for fire-and-forget work inside sender pipelines.
