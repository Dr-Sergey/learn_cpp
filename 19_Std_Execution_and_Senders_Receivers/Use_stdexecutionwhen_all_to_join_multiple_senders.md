# Use std::execution::when_all to join multiple senders

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

`when_all` is the primary way to run multiple senders concurrently and collect all their results. If you have used `std::async` + `future::get` before, think of `when_all` as the composable, cancellation-aware, scheduler-controlled replacement for that pattern.

The key properties you need to remember:

| Property | Detail |
| --- | --- |
| Completes when | ALL child senders complete |
| Value result | `tuple<Val1, Val2, ...>` in declaration order |
| Error behavior | First error cancels remaining, propagates error |
| Stopped behavior | Any stopped cancels remaining, propagates stopped |

---

## Self-Assessment

### Q1: Join two independent computations with `when_all`

This is the bread-and-butter use case. Two tasks run concurrently on a thread pool, and `when_all` waits for both before returning the results. Structured bindings let you unpack the tuple immediately:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Two independent computations on the pool:
    auto compute_a = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "[A] thread " << std::this_thread::get_id() << '\n';
            return 100;
        }));

    auto compute_b = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "[B] thread " << std::this_thread::get_id() << '\n';
            return 200;
        }));

    // when_all runs both concurrently, collects results:
    auto joined = stdexec::when_all(
        std::move(compute_a),
        std::move(compute_b)
    );

    auto [a, b] = stdexec::sync_wait(std::move(joined)).value();
    std::cout << "A=" << a << ", B=" << b << '\n';       // 100, 200
    std::cout << "Sum=" << (a + b) << '\n';               // 300

    // Three senders:
    auto [x, y, z] = stdexec::sync_wait(
        stdexec::when_all(
            stdexec::just(1),
            stdexec::just(2),
            stdexec::just(3)
        )
    ).value();
    std::cout << x << ", " << y << ", " << z << '\n';  // 1, 2, 3
}
```

Notice that `[A]` and `[B]` will print on different thread IDs - they really do run in parallel. The result tuple is `(100, 200)` regardless of which task finished first.

### Q2: Error/cancellation propagation in `when_all`

When any child fails, `when_all` sends a stop signal to the remaining children and then propagates the error. Crucially, it still waits for all children to acknowledge the cancellation before completing - there are no dangling operations. The diagram after the code makes the priority ordering clear:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // One sender errors:
    auto ok = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() { return 42; }));

    auto fails = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() -> int {
            throw std::runtime_error("task failed");
        }));

    try {
        stdexec::sync_wait(
            stdexec::when_all(std::move(ok), std::move(fails))
        );
    } catch (const std::runtime_error& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }

    // Cancellation: if one sender is stopped, when_all stops
    auto stopped = stdexec::just_stopped();
    auto normal = stdexec::just(42);

    auto result = stdexec::sync_wait(
        stdexec::when_all(std::move(stopped), std::move(normal))
    );
    if (!result) {
        std::cout << "when_all stopped (nullopt)\n";
    }
}
// Output:
// Caught: task failed
// when_all stopped (nullopt)
```

The priority ordering that governs the final completion signal is: error beats stopped, which beats value. If you mix one error and one stopped, you get the error.

```cpp
Error propagation in when_all:

  sender A (ok)  --+-- when_all --> error (from B)
  sender B (err) --+        |
                            +-> cancels A (stop token)

Priority: error > stopped > value
```

### Q3: `when_all` vs `std::async` + `future::get`

It is worth seeing the two approaches side by side because the advantages of `when_all` are not just theoretical. The old `std::async` approach has no cancellation, no scheduler control, and composes poorly. `when_all` solves all of those:

```cpp
// OLD way: std::async + future::get
auto f1 = std::async(std::launch::async, compute_a);
auto f2 = std::async(std::launch::async, compute_b);
int a = f1.get();  // blocks
int b = f2.get();  // blocks
// Problems: no cancellation, no composition, no scheduler control

// NEW way: when_all
auto [a2, b2] = stdexec::sync_wait(
    stdexec::when_all(
        stdexec::starts_on(sched, stdexec::just() | stdexec::then(compute_a)),
        stdexec::starts_on(sched, stdexec::just() | stdexec::then(compute_b))
    )
).value();
```

The comparison table lays out every dimension:

| Feature | `std::async` + `get()` | `when_all` |
| --- | --- | --- |
| Composition | No | Yes (pipeline) |
| Scheduler control | No | Yes |
| Cancellation | No | Automatic |
| Error handling | Exception from get() | Error channel |
| Result order | Call-order dependent | Declaration order |
| Nested parallelism | Manual | `when_all` inside `let_value` |
| Lazy? | No (starts immediately) | Yes |
| Heap allocation | Future state | Operation state (stack possible) |

---

## Notes

- Results are always in **declaration order**, not completion order.
- `when_all()` with zero arguments completes immediately with `set_value()`.
- `when_all_with_variant` handles senders with different completion signatures.
- `when_all` provides structured concurrency: it never completes until ALL children finish.
- Combine with `split` for diamond dependency patterns.
