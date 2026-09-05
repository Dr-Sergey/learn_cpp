# Use the Pipeline pattern with coroutines for streaming data processing

**Category:** Design Patterns Modern Takes  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/coroutine/generator>  

---

## Topic Overview

The **Pipeline pattern** composes a series of transformation stages where each stage processes one element at a time and passes it to the next - no intermediate containers needed. C++20 coroutines make this natural with `co_yield`: each stage is a generator that lazily produces values on demand.

The key mental model is that each pipeline stage is a function that is paused and resumed. When the consumer asks for the next value, the generator runs until it hits `co_yield`, hands the value back, and suspends itself. The next call picks up right where it left off. No intermediate vectors are allocated; elements flow through one at a time.

### Pipeline vs Eager Processing

The difference is significant for large data: eager processing creates full intermediate collections at every stage, while the pipeline processes one element at a time end-to-end.

```cpp
Eager (materializes intermediates):    Pipeline (lazy, streaming):
  auto parsed  = parse_all(input);       for (auto x : parse(input)
  auto filtered = filter_all(parsed);                  | filter(pred)
  auto output  = format_all(filtered);                 | format()) {
  // 3 full vectors in memory              process(x);  // 1 element at a time
                                         }
```

---

## Self-Assessment

### Q1: Implement a pipeline stage as a coroutine generator that takes an input range and yields transformed values

Before you can write pipeline stages, you need a `Generator<T>` type that implements the coroutine machinery. This boilerplate defines the promise type, the iterator interface for range-for, and the handle lifecycle. C++23 gives you `std::generator<T>` for free, but understanding the manual version is worthwhile.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <coroutine>
#include <optional>
#include <cstdint>

// Minimal Generator<T> coroutine type
template<typename T>
class Generator {
public:
    struct promise_type {
        T current_value;
        std::suspend_always yield_value(T value) {
            current_value = std::move(value);
            return {};
        }
        Generator get_return_object() {
            return Generator{Handle::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    using Handle = std::coroutine_handle<promise_type>;

    explicit Generator(Handle h) : handle_(h) {}
    ~Generator() { if (handle_) handle_.destroy(); }

    Generator(Generator&& other) noexcept : handle_(other.handle_) { other.handle_ = nullptr; }
    Generator& operator=(Generator&&) = delete;

    // Iterator interface for range-for
    struct iterator {
        Handle handle;
        bool operator!=(std::default_sentinel_t) const { return !handle.done(); }
        iterator& operator++() { handle.resume(); return *this; }
        T& operator*() { return handle.promise().current_value; }
    };

    iterator begin() { handle_.resume(); return {handle_}; }
    std::default_sentinel_t end() { return {}; }

private:
    Handle handle_;
};

// Pipeline stage: transform
Generator<int> multiply_by(Generator<int> input, int factor) {
    for (auto value : input) {
        co_yield value * factor;  // Lazy: one at a time
    }
}

// Pipeline stage: source from vector
Generator<int> from_vector(const std::vector<int>& data) {
    for (int v : data) {
        co_yield v;
    }
}

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // Pipeline: source -> multiply by 3
    auto pipeline = multiply_by(from_vector(data), 3);

    for (auto value : pipeline) {
        std::cout << value << ' ';  // 3 6 9 12 15
    }
    std::cout << '\n';
}
```

Each `co_yield` in `multiply_by` suspends the coroutine and hands one value to the caller. The `from_vector` generator feeds values one at a time into `multiply_by`, which transforms and yields each one. The whole thing runs without ever building an intermediate collection.

### Q2: Chain three coroutine stages: parse -> filter -> format without materializing intermediates

Here's a realistic three-stage pipeline: raw strings come in, get parsed to integers (bad parses are silently dropped), even numbers pass the filter, and the format stage wraps results in a display string. Because every stage is a generator, the chain `format(filter_even(parse(raw_data)))` evaluates lazily.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <coroutine>
#include <sstream>
#include <charconv>

// (Generator<T> definition same as Q1 - omitted for brevity)
template<typename T>
class Generator {
public:
    struct promise_type {
        T current_value;
        std::suspend_always yield_value(T value) { current_value = std::move(value); return {}; }
        Generator get_return_object() { return Generator{Handle::from_promise(*this)}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    using Handle = std::coroutine_handle<promise_type>;
    explicit Generator(Handle h) : handle_(h) {}
    ~Generator() { if (handle_) handle_.destroy(); }
    Generator(Generator&& o) noexcept : handle_(o.handle_) { o.handle_ = nullptr; }
    struct iterator {
        Handle handle;
        bool operator!=(std::default_sentinel_t) const { return !handle.done(); }
        iterator& operator++() { handle.resume(); return *this; }
        T& operator*() { return handle.promise().current_value; }
    };
    iterator begin() { handle_.resume(); return {handle_}; }
    std::default_sentinel_t end() { return {}; }
private:
    Handle handle_;
};

// Stage 1: Parse (string -> int)
Generator<int> parse(const std::vector<std::string>& lines) {
    for (const auto& line : lines) {
        int value{};
        auto [ptr, ec] = std::from_chars(line.data(), line.data() + line.size(), value);
        if (ec == std::errc{}) {
            co_yield value;  // Only yield valid parses
        }
    }
}

// Stage 2: Filter (keep even numbers)
Generator<int> filter_even(Generator<int> input) {
    for (auto value : input) {
        if (value % 2 == 0) {
            co_yield value;
        }
    }
}

// Stage 3: Format (int -> string)
Generator<std::string> format(Generator<int> input) {
    int index = 0;
    for (auto value : input) {
        co_yield "[" + std::to_string(index++) + "] = " + std::to_string(value);
    }
}

int main() {
    std::vector<std::string> raw_data = {
        "10", "abc", "20", "15", "30", "xyz", "40"
    };

    // Chain: parse -> filter_even -> format
    // NO intermediate vectors created!
    auto pipeline = format(filter_even(parse(raw_data)));

    for (auto& line : pipeline) {
        std::cout << line << '\n';
    }
    // [0] = 10
    // [1] = 20
    // [2] = 30
    // [3] = 40
}
```

Watch how `"abc"` and `"xyz"` are silently dropped by the parse stage - they never even reach the filter. The pipeline handles this gracefully because `co_yield` is simply not called for invalid input. You get clean error handling without needing exceptions or sentinel values.

### Q3: Compare coroutine pipelines with ranges::views pipelines for lazy streaming processing

Both approaches are lazy - neither materializes intermediate results. The difference is where they shine. Ranges views are ideal for pure data transformations with no state between elements. Coroutines are better when a stage needs to remember something across calls, or when the source involves async I/O.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <string>
#include <algorithm>

// APPROACH 1: ranges::views pipeline
void ranges_pipeline() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Lazy pipeline: no intermediate containers
    auto result = data
        | std::views::filter([](int x) { return x % 2 == 0; })
        | std::views::transform([](int x) { return x * x; })
        | std::views::take(3);

    std::cout << "Ranges: ";
    for (auto x : result)
        std::cout << x << ' ';  // 4 16 36
    std::cout << '\n';
}

// APPROACH 2: Coroutine pipeline (conceptual)
// See Q1-Q2 for full Generator implementation
// auto result = take(3, square(filter_even(from_vector(data))));

int main() {
    ranges_pipeline();

    /*
    Feature comparison:

    Feature            | ranges::views        | Coroutine generator
    -------------------|----------------------|----------------------
    Laziness           | Yes, pull-based      | Yes, pull-based
    No intermediates   | Yes                  | Yes
    Composable         | Yes, pipe operator | | Yes, function nesting
    Stateful stages    | Limited              | Yes, full (local vars)
    Async I/O          | No                   | Yes (with co_await)
    Infinite streams   | Yes (iota, generate) | Yes (while(true) co_yield)
    Compile time       | Heavy template meta  | Lighter
    Runtime overhead   | Zero (inlined)       | Coroutine frame alloc
    Debugging          | Hard (deep templates)| Easier (step through)
    Standard support   | C++20 (full)         | C++23 std::generator

    When to use each:

    - ranges::views: pure data transformations, no I/O, no complex state
    - Coroutines: stateful processing, async I/O, complex control flow,
                  when stages need to maintain context between yields
    */
}
```

The ranges approach has zero per-element overhead because everything gets inlined. The coroutine approach allocates a small frame per stage (~100-200 bytes), but it can do things ranges simply can't - like remembering a running average, deduplicating consecutive values, or waiting on a network read between yields.

---

## Notes

- C++23 adds `std::generator<T>` so you don't need to write your own; before C++23, use a custom Generator or the cppcoro library.
- Coroutine pipelines allocate a frame per stage (~100-200 bytes); ranges views are fully stack-allocated and inlined.
- Coroutines shine for **stateful stages** (deduplication, running averages, buffering) where ranges views become awkward or impossible.
- For simple filter/transform chains with no state, prefer `std::views` - it's zero overhead and fully standard.
