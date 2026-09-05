# Use coroutines for async I/O and task chaining

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

Coroutines enable writing async I/O code that looks synchronous. Each `co_await` suspends the coroutine until the I/O completes, avoiding callback nesting ("callback hell"). The result is code that reads top-to-bottom like ordinary sequential logic, even though it is non-blocking under the hood.

### Async I/O with Coroutines vs Callbacks

Here is the same multi-step async operation written both ways. The left side is the classic callback pyramid - every step nests inside the previous one. The right side is the coroutine version, where each step is just another `co_await` on the next line:

```cpp
CALLBACK STYLE (deeply nested):       COROUTINE STYLE (flat):
read_file(path, [](data) {             auto data = co_await read_file(path);
  parse(data, [](parsed) {             auto parsed = co_await parse(data);
    validate(parsed, [](ok) {          auto ok = co_await validate(parsed);
      save(ok, [](result) {            auto result = co_await save(ok);
        done(result);                  done(result);
      });
    });
  });
});
```

### Task Chaining Pattern

When each step produces a value that feeds into the next, you get a pipeline. Coroutines make that pipeline look like a straight line:

```cpp
task<A> step1()    --> co_await --> task<B> step2(A) --> co_await --> task<C> step3(B)
                                    result of step1        result of step2
                                    feeds into step2       feeds into step3
```

---

## Self-Assessment

### Q1: Write a `co_await`-based async file read that suspends until I/O completes

The key piece here is `AsyncFileRead`. It is an awaitable whose `await_suspend` spins up a background thread to do the actual file read, and then resumes the coroutine handle when the work is done. From the coroutine's perspective, `co_await async_read(...)` looks like a synchronous call - it simply produces the file contents as a `std::string`.

```cpp
#include <coroutine>
#include <fstream>
#include <iostream>
#include <optional>
#include <sstream>
#include <string>
#include <thread>
#include <utility>
#include <algorithm>
#include <exception>

// Minimal task type
template<typename T>
struct Task {
    struct promise_type {
        std::optional<T> value;
        std::exception_ptr exception;
        Task get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(T v) { value = std::move(v); }
        void unhandled_exception() { exception = std::current_exception(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~Task() { if (handle) handle.destroy(); }
    Task(Task&& o) noexcept : handle(std::exchange(o.handle, {})) {}
    T get() {
        handle.resume();
        if (handle.promise().exception)
            std::rethrow_exception(handle.promise().exception);
        return std::move(*handle.promise().value);
    }
};

// Awaitable that simulates async file read
struct AsyncFileRead {
    std::string path;
    std::string result;

    bool await_ready() { return false; }  // always suspend

    void await_suspend(std::coroutine_handle<> handle) {
        // Simulate async I/O on a background thread
        std::thread([this, handle]() {
            std::ifstream file(path);
            if (file) {
                std::ostringstream oss;
                oss << file.rdbuf();
                result = oss.str();
            } else {
                result = "[ERROR: file not found]";
            }
            handle.resume();  // resume coroutine when I/O done
        }).detach();
    }

    std::string await_resume() {
        return std::move(result);
    }
};

AsyncFileRead async_read(const std::string& path) {
    return AsyncFileRead{path};
}

// Coroutine that reads a file
Task<std::string> read_and_process() {
    // This looks synchronous but is actually async!
    std::string content = co_await async_read("example.txt");
    co_return "Read " + std::to_string(content.size()) + " bytes";
}

int main() {
    // Create a test file
    { std::ofstream("example.txt") << "Hello, async world!"; }

    auto task = read_and_process();
    std::string result = task.get();
    std::cout << result << '\n';

    // Cleanup
    std::remove("example.txt");
}
// Expected output:
// Read 19 bytes
```

Notice how `read_and_process` reads exactly like synchronous code - there is no callback, no future polling, no explicit thread join. The coroutine suspends at `co_await async_read(...)`, the background thread does its work, then the coroutine wakes up and continues on the next line.

### Q2: Chain two coroutines so that the result of one feeds into the next using `co_await`

This example builds a three-stage pipeline: fetch raw CSV text, parse it into a vector of integers, then sum the integers. Each stage is its own coroutine, and the `pipeline()` coroutine chains them together with three `co_await` expressions. The result of each step flows directly into the parameter of the next.

```cpp
#include <coroutine>
#include <iostream>
#include <optional>
#include <sstream>
#include <string>
#include <utility>
#include <vector>
#include <exception>

template<typename T>
struct Task {
    struct promise_type {
        std::optional<T> value;
        Task get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_never initial_suspend() noexcept { return {}; }  // eager
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(T v) { value = std::move(v); }
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~Task() { if (handle) handle.destroy(); }
    Task(Task&& o) noexcept : handle(std::exchange(o.handle, {})) {}

    // Awaitable for chaining
    bool await_ready() { return handle.done(); }
    void await_suspend(std::coroutine_handle<> h) { h.resume(); }
    T await_resume() { return std::move(*handle.promise().value); }
};

// Step 1: Fetch raw data
Task<std::string> fetch_data() {
    co_return std::string("100,200,300,400,500");
}

// Step 2: Parse CSV into numbers (takes result of step 1)
Task<std::vector<int>> parse_csv(std::string raw) {
    std::vector<int> nums;
    std::istringstream ss(raw);
    std::string token;
    while (std::getline(ss, token, ','))
        nums.push_back(std::stoi(token));
    co_return nums;
}

// Step 3: Compute sum (takes result of step 2)
Task<int> compute_sum(std::vector<int> nums) {
    int total = 0;
    for (int n : nums) total += n;
    co_return total;
}

// Chained pipeline: each step feeds into the next
Task<std::string> pipeline() {
    // Each co_await gets the result of the previous step
    auto raw    = co_await fetch_data();        // string
    auto nums   = co_await parse_csv(raw);       // vector<int>
    auto total  = co_await compute_sum(nums);    // int

    co_return "Sum of " + std::to_string(nums.size())
             + " numbers = " + std::to_string(total);

}

int main() {
    auto result = pipeline();
    std::cout << *result.handle.promise().value << '\n';
}
// Expected output:
// Sum of 5 numbers = 1500
```

The pipeline reads exactly like a series of function calls, with each result named and immediately available on the next line. Tracing the data flow is straightforward:

```bash
pipeline() -- co_await fetch_data()    -> "100,200,..."
           -- co_await parse_csv(raw)  -> {100, 200, ...}
           -- co_await compute_sum(v)  -> 1500
           -- co_return formatted string
```

### Q3: Explain why coroutines avoid callback hell compared to traditional async code

The callback style forces you to express sequential logic as nested closures. Every level of nesting is another step in the sequence, and because each callback is a separate lambda, error handling has to be repeated at every level - or scattered into separate error paths that are easy to miss. Here is what a real multi-step request handler looks like in callback style:

```cpp
void process_request(Request req) {
    db.query(req.user_id, [&](auto user) {           // level 1
        auth.check(user, [&](bool ok) {               // level 2
            if (!ok) { handle_error(); return; }
            cache.get(user.key, [&](auto data) {      // level 3
                transform(data, [&](auto result) {    // level 4
                    response.send(result, [&]() {     // level 5
                        log.write("done");             // lost in nesting
                    });
                });
            });
        });
    });
    // Error handling? Scattered across 5 callbacks!
    // Control flow? Nearly impossible to follow
}
```

The coroutine version of the same logic is flat. Every step is one line. Errors from any step propagate to a single `catch` block at the top level:

```cpp
Task<void> process_request(Request req) {
    try {
        auto user   = co_await db.query(req.user_id);   // flat!
        bool ok     = co_await auth.check(user);         // flat!
        if (!ok) { handle_error(); co_return; }
        auto data   = co_await cache.get(user.key);      // flat!
        auto result = co_await transform(data);          // flat!
        co_await response.send(result);                  // flat!
        co_await log.write("done");                      // flat!
    } catch (const std::exception& e) {
        // ALL errors caught in ONE place!
        handle_error(e);
    }
}
```

The difference in readability is dramatic once you go beyond two or three steps. Here is the full comparison across the properties that matter most:

| Aspect | Callbacks | Coroutines |
| --- | --- | --- |
| Nesting | O(N) levels deep | Flat |
| Error handling | Per-callback, scattered | `try/catch` (structured) |
| Control flow | Inversion of control | Natural, top-to-bottom |
| Resource lifetime | Tricky (shared_ptr, weak_ptr) | RAII (scope-based) |
| Debugging | Hard (fragmented stack) | Easier (sequential logic) |
| Cancellation | Manual token passing | Destroy coroutine handle |

---

## Notes

- Coroutines are **not threads** - they run on whatever thread resumes them. Use an executor to control which thread.
- For real async I/O, integrate with OS-level mechanisms (IOCP on Windows, epoll on Linux).
- Libraries like Boost.Asio provide `co_await`-compatible async operations.
- `co_await` expresses **sequential** async operations. For **concurrent** operations, use a `when_all()` combinator.
