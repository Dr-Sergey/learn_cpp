# Understand final_suspend Customization for Detached vs Awaited Tasks

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** [cppreference - Coroutines (final_suspend)](https://en.cppreference.com/w/cpp/language/coroutines#co_await)  

---

## Topic Overview

When a coroutine body finishes - either by reaching the closing brace, hitting `co_return`, or having an unhandled exception captured - the compiler executes `co_await promise.final_suspend()`. The awaiter returned by `final_suspend()` determines **what happens to the coroutine frame after completion**. This is the single most impactful customization point for coroutine lifetime management, and it's the one you need to get right before anything else works reliably.

Here's a quick map of your three canonical choices:

| `final_suspend()` returns | Behaviour | Frame cleanup | Pattern name |
| --- | --- | --- | --- |
| `std::suspend_always{}` | Coroutine stays suspended; caller destroys handle | Caller calls `handle.destroy()` | **Awaited / caller-driven** |
| `std::suspend_never{}` | Coroutine self-destructs immediately | Automatic | **Fire-and-forget / detached** |
| Custom awaiter | Transfers to a continuation or scheduler | Continuation or scheduler | **Symmetric transfer / continuation** |

If the table feels abstract, here is what each choice actually means in practice:

**`suspend_always` (caller-driven):** The coroutine frame remains valid after completion. The caller can inspect the promise (read result, check exceptions) and must eventually call `destroy()`. This is the correct choice for `Task<T>`, generators, and any coroutine whose lifetime is managed by a RAII wrapper. Think of it as the coroutine politely waiting for you to come collect the result.

**`suspend_never` (fire-and-forget):** The frame is destroyed before the caller can inspect it. There is no handle to destroy - and destroying an already-destroyed handle is **undefined behaviour**. Use this only for truly detached coroutines where no one holds the handle. The reason this trips people up is that it looks safe until you accidentally store the handle somewhere.

**Custom awaiter (symmetric transfer):** The awaiter's `await_suspend` returns a `coroutine_handle<>` - the continuation. The current coroutine is suspended (frame stays alive) and the continuation is resumed **without adding a stack frame** (symmetric transfer). The continuation typically destroys the completed coroutine's handle. This is the gold standard for task systems that chain coroutines.

The diagram below shows where control goes at the end of each path:

```cpp
               ┌──────────────┐
               │ Coroutine    │
               │ body done    │
               └──────┬───────┘
                      │
         co_await promise.final_suspend()
                      │
        ┌─────────────┼─────────────────┐
        ▼             ▼                  ▼
  suspend_always   suspend_never    Custom awaiter
   (suspended)    (frame destroyed)  (symmetric transfer)
        │                               │
   caller calls                    resume continuation
   handle.destroy()                     │
                                   continuation does
                                   handle.destroy()
```

**Critical constraint:** `final_suspend()` must be `noexcept`. An exception here is instant `std::terminate()` - there is no recovery path.

---

## Self-Assessment

### Q1: Demonstrate `suspend_always` for a caller-driven Task with RAII cleanup, and show what goes wrong with `suspend_never` if a handle is held

The key thing to watch here is how `final_suspend` choice determines who is responsible for the coroutine frame. Notice that `CallerDrivenTask` stores a handle and cleans it up in the destructor - that only works because `suspend_always` keeps the frame alive long enough for the destructor to find it.

```cpp
#include <coroutine>
#include <cstdio>
#include <utility>
#include <exception>

// --- Caller-driven Task (suspend_always) ---
struct CallerDrivenTask {
    struct promise_type {
        int result = 0;

        CallerDrivenTask get_return_object() {
            return CallerDrivenTask{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; } // KEY
        void return_value(int v) { result = v; }
        void unhandled_exception() { std::terminate(); }
    };

    int get() {
        handle_.resume();
        return handle_.promise().result;  // safe: frame is still alive
    }

    explicit CallerDrivenTask(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~CallerDrivenTask() {
        if (handle_) handle_.destroy();  // RAII cleanup
    }
    CallerDrivenTask(CallerDrivenTask&& o) noexcept
        : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    std::coroutine_handle<promise_type> handle_;
};

CallerDrivenTask safe_coro() { co_return 42; }

// --- Fire-and-forget (suspend_never) ---
struct FireAndForget {
    struct promise_type {
        FireAndForget get_return_object() { return {}; }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; } // KEY
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };
    // No handle stored - frame is already destroyed when we get here
};

FireAndForget detached_coro() {
    std::printf("fire-and-forget running\n");
    co_return;
    // Frame destroyed here - no one needs to call destroy()
}

int main() {
    // Caller-driven: safe to inspect result
    auto task = safe_coro();
    std::printf("result = %d\n", task.get());  // 42

    // Fire-and-forget: runs and cleans up on its own
    detached_coro();

    // DANGER: if you stored a handle from a suspend_never coroutine:
    // auto h = ...; h.destroy();  // UNDEFINED BEHAVIOUR - double destroy
}
```

Because `CallerDrivenTask` uses `suspend_always`, the frame is alive after `co_return 42`, and `handle_.promise().result` safely reads `42`. The `FireAndForget` coroutine owns nothing - the frame is gone the moment it finishes, so there is nothing to destroy and nothing to read.

---

### Q2: Implement a custom `final_suspend` awaiter that performs symmetric transfer to resume a waiting continuation

This is where it gets interesting. Instead of just suspending or self-destructing, the `final_suspend` awaiter can hand control directly to whoever was waiting for this coroutine - without pushing another stack frame. The `continuation` field in the promise is the key: it stores the handle of the outer coroutine that did `co_await inner_work()`.

```cpp
#include <coroutine>
#include <cstdio>
#include <utility>
#include <exception>

struct ContinuationTask {
    struct promise_type {
        int result = 0;
        std::coroutine_handle<> continuation{};  // who is waiting for us

        ContinuationTask get_return_object() {
            return ContinuationTask{
                std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() noexcept { return {}; }

        // Custom final_suspend: transfer to continuation
        auto final_suspend() noexcept {
            struct FinalAwaiter {
                bool await_ready() noexcept { return false; }

                // Symmetric transfer: resume continuation without stack growth
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> self) noexcept {
                    auto cont = self.promise().continuation;
                    if (cont) return cont;              // transfer to waiter
                    return std::noop_coroutine();       // no waiter -> return to caller
                }

                void await_resume() noexcept {}
            };
            return FinalAwaiter{};
        }

        void return_value(int v) { result = v; }
        void unhandled_exception() { std::terminate(); }
    };

    using handle_type = std::coroutine_handle<promise_type>;

    // Make ContinuationTask awaitable
    bool await_ready() noexcept { return false; }

    std::coroutine_handle<> await_suspend(std::coroutine_handle<> caller) noexcept {
        handle_.promise().continuation = caller;  // register who waits
        return handle_;  // symmetric transfer: start the inner task
    }

    int await_resume() {
        return handle_.promise().result;
    }

    explicit ContinuationTask(handle_type h) : handle_(h) {}
    ~ContinuationTask() { if (handle_) handle_.destroy(); }
    ContinuationTask(ContinuationTask&& o) noexcept
        : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    handle_type handle_;
};

ContinuationTask inner_work() {
    std::printf("inner: computing\n");
    co_return 42;
}

ContinuationTask outer_work() {
    std::printf("outer: awaiting inner\n");
    int val = co_await inner_work();  // symmetric transfer to inner
    // inner completes -> symmetric transfer back here
    std::printf("outer: got %d\n", val);
    co_return val * 2;
}

// Synchronous runner
void sync_run(ContinuationTask t) {
    auto h = t.handle_;
    h.resume();  // kick off outer -> transfers to inner -> transfers back -> outer finishes
    std::printf("final result: %d\n", h.promise().result);
}

int main() {
    auto t = outer_work();
    sync_run(std::move(t));
    // Output:
    // outer: awaiting inner
    // inner: computing
    // outer: got 42
    // final result: 84
}
```

The entire chain runs with a single call to `h.resume()` - no stack growth, no nested calls. The call flow looks like this:

```cpp
sync_run         outer_work        inner_work
  resume() --->    │
                    ├─ co_await --->    │
                    │                  ├─ co_return 42
                    │    <-- final_suspend transfer --┘
                    ├─ got 42
                    ├─ co_return 84
  <-- final_suspend transfer (to noop_coroutine) --┘
  reads result
```

---

### Q3: What happens if `final_suspend` throws? Demonstrate and explain `noexcept` requirement

The short answer is: it calls `std::terminate()`. The longer answer is that by the time `final_suspend` runs, the coroutine has already unwound its locals and reported its result via `return_value` or `return_void`. There is no catch handler left in scope. An exception here genuinely has nowhere to go - hence the hard requirement that the entire `co_await promise.final_suspend()` expression be non-throwing.

```cpp
#include <coroutine>
#include <cstdio>
#include <exception>

struct BadTask {
    struct promise_type {
        BadTask get_return_object() { return {}; }
        std::suspend_never initial_suspend() noexcept { return {}; }

        // THIS IS ILL-FORMED: final_suspend must be noexcept
        // The compiler may reject this or it leads to std::terminate at runtime.
        //
        // auto final_suspend() {    // <-- missing noexcept!
        //     throw 42;             // instant std::terminate
        //     return std::suspend_never{};
        // }

        // CORRECT:
        std::suspend_never final_suspend() noexcept { return {}; }

        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };
};

// Key points about final_suspend:
//
// 1. The standard requires final_suspend() to be noexcept.
//    [dcl.fct.def.coroutine]/15: "The expression co_await promise.final_suspend()
//    shall not be potentially-throwing."
//
// 2. If any part of the co_await on final_suspend can throw:
//    - await_ready() throws -> std::terminate
//    - await_suspend() throws -> std::terminate
//    - await_resume() throws -> std::terminate
//    All three must be noexcept.
//
// 3. WHY: At final_suspend, the coroutine has already completed.
//    Local variables are destroyed. The promise's return_value/return_void
//    has been called. There is no meaningful catch handler left.
//    An exception here has nowhere to propagate - hence terminate.
//
// 4. Compilers enforce this with varying strictness:
//    - MSVC: hard error if final_suspend is not noexcept
//    - GCC/Clang: may warn or error depending on version

int main() {
    std::printf("final_suspend must always be noexcept.\n");
    std::printf("Violating this is either ill-formed or calls std::terminate.\n");
}
```

The table below summarises which parts of the `final_suspend` protocol must be `noexcept` - and the answer is all of them:

| `final_suspend()` property | Requirement |
| --- | --- |
| `noexcept` specifier | Mandatory - `[dcl.fct.def.coroutine]/15` |
| `await_ready()` | Must be `noexcept` |
| `await_suspend()` | Must be `noexcept` |
| `await_resume()` | Must be `noexcept` |
| Return type | Any awaiter type (including custom) |

---

## Notes

- `std::noop_coroutine()` returns a handle that, when resumed, does nothing and returns control to the caller of `resume()`. It is the correct "nowhere to go" target for symmetric transfer when there is no continuation.
- The `suspend_always` vs `suspend_never` choice affects **exception safety**: with `suspend_always`, the caller can check `promise().eptr` before destroying. With `suspend_never`, the exception is either handled in `unhandled_exception()` or causes `terminate()` - the caller never sees it.
- Symmetric transfer was added specifically to solve the **stack overflow problem** in coroutine chains. Without it, `A awaits B awaits C awaits D` would nest four stack frames on `resume()`. With symmetric transfer, each `await_suspend` returns the next handle, and the compiler implements this as a tail-call loop.
- In a thread-pool scheduler, the custom `final_suspend` awaiter can enqueue the continuation onto the pool rather than resuming it immediately. This decouples completion from the thread that ran the coroutine.
- Never call `handle.destroy()` on a coroutine that has not reached final suspension (unless you are intentionally cancelling it and understand the destruction sequence of local variables).
- For generators, `final_suspend` returning `suspend_always` is **mandatory** - the iterator's `operator==` checks `handle.done()`, which requires the handle to be valid.
