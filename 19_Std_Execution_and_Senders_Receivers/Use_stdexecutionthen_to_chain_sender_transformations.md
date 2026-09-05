# Use std::execution::then to chain sender transformations

**Category:** Std Execution and Senders Receivers  
**Standard:** C++20  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

`then()` is the workhorse of sender pipelines. It transforms the value produced by a sender through a callable, producing a new sender with the transformed type. You can think of it as the async equivalent of calling a function on the result - each `then()` in a chain is one transformation step.

The pipeline is lazy - no work happens until the sender is connected to a receiver and started. Building the chain is just describing what to do; nothing runs yet.

```cpp
Type flow through then():

  just(42)        -> sender<int>
  | then(to_str)  -> sender<string>      (int -> string)
  | then(length)  -> sender<size_t>      (string -> size_t)
  | then(double)  -> sender<size_t>      (size_t -> size_t)
```

---

## Self-Assessment

### Q1: Pipe through `then` and verify the value reaches the receiver

Here is the simplest pipeline you can write: a single `just` producing a value, piped into a `then` that transforms it, and `sync_wait` to get the result back. Notice the multi-value example at the end - when the predecessor produces multiple values, the `then` callable receives them all as separate arguments:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    // Basic then:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Transforming: " << x << " * 2\n";
            return x * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 84

    // Multi-value then:
    auto multi = stdexec::just(10, 20)
        | stdexec::then([](int a, int b) {
            return a + b;  // receives ALL predecessor values
        });
    auto [sum] = stdexec::sync_wait(std::move(multi)).value();
    std::cout << "Sum: " << sum << '\n';  // 30

    // Manual connect + start (low-level):
    // auto op = stdexec::connect(sender, receiver);
    // stdexec::start(op);  // triggers the pipeline
    // sync_wait does this internally.
}
```

The comment about `connect` and `start` is worth noting. `sync_wait` is just a convenience wrapper that does those two things and then blocks. Understanding that it delegates to `connect` + `start` helps when you need to do lower-level work later.

### Q2: `then()` is lazy - no work until connected and started

This is the property that makes pipelines composable and storage-safe. When you write `just(42) | then(f)` you are constructing a description of work, not performing it. The counter stays at zero until `sync_wait` actually triggers execution:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    int counter = 0;

    // Build the pipeline (no work yet!):
    auto pipeline = stdexec::just(42)
        | stdexec::then([&counter](int x) {
            ++counter;
            return x * 2;
        })
        | stdexec::then([&counter](int x) {
            ++counter;
            return x + 1;
        });

    std::cout << "After build:  counter = " << counter << '\n';  // 0

    // sync_wait connects and starts the pipeline:
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();

    std::cout << "After run:    counter = " << counter << '\n';  // 2
    std::cout << "Result: " << result << '\n';  // 85
}
// Output:
// After build:  counter = 0
// After run:    counter = 2
// Result: 85
```

Laziness is not just a technicality - it enables a whole set of practical capabilities:

- Building pipelines without side effects
- Storing pipelines in data structures
- Choosing when/where to execute
- Inspecting completion signatures at compile time

### Q3: Chain three `then()` - type changes at each step

One of the nicest things about `then()` is that the type can change at every step. The compiler tracks the full type chain at compile time, which means type errors are caught before any code runs. This example walks through three type transformations - `int` to `double`, `double` to `string`, `string` to `size_t`:

<!-- compile: needs third-party library header `stdexec/execution.hpp` -->
```cpp
#include <stdexec/execution.hpp>
#include <iostream>
#include <string>
#include <cmath>

int main() {
    auto pipeline = stdexec::just(42)        // sender<int>
        | stdexec::then([](int x) -> double {
            // Step 1: int -> double
            return std::sqrt(static_cast<double>(x));
        })                                    // sender<double>
        | stdexec::then([](double x) -> std::string {
            // Step 2: double -> string
            return "sqrt = " + std::to_string(x);
        })                                    // sender<string>
        | stdexec::then([](const std::string& s) -> std::size_t {
            // Step 3: string -> size_t
            std::cout << s << '\n';
            return s.size();
        });                                   // sender<size_t>

    auto [length] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "String length: " << length << '\n';
}
// Output:
// sqrt = 6.480741
// String length: 15
```

The diagram below shows the compile-time type flow. If you accidentally pass the wrong lambda signature at any step, the compiler will tell you immediately - it knows the expected type at each stage before a single byte of the executable is generated:

```cpp
Type transformation at each then():

  just(42)                  | completion_signatures<set_value(int)>
     |                       |
  then([](int) -> double)  | completion_signatures<set_value(double)>
     |                       |
  then([](double) -> str)  | completion_signatures<set_value(string)>
     |                       |
  then([](str) -> size_t)  | completion_signatures<set_value(size_t)>

All type deduction happens at COMPILE TIME.
The compiler knows the full type chain before any code runs.
```

---

## Notes

- `then` only operates on the value channel; errors skip `then` nodes.
- If the callable throws, the exception routes to `set_error`.
- `then` returns a move-only sender; use `split()` to share.
- For void predecessors: `then([]() { return 42; })`.
- `upon_error` and `upon_stopped` are the error/cancellation equivalents of `then`.
