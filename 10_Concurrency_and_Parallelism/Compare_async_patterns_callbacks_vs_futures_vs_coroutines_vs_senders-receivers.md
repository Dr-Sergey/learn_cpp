# Compare async patterns: callbacks vs futures vs coroutines vs senders/receivers

**Category:** Concurrency and Parallelism  
**Standard:** C++20, C++26  
**Reference:** <https://wg21.link/P2300>  

---

## Topic Overview

C++ has evolved through multiple async programming models over the years, and each generation fixed real problems with the previous one - while introducing its own tradeoffs. Understanding the landscape helps you choose the right tool and also helps you read code that uses any of these patterns.

### Comparison Matrix

Here is the quick comparison. If the table feels like a lot, the short version is: callbacks are low-overhead but painful to compose; futures are simple but limited; coroutines give you readable sequential code; senders give you the most power and safety at the cost of a steeper learning curve.

| Pattern | Allocation | Composability | Cancellation | Error Handling |
| --- | --- | --- | --- | --- |
| Callbacks | None | Poor (nesting) | Manual | Manual |
| std::future | Shared state (heap) | Poor (no chaining) | None | exception_ptr |
| Coroutines | Frame (heap, elidable) | Good (co_await) | Via token | co_await + try/catch |
| Senders/Receivers | Customizable | Excellent (pipe) | Built-in | set_error channel |

### Callbacks (Oldest)

Callbacks are the oldest approach. They are zero-overhead and universally portable, but they do not compose well. Once you have more than two or three levels of nested callbacks, the code becomes very hard to follow and even harder to handle errors in consistently.

```cpp
void read_file(const char* path,
               std::function<void(std::string)> on_success,
               std::function<void(int)> on_error) {
    // Pros: zero overhead, simple
    // Cons: callback hell, hard to compose, error handling scattered
}
```

### std::future (C++11)

`std::future` made async results feel like values you can pass around and wait on. The syntax is simple. The problem is that it is eager (work starts immediately), it always allocates a shared state on the heap, and there is no way to cancel or chain operations without blocking.

```cpp
#include <future>
auto f = std::async(std::launch::async, []{ return compute(); });
int result = f.get();  // Blocks until ready
// Pros: simple syntax
// Cons: no chaining, always allocates shared state, no cancellation
```

### Coroutines (C++20)

Coroutines solve the composability and readability problems. Sequential async logic now reads like sequential synchronous code, and `co_await` chains operations without nesting. The coroutine frame is heap-allocated once (and compilers can often elide the allocation), and error handling works naturally with try/catch.

```cpp
Task<int> compute_async() {
    auto data = co_await read_file_async("input.txt");
    auto result = co_await process_async(data);
    co_return result;
}
// Pros: natural sequential syntax, composable
// Cons: heap allocation for frame, tooling immature
```

### Senders/Receivers (C++26)

Senders/receivers are the most powerful and most principled model. The work is lazy - nothing starts until you explicitly connect and start the sender graph. Cancellation is a first-class concept. The execution context (thread pool, inline, etc.) is part of the type. Composition via the pipe operator builds up a complete description of the async work before any of it runs.

```cpp
auto work = ex::just("input.txt")
    | ex::then(read_file)
    | ex::then(process)
    | ex::upon_error([](auto err) { log(err); return default_value(); });

ex::sync_wait(std::move(work));
// Pros: lazy, composable, cancellation, custom execution context
// Cons: complex mental model, verbose
```

---

## Self-Assessment

### Q1: When to use each pattern

- Callbacks: C interop, simple one-shot async operations where you control the full call stack and do not need composability.
- Futures: Simple "fire and forget" parallel computation where you just need to wait for one result and do not need to chain or cancel.
- Coroutines: Complex async control flow with branching, loops, and sequential dependencies; also the right choice for generators and async I/O in application code.
- Senders: Library-level async composition, custom schedulers, structured concurrency, and anywhere cancellation and error propagation need to be first-class.

### Q2: Why are senders better than futures for composition

Senders are lazy - they do not start until connected to a receiver via `connect()`. This means you can build up a pipeline description, attach a scheduler, add error handling, and only start execution when you are ready. Futures are eager - they start immediately when created, so by the time you have a `std::future<T>` in hand, the work is already running with no way to redirect it or cancel it. Senders are also type-safe: the scheduler becomes part of the sender's type, so the compiler can verify that operations are connected to compatible schedulers. And senders carry stop tokens natively, so cancellation propagates through the entire pipeline without any extra plumbing.

### Q3: Can coroutines and senders work together

Yes, and that combination is arguably the intended end state. You can `co_await` a sender, and senders can wrap coroutines. P2300 defines `as_awaitable` to adapt a sender into an awaiter that coroutines understand. The idiomatic pattern is to use senders to build the async infrastructure (schedulers, pipelines, retry logic) and coroutines to consume that infrastructure with readable, sequential-looking business logic.

---

## Notes

- The industry is converging on coroutines for application code combined with senders for library plumbing. That combination gets you both readability and structured safety.
- `std::future` is widely regarded as a design mistake - the eagerness, the mandatory heap allocation, and the lack of chaining make it unsuitable for serious async code. Avoid it in new code.
- Senders/receivers (P2300) are targeting C++26 as `std::execution`. The stdexec (NVIDIA) and libunifex (Meta) libraries let you use them today.
- Each pattern can be layered: senders can be built internally using coroutines, which may themselves use callbacks at the lowest level. Understanding the full stack helps you debug problems at any layer.
