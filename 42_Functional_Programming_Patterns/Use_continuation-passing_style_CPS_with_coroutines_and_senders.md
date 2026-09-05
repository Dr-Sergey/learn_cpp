# Use continuation-passing style (CPS) with coroutines and senders

**Category:** Functional Programming Patterns  
**Standard:** C++20, C++26  
**Reference:** <https://wg21.link/P2300>  

---

## Topic Overview

Continuation-passing style (CPS) is a pattern where instead of returning a result, a function takes a "continuation" (a callback) that receives the result. Coroutines and the sender/receiver model (P2300) are structured forms of CPS that make asynchronous code readable without sacrificing composability.

The reason this concept matters is that async I/O fundamentally cannot "return" a value in the traditional sense - the value isn't available yet. CPS solves this by flipping the model: instead of "give me the result," you say "here's what to do with the result when it's ready." Coroutines and senders are ways to write that same idea with much nicer syntax.

### Manual CPS (Callback Style)

Here's what CPS looks like in its raw form. The function takes the continuation as a parameter and calls it when the result is available. The problem with this approach becomes clear quickly:

```cpp
#include <functional>
#include <string>
#include <iostream>

// CPS: instead of returning, pass result to continuation
void read_file_cps(const std::string& path,
                   std::function<void(std::string)> on_success,
                   std::function<void(std::string)> on_error) {
    // Simulate async IO
    if (path.empty()) {
        on_error("empty path");
    } else {
        on_success("file contents of " + path);
    }
}

// Nesting becomes "callback hell":
void process() {
    read_file_cps("config.json",
        [](std::string config) {
            read_file_cps(config, // pretend config contains a path
                [](std::string data) {
                    std::cout << "Got: " << data << "\n";
                },
                [](std::string err) { std::cerr << err << "\n"; });
        },
        [](std::string err) { std::cerr << err << "\n"; });
}
```

Each sequential step requires another level of nesting. Error handling at each level is duplicated. This is the classic "callback hell" or "pyramid of doom" problem that drove the creation of coroutines.

### Coroutines: Structured CPS

Coroutines solve the readability problem while keeping the underlying CPS mechanism. When the compiler encounters `co_await`, it saves the current function's state as a continuation and hands it to the awaitable. When the awaitable has the result ready, it calls that continuation. The programmer writes sequential code; the compiler generates the CPS:

```cpp
#include <coroutine>
#include <string>
#include <iostream>

// A coroutine IS CPS — co_await is "pass the continuation"
// The compiler transforms:
//   auto result = co_await async_read("path");
//   process(result);
// Into:
//   async_read("path", [](auto result) { process(result); });

// But with coroutines, it LOOKS like sequential code:
Task<void> process_structured() {
    auto config = co_await async_read("config.json");
    auto data = co_await async_read(config);
    std::cout << "Got: " << data << "\n";
    // No nesting! Same logic, linear structure
}
```

The reason this trips people up is that `co_await` doesn't actually block the thread - it suspends the coroutine and lets the thread do other work. The coroutine resumes later when the result arrives, exactly as if the continuation was called. It's CPS in disguise, written to look synchronous.

### Senders/Receivers: Composable CPS

The P2300 sender/receiver model goes further, making CPS pipelines composable with pipe operators. A sender describes a computation without starting it. Connecting it to a receiver starts execution. This laziness enables powerful optimizations and structured cancellation:

```cpp
// P2300 std::execution — structured, composable CPS:
#include <execution>  // Future standard

auto work = ex::just("config.json")
    | ex::then([](auto path) { return read_file(path); })
    | ex::then([](auto config) { return parse_config(config); })
    | ex::upon_error([](auto err) { log_error(err); return default_config(); });

// Each `then` attaches a continuation
// The pipeline is a "sender" — computation doesn't start until connected to a receiver
// ex::sync_wait(work); starts execution
```

The pipeline reads left to right like a data transformation chain. Each `then` call wraps the previous sender in a new sender that will call the next continuation when ready. Nothing runs until you connect a receiver - for example, `ex::sync_wait` which blocks until the work completes.

---

## Self-Assessment

### Q1: What's the relationship between CPS and coroutines

Coroutines are syntactic sugar for CPS. When the compiler encounters `co_await expr`, it saves the current function state (the continuation) and passes it to the awaitable. The awaitable calls the continuation when the result is ready. The programmer writes sequential code; the compiler generates the callback-passing structure automatically. This is why coroutines feel synchronous even though they're fundamentally async.

### Q2: How do senders improve over raw callbacks

Senders are lazy (computation doesn't start until connected to a receiver), composable (pipe operators chain operations naturally), and structured (the lifetime of the work and its cancellation are explicit). Raw callbacks are eager (side effects start immediately), hard to compose (you manually nest them), and error-prone (callback hell, missed error handling paths, dangling reference bugs when captures outlive their objects).

### Q3: Show CPS transform for a simple function

Here's the mechanical transformation from direct style to CPS for the simplest possible case - an addition function. This shows exactly what the compiler is doing behind the scenes with coroutines:

```cpp
// Direct style:
int add(int a, int b) { return a + b; }
int result = add(1, 2);
process(result);

// CPS transform:
void add_cps(int a, int b, std::function<void(int)> k) {
    k(a + b);  // Pass result to continuation k
}
add_cps(1, 2, [](int result) { process(result); });
```

In direct style, you call `add` and use its return value. In CPS, there is no return value - instead you pass a function `k` (the continuation, traditionally named `k` in the literature) that receives the result and continues the computation.

---

## Notes

- CPS is the theoretical foundation for async/await in C++, JavaScript, Rust, and C#. All of these languages compile their `await` expressions to essentially the same CPS structure under the hood.
- Senders/receivers (P2300) are targeting C++26 as the standard C++ async model, providing a structured and composable way to express asynchronous work.
- CPS enables tail-call optimization in languages that support it, but C++ does not guarantee tail-call elimination, so deeply recursive CPS in C++ can still overflow the stack.
- Coroutines make CPS ergonomic while maintaining the performance benefits - the suspension and resumption overhead is minimal compared to thread context switches.
