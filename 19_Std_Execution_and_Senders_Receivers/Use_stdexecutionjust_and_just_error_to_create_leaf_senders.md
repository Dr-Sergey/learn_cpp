# Use std::execution::just and just_error to create leaf senders

**Category:** Std Execution and Senders Receivers  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

Every sender pipeline has to start somewhere. Leaf senders are those starting points. They don't depend on any predecessor - they simply inject a value, an error, or a cancellation signal into the pipeline. Think of them as the `return`, `throw`, and `cancel` of the sender world, respectively.

| Factory | Completes via | Carries |
| --- | --- | --- |
| `just(vals...)` | `set_value(vals...)` | Zero or more values |
| `just_error(err)` | `set_error(err)` | An error value |
| `just_stopped()` | `set_stopped()` | Cancellation signal |

```cpp
Leaf senders (no predecessor):

  just(42, "hi")  --->  set_value(42, "hi")
  just_error(ep)  --->  set_error(ep)
  just_stopped()  --->  set_stopped()

They are the "return" / "throw" / "cancel" of the sender world.
```

---

## Self-Assessment

### Q1: `just(42)` creates a sender that produces a value

`just` can carry any number of values, including zero. When you pass multiple values they all travel together as a bundle to the next step in the pipeline:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <string>

int main() {
    // Single value:
    auto s1 = stdexec::just(42);
    auto [val] = stdexec::sync_wait(std::move(s1)).value();
    std::cout << "Single: " << val << '\n';  // 42

    // Multiple values:
    auto s2 = stdexec::just(42, std::string("hello"), 3.14);
    auto [i, str, d] = stdexec::sync_wait(std::move(s2)).value();
    std::cout << i << ", " << str << ", " << d << '\n';  // 42, hello, 3.14

    // Zero values (void sender):
    auto s3 = stdexec::just();
    auto result = stdexec::sync_wait(std::move(s3));
    if (result) std::cout << "Void sender completed\n";

    // In a pipeline:
    auto pipeline = stdexec::just(10, 20)
        | stdexec::then([](int a, int b) {
            return a + b;
        })
        | stdexec::then([](int sum) {
            std::cout << "Sum: " << sum << '\n';  // 30
            return sum;
        });

    stdexec::sync_wait(std::move(pipeline));
}
// Output:
// Single: 42
// 42, hello, 3.14
// Void sender completed
// Sum: 30
```

The multi-value case (`just(10, 20)`) is worth noting - both values arrive at the `then` callback together as separate arguments. This is different from wrapping them in a `pair` or `tuple`; the values are genuinely separate and the callback receives them as individual parameters.

### Q2: `just_error` creates a sender that completes with an error

`just_error` is how you inject a pre-known failure into a pipeline. You'll use it in tests, in mock implementations, and in `let_error` recovery handlers that want to re-signal an error of a different type. The error type is not restricted to `exception_ptr` - it can be a `std::error_code`, a custom enum, or anything the pipeline is prepared to handle:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <stdexcept>
#include <system_error>

int main() {
    // just_error with exception_ptr:
    auto s1 = stdexec::just_error(
        std::make_exception_ptr(std::runtime_error("db connection failed")));

    try {
        stdexec::sync_wait(std::move(s1));
    } catch (const std::runtime_error& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }

    // just_error with error_code:
    auto s2 = stdexec::just_error(
        std::make_error_code(std::errc::permission_denied));
    // This sender's error type is std::error_code, not exception_ptr

    // Use in a pipeline with upon_error recovery:
    auto with_recovery = stdexec::just_error(
            std::make_exception_ptr(std::runtime_error("fail")))
        | stdexec::upon_error([](std::exception_ptr ep) {
            std::cout << "Recovering from error\n";
            return 0;  // fallback value
        });

    auto [val] = stdexec::sync_wait(std::move(with_recovery)).value();
    std::cout << "Recovered: " << val << '\n';  // 0

    // just_stopped for cancellation:
    auto s3 = stdexec::just_stopped();
    auto result = stdexec::sync_wait(std::move(s3));
    if (!result) {
        std::cout << "Sender was stopped (nullopt)\n";
    }
}
// Output:
// Caught: db connection failed
// Recovering from error
// Recovered: 0
// Sender was stopped (nullopt)
```

Notice how `just_stopped()` produces `nullopt` from `sync_wait`. The stopped channel is the sender world's cancellation signal - it's neither a value nor an error, it's a distinct third outcome meaning "the operation was cancelled."

### Q3: `just` is the async equivalent of a completed `future<T>`

If you've used `std::future`, this comparison can help anchor your mental model. `just` is like a `future` that's already resolved - except it's lazy, composable, and capable of carrying multiple values:

| Concept | `std::future<T>` | `just(value)` |
| --- | --- | --- |
| Represents | An eventually-available value | An immediately-available sender |
| Blocking? | `get()` blocks | `sync_wait` blocks |
| Composable? | No (no `then`) | Yes (`\| then(...)`) |
| Lazy? | No (work starts immediately) | Yes (no work until connected) |
| Multiple values? | No (single T) | Yes (`just(a, b, c)`) |
| Error channel? | Exception only | `just_error` for typed errors |
| Cancel? | No standard cancel | `just_stopped()` |

```cpp
// future world:
std::future<int> f = std::async([] { return 42; });  // starts immediately
int val = f.get();  // blocks, extracts value

// sender world:
auto s = stdexec::just(42);  // lazy, no work
auto [val2] = stdexec::sync_wait(
    s | stdexec::then([](int x) { return x * 2; })
).value();  // blocks, drives pipeline

// just(42) is like make_ready_future(42) -- already completed,
// but unlike future, it composes into pipelines.

// The equivalences:
//   just(v)           ~  make_ready_future(v)
//   just_error(e)     ~  make_exceptional_future(e)
//   just_stopped()    ~  (no future equivalent)
//   just(a, b, c)     ~  (no future equivalent -- futures carry one value)
```

The fact that `just` is lazy even though its value is known immediately might seem odd at first. The reason is uniformity: the entire sender framework is lazy. `just` follows the same protocol as every other sender, which means you can slot it into any pipeline without special cases.

---

## Notes

- `just` copies its arguments; use `std::move` for move-only types.
- `just()` (no args) is useful as a pipeline trigger that carries no data.
- Leaf senders have known completion signatures at compile time.
- `just` + `then` is the minimal sender pipeline.
- The standard header is `<execution>`; NVIDIA's implementation uses `<stdexec/execution.hpp>`.
