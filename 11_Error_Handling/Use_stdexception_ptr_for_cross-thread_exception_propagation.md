# Use std::exception_ptr for cross-thread exception propagation

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/error/exception_ptr>  

---

## Topic Overview

`std::exception_ptr` is a reference-counted smart pointer that can hold a copy of any exception. It enables **capturing** an exception in one context (a worker thread, a callback, a coroutine) and **rethrowing** it in a completely different context - most commonly the main thread. Without `exception_ptr`, exceptions thrown in worker threads would terminate the program.

### The Problem: Exceptions Don't Cross Thread Boundaries

The reason this matters is that C++ exceptions are fundamentally a per-thread concept: an exception thrown in thread A can only be caught in thread A. If it escapes the thread's top-level function uncaught, the runtime calls `std::terminate`. `exception_ptr` is the bridge that lets you safely carry an exception across that boundary:

```cpp
Thread A (worker)                    Thread B (main)
+------------------+                +---------------------------+
| throw runtime_err | ------X------ | try { t.join(); }         |
| (uncaught!)       |  Can't cross  | // exception is LOST      |
| -> std::terminate |               | // or worse: terminate()  |
+------------------+                +---------------------------+

WITH exception_ptr:
Thread A (worker)                    Thread B (main)
+------------------+                +---------------------------+
| catch (...)       |                | if (eptr)                 |
|  eptr = current_  | ------OK----- |   rethrow_exception(eptr) |
|   exception()     |  Safe to copy | // exception re-thrown!   |
+------------------+                +---------------------------+
```

### Core API

| Function | Purpose |
| --- | --- |
| `std::current_exception()` | Captures the currently-handled exception into an `exception_ptr` (inside a `catch` block) |
| `std::rethrow_exception(eptr)` | Rethrows the exception held by `eptr` - can be caught normally |
| `std::make_exception_ptr(e)` | Creates an `exception_ptr` from any exception object (no need to throw first) |
| `eptr == nullptr` | Check if `exception_ptr` holds an exception |

### Lifecycle

Here is the simplest possible demonstration: capture an exception in one place, then rethrow it somewhere else entirely. The exception object lives as long as the `exception_ptr` that holds it:

```cpp
#include <exception>
#include <iostream>
#include <stdexcept>

int main() {
    std::exception_ptr eptr;  // null - no exception held

    // Capture an exception
    try {
        throw std::runtime_error("something failed");
    } catch (...) {
        eptr = std::current_exception();  // captures a COPY
    }

    // eptr is valid even though we left the catch block
    // Can be stored, copied, passed to another thread, etc.

    // Rethrow later (possibly in a different thread)
    if (eptr) {
        try {
            std::rethrow_exception(eptr);
        } catch (const std::runtime_error& e) {
            std::cout << "Re-caught: " << e.what() << "\n";
        }
    }
}
// Expected output:
//   Re-caught: something failed
```

### Important Notes

- `exception_ptr` is **reference-counted** - cheap to copy, multiple copies share the same exception object.
- `current_exception()` called **outside** a catch block returns a null `exception_ptr`.
- The exception object's lifetime extends as long as any `exception_ptr` refers to it.
- `rethrow_exception()` is `[[noreturn]]` - it always throws.

---

## Self-Assessment

### Q1: Capture an exception with `std::current_exception()` and rethrow it in another thread

**Solution - Worker Thread Exception Propagation:**

This is the pattern you will use most often. The rule is simple: always wrap your worker function body in a try/catch, capture with `current_exception()`, and check it in the main thread after `join()`. Never let an exception escape the thread function uncaught:

```cpp
#include <iostream>
#include <thread>
#include <exception>
#include <stdexcept>
#include <vector>
#include <mutex>
#include <string>

std::exception_ptr worker_exception;  // shared between threads

void worker_task(int task_id) {
    try {
        // Simulate work that might fail
        if (task_id == 3)
            throw std::runtime_error("Task #3 failed: database connection lost");

        std::cout << "Task #" << task_id << " completed successfully\n";
    } catch (...) {
        // Capture the exception - don't let it escape the thread!
        worker_exception = std::current_exception();
    }
}

int main() {
    // Launch worker thread
    std::thread t(worker_task, 3);
    t.join();  // wait for worker to finish

    // Check if worker reported an exception
    if (worker_exception) {
        std::cout << "Worker thread threw an exception, re-throwing...\n";
        try {
            std::rethrow_exception(worker_exception);
        } catch (const std::runtime_error& e) {
            std::cout << "Caught in main: " << e.what() << "\n";
        }
    }

    // --- Multiple workers with per-thread exception storage ---
    std::cout << "\n--- Multiple workers ---\n";
    constexpr int N = 5;
    std::vector<std::exception_ptr> exceptions(N);
    std::vector<std::thread> threads;

    for (int i = 0; i < N; ++i) {
        threads.emplace_back([i, &exceptions] {
            try {
                if (i % 2 == 0)
                    throw std::runtime_error("Worker " + std::to_string(i) + " failed");
                std::cout << "Worker " << i << " OK\n";
            } catch (...) {
                exceptions[i] = std::current_exception();
            }
        });
    }

    for (auto& t : threads) t.join();

    // Collect and report all exceptions
    for (int i = 0; i < N; ++i) {
        if (exceptions[i]) {
            try {
                std::rethrow_exception(exceptions[i]);
            } catch (const std::exception& e) {
                std::cout << "Thread " << i << " error: " << e.what() << "\n";
            }
        }
    }
}
// Expected output:
//   Worker thread threw an exception, re-throwing...
//   Caught in main: Task #3 failed: database connection lost
//
//   --- Multiple workers ---
//   Worker 1 OK
//   Worker 3 OK
//   Thread 0 error: Worker 0 failed
//   Thread 2 error: Worker 2 failed
//   Thread 4 error: Worker 4 failed
```

The per-thread vector of `exception_ptr` objects is the right approach when you spawn multiple workers. Each thread owns its own slot - no mutex needed for the captures themselves, because each thread only writes to its own index.

---

### Q2: Show how `std::promise` uses `exception_ptr` internally when `set_exception` is called

**Solution - Promise/Future Exception Propagation:**

`std::promise` is really just a structured wrapper around `exception_ptr` plus a value. When you call `set_exception`, it stores the pointer. When you call `future::get()`, it rethrows if a pointer is present. This example shows all three ways to get an exception into a future:

```cpp
#include <iostream>
#include <thread>
#include <future>
#include <stdexcept>
#include <cmath>

// std::promise::set_exception stores an exception_ptr internally.
// When the corresponding future::get() is called, the exception is rethrown.

double async_divide(double a, double b, std::promise<double>& prom) {
    try {
        if (b == 0.0)
            throw std::invalid_argument("Division by zero");
        if (std::isnan(a) || std::isnan(b))
            throw std::domain_error("NaN input");
        prom.set_value(a / b);
    } catch (...) {
        // Internally, set_exception stores current_exception()
        prom.set_exception(std::current_exception());
    }
    return 0;
}

int main() {
    // ===== Direct promise/future usage =====
    {
        std::promise<double> prom;
        std::future<double> fut = prom.get_future();
        std::thread t(async_divide, 10.0, 0.0, std::ref(prom));

        try {
            double result = fut.get();  // rethrows the stored exception!
            std::cout << "Result: " << result << "\n";
        } catch (const std::invalid_argument& e) {
            std::cout << "Caught from future: " << e.what() << "\n";
        }
        t.join();
    }

    // ===== make_exception_ptr (no throw needed) =====
    {
        std::promise<int> prom;
        auto fut = prom.get_future();

        // make_exception_ptr creates an exception_ptr WITHOUT throwing
        prom.set_exception(
            std::make_exception_ptr(std::runtime_error("pre-built error"))
        );

        try {
            fut.get();
        } catch (const std::runtime_error& e) {
            std::cout << "Caught make_exception_ptr: " << e.what() << "\n";
        }
    }

    // ===== std::async does this automatically =====
    {
        auto fut = std::async(std::launch::async, [] {
            throw std::logic_error("async failed");
            return 42;
        });

        try {
            fut.get();  // exception propagated automatically
        } catch (const std::logic_error& e) {
            std::cout << "Caught from async: " << e.what() << "\n";
        }
    }
}
// Expected output:
//   Caught from future: Division by zero
//   Caught make_exception_ptr: pre-built error
//   Caught from async: async failed
```

### How `promise`/`future` Uses `exception_ptr` Internally:

```cpp
promise::set_exception(eptr)     future::get()
+--------------------------+     +-------------------------------+
| shared_state->exception  |     | if (shared_state->exception)  |
|   = eptr;                |---->|   rethrow_exception(          |
| shared_state->ready      |     |     shared_state->exception)  |
|   = true;                |     | else                          |
| notify waiting thread    |     |   return shared_state->value  |
+--------------------------+     +-------------------------------+
```

---

### Q3: Explain why `exception_ptr` is safe to copy and transmit across thread boundaries

**Solution - Thread Safety of `exception_ptr`:**

You might wonder whether it is safe to hand the same `exception_ptr` to multiple threads simultaneously. It is, and this example shows why: each copy only increments an atomic reference count, and the exception object itself is immutable after capture:

```cpp
#include <iostream>
#include <thread>
#include <exception>
#include <stdexcept>
#include <vector>

int main() {
    // Create an exception_ptr
    auto eptr = std::make_exception_ptr(std::runtime_error("shared error"));

    // COPY it to multiple threads - this is safe!
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([eptr, i] {  // captured by VALUE (copied)
            try {
                std::rethrow_exception(eptr);  // each thread rethrows safely
            } catch (const std::exception& e) {
                std::cout << "Thread " << i << " caught: " << e.what() << "\n";
            }
        });
    }

    for (auto& t : threads) t.join();
}
// Expected output (order may vary):
//   Thread 0 caught: shared error
//   Thread 1 caught: shared error
//   Thread 2 caught: shared error
//   Thread 3 caught: shared error
```

**Why `exception_ptr` Is Thread-Safe:**

| Property | Explanation |
| --- | --- |
| **Reference-counted** | Like `shared_ptr` - uses atomic ref counting internally. Copies increment the count, destruction decrements it. |
| **Immutable exception object** | The stored exception is never modified after capture - read-only access from all threads. |
| **No data races** | Copying an `exception_ptr` only touches the atomic ref count. The pointed-to exception is shared but immutable. |
| **Lifetime management** | The exception object lives as long as at least one `exception_ptr` refers to it - no dangling. |
| **Standard guarantee** | [except.propagation] SS14.7 explicitly states `exception_ptr` can be used to propagate exceptions across threads. |

```cpp
exception_ptr Internals (conceptual):
+------------+   +------------+   +------------+
| eptr copy 1|   | eptr copy 2|   | eptr copy 3|
| (Thread A) |   | (Thread B) |   | (Thread C) |
+-----+------+   +-----+------+   +-----+------+
      |                |                |
      +----------------+----------------+
                       |
              +----------------+
              | Exception obj  |  <- atomic refcount = 3
              | (immutable)    |  <- allocated once, shared
              | runtime_error  |
              | "shared error" |
              +----------------+
```

Here is a quick summary of what is and is not safe to do with a shared `exception_ptr`:

```cpp
// Safe: copying exception_ptr across threads
auto eptr2 = eptr;  // atomic ref increment

// Safe: rethrowing from any thread
std::rethrow_exception(eptr);

// Safe: comparing
if (eptr == nullptr) { /* ... */ }

// NOT safe: writing to the SAME exception_ptr variable from multiple threads
// (same as any shared non-atomic variable)
// Use a mutex or per-thread exception_ptrs instead.
```

---

## Notes

- **Always capture exceptions in worker threads** - uncaught exceptions in a `std::thread` call `std::terminate`.
- **`std::async` propagates automatically** - it internally uses `exception_ptr` + `promise`/`future`.
- **`std::current_exception()` in a catch block** - if called outside a catch block, returns null.
- **`make_exception_ptr(e)`** is more efficient than `try { throw e; } catch (...) { return current_exception(); }` - avoids an actual throw/catch cycle on some implementations.
- **`exception_ptr` keeps the exception alive** - be careful about long-lived `exception_ptr` objects holding large exception hierarchies in memory.
- **`packaged_task`** also uses `exception_ptr` behind the scenes - any exception thrown inside a `packaged_task` is automatically stored and re-thrown on `future::get()`.
