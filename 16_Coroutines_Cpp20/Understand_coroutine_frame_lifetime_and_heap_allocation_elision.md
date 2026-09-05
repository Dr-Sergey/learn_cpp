# Understand coroutine frame lifetime and heap allocation elision

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

This topic focuses on **coroutine frame lifetime management** - when frames are created, when they're destroyed, and the specific scenarios that prevent the compiler from eliding heap allocation.

Understanding this well pays off in two ways. First, you avoid lifetime bugs (using a destroyed frame, double-destroying a handle). Second, you can write coroutines that the compiler's HALO optimisation can actually act on, avoiding heap allocation entirely for short-lived generators.

### Coroutine Frame Lifetime

Every coroutine frame goes through a fixed sequence of events. The key thing to notice is that the frame is alive all the way through `final_suspend` - you must read your result from the promise before you call `destroy()`:

```cpp
create coroutine
    |
    +- operator new (allocate frame)   <- or HALO (stack)
    +- copy/move parameters into frame
    +- construct promise_type
    +- call get_return_object()
    +- co_await initial_suspend()
    |
    +- [coroutine body runs / suspends / resumes]
    |
    +- co_await final_suspend()        <- frame still alive
    |                                     (must read result before destroy)
    +- destroy promise_type
    +- destroy local variables
    +- operator delete (free frame)    <- or automatic (stack)
```

### Lifetime vs Scope

These two terms are easy to confuse. The scope of the handle variable is where the `coroutine_handle` object lives in your code. The lifetime of the frame is the actual period of allocation. HALO can only fire when the lifetime is contained entirely within the immediate caller's scope:

| Concept | Description |
| --- | --- |
| **Scope** | Where the coroutine handle variable lives (caller's function) |
| **Lifetime** | From `operator new` to `handle.destroy()` or auto-destroy |
| **HALO condition** | Lifetime is within the scope of the immediate caller |

---

## Self-Assessment

### Q1: Explain when HALO can place the coroutine frame on the stack instead of the heap

**HALO (Heap Allocation Elision Optimization) requirements:**

The reason HALO is so picky is that "putting the frame on the caller's stack" only works if the compiler can guarantee the frame will never be accessed after the caller's stack frame is gone. That rules out several common patterns:

1. **The coroutine is called and consumed in the same scope:**

```cpp
// HALO-eligible: gen lives and dies in process()
void process() {
    auto gen = my_generator();  // create
    for (auto& val : gen) {     // consume
        use(val);
    }                           // destroy
}
```

1. **The handle does not escape:**

```cpp
// NOT HALO-eligible: handle escapes via return
auto create() {
    return my_generator();  // handle escapes to caller!
}
```

1. **No cross-TU opacity:**

```cpp
// NOT HALO-eligible without LTO: coroutine defined in another .cpp
extern Generator make_gen();  // opaque to compiler
void use() {
    auto g = make_gen();  // compiler can't see inside make_gen
}
```

1. **The compiler can determine the frame size at the call site:**

```cpp
// NOT HALO-eligible: virtual/indirect call
struct Base {
    virtual Generator gen() = 0;  // frame size unknown at call site
};
```

### Q2: Show a case where heap elision is prevented: the coroutine outlives its call site

This example shows three cases in the same file - two that prevent HALO and one that allows it. The contrast makes the rule concrete:

```cpp
#include <coroutine>
#include <iostream>
#include <memory>
#include <exception>
#include <utility>

struct Generator {
    struct promise_type {
        int val;
        Generator get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(int v) { val = v; return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~Generator() { if (handle) handle.destroy(); }
    Generator(Generator&& o) noexcept : handle(std::exchange(o.handle, {})) {}
    Generator& operator=(Generator&&) = delete;
};

Generator iota(int start) {
    for (int i = start; ; ++i)
        co_yield i;
}

// Case 1: HALO PREVENTED -- coroutine outlives creation scope
Generator create_generator() {
    return iota(0);  // returned to caller -> outlives this function
}

// Case 2: HALO PREVENTED -- stored in heap-allocated object
struct Widget {
    Generator gen;
    Widget() : gen(iota(100)) {}  // gen outlives the constructor scope
};

// Case 3: HALO ELIGIBLE -- consumed in same scope
void consume_locally() {
    auto gen = iota(0);
    for (int i = 0; i < 5; ++i) {
        gen.handle.resume();
        std::cout << gen.handle.promise().val << ' ';
    }
    std::cout << '\n';
    // gen destroyed here -- same scope as creation
}

int main() {
    consume_locally();

    // The returned generator required heap allocation
    auto g = create_generator();
    g.handle.resume();
    std::cout << "From factory: " << g.handle.promise().val << '\n';
}
// Expected output:
// 0 1 2 3 4
// From factory: 0
```

Cases 1 and 2 are the patterns that trip people up most often. Returning a generator from a factory function is a very common pattern, and it's also exactly the pattern that forces heap allocation. If you write a lot of small, short-lived generators, prefer consuming them immediately in the same function.

### Q3: Measure the overhead of coroutine frame heap allocation vs a comparable state machine

This benchmark measures the per-resume overhead for a simple counting generator against a plain class-based state machine doing the same work. The ratio shows the cost of the heap allocation plus the indirect jump on resume:

```cpp
#include <chrono>
#include <coroutine>
#include <iostream>

// Coroutine generator
struct CoroGen {
    struct promise_type {
        int val;
        CoroGen get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(int v) { val = v; return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~CoroGen() { if (handle) handle.destroy(); }
};

CoroGen coro_counter(int n) {
    for (int i = 0; i < n; ++i)
        co_yield i;
}

// Manual state machine equivalent
class ManualCounter {
    int current_ = 0;
    int limit_;
public:
    ManualCounter(int n) : limit_(n) {}
    bool done() const { return current_ >= limit_; }
    int next() { return current_++; }
};

int main() {
    constexpr int N = 10'000'000;
    long long sink = 0;

    // Benchmark coroutine
    auto t1 = std::chrono::high_resolution_clock::now();
    {
        auto gen = coro_counter(N);
        for (int i = 0; i < N; ++i) {
            gen.handle.resume();
            sink += gen.handle.promise().val;
        }
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark state machine
    auto t3 = std::chrono::high_resolution_clock::now();
    {
        ManualCounter mc(N);
        while (!mc.done())
            sink += mc.next();
    }
    auto t4 = std::chrono::high_resolution_clock::now();

    auto coro_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto manual_ms = std::chrono::duration<double, std::milli>(t4 - t3).count();

    std::cout << "Coroutine:     " << coro_ms << " ms\n";
    std::cout << "State machine: " << manual_ms << " ms\n";
    std::cout << "Ratio:         " << (coro_ms / manual_ms) << "x\n";
    std::cout << "(sink=" << sink << ")\n";
}
// Typical results (Release build, x64):
// Coroutine:     ~80 ms   (1 heap alloc + resume overhead)
// State machine: ~20 ms   (pure stack, no suspension)
// Ratio:         ~4x
//
// With HALO (coroutine inlined): ratio approaches 1x
```

**Analysis:** The bulk of coroutine overhead is `resume()` call overhead (indirect jump through coroutine state), not the single heap allocation. The heap allocation happens once per creation; the indirect jump happens on every resume. HALO eliminates the allocation but the suspension/resumption overhead remains. If you need zero overhead, you need the coroutine to be fully inlined and HALO to fire.

---

## Notes

- A coroutine frame is destroyed when: (a) `handle.destroy()` is called explicitly, or (b) the coroutine runs to completion and `final_suspend()` returns `suspend_never`.
- **Never** destroy a coroutine handle while the coroutine is actively running - that is undefined behavior.
- Custom `operator new`/`delete` in promise_type lets you use pool allocators to avoid per-coroutine `malloc`. This is the standard approach for high-throughput servers where heap allocation pressure matters.
- With `-O2` and full inlining, HALO can make coroutines zero-cost. Without optimization, the compiler always heap-allocates.
- GCC 14+ and Clang 17+ have improved HALO; MSVC has limited HALO support as of current releases.
