# Understand co_yield Patterns and Lazy Generator Pipelines

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** [cppreference - co_yield](https://en.cppreference.com/w/cpp/language/co_yield)  

---

## Topic Overview

`co_yield expr` is syntactic sugar for `co_await promise.yield_value(expr)`. The promise type's `yield_value` stores the yielded value and returns an awaiter - almost always `std::suspend_always` - so the coroutine suspends and the caller can pull the value. This is the foundation of the **pull-based lazy generator** pattern: values are produced one at a time, on demand, with zero heap allocation for the sequence itself.

A **Generator\<T\>** wraps a `coroutine_handle<promise_type>` and exposes an iterator interface (`begin`/`end` or `operator()`) so it can be used in range-for loops. Because the coroutine is suspended between yields, **memory usage is O(1)** regardless of how many values the logical sequence contains - a radical improvement over materialising an entire `std::vector`.

The real power emerges when generators **compose**. A `transform` generator takes an input generator and a function, co_yielding mapped values. A `filter` generator skips values that fail a predicate. A `take` generator yields at most N values then returns. These combinators form **lazy pipelines** analogous to ranges but implemented entirely with coroutines - and they work on compilers that don't yet ship full `<ranges>` support.

Here is a quick reference for the mechanism and where suspension happens at each stage:

| Concept | Mechanism | Suspension Point |
| --- | --- | --- |
| `co_yield v` | `co_await promise.yield_value(v)` | After value stored |
| `initial_suspend` | `suspend_always` for lazy start | Before first resume |
| `final_suspend` | `suspend_always` to keep handle alive | After coroutine body ends |
| Pipeline stage | Generator consuming another Generator | Each stage suspends independently |

The data flow in a two-stage pipeline looks like this - each stage pulls from the one below it, and suspension propagates back up to the caller on demand:

```cpp
Caller --resume()--> Generator A --resume()--> Generator B
       <--yield--   (transform)   <--yield--   (source)
```

---

## Self-Assessment

### Q1: Implement a basic `Generator<T>` that supports range-for and demonstrate `co_yield` with a fibonacci sequence

This is the building block for everything else in this topic. Pay attention to two details in the promise type: `initial_suspend` returns `suspend_always` (lazy start - the body doesn't run until you call `begin()`), and `final_suspend` also returns `suspend_always` (keeps the frame alive long enough for the destructor to call `destroy()`):

```cpp
#include <coroutine>
#include <cstdint>
#include <iostream>
#include <utility>

template <typename T>
class Generator {
public:
    struct promise_type {
        T current_value;

        Generator get_return_object() {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T value) noexcept {
            current_value = std::move(value);
            return {};
        }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };

    // --- Iterator for range-for ---
    struct sentinel {};
    struct iterator {
        std::coroutine_handle<promise_type> h;

        iterator& operator++() { h.resume(); return *this; }
        T const& operator*() const { return h.promise().current_value; }
        bool operator==(sentinel) const { return h.done(); }
    };

    iterator begin() {
        handle_.resume();          // lazy start: advance to first yield
        return {handle_};
    }
    sentinel end() { return {}; }

    // --- RAII ---
    explicit Generator(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~Generator() { if (handle_) handle_.destroy(); }
    Generator(Generator&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}
    Generator& operator=(Generator&&) = delete;

private:
    std::coroutine_handle<promise_type> handle_;
};

// Infinite fibonacci sequence -- O(1) memory
Generator<std::uint64_t> fibonacci() {
    std::uint64_t a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

int main() {
    int count = 0;
    for (auto v : fibonacci()) {
        std::cout << v << ' ';
        if (++count == 15) break;   // must break -- sequence is infinite
    }
    // Output: 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377
}
```

`initial_suspend` returns `suspend_always` so the coroutine does not execute until `begin()` calls `resume()`. `final_suspend` returns `suspend_always` so the handle stays valid for the destructor to call `destroy()`. If you got these backwards, you'd have a use-after-free on one end and a hanging iterator on the other.

---

### Q2: Build composable pipeline combinators - `transform`, `filter`, `take` - and chain them over a generator

Each combinator is itself a coroutine that consumes one generator and produces another. The result is a lazy pipeline where no intermediate buffer is ever created. Pull from the end, and the pull ripples back through every stage:

```cpp
// Reuses Generator<T> from Q1.

// --- Combinator: transform ---
template <typename T, typename F>
Generator<std::invoke_result_t<F, T const&>> transform(Generator<T> source, F func) {
    for (auto&& v : source) {
        co_yield func(v);
    }
}

// --- Combinator: filter ---
template <typename T, typename Pred>
Generator<T> filter(Generator<T> source, Pred pred) {
    for (auto&& v : source) {
        if (pred(v)) co_yield v;
    }
}

// --- Combinator: take ---
template <typename T>
Generator<T> take(Generator<T> source, std::size_t n) {
    std::size_t i = 0;
    for (auto&& v : source) {
        if (i++ >= n) co_return;
        co_yield v;
    }
}

// --- Source generator ---
Generator<int> iota(int start) {
    while (true) co_yield start++;
}

int main() {
    // Pipeline: natural numbers -> keep evens -> square them -> take 8
    auto pipeline = take(
        transform(
            filter(iota(1), [](int v) { return v % 2 == 0; }),
            [](int v) { return v * v; }
        ),
        8
    );

    for (auto v : pipeline) {
        std::cout << v << ' ';
    }
    // Output: 4 16 36 64 100 144 196 256
}
```

Here is how the data flows through the pipeline one step at a time. Each stage only produces a value when the stage above it asks:

```cpp
iota(1) --> filter(even) --> transform(square) --> take(8) --> caller
  1            skip
  2            2               4                     4          print
  3            skip
  4            4               16                    16         print
  ...
```

Each stage is a separate coroutine frame. When `take` asks for the next value, it resumes `transform`, which resumes `filter`, which resumes `iota` - a chain of symmetric pulls with **no intermediate buffering**. You can compose as many stages as you want without ever materialising the full sequence.

---

### Q3: What happens if `yield_value` returns a custom awaiter instead of `suspend_always`? Demonstrate a buffered generator that batches yields

This is one of the more subtle points about `co_yield`: `yield_value` is not required to return `suspend_always`. The awaiter it returns controls whether the coroutine actually suspends. If `await_ready()` on that awaiter returns `true`, the coroutine keeps running without giving control back to the caller. This is how you implement batched or buffered yields:

```cpp
#include <coroutine>
#include <iostream>
#include <vector>

template <typename T>
class BatchGenerator {
public:
    struct promise_type;
    using handle_type = std::coroutine_handle<promise_type>;

    struct batch_awaiter {
        promise_type& p;
        // Don't suspend if buffer isn't full yet
        bool await_ready() noexcept { return p.buffer.size() < p.batch_size; }
        void await_suspend(std::coroutine_handle<>) noexcept {}
        void await_resume() noexcept {}
    };

    struct promise_type {
        std::vector<T> buffer;
        std::size_t batch_size = 4;

        BatchGenerator get_return_object() {
            return BatchGenerator{handle_type::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        // Custom awaiter: only suspend when batch is full
        batch_awaiter yield_value(T value) {
            buffer.push_back(std::move(value));
            return {*this};
        }
        void return_void() noexcept {}
        void unhandled_exception() { std::terminate(); }
    };

    // Pull one batch at a time
    std::vector<T> const* next_batch() {
        auto& p = handle_.promise();
        p.buffer.clear();
        while (!handle_.done() && p.buffer.size() < p.batch_size) {
            handle_.resume();
        }
        return p.buffer.empty() ? nullptr : &p.buffer;
    }

    explicit BatchGenerator(handle_type h) : handle_(h) {}
    ~BatchGenerator() { if (handle_) handle_.destroy(); }
    BatchGenerator(BatchGenerator&& o) noexcept : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    handle_type handle_;
};

BatchGenerator<int> numbers(int n) {
    for (int i = 0; i < n; ++i) {
        co_yield i;
    }
}

int main() {
    auto gen = numbers(10);
    while (auto* batch = gen.next_batch()) {
        std::cout << "batch:";
        for (auto v : *batch) std::cout << ' ' << v;
        std::cout << '\n';
    }
    // Output:
    // batch: 0 1 2 3
    // batch: 4 5 6 7
    // batch: 8 9
}
```

The reason this works: `yield_value` is a customisation point that returns an awaiter, and `await_ready()` on that awaiter controls whether the coroutine pauses. When `await_ready()` returns `true` (buffer not full yet), `co_yield` does **not** suspend - the coroutine immediately continues to produce the next value. The caller only regains control when the batch is complete (or the coroutine finishes). This technique is useful for I/O batching and vectorised processing where per-element suspension overhead would be wasteful.

---

## Notes

- `co_yield` is purely a promise-level customisation point - the compiler always rewrites it to `co_await promise.yield_value(expr)`. Understanding this rewrite is the key to understanding everything else about generators.
- Generators are move-only; copying a `coroutine_handle` would create a double-destroy bug when both copies try to call `destroy()` in their destructors.
- Pipeline combinators are themselves coroutines, so each stage adds one coroutine frame (~frame-size bytes on the heap, often 64-256 bytes). For very deep pipelines, measure allocation overhead before assuming it's free.
- C++23 `std::generator` (P2502) standardises the pattern shown in Q1 and integrates with `<ranges>`. Prefer it when available; hand-roll only for custom suspension logic (e.g., batching in Q3).
- Infinite generators are safe as long as the consumer eventually `break`s or the pipeline includes `take`. Forgetting to break causes an infinite loop, **not** infinite memory growth - the coroutine never advances beyond one yield at a time.
- Allocation elision (HALO - Heap Allocation eLision Optimization) can eliminate coroutine frame allocation when the compiler can prove the frame's lifetime is bounded. Short-lived pipeline stages are prime candidates.
