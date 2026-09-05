# Use `std::osyncstream` (C++20) for Synchronized Concurrent Output

**Category:** Standard Library New Cpp23 26  
**Standard:** C++20 (often overlooked, included here for completeness)  
**Reference:** [cppreference — std::osyncstream](https://en.cppreference.com/w/cpp/io/basic_osyncstream)  

---

## Topic Overview

`std::osyncstream`, defined in `<syncstream>`, wraps an output stream (typically `std::cout` or a file stream) and guarantees that the entire sequence of output operations performed on it appears atomically in the wrapped stream. Without it, concurrent writes from multiple threads to `std::cout` produce interleaved, garbled output - a problem familiar to every multithreaded C++ developer.

The key mechanism is this: `osyncstream` buffers all output internally and flushes it as a single atomic unit to the underlying stream upon destruction (or explicit `.emit()`). You don't need a mutex - synchronization is handled internally using an unspecified mechanism tied to the underlying `streambuf`. This is exactly the kind of quality-of-life improvement that makes concurrent output much less of a headache.

| Approach | Thread-safe | Atomic lines | Requires mutex | Performance |
| --- | --- | --- | --- | --- |
| Raw `std::cout` | No - data races | No | No | Fast but broken |
| `std::cout` + mutex | Yes | Depends on lock scope | Yes - manual | Serialized |
| `std::osyncstream` | Yes | Yes - by design | No | Buffered, efficient |
| `std::println` (C++23) | Yes (implementation) | Single call | No | Single line |

The diagram below shows what happens at the stream level. Each thread owns its own `osyncstream` that buffers locally; the underlying `std::cout` only receives complete blocks:

```cpp
Thread A                    Thread B
┌────────────────┐          ┌────────────────┐
│ osyncstream os1 │          │ osyncstream os2 │
│ os1 << "A1\n"; │          │ os2 << "B1\n"; │
│ os1 << "A2\n"; │          │ os2 << "B2\n"; │
│ ~os1() -> emit  │          │ ~os2() -> emit  │
└──────┬─────────┘          └──────┬─────────┘
       │                           │
       ▼                           ▼
┌─────────────────────────────────────────┐
│ std::cout                               │
│ Output: "A1\nA2\n" then "B1\nB2\n"     │  <- never interleaved
│    OR:  "B1\nB2\n" then "A1\nA2\n"     │
└─────────────────────────────────────────┘
```

The order between different `osyncstream` objects is unspecified, but each stream's output appears as a contiguous block. That's the guarantee you actually care about.

---

## Self-Assessment

### Q1: How does `std::osyncstream` prevent interleaved output from multiple threads

The key insight is that individual `<<` calls on `std::cout` have been thread-safe since C++11 - each call won't corrupt internal state - but the *sequence* of calls is not atomic. Two threads can interleave their sequences. `osyncstream` makes the entire sequence from one thread appear as a unit. Here's the contrast:

```cpp
#include <syncstream>
#include <iostream>
#include <thread>
#include <vector>
#include <string>

// WITHOUT osyncstream: interleaved output
void print_without_sync(int id) {
    // Each << is individually thread-safe (since C++11),
    // but the SEQUENCE is not atomic
    std::cout << "Thread " << id << ": Hello from thread " << id << '\n';
    // Output can be: "Thread Thread 12: Hello: Hello from thread from thread 21\n\n"
}

// WITH osyncstream: atomic output blocks
void print_with_sync(int id) {
    // All output to os is buffered, then emitted atomically on destruction
    std::osyncstream os(std::cout);
    os << "Thread " << id << ": Hello from thread " << id << '\n';
    os << "Thread " << id << ": Second line\n";
    // ~os() emits both lines as one atomic block
}

int main() {
    constexpr int num_threads = 8;

    std::cout << "=== Without synchronization (may interleave) ===\n";
    {
        std::vector<std::jthread> threads;
        for (int i = 0; i < num_threads; ++i)
            threads.emplace_back(print_without_sync, i);
    }

    std::cout << "\n=== With osyncstream (never interleaves) ===\n";
    {
        std::vector<std::jthread> threads;
        for (int i = 0; i < num_threads; ++i)
            threads.emplace_back(print_with_sync, i);
    }
}
```

Notice that `print_with_sync` creates an `osyncstream` on the stack at the top of the function and lets the destructor do the work. This is the idiomatic pattern - you don't need to call `.emit()` manually when a scope boundary is a natural flush point.

---

### Q2: How do you control when output is emitted and use `osyncstream` with file streams

You're not limited to `std::cout`. `osyncstream` can wrap any `std::ostream`, including file streams opened by multiple threads. You can also call `.emit()` explicitly if you need the output to appear before the object is destroyed:

```cpp
#include <syncstream>
#include <iostream>
#include <fstream>
#include <thread>
#include <vector>
#include <chrono>

// Manual emit control
void manual_emit_example() {
    std::osyncstream os(std::cout);

    os << "Phase 1 output\n";
    os.emit();  // explicit flush: "Phase 1 output\n" appears now

    os << "Phase 2 output\n";
    // Phase 2 will be emitted on destruction
}

// File logging from multiple threads
void log_to_file(std::ostream& file, int thread_id, int iterations) {
    for (int i = 0; i < iterations; ++i) {
        // Each iteration's log entry is atomic
        std::osyncstream os(file);
        os << "[T" << thread_id << "] Iteration " << i
           << " at " << std::chrono::steady_clock::now()
                         .time_since_epoch().count() << '\n';
        // Destructor emits - entry is never split
    }
}

// Scoped blocks for multi-line atomic output
void report_status(int worker_id, int processed, int errors) {
    // Use a block scope to control emit timing
    {
        std::osyncstream os(std::cout);
        os << "╔══════════════════════════╗\n";
        os << "║ Worker " << worker_id << " Status Report ║\n";
        os << "║ Processed: " << processed << "            ║\n";
        os << "║ Errors:    " << errors    << "            ║\n";
        os << "╚══════════════════════════╝\n";
    }  // entire box emitted atomically here
}

int main() {
    manual_emit_example();

    // File output from multiple threads
    std::ofstream logfile("concurrent.log");
    {
        std::vector<std::jthread> workers;
        for (int i = 0; i < 4; ++i)
            workers.emplace_back(log_to_file, std::ref(logfile), i, 10);
    }
    logfile.close();

    // Console status reports from multiple workers
    {
        std::vector<std::jthread> workers;
        for (int i = 0; i < 3; ++i)
            workers.emplace_back(report_status, i, (i + 1) * 100, i);
    }
}
```

The scoped-block pattern in `report_status` is worth highlighting. By opening a block explicitly, you control exactly when the destructor fires and therefore exactly when the entire box appears on the console. This makes multi-line atomic output easy to reason about.

---

### Q3: What are the performance implications and when should you prefer alternatives

`osyncstream` is not always the right tool. For single-line output in C++23, `std::println` is simpler. For high-throughput logging, a lock-free queue with a dedicated writer thread beats any approach that synchronizes per message. Here's how the main options compare:

```cpp
#include <syncstream>
#include <iostream>
#include <sstream>
#include <mutex>
#include <thread>
#include <vector>
#include <format>
#include <print>
#include <chrono>

// Alternative 1: mutex + cout
std::mutex cout_mutex;

void log_with_mutex(int id, int n) {
    for (int i = 0; i < n; ++i) {
        std::lock_guard lock(cout_mutex);
        std::cout << "Thread " << id << ": message " << i << '\n';
    }
}

// Alternative 2: osyncstream
void log_with_syncstream(int id, int n) {
    for (int i = 0; i < n; ++i) {
        std::osyncstream(std::cout)
            << "Thread " << id << ": message " << i << '\n';
    }
}

// Alternative 3: std::println (C++23) - per-line atomic
void log_with_println(int id, int n) {
    for (int i = 0; i < n; ++i) {
        std::println("Thread {}: message {}", id, i);
    }
}

// Alternative 4: pre-format then single write
void log_preformat(int id, int n) {
    for (int i = 0; i < n; ++i) {
        auto msg = std::format("Thread {}: message {}\n", id, i);
        std::cout.write(msg.data(), msg.size());
        // Single write call - often atomic for small buffers on POSIX
    }
}

// Benchmark harness
template <typename F>
auto bench(std::string_view label, F&& func, int threads, int msgs) {
    auto t0 = std::chrono::high_resolution_clock::now();
    {
        std::vector<std::jthread> pool;
        for (int i = 0; i < threads; ++i)
            pool.emplace_back(func, i, msgs);
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    // Use syncstream here to avoid interleaved benchmark output
    std::osyncstream(std::cerr)
        << label << ": " << ms << " ms\n";
}

// When to use what:
//
// | Scenario                              | Best choice           |
// |---------------------------------------|-----------------------|
// | Multi-line atomic block               | osyncstream           |
// | Single line per thread                | std::println          |
// | Need custom mutex logic               | mutex + manual lock   |
// | High-throughput logging               | Lock-free queue + writer thread |
// | Debug output (don't care about order) | Raw cout is OK        |

int main() {
    constexpr int T = 4, N = 1000;

    // Redirect stdout to /dev/null for benchmarking
    bench("mutex",      log_with_mutex,      T, N);
    bench("syncstream", log_with_syncstream, T, N);
    bench("println",    log_with_println,    T, N);
}
```

The decision table in the comments is the practical takeaway. `osyncstream` shines for multi-line atomic blocks like status reports or formatted tables. For single lines, `std::println` is less ceremony. For serious throughput, get logging off the hot path entirely with a dedicated writer thread.

---

## Notes

- `std::osyncstream` is in `<syncstream>` (C++20). The alias `std::osyncstream` stands for `basic_osyncstream<char>`.
- Output is buffered internally and emitted atomically on destruction or explicit `.emit()`.
- Each `osyncstream` instance synchronizes with all other instances wrapping the same underlying stream - you don't need to coordinate them manually.
- This is not a replacement for proper logging frameworks, which provide timestamps, log levels, and async I/O.
- For single-line output in C++23+, `std::println` is simpler and implicitly synchronized per call.
- `osyncstream` excels at **multi-line atomic blocks** - entire tables, status reports, or formatted sections where all lines must appear together.
- The internal buffer may allocate heap memory. For extreme performance-critical paths, consider a lock-free queue with a dedicated writer thread instead.
- `osyncstream` can wrap any `std::ostream`: `cout`, `cerr`, `ofstream`, `ostringstream`.
- Moving an `osyncstream` transfers the buffered content - the moved-from object becomes empty.
- Compilers: GCC 11+, Clang 14+ (libstdc++), MSVC 19.29+ (VS 2019 16.10).
