# Use `std::osyncstream` (C++20) for Synchronized Output Without Interleaving

**Category:** Concurrency and Parallelism  
**Standard:** C++20 (Header `<syncstream>`)  
**Reference:** [cppreference - std::basic_osyncstream](https://en.cppreference.com/w/cpp/io/basic_osyncstream)  

---

## Topic Overview

Here is a problem you will run into the moment you start writing multithreaded code that prints anything: individual `operator<<` calls on `std::cout` are thread-safe in the sense that they won't corrupt memory, but that guarantee is much weaker than it sounds. A single logical line like `std::cout << "ID: " << id << "\n";` compiles into multiple separate calls, and the thread scheduler is free to interleave another thread's output between any two of them. The result is garbled, unreadable output even though no undefined behavior occurred.

The traditional fix is to wrap every print statement in a `std::mutex` lock. That works, but it's error-prone (you have to remember the lock at every call site), it's hard to compose (every helper function has to know about the mutex), and it means you hold the lock for the entire duration of potentially slow formatting.

`std::osyncstream` solves this at the library level. It wraps any `std::basic_ostream` and buffers all output locally in a `std::basic_syncbuf`. The buffered content is then atomically transferred to the underlying stream when the `osyncstream` object is destroyed or when `.emit()` is called explicitly. The C++ standard guarantees that characters transferred from different `osyncstream` objects targeting the same stream do not interleave - each emission appears as one contiguous block.

| Aspect | `std::mutex` + `cout` | `std::osyncstream` |
| --- | --- | --- |
| Granularity of atomicity | Manual (lock scope) | Automatic (object lifetime) |
| Composability | Poor - callee must know the mutex | Excellent - just pass an `osyncstream` |
| Risk of forgetting the lock | High | None - RAII guarantees flush |
| Performance model | Contention on mutex every insertion | Local buffering, one sync on emit |
| Available since | C++11 | C++20 |

Here is the mental model for what happens under the hood. Internally, `std::basic_syncbuf<CharT, Traits, Allocator>` holds a pointer to the wrapped streambuf and an internal buffer. On `emit()`, it locks an implementation-defined mutex associated with the wrapped streambuf, copies the buffer contents, and unlocks. The key insight is that the lock is held only during the copy step - not during the potentially slow formatting work that filled the buffer. That is why contention is so much lower than holding a user-level mutex for the entire print statement.

The diagram below shows the guarantee visually:

```cpp
Thread A                         Thread B
─────────────────────────────    ─────────────────────────────
osyncstream out(cout);           osyncstream out(cout);
out << "Hello ";                 out << "World ";
out << "from A\n";               out << "from B\n";
~osyncstream -> emit()           ~osyncstream -> emit()
   ┌─ lock ──────────┐             ┌─ lock ──────────┐
   │ "Hello from A\n" │             │ "World from B\n" │
   └──────────────────┘             └──────────────────┘
         ▼                                ▼
    cout receives complete blocks - never "Hello World from from A B\n"
```

---

## Self-Assessment

### Q1: What is the output guarantee of `std::osyncstream` and how does it compare to raw `cout` under concurrency

The code below runs two batches of threads - one using raw `std::cout`, one using `std::osyncstream` - so you can see the difference directly. Pay attention to how `print_synced` works: all the `<<` operations go into an internal buffer, and only when the local `out` object is destroyed does the entire buffer transfer atomically to `std::cout`.

```cpp
// Compile: g++ -std=c++20 -pthread q1_osyncstream_basic.cpp -o q1
#include <iostream>
#include <syncstream>
#include <thread>
#include <vector>

void print_raw(int id) {
    // Multiple operator<< calls: interleaving is possible
    std::cout << "Thread " << id << ": raw output line\n";
}

void print_synced(int id) {
    // Entire sequence is emitted atomically on destruction
    std::osyncstream out(std::cout);
    out << "Thread " << id << ": synced output line\n";
    // ~osyncstream calls emit() here
}

int main() {
    constexpr int N = 8;
    std::vector<std::jthread> threads;

    std::cout << "=== Raw cout (may interleave) ===\n";
    for (int i = 0; i < N; ++i)
        threads.emplace_back(print_raw, i);
    threads.clear();  // join all

    std::cout << "\n=== osyncstream (no interleaving) ===\n";
    for (int i = 0; i < N; ++i)
        threads.emplace_back(print_synced, i);
    threads.clear();

    return 0;
}
```

Each `osyncstream` temporary buffers all `<<` operations locally. When the object is destroyed at the end of `print_synced`, `emit()` atomically transfers the buffer. Two threads' emissions targeting the same `cout` are serialised - the standard says they "do not interleave." The raw version has no such guarantee, so expect jumbled output there.

---

### Q2: How does explicit `emit()` let you control when the atomic flush happens, and what is the state of the `osyncstream` after it

Sometimes you want to flush part of your output mid-function - for example, after completing phase one of a long-running task - without destroying the `osyncstream` object yet. The explicit `emit()` call handles that. Notice in the code below that after calling `emit()`, the `osyncstream` is empty but still alive and ready to accumulate more output.

```cpp
// Compile: g++ -std=c++20 -pthread q2_explicit_emit.cpp -o q2
#include <iostream>
#include <syncstream>
#include <thread>
#include <string>

void worker(int id) {
    std::osyncstream out(std::cout);

    // Phase 1: accumulate a progress report
    out << "[Worker " << id << "] Phase 1 complete. ";
    out << "Partial results ready.\n";
    out.emit();  // flush to cout atomically NOW

    // After emit(), the osyncstream is empty but still usable
    // Phase 2: accumulate more output
    out << "[Worker " << id << "] Phase 2 complete. ";
    out << "Final results ready.\n";
    // implicit emit() on destruction
}

int main() {
    std::jthread t1(worker, 1);
    std::jthread t2(worker, 2);
    std::jthread t3(worker, 3);
    return 0;
}
```

After `emit()`, the internal buffer is cleared and the `osyncstream` can be reused. Each `emit()` is a separate atomic transfer, so Phase 1 and Phase 2 outputs are independently non-interleaving blocks. If you need two logically separate messages to never split another thread's message, call `emit()` between them.

---

### Q3: How does `osyncstream` interact with `std::flush`, `std::endl`, and the `emit_on_flush` policy

This is a subtle but important detail. By default, `osyncstream` sets `emit_on_sync(true)` on its underlying `syncbuf`, which means that calling `std::flush` or `std::endl` through the `osyncstream` triggers an atomic `emit()`. You can turn this off if you want more manual control. The code below demonstrates both behaviors.

```cpp
// Compile: g++ -std=c++20 -pthread q3_flush_policy.cpp -o q3
#include <iostream>
#include <syncstream>
#include <thread>
#include <sstream>

void demonstrate_flush_policies() {
    // --- Default: emit_on_flush is true for osyncstream ---
    {
        std::osyncstream out(std::cout);
        out << "Line A with endl" << std::endl;
        // std::endl calls flush() on the osyncstream.
        // Because emit_on_flush is true (default), this triggers emit().
        // "Line A with endl\n" is now in cout.

        out << "Line B after emit";
        // Will be emitted on destruction
    }

    std::cout << "\n--- syncbuf with emit_on_flush(false) ---\n";

    // --- Manual syncbuf control ---
    {
        std::syncbuf sbuf(std::cout.rdbuf());
        sbuf.set_emit_on_sync(false);  // flush won't trigger emit

        std::ostream out(&sbuf);
        out << "This flush won't emit." << std::flush;
        // Nothing appears in cout yet - still in syncbuf's buffer.

        out << " Still buffered.\n";
        sbuf.emit();  // NOW both messages transfer atomically
    }
}

void worker_with_endl(int id) {
    // Common pattern: use temporary osyncstream with endl
    // endl triggers both '\n' AND emit(), giving immediate atomic output
    std::osyncstream(std::cout) << "Worker " << id << " done." << std::endl;
}

int main() {
    demonstrate_flush_policies();

    std::cout << "\n--- Workers with endl ---\n";
    std::jthread t1(worker_with_endl, 1);
    std::jthread t2(worker_with_endl, 2);
    std::jthread t3(worker_with_endl, 3);

    return 0;
}
```

The one-liner `std::osyncstream(std::cout) << ... << std::endl;` at the bottom of `worker_with_endl` is the idiomatic pattern you will see most often: a temporary `osyncstream` is created, accumulates the entire message, and is then destroyed at the semicolon - guaranteeing atomic output. You can disable the `emit_on_flush` behavior via the underlying `syncbuf` if you want `flush` to be a no-op and prefer to control emission entirely manually.

---

## Notes

- No interleaving is not the same as ordering. `osyncstream` guarantees each emission is contiguous but does not impose a total order on which thread's output appears first. Use external synchronisation (latches, barriers) if ordering matters.
- Performance is excellent: formatting happens into a thread-local buffer with zero contention. Only the final `emit()` acquires the implementation's internal mutex briefly. This is dramatically better than holding a `std::mutex` around the entire `<<` chain.
- You can wrap an `osyncstream` in another `osyncstream`. Each level adds its own buffering. This is rarely useful but not undefined behavior.
- `osyncstream` works with any `ostream` - `ofstream`, `ostringstream`, etc. - not just `cout`. This is useful for multi-threaded logging to a single file.
- `std::osyncstream(std::cout) << "msg\n";` is the most concise pattern. The temporary lives until the end of the full expression, so the entire `<<` chain is captured before the emit.
- `syncbuf` may allocate heap memory for its internal buffer. For extremely hot paths, profile to confirm this isn't a bottleneck.
