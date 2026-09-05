# Use std::execution::bulk for parallel loop execution over a sender

**Category:** Std Execution and Senders Receivers  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

`bulk` is the sender equivalent of a parallel for-loop. Instead of launching threads manually or reaching for `std::for_each` with a parallel policy, you describe the loop as a step in your sender pipeline and let the scheduler decide how to distribute the work across its thread pool.

| Property | Detail |
| --- | --- |
| Signature | `bulk(shape, f)` |
| Callback | `f(index, shared_state...)` |
| Concurrency | Up to `shape` invocations in parallel |
| Shared state | Passed through from predecessor sender |
| Ordering | No guaranteed order between indices |

The mental model is straightforward: `bulk(N, f)` says "call `f(0, ...)` through `f(N-1, ...)`, as concurrently as the scheduler wants." The scheduler maps those index calls onto its threads. After all `N` invocations finish, the pipeline continues:

```cpp
bulk(N, f) execution:

  predecessor sender
        |
        v
  +-----------------------+
  |  f(0, data)  | f(1, data)  |
  |  f(2, data)  | f(3, data)  |  (concurrent)
  |  ...         | f(N-1,data) |
  +-----------------------+
        |
        v
  data (passed through unchanged)
```

---

## Self-Assessment

### Q1: `bulk(n, f)` - execute `f(index, data)` for each index

Here's a concrete example - fill a vector with squares in parallel, then sum the results:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <vector>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    constexpr int N = 8;
    std::vector<int> data(N, 0);

    // bulk: execute f(index, data) for each index in [0, N)
    auto pipeline = stdexec::schedule(sched)
        | stdexec::then([&data]() -> std::vector<int>& {
            return data;
        })
        | stdexec::bulk(N, [](int index, std::vector<int>& vec) {
            // Each index runs potentially on a different thread:
            vec[index] = index * index;
            std::cout << "index " << index << " on thread "
                      << std::this_thread::get_id() << '\n';
        })
        | stdexec::then([](std::vector<int>& vec) {
            int sum = 0;
            for (int v : vec) sum += v;
            return sum;
        });

    auto [total] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Sum of squares: " << total << '\n';  // 0+1+4+9+16+25+36+49 = 140
}
// Output (order may vary):
// index 0 on thread 14000001
// index 3 on thread 14000003
// index 1 on thread 14000001
// ...
// Sum of squares: 140
```

The output order of the index lines is non-deterministic - that's the parallel execution showing through. But the final sum is always 140 because `bulk` waits for all indices to complete before the next `then` step runs.

### Q2: `bulk` maps onto a scheduler's thread pool

The scheduler decides how to assign indices to threads. A 4-thread pool with 8 items might process them in two waves, or via work-stealing, or any other strategy it sees fit:

```cpp
Scheduler thread pool mapping:

  Thread pool (4 threads):
  +----------+----------+----------+----------+
  | Thread 0 | Thread 1 | Thread 2 | Thread 3 |
  +----------+----------+----------+----------+
  | f(0)     | f(1)     | f(2)     | f(3)     |  <- wave 1
  | f(4)     | f(5)     | f(6)     | f(7)     |  <- wave 2
  +----------+----------+----------+----------+

  bulk(8, f) with 4-thread pool: scheduler decides how to map
  indices to threads. May use work-stealing, round-robin, etc.
```

Compare this to doing it manually with threads or `std::for_each`:

| Manual threads | `bulk` |
| --- | --- |
| Create/join threads yourself | Scheduler manages threads |
| Choose partitioning strategy | Scheduler decides |
| Manual synchronization | Data passed through sender |
| Cannot compose | Pipeline-composable |
| Thread affinity = your problem | Scheduler handles it |

The composability point is worth emphasizing. Because `bulk` is just another sender adaptor, you can chain multiple parallel passes back to back in the same pipeline:

```cpp
// bulk is composable in a pipeline:
auto pipeline = stdexec::schedule(sched)
    | stdexec::then([]() { return std::vector<float>(1000, 1.0f); })
    | stdexec::bulk(1000, [](int i, std::vector<float>& v) {
        v[i] *= 2.0f;  // parallel transform
    })
    | stdexec::bulk(1000, [](int i, std::vector<float>& v) {
        v[i] += 1.0f;  // another parallel pass
    })
    | stdexec::then([](std::vector<float>& v) {
        return v;  // all elements are 3.0f
    });
```

Two independent parallel passes, each complete before the next begins, all in one readable chain.

### Q3: `bulk` vs `std::for_each` with parallel policy

Let's compare `bulk` directly against the classic parallel STL approach to see why the extra machinery is worth it for async-heavy code:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <algorithm>
#include <execution>
#include <vector>
#include <iostream>

int main() {
    constexpr int N = 1000;
    std::vector<double> data(N);

    // Approach 1: std::for_each with parallel policy
    std::iota(data.begin(), data.end(), 0.0);
    std::for_each(std::execution::par, data.begin(), data.end(),
        [](double& x) { x = x * x; });
    // Simple but:
    // - No control over which thread pool
    // - No composition with async I/O
    // - No cancellation support
    // - Implementation-defined parallelism

    // Approach 2: bulk sender
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto sender_work = stdexec::schedule(sched)
        | stdexec::then([N]() { return std::vector<double>(N); })
        | stdexec::bulk(N, [](int i, std::vector<double>& v) {
            v[i] = static_cast<double>(i) * i;
        })
        | stdexec::then([](std::vector<double>& v) {
            std::cout << "v[42] = " << v[42] << '\n';  // 1764.0
            return v;
        });

    auto [result] = stdexec::sync_wait(std::move(sender_work)).value();
}
```

| Feature | `std::for_each(par, ...)` | `bulk(N, f)` |
| --- | --- | --- |
| Scheduler control | No (impl-defined) | Yes (explicit) |
| Composable | No | Yes (pipeline) |
| Cancellation | No | Yes (stop token) |
| Error propagation | Exception only | Error channel |
| Async I/O integration | No | Yes |
| Index-based | No (iterator) | Yes (0..N-1) |

The parallel STL approach is fine for isolated data transformation with no async dependencies. Once you need to mix parallel CPU work with async I/O, or thread pool control, or cancellation, `bulk` in a sender pipeline is the cleaner solution.

---

## Notes

- `bulk` passes the predecessor's values **by reference** to each invocation.
- The shared data must be thread-safe for concurrent access (no overlapping writes).
- `bulk` completes only after ALL indices have finished.
- If any invocation throws, the error propagates to the error channel.
- The shape can be any integral type (not just `int`).
- `bulk` is the primary way to express data parallelism in P2300.
