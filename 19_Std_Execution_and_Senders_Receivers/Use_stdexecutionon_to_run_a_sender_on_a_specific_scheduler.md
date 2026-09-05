# Use std::execution::on to run a sender on a specific scheduler

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

One of the most important questions in async programming is not *what* runs, but *where* it runs. The scheduler APIs in P2300 answer exactly that question. Here is a quick map of the four tools and what each one controls:

| API | Meaning |
| --- | --- |
| `starts_on(sched, sender)` | Run the sender on `sched` from the start |
| `continues_on(sched)` | Later work continues on `sched` |
| `on(sched, sender)` | Run on `sched`, then return to original context |
| `schedule(sched)` | Create an entry-point sender on `sched` |

The difference between `on`, `starts_on`, and `continues_on` trips people up, so here is a plain-language restatement. `starts_on` says "launch this work on that scheduler and keep running there." `continues_on` says "from this point in the pipeline, switch over to that scheduler." `on` is a round-trip - it switches in and then back out again. The diagram below makes this concrete:

```cpp
on(sched, sender):
  original_ctx -> [switch to sched] -> run sender -> [switch back] -> original_ctx

starts_on(sched, sender):
  [run on sched] -> stay on sched
```

---

## Self-Assessment

### Q1: `starts_on` (formerly `on`) to force execution on a pool thread

This example uses `starts_on` to pin work onto a thread pool. Notice how `work` is built first as a plain pipeline, and then `starts_on` wraps it to say "execute this pipeline on the pool." The second half of the example shows two pipelines running on the same pool and collected with `when_all`.

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Create work that runs on the pool:
    auto work = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Thread: " << std::this_thread::get_id()
                      << " value: " << x << '\n';
            return x * 2;
        });

    // starts_on forces this work onto the pool's threads:
    auto on_pool = stdexec::starts_on(sched, std::move(work));

    auto [result] = stdexec::sync_wait(std::move(on_pool)).value();
    std::cout << "Result: " << result << '\n';  // 84

    // Multiple senders on the same pool:
    auto a = stdexec::starts_on(sched,
        stdexec::just(10) | stdexec::then([](int x) { return x + 1; }));
    auto b = stdexec::starts_on(sched,
        stdexec::just(20) | stdexec::then([](int x) { return x + 2; }));

    auto [ra, rb] = stdexec::sync_wait(
        stdexec::when_all(std::move(a), std::move(b))
    ).value();
    std::cout << ra << ", " << rb << '\n';  // 11, 22
}
```

The thread ID printed in the first lambda will not match `main`'s thread - that is the whole point. The work genuinely runs on the pool.

### Q2: `starts_on` vs `continues_on` (transfer)

The table makes the differences concrete, but the key mental model is: `starts_on` sets the launch context, while `continues_on` changes context mid-flight. You can think of `continues_on` as a lane-change on a highway - you are already moving, and you just switch lanes at a specific point.

| Aspect | `starts_on(sched, sender)` | `continues_on(sched)` |
| --- | --- | --- |
| When | Before sender runs | Mid-pipeline |
| Effect | Sets initial execution context | Changes context for subsequent work |
| Stays? | Sender runs on sched | Continuation runs on sched |
| Returns to original? | No | No |
| Use case | "Run this work there" | "Continue the rest there" |

The code below shows both patterns side by side - `p1` uses `starts_on` so everything stays on pool A, while `p2` uses `continues_on` to cross from pool A to pool B halfway through the pipeline:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool_a(2);
    exec::static_thread_pool pool_b(2);
    auto sched_a = pool_a.get_scheduler();
    auto sched_b = pool_b.get_scheduler();

    // starts_on: everything runs on pool_a
    auto p1 = stdexec::starts_on(sched_a,
        stdexec::just()
        | stdexec::then([]() {
            std::cout << "[A] " << std::this_thread::get_id() << '\n';
            return 1;
        })
        | stdexec::then([](int x) {
            std::cout << "[A] " << std::this_thread::get_id() << '\n';
            return x + 1;
        })
    );

    // continues_on: switch mid-pipeline
    auto p2 = stdexec::schedule(sched_a)
        | stdexec::then([]() {
            std::cout << "[A] " << std::this_thread::get_id() << '\n';
            return 1;
        })
        | stdexec::continues_on(sched_b)  // switch!
        | stdexec::then([](int x) {
            std::cout << "[B] " << std::this_thread::get_id() << '\n';
            return x + 1;
        });

    stdexec::sync_wait(std::move(p1));
    stdexec::sync_wait(std::move(p2));
}
```

Watch the thread IDs in the output. In `p1` both lambdas print an `[A]` thread. In `p2` the second lambda prints a `[B]` thread - that is the context switch happening at `continues_on(sched_b)`.

### Q3: Full pipeline: pool -> I/O -> pool

Real systems often need to move work between different execution contexts - compute work goes to a CPU thread pool, I/O goes to a dedicated I/O pool, and then the result callback bounces back to the compute pool. This pipeline demonstrates that three-phase pattern:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <string>

int main() {
    exec::static_thread_pool compute(4);
    exec::static_thread_pool io(2);
    auto compute_sched = compute.get_scheduler();
    auto io_sched = io.get_scheduler();

    auto pipeline =
        // Step 1: Compute on thread pool
        stdexec::schedule(compute_sched)
        | stdexec::then([]() {
            std::cout << "[compute] Preparing data on "
                      << std::this_thread::get_id() << '\n';
            return std::string("processed_data");
        })
        // Step 2: Transfer to I/O pool for write
        | stdexec::continues_on(io_sched)
        | stdexec::then([](std::string data) {
            std::cout << "[io] Writing '" << data << "' on "
                      << std::this_thread::get_id() << '\n';
            return data.size();
        })
        // Step 3: Transfer back to compute pool for callback
        | stdexec::continues_on(compute_sched)
        | stdexec::then([](size_t bytes) {
            std::cout << "[compute] Wrote " << bytes << " bytes, callback on "
                      << std::this_thread::get_id() << '\n';
            return bytes;
        });

    auto [written] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Done: " << written << " bytes\n";
}
// Output:
// [compute] Preparing data on 14000001
// [io] Writing 'processed_data' on 14000050
// [compute] Wrote 14 bytes, callback on 14000002
// Done: 14 bytes
```

The execution flow makes it clear where each step lives - and crucially, each `continues_on` is the only thing you need to change if you want to redirect a phase to a different context later:

```cpp
Execution flow:

  Compute pool        I/O pool          Compute pool
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ Prepare     │-->│ Write       │-->│ Callback    │
│ data        │   │ to disk     │   │ (notify)    │
└─────────────┘   └─────────────┘   └─────────────┘
  schedule()    continues_on()   continues_on()
```

---

## Notes

- In P2300 R7, `on()` was split into `starts_on` (before) and `continues_on` (after).
- `on(sched, sender)` means `starts_on(sched, sender) | continues_on(original)`.
- `continues_on` was previously called `transfer`.
- `schedule(sched)` creates an empty sender on the scheduler (alternative to `starts_on`).
- Never nest `sync_wait` calls - use `let_value` for nested async work.
