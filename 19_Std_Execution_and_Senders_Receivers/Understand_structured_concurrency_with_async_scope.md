# Understand structured concurrency with async_scope

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

This topic focuses on practical patterns with `async_scope`: spawning child senders, preventing dangling work, and comparing the approach with detached threads. The core guarantee is simple but powerful - the parent always outlives its children. `async_scope` enforces this by counting in-flight tasks and giving you a way to wait until that count reaches zero.

The lifecycle looks like this:

```cpp
async_scope lifecycle:

  scope created
       │
  spawn(sender1)  spawn(sender2)  spawn(sender3)
       │               │               │
       └───────────────┼───────────────┘
                       │
               on_empty() -> waits until ALL done
                       │
               scope destroyed safely
```

Every sender you pass to `spawn` is tracked. The scope will not let you leave until all of them have finished - whether they completed with a value, an error, or a cancellation signal.

---

## Self-Assessment

### Q1: Spawn child senders and wait before scope destruction

The atomic counter here is just a way to verify that all five tasks actually ran. The important thing to watch is the output order - tasks complete in reverse order because each one sleeps for a duration proportional to `(5 - i)`, so child 4 sleeps the least and finishes first:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <thread>
#include <chrono>
#include <atomic>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    std::atomic<int> counter{0};

    // Spawn 5 child senders:
    for (int i = 0; i < 5; ++i) {
        scope.spawn(
            stdexec::schedule(sched)
            | stdexec::then([i, &counter]() {
                // Simulate varying work duration:
                std::this_thread::sleep_for(
                    std::chrono::milliseconds(10 * (5 - i)));
                counter.fetch_add(1, std::memory_order_relaxed);
                std::cout << "Child " << i << " done\n";
            })
        );
    }

    std::cout << "All children spawned, waiting...\n";

    // Wait for ALL children to complete:
    stdexec::sync_wait(scope.on_empty());

    // All children guaranteed complete:
    std::cout << "Counter: " << counter.load() << '\n';  // always 5
}
// Output:
// All children spawned, waiting...
// Child 4 done
// Child 3 done
// Child 2 done
// Child 1 done
// Child 0 done
// Counter: 5
```

After `on_empty()` returns, you can access `counter` without any synchronization beyond what the scope already provided. That guarantee is the whole point.

### Q2: Scope prevents dangling: work completes before scope exits

A `unique_ptr` is a good illustration here because its destructor is visible in the output. You can see that `~Resource(database)` appears only after both tasks have printed their messages - the scope's `on_empty()` acts as the sequencing fence that makes this deterministic:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <memory>

struct Resource {
    std::string name;
    ~Resource() { std::cout << "~Resource(" << name << ")\n"; }
};

void process(exec::static_thread_pool& pool) {
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    // Resource with a limited lifetime:
    auto resource = std::make_unique<Resource>(Resource{"database"});

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&resource]() {
            // Safe: scope guarantees this completes
            // before resource is destroyed
            std::cout << "Using: " << resource->name << '\n';
        })
    );

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&resource]() {
            std::cout << "Also using: " << resource->name << '\n';
        })
    );

    // on_empty() ensures all spawned work is complete:
    stdexec::sync_wait(scope.on_empty());

    // NOW resource can safely be destroyed (unique_ptr dtor runs)
}
// Output:
// Using: database
// Also using: database
// ~Resource(database)   <-- destroyed AFTER all tasks complete

int main() {
    exec::static_thread_pool pool(4);
    process(pool);
    std::cout << "Done\n";
}
```

If you removed the `on_empty()` call, the `unique_ptr` could be destroyed while a task is still inside `resource->name`, which is a use-after-free. The scope makes the safe version the only version you can accidentally write.

### Q3: `async_scope` vs detached `std::thread`

The first code block shows the unsafe pattern. Even though this particular example uses a sleep to make the bug more likely to manifest, the underlying problem is structural - the thread captures a reference to a local variable, and `detach` gives you no guarantee about ordering:

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>

// ===== DANGEROUS: detached threads =====
void bad_pattern() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    std::thread t([&data]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        // DANGER: data might be destroyed already!
        for (int x : data) std::cout << x << ' ';
    });
    t.detach();
    // data destroyed here -> thread has dangling reference
}

// ===== SAFE: async_scope =====
// void good_pattern(exec::static_thread_pool& pool) {
//     auto sched = pool.get_scheduler();
//     exec::async_scope scope;
//     std::vector<int> data = {1, 2, 3, 4, 5};
//
//     scope.spawn(stdexec::schedule(sched)
//         | stdexec::then([&data]() {
//             for (int x : data) std::cout << x << ' ';
//         }));
//
//     stdexec::sync_wait(scope.on_empty());
//     // data destroyed here -> AFTER task completes, SAFE
// }
```

Here is the side-by-side comparison of the key properties:

| Problem | `thread::detach()` | `async_scope` |
| --- | --- | --- |
| Dangling references | Yes - no lifetime tracking | No - scope joins first |
| Resource leaks | Thread runs after cleanup | Tasks complete before cleanup |
| Unhandled exceptions | `std::terminate` | Error channel propagation |
| Cancellation | Ad-hoc, manual | Built-in stop_token |
| Debugging | Hard (no ownership) | Clear parent-child relationship |

The rule-of-thumb summary for choosing between these options:

```cpp
Rule of thumb:
  // BAD: Never use thread::detach()
  // BAD: Never use raw new + callback (same problem)
  // GOOD: Use jthread (C++20) for simple cases
  // GOOD: Use async_scope for fire-and-forget in sender pipelines
  // GOOD: Use when_all for fan-out with results
```

---

## Notes

- `async_scope::spawn()` takes full ownership of the sender and manages the operation state internally - you hand it off and the scope handles the rest.
- `on_empty()` is idempotent and can be called multiple times safely, making it convenient to check progress at multiple points.
- `request_stop()` sends a cooperative cancellation signal to all spawned-but-not-yet-completed tasks via the stop token mechanism.
- In `stdexec`, the type is `exec::async_scope`; the C++26 standard aims to provide `std::execution::async_scope` once P3149 is accepted.
- Structured concurrency is a paradigm shift worth internalizing: every piece of async work has a clear owner, a clear lifetime, and a clear place where you wait for it. That clarity is what makes large async codebases maintainable.
