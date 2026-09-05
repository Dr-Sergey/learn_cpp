# Use -fsanitize=thread (ThreadSanitizer) to detect data races systematically

**Category:** Tooling and Debugging  
**Standard:** C++11  
**Reference:** <https://clang.llvm.org/docs/ThreadSanitizer.html>  

---

## Topic Overview

ThreadSanitizer (TSan) detects **data races** at runtime - situations where two threads access the same memory location concurrently, at least one of those accesses is a write, and there is no synchronization between them. Data races are undefined behavior in C++, and the bugs they produce are among the hardest to diagnose: the program might produce wrong results silently, crash unpredictably, or work perfectly in debug mode and fail only under load.

TSan makes these races loud and precise. Instead of a mysterious crash you cannot reproduce, you get a report that names both threads, both stack traces, and the exact memory location.

```cpp
Compile:  clang++ -fsanitize=thread -g -O1 prog.cpp -lpthread
Run:      ./a.out
```

| Feature | TSan |
| --- | --- |
| Overhead | 5-15x slowdown, 5-10x memory |
| False positives | Very rare |
| Platforms | Linux (Clang/GCC), macOS (Clang) |
| Combinable? | NOT with ASAN or MSAN |

The overhead is higher than ASAN because TSan needs to track the history of every memory access across all threads to determine whether a given pair of accesses constitutes a race. That history has to be stored somewhere, which explains the 5-10x memory increase.

---

## Self-Assessment

### Q1: Introduce a data race and observe the TSan report

The classic data race is a shared counter incremented by multiple threads without any protection. The `++` operator on a plain `int` compiles to a read-modify-write sequence that is not atomic - two threads can interleave those steps and both write back a stale value, losing increments:

```cpp
#include <iostream>
#include <thread>

int shared_counter = 0;  // Unprotected shared state!

void increment(int n) {
    for (int i = 0; i < n; ++i)
        ++shared_counter;  // DATA RACE: concurrent read-modify-write
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Counter: " << shared_counter << '\n';
    return 0;
}
// Compile: clang++ -fsanitize=thread -g -O1 -o race race.cpp -lpthread
// TSan output (approximate):
//   ==================
//   WARNING: ThreadSanitizer: data race (pid=12345)
//     Write of size 4 at 0x... by thread T2:
//       #0 increment(int) race.cpp:7
//       #1 ...
//     Previous write of size 4 at 0x... by thread T1:
//       #0 increment(int) race.cpp:7
//       #1 ...
//     Location is global 'shared_counter' of size 4 at 0x...
//   ==================
//   Counter: 143782  (wrong! expected 200000)
```

Without TSan, you might run this a hundred times and it usually produces a "wrong" count, but occasionally produces 200000 by chance. That non-determinism makes the bug very hard to pin down. TSan removes the non-determinism - it catches the race on the first run.

### Q2: Fix the race with an atomic

The cleanest fix for a simple counter race is `std::atomic`. The `fetch_add` with `memory_order_relaxed` is sufficient here because the only requirement is that each increment is atomic - we do not need ordering guarantees with respect to other variables:

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<int> shared_counter{0};  // Now thread-safe!

void increment(int n) {
    for (int i = 0; i < n; ++i)
        shared_counter.fetch_add(1, std::memory_order_relaxed);
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Counter: " << shared_counter.load() << '\n';
    return 0;
}
// Compile: clang++ -fsanitize=thread -g -O1 -o norace norace.cpp -lpthread
// TSan output: CLEAN (no warnings!)
// Expected output:
// Counter: 200000
```

TSan running clean is exactly what you want to see. It confirms that the synchronization you added is correct.

If you prefer a mutex-based approach, TSan also understands that and will verify the synchronization is used consistently:

```cpp
#include <mutex>
std::mutex mtx;
int shared_counter = 0;

void increment(int n) {
    for (int i = 0; i < n; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        ++shared_counter;
    }
}
// TSan also verifies mutex-based synchronization is correct.
```

### Q3: TSan's instrumentation and overhead

The reason TSan has higher overhead than ASAN is that it solves a harder problem: it has to detect whether two accesses from different threads are correctly ordered, which requires remembering the history of accesses and tracking the happens-before relationship between threads.

TSan instruments **every memory access** and **every synchronization operation**:

```cpp
For each memory access, TSan:

1. Records: { thread_id, address, size, is_write, timestamp }
2. Stores in shadow memory (4 shadow words per 8 bytes of app memory)
3. Checks against recent accesses from other threads
4. If conflict found (same addr, different thread, one write, no sync):

   => REPORT DATA RACE

Shadow memory layout (per 8 app bytes):
┌────────────────┬────────────────┬────────────────┬────────────────┐
│ Shadow word 0  │ Shadow word 1  │ Shadow word 2  │ Shadow word 3  │
│ thread+clock   │ thread+clock   │ thread+clock   │ thread+clock   │
└────────────────┴────────────────┴────────────────┴────────────────┘
  = 4 x 8 bytes = 32 bytes shadow for 8 bytes app = 4:1 ratio
```

The "clock" stored alongside each thread ID is a logical (vector) clock. TSan uses vector clocks to determine the happens-before relationship between threads - whether an acquire on a mutex in thread B is guaranteed to see all writes that thread A did before its release on the same mutex. If two accesses cannot be ordered by the happens-before relation and one is a write, that is a data race.

**Why 5-15x overhead:**

- **Memory**: 4:1 shadow ratio + vector clock per thread + stack trace storage = ~5-10x
- **CPU**: Every load/store is instrumented with shadow checks = ~5-15x
- **Synchronization**: TSan intercepts all pthread/atomic ops to maintain the happens-before graph

---

## Notes

- TSan and ASAN are **mutually exclusive** - run in separate CI jobs.
- TSan uses `-O1` or `-O2` (not `-O0`; optimizations help reduce instrumentation overhead).
- TSan can detect: data races, lock-order inversions, use-after-destroy of mutexes.
- Use `TSAN_OPTIONS=halt_on_error=1` to fail CI on the first race found.
- TSan works with `std::thread`, `std::mutex`, `std::atomic`, and pthreads.
- Suppress known third-party races with a suppressions file.
