# Use stdexec::on and stdexec::starts_on for scheduler affinity

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

One of the most important things the senders/receivers model gives you is explicit control over *where* work runs. You can say "this part of the pipeline belongs on the thread pool" and "this other part belongs on the I/O pool," and the framework enforces that for you. The three tools for doing this are:

| API | Purpose | Direction |
| --- | --- | --- |
| `starts_on(sched, sender)` | Run entire sender on scheduler | Sets initial context |
| `continues_on(sched)` | Transfer subsequent work to scheduler | Mid-pipeline switch |
| `schedule(sched)` | Create entry-point sender on scheduler | Pipeline start |

Think of `starts_on` as "launch this whole thing on scheduler X" and `continues_on` as "from this point forward, keep going on scheduler Y." Together they give you clean, composable control over which execution context handles each stage of work.

---

## Self-Assessment

### Q1: `starts_on` - ensure a pipeline begins on a specific scheduler

Without `starts_on`, `sync_wait` uses its own internal `run_loop` to drive the pipeline - usually the calling thread. With `starts_on`, you hand the work off to a real scheduler before it even begins. Here's what that looks like:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // A plain sender (runs on whatever context drives it):
    auto computation = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Computing on thread: "
                      << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    // starts_on: force this sender to run on the pool:
    auto on_pool = stdexec::starts_on(sched, std::move(computation));

    auto [result] = stdexec::sync_wait(std::move(on_pool)).value();
    std::cout << "Result: " << result << '\n';  // 84

    // Without starts_on, sync_wait runs it on its internal run_loop.
    // With starts_on, it runs on the thread pool.
}
```

Notice that the thread ID printed inside the lambda will be one of the pool's threads, not the main thread. That's the whole point - `starts_on` redirects execution before any callback runs.

### Q2: `continues_on` - move continuations to a different scheduler

Once a pipeline is running, you may want to hand off to a different context part-way through. For example, heavy CPU work belongs on a compute pool, but writing results to disk or a socket belongs on an I/O pool. `continues_on` inserts that handoff directly in the pipeline chain:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool compute_pool(4);
    exec::static_thread_pool io_pool(1);
    auto compute = compute_pool.get_scheduler();
    auto io = io_pool.get_scheduler();

    auto pipeline =
        stdexec::schedule(compute)
        | stdexec::then([]() -> int {
            std::cout << "[compute] " << std::this_thread::get_id() << '\n';
            return 42;  // heavy computation
        })
        | stdexec::continues_on(io)   // switch to I/O pool
        | stdexec::then([](int x) -> int {
            std::cout << "[io]      " << std::this_thread::get_id() << '\n';
            // write to disk / send over network
            return x;
        })
        | stdexec::continues_on(compute)  // switch back
        | stdexec::then([](int x) {
            std::cout << "[compute] " << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 84
}
// Thread IDs show the context switches between pools
```

The thread IDs in the output will change at each `continues_on` call. That's the scheduler switch happening in real time. Each `then` after a `continues_on` runs on the new scheduler.

### Q3: Full pipeline: compute -> I/O -> compute

Here's a more realistic example - an image processing pipeline that deliberately assigns each phase to the right kind of resource. Reading and writing are I/O-bound, so they go on the I/O pool. The pixel transformation is CPU-bound, so it goes on the compute pool:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <string>
#include <vector>

int main() {
    exec::static_thread_pool compute_pool(4);
    exec::static_thread_pool io_pool(2);
    auto compute = compute_pool.get_scheduler();
    auto io = io_pool.get_scheduler();

    // Realistic example: image processing pipeline
    auto pipeline =
        // Phase 1: Load image data (I/O pool)
        stdexec::starts_on(io,
            stdexec::just(std::string("/photos/image.raw"))
            | stdexec::then([](const std::string& path) {
                std::cout << "[IO] Loading " << path
                          << " on " << std::this_thread::get_id() << '\n';
                return std::vector<uint8_t>(1024, 128);  // simulated image data
            })
        )
        // Phase 2: Process image (compute pool)
        | stdexec::continues_on(compute)
        | stdexec::then([](std::vector<uint8_t> data) {
            std::cout << "[Compute] Processing "
                      << data.size() << " bytes on "
                      << std::this_thread::get_id() << '\n';
            for (auto& pixel : data) pixel = 255 - pixel;  // invert
            return data;
        })
        // Phase 3: Save result (I/O pool)
        | stdexec::continues_on(io)
        | stdexec::then([](std::vector<uint8_t> data) {
            std::cout << "[IO] Saving " << data.size()
                      << " bytes on " << std::this_thread::get_id() << '\n';
            return data.size();
        });

    auto [bytes] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Saved " << bytes << " bytes\n";
}
// Output:
// [IO] Loading /photos/image.raw on 14000050
// [Compute] Processing 1024 bytes on 14000001
// [IO] Saving 1024 bytes on 14000051
// Saved 1024 bytes
```

The diagram below shows the scheduler transitions at a glance:

```cpp
Pipeline scheduler flow:

  IO pool          Compute pool        IO pool
+----------+    +--------------+    +----------+
| Load     |-->| Process      |-->| Save     |
| image    |    | (invert)     |    | result   |
+----------+    +--------------+    +----------+
  starts_on    continues_on       continues_on
```

Each phase runs on exactly the resource it needs. The pipeline expresses the entire scheduling policy in one readable chain of operators.

---

## Notes

- `starts_on(sched, sender)` sets the initial context; `continues_on(sched)` changes it mid-pipeline.
- `continues_on` was called `transfer` in earlier P2300 drafts.
- `on(sched, sender)` is a shorthand for `starts_on(sched, sender) | continues_on(original_scheduler)`.
- Use I/O schedulers for file/network operations and compute schedulers for CPU-heavy work.
- Scheduler affinity is a zero-cost abstraction - the pipeline compiles to direct task submissions.
