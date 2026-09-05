# Survey coroutine libraries: cppcoro, libunifex, folly::coro, stdexec

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

The C++20 coroutine specification provides **low-level machinery** (compiler transforms, `coroutine_handle`, promise types) but no ready-to-use coroutine types like `task<T>` or `generator<T>`. This is by design - the standard defines the protocol, and libraries build the ergonomic types on top of it. Several libraries fill this gap, each with a different philosophy and set of trade-offs.

### Library Comparison

Here is a quick orientation before diving into each one. The key axes are maturity, design philosophy (sender/receiver vs coroutine-first), and whether structured concurrency is a first-class concept:

| Library | Maintainer | Status | Key Types | Structured Concurrency |
| --- | --- | --- | --- | --- |
| **cppcoro** | Lewis Baker | Archived (reference impl) | `task<T>`, `generator<T>`, `async_generator<T>` | Basic |
| **libunifex** | Facebook/Meta | Active (P2300 reference) | `task<T>`, schedulers, senders | Yes |
| **folly::coro** | Facebook/Meta | Production | `Task<T>`, `AsyncGenerator<T>`, `co_invoke` | Yes |
| **stdexec** | NVIDIA | Active (P2300 reference) | Senders/receivers, `sync_wait` | Yes |
| **std::generator** | Standard | C++23 | `std::generator<T>` | N/A |

### cppcoro

The original C++20 coroutine library by Lewis Baker. It's now archived, but it remains the most widely cited reference for learning the patterns, and many newer libraries follow its API conventions. If you want to understand how coroutine libraries are supposed to feel to use, start here:

<!-- compile: needs third-party library header `cppcoro/task.hpp` -->
```cpp
#include <cppcoro/task.hpp>
#include <cppcoro/sync_wait.hpp>
#include <cppcoro/when_all.hpp>

cppcoro::task<int> compute() {
    co_return 42;
}

cppcoro::task<std::string> fetch() {
    co_return "hello";
}

int main() {
    // sync_wait drives a coroutine from synchronous code
    auto [val, str] = cppcoro::sync_wait(
        cppcoro::when_all(compute(), fetch())
    );
    // val == 42, str == "hello"
}
```

`sync_wait` is the bridge between coroutine world and the rest of your program - it blocks the current thread until the coroutine completes and returns its result. **Key features:** `task<T>`, `shared_task<T>`, `generator<T>`, `async_generator<T>`, `async_mutex`, `when_all`, `sync_wait`.

### libunifex

Facebook's library that implements the P2300 sender/receiver model with coroutine integration. The interesting thing about libunifex is that it treats coroutines as one composition mechanism among several - you can mix coroutine-style code with sender/receiver pipelines, which makes it very flexible:

<!-- compile: needs third-party library header `unifex/task.hpp` -->
```cpp
#include <unifex/task.hpp>
#include <unifex/sync_wait.hpp>
#include <unifex/just.hpp>
#include <unifex/then.hpp>
#include <unifex/thread_unsafe_event_loop.hpp>

unifex::task<int> async_add(int a, int b) {
    co_return a + b;
}

int main() {
    unifex::thread_unsafe_event_loop loop;
    auto result = unifex::sync_wait(
        unifex::on(loop.get_scheduler(), async_add(3, 4))
    );
    // result == 7
}
```

Notice the `unifex::on(scheduler, work)` pattern - you explicitly choose which scheduler runs the coroutine. **Key features:** P2300 sender/receiver, schedulers, `let_value`, `when_all`, cancellation via stop tokens.

### folly::coro

Production-grade library used within Meta. If you need something battle-tested with a rich set of async primitives (retries, timeouts, scoped tasks, async generators), this is the answer. The API is coroutine-first and the ergonomics are polished from years of internal use:

<!-- compile: needs third-party library header `folly/experimental/coro/Task.h` -->
```cpp
#include <folly/experimental/coro/Task.h>
#include <folly/experimental/coro/BlockingWait.h>
#include <folly/experimental/coro/Collect.h>

folly::coro::Task<int> compute_async() {
    co_return 42;
}

folly::coro::Task<void> example() {
    auto result = co_await compute_async();
    // result == 42

    // Concurrent fan-out
    auto [a, b] = co_await folly::coro::collectAll(
        compute_async(), compute_async());
}

int main() {
    folly::coro::blockingWait(example());
}
```

**Key features:** `Task<T>`, `AsyncGenerator<T>`, `collectAll`, `collectAny`, timeouts, `co_invoke`, retry policies, `AsyncScope`.

### stdexec (P2300 Reference Implementation)

NVIDIA's reference implementation of `std::execution` (P2300). Where cppcoro and folly are coroutine-first, stdexec is sender/receiver-first, meaning you compose work as a pipeline of senders and only drop to coroutines when the imperative style helps. It also supports heterogeneous execution targets (CPU and GPU schedulers):

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto work = stdexec::starts_on(sched,
        stdexec::just(40)
        | stdexec::then([](int x) { return x + 2; })
    );

    auto [result] = stdexec::sync_wait(std::move(work)).value();
    // result == 42
}
```

### Choosing a Library

If you're not sure which to pick, here is the decision table. Library choice is a real architectural commitment - pick deliberately:

| Need | Recommendation |
| --- | --- |
| Learning / prototyping | cppcoro (simple API) |
| P2300 sender/receiver model | stdexec or libunifex |
| Production async framework | folly::coro |
| Standard-only (no deps) | `std::generator` (C++23) + hand-rolled `task<T>` |
| GPU / heterogeneous | stdexec (NVIDIA backing) |

---

## Self-Assessment

### Q1: Write a minimal coroutine using cppcoro-style `task<T>`

Here is a recursive coroutine computing Fibonacci numbers with cppcoro. Each recursive call becomes a `co_await`, and `sync_wait` drives the whole thing from `main`. This is a clean demonstration of how a coroutine-based `task<T>` naturally replaces a plain function call for async work:

<!-- compile: needs third-party library header `cppcoro/task.hpp` -->
```cpp
#include <cppcoro/task.hpp>
#include <cppcoro/sync_wait.hpp>
#include <iostream>

cppcoro::task<int> fibonacci(int n) {
    if (n <= 1) co_return n;
    auto a = co_await fibonacci(n - 1);
    auto b = co_await fibonacci(n - 2);
    co_return a + b;
}

int main() {
    int result = cppcoro::sync_wait(fibonacci(10));
    std::cout << "fib(10) = " << result << "\n";  // 55
}
```

### Q2: Show concurrent fan-out with folly::coro

`collectAll` is the workhorse for running a group of tasks concurrently and waiting for all of them. All three `fetch_price` calls are in flight at the same time; `collectAll` collects their results in order:

```cpp
folly::coro::Task<int> fetch_price(std::string symbol) {
    co_await folly::coro::sleep(std::chrono::milliseconds(100));
    co_return symbol.size() * 10;  // dummy
}

folly::coro::Task<void> portfolio() {
    auto [a, b, c] = co_await folly::coro::collectAll(
        fetch_price("AAPL"),
        fetch_price("GOOG"),
        fetch_price("MSFT")
    );
    std::cout << "Total: " << a + b + c << "\n";
}
```

### Q3: When would you choose stdexec over folly::coro

Choose **stdexec** when:

- You want alignment with the upcoming **C++ standard** (`std::execution`). stdexec is tracking that proposal closely, so code written against it today will be easier to migrate when the standard ships.
- You need **heterogeneous execution** (CPU + GPU) - stdexec supports GPU schedulers, which is unique among these libraries.
- You prefer the **sender/receiver** composition model over coroutine-first design - senders compose more naturally with functional-style pipelines.
- You want to avoid the folly dependency ecosystem, which can be heavyweight.

Choose **folly::coro** when:

- You need a **battle-tested production** library that has been run at scale inside Meta for years.
- You want rich async primitives (retries, timeouts, scoped tasks) out of the box without having to build them yourself.
- Your team is already using the folly ecosystem and the integration cost is low.

---

## Notes

- C++23 added `std::generator<T>` to the standard - the first standard coroutine type. If you only need lazy generation, you no longer need a third-party library.
- P2300 (`std::execution`) is targeting C++26 and will provide standard sender/receiver types, making libunifex and stdexec effectively reference implementations for what's coming to the language.
- Most libraries define their own `task<T>` - they are not interchangeable without adapters. This means switching libraries later is a significant refactor, so pick early and stick with it.
- Coroutine library choice is a major architectural decision. The API shapes how you structure your entire async codebase.
