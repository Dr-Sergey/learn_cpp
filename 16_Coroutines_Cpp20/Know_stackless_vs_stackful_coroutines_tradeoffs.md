# Know Stackless vs Stackful Coroutines Tradeoffs

**Category:** Coroutines Cpp20  
**Standard:** C++20 (stackless)  
**Reference:** [cppreference - Coroutines](https://en.cppreference.com/w/cpp/language/coroutines), [Boost.Context](https://www.boost.org/doc/libs/release/libs/context/doc/html/index.html)  

---

## Topic Overview

C++20 standardised **stackless coroutines**: each coroutine gets a heap-allocated frame that holds only its own local variables and suspension metadata. The coroutine **cannot suspend from a nested call** - only the top-level coroutine function can contain `co_await`/`co_yield`/`co_return`. This design gives the compiler maximum optimisation freedom (inlining, HALO) at the cost of requiring every function in a suspension chain to be a coroutine.

**Stackful coroutines** (Boost.Context, Boost.Fiber, OS `CreateFiber`/`ucontext_t`) allocate an entire **alternate stack** (typically 64 KiB - 1 MiB). Any function in the call chain can suspend - the entire stack is preserved. This is transparent to the callee, which needs no special annotations, but the stack memory cost is high and the compiler cannot optimise across the context switch boundary.

Here is the full comparison at a glance. If the table feels like a lot, the practical summary is: stackless wins on memory and speed when you control the whole call chain; stackful wins when you need to suspend from inside code you don't control:

| Property | Stackless (C++20) | Stackful (Boost.Context / fibers) |
| --- | --- | --- |
| Frame size | Exact (compiler-computed, often 64-512 B) | Full stack (64 KiB - 1 MiB default) |
| Suspend from nested call | No - only coroutine body | Yes - any depth |
| Compiler optimisation | Full (inlining, HALO, register alloc) | Limited (opaque context switch) |
| Allocation | Per-coroutine frame (HALO can elide) | Pre-allocated stack per fiber |
| Context switch cost | ~ns (pointer swap + resume) | ~10-100 ns (save/restore registers, stack pointer) |
| Standard support | C++20 `<coroutine>` | No standard; Boost, POSIX `ucontext`, Win32 fibers |
| Migration between threads | Easy (handle is just a pointer) | Complex (stack may have TLS references) |
| Deep recursion while suspended | Must make every level a coroutine | Works transparently |

```cpp
Stackless (C++20):                      Stackful (Boost.Context):

  main stack                              main stack        fiber stack
  +----------+                            +----------+     +----------+
  | caller() |                            | caller() |     | deep_fn()|
  |          |                            |          |     | helper() |
  | resume()-+---> coroutine frame        | resume()-+---->| coro()   |
  |          |    (heap, ~200 B)          |          |     | ...      |
  |          |<-- suspend                 |          |<----| suspend  |
  +----------+                            +----------+     +----------+
                                                          (64 KiB)
```

The choice between models depends on the **suspension pattern**: if you control the entire call chain and can make every level a coroutine, stackless wins on memory and performance. If you need to suspend from inside library code, callbacks, or deep recursion you don't control, stackful is the pragmatic choice.

---

## Self-Assessment

### Q1: Demonstrate that C++20 stackless coroutines cannot suspend from a nested (non-coroutine) function, and show the workaround

This is the most important limitation to internalise about stackless coroutines. The restriction is not a library decision - it is baked into the C++ language. `co_await` is a compile-time transformation, and it can only appear inside a function that the compiler has transformed into a coroutine. If you try to put `co_await` in a regular function, the compiler rejects it outright.

The workaround is to make the helper a coroutine too. That means every function in the suspension chain needs to be a coroutine and return an awaitable type - the so-called "viral `co_await`" problem:

```cpp
#include <coroutine>
#include <cstdio>
#include <exception>

struct Task {
    struct promise_type {
        Task get_return_object() { return {}; }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };
};

// An awaitable
struct MyAwaitable {
    bool await_ready() noexcept { return false; }
    void await_suspend(std::coroutine_handle<> h) noexcept { h.resume(); }
    int await_resume() noexcept { return 42; }
};

// --- THIS DOES NOT COMPILE ---
// Regular function cannot contain co_await
// void helper() {
//     int x = co_await MyAwaitable{};  // ERROR: not a coroutine
// }
//
// Task broken_coro() {
//     helper();  // intended to suspend inside helper -- impossible
//     co_return;
// }

// --- WORKAROUND 1: Make helper a coroutine too ---
Task helper_coro() {
    int x = co_await MyAwaitable{};
    std::printf("helper got: %d\n", x);
    co_return;
}

Task fixed_coro() {
    co_await helper_coro();  // chain of coroutines
    co_return;
}

// --- WORKAROUND 2: Pass a callback / awaitable up ---
// Transform the nested logic so the suspension point
// is always at the coroutine level

template <typename F>
Task callback_pattern(F async_op) {
    int x = co_await async_op();  // suspension at top-level coroutine
    std::printf("callback pattern got: %d\n", x);
    co_return;
}

int main() {
    fixed_coro();
    callback_pattern([] { return MyAwaitable{}; });
}
```

**Key insight:** In stackless coroutines, `co_await` is a syntactic construct that the compiler rewrites. It can only appear in a function declared as a coroutine. This is a **fundamental limitation**, not a library restriction.

---

### Q2: Implement a minimal stackful coroutine / fiber using Boost.Context and demonstrate suspension from a deeply nested non-coroutine call

This is the example that makes the power of stackful coroutines concrete. Notice that `level3` is a perfectly ordinary function - no coroutine keywords, no special return type - yet it suspends the entire fiber from three call levels deep. With C++20 stackless coroutines, this is structurally impossible without rewriting all three helper functions:

<!-- compile: needs third-party library header `boost/context/fiber.hpp` -->
```cpp
// Requires: Boost.Context (link with -lboost_context)
#include <boost/context/fiber.hpp>
#include <cstdio>

namespace ctx = boost::context;

// Deep call chain -- none of these are "coroutines" in the C++ sense
int level3(ctx::fiber& main_fiber) {
    std::printf("  level3: about to suspend\n");
    main_fiber = std::move(main_fiber).resume();  // SUSPEND from depth 3!
    std::printf("  level3: resumed\n");
    return 99;
}

int level2(ctx::fiber& main_fiber) {
    std::printf(" level2: calling level3\n");
    return level3(main_fiber);
}

int level1(ctx::fiber& main_fiber) {
    std::printf("level1: calling level2\n");
    int result = level2(main_fiber);
    std::printf("level1: level3 returned %d\n", result);
    return result;
}

int main() {
    ctx::fiber fiber([](ctx::fiber&& sink) -> ctx::fiber {
        std::printf("fiber: starting\n");
        int r = level1(sink);  // deep call chain suspends in level3
        std::printf("fiber: finished with %d\n", r);
        return std::move(sink);
    });

    std::printf("main: created fiber, resuming\n");
    fiber = std::move(fiber).resume();  // run until level3 suspends

    std::printf("main: fiber suspended, doing other work...\n");

    fiber = std::move(fiber).resume();  // resume level3
    std::printf("main: fiber completed\n");
}

// Output:
// main: created fiber, resuming
// fiber: starting
// level1: calling level2
//  level2: calling level3
//   level3: about to suspend
// main: fiber suspended, doing other work...
//   level3: resumed
// level1: level3 returned 99
// fiber: finished with 99
// main: fiber completed
```

**This is impossible with C++20 stackless coroutines** - `level3` would need to be a coroutine, which means `level2` and `level1` would also need changes (to `co_await` the result). Stackful fibers preserve the entire call stack, so any depth can suspend transparently. The reason this is possible is that Boost.Context literally saves and restores the CPU stack pointer and all callee-saved registers on every switch - the whole call stack is intact when it resumes.

---

### Q3: Quantify the memory and performance tradeoffs. When should you choose each model

This conceptual comparison table and benchmark skeleton give you the numbers to reason about. The memory difference is the most dramatic part - 10,000 stackless coroutines fit in 2 MB; the equivalent in fibers needs 640 MB:

```cpp
// This is a conceptual comparison -- compile-time analysis, not runnable as-is.

/*
+---------------------------------------------------------------------+
|                    MEMORY COMPARISON (10,000 concurrent coroutines) |
+------------------+------------------+-------------------------------+
| Metric           | Stackless (C++20)| Stackful (fibers)             |
+------------------+------------------+-------------------------------+
| Per-coroutine    | ~200 bytes       | ~64 KiB (default stack)       |
| 10K coroutines   | ~2 MB            | ~640 MB                       |
| 100K coroutines  | ~20 MB           | ~6.4 GB (!) or use guard pg)  |
| HALO-eligible    | 0 bytes (stack)  | N/A                           |
| Stack overflow   | Impossible       | Possible if stack too small   |
+------------------+------------------+-------------------------------+
|                                                                     |
|                  CONTEXT SWITCH OVERHEAD                             |
+------------------+------------------+-------------------------------+
| Metric           | Stackless        | Stackful                      |
+------------------+------------------+-------------------------------+
| Suspend          | Store index,     | Save all callee-saved regs    |
|                  | return to caller | (RSP, RBP, R12-R15, XMM)     |
| Resume           | Jump to label    | Restore registers, jump       |
| Typical latency  | 1-5 ns           | 20-100 ns                     |
| Symmetric xfer   | Tail call        | Not applicable                |
+------------------+------------------+-------------------------------+
|                                                                     |
|                  DECISION MATRIX                                    |
+-------------------------------------+-------------------------------+
| Scenario                            | Recommendation                |
+-------------------------------------+-------------------------------+
| High-concurrency I/O (100K+ tasks)  | Stackless -- memory wins      |
| Adapting legacy sync libraries      | Stackful -- transparent suspend|
| Game loop / real-time budgets       | Stackless -- predictable cost |
| Recursive algorithms with suspend   | Stackful -- natural fit       |
| Prototyping / rapid development     | Stackful -- less boilerplate  |
| Library API (public interface)      | Stackless -- standard, portable|
| Deep call chains with co_await      | Stackful avoids viral co_await|
| Embedded / memory-constrained       | Stackless -- tiny frames      |
+-------------------------------------+-------------------------------+
*/

// Benchmark skeleton: measure coroutine creation + resume overhead
#include <chrono>
#include <coroutine>
#include <cstdio>

struct BenchTask {
    struct promise_type {
        BenchTask get_return_object() {
            return BenchTask{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> h;
    ~BenchTask() { if (h) h.destroy(); }
};

BenchTask noop_coro() { co_return; }

int main() {
    constexpr int N = 1'000'000;

    auto t0 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        auto t = noop_coro();
        t.h.resume();  // resume to completion
        // destructor destroys frame
    }
    auto t1 = std::chrono::steady_clock::now();

    auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count();
    std::printf("Stackless: %d iterations, %.1f ns/iter\n", N, double(ns) / N);

    // Typical results (optimised):
    // Stackless: 1000000 iterations, 15-40 ns/iter (incl. alloc+dealloc)
    // With HALO:  ~2-5 ns/iter
    // Boost.Fiber: ~50-150 ns/iter for create+switch+destroy
}
```

---

## Notes

- C++20 coroutines are **not** a replacement for stackful coroutines - they are a different tool. The "viral `co_await`" problem (every function in the chain must be a coroutine) is the primary usability cost of the stackless model.
- Stackful coroutines can be implemented with **segmented stacks** (GCC's split stacks) to reduce memory waste, but this adds complexity and hurts cache locality.
- Windows fibers (`CreateFiber`/`SwitchToFiber`) and POSIX `ucontext_t` are OS-level stackful mechanisms. Boost.Context provides a portable, optimised abstraction over both.
- Boost.Fiber builds on Boost.Context and adds cooperative scheduling, channels, and synchronisation primitives - a full fiber runtime. Consider it when you need stackful coroutines with structured concurrency.
- The C++ standardisation committee considered stackful coroutines (N4024, P0876) but chose the stackless model for C++20 because of its zero-overhead abstraction properties. Stackful support may come in a future standard or via `std::execution`.
- **Hybrid approach:** Use C++20 stackless coroutines at the top level (scheduler, I/O multiplexer) and stackful fibers for CPU-bound traversals or legacy library integration. Many production systems (e.g., Amazon, Facebook) use this pattern.
- On ARM and RISC-V, the register save/restore cost of stackful context switches is higher than on x86-64 due to more callee-saved registers. This tilts the tradeoff further toward stackless on those architectures.
