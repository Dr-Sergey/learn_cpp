# Understand coroutine-based concurrency vs thread-based concurrency

**Category:** Concurrency and Parallelism  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

C++20 introduced coroutines - functions that can suspend and resume execution. Coroutines enable **cooperative concurrency**: a coroutine voluntarily yields control at `co_await`/`co_yield` points. Threads provide **preemptive concurrency**: the OS scheduler can interrupt any thread at any time.

The reason this distinction matters in practice is cost. Creating a thread means a system call, allocating a full OS stack (often 1-8 MB), and paying for kernel-level context switches on every switch. A coroutine is just a small heap allocation (a few hundred bytes for its frame) and a function call to resume. When you need to manage thousands of concurrent tasks, that difference is enormous.

### Comparison Table

| Aspect | Threads | Coroutines |
| --- | --- | --- |
| Scheduling | Preemptive (OS) | Cooperative (programmer) |
| Stack | Full OS stack (~1-8 MB) | Coroutine frame (~hundreds of bytes) |
| Context switch | Kernel transition (~1-5 us) | Function call (~10-50 ns) |
| Creation cost | Heavy (syscall) | Lightweight (heap alloc) |
| Concurrency model | 1:1 (thread:OS thread) | M:N (many coroutines:few threads) |
| Data races | Common (preemption anywhere) | Rare (yield only at explicit points) |
| Scalability | 100s-1000s threads | 100,000s+ coroutines |
| Best for | CPU-bound work | I/O-bound work |

### Visual Model

The diagram below shows the difference in how threads and coroutines map to OS threads:

```cpp
Thread-based (1:1):                 Coroutine-based (M:N):
┌─────┐ ┌─────┐ ┌─────┐           ┌───┐┌───┐┌───┐┌───┐┌───┐
│ T1  │ │ T2  │ │ T3  │           │C1 ││C2 ││C3 ││C4 ││C5 │ ...1000s
└──┬──┘ └──┬──┘ └──┬──┘           └─┬─┘└─┬─┘└─┬─┘└─┬─┘└─┬─┘
   │       │       │                └──┬──┘    │  ┌─┘    │
   ▼       ▼       ▼                  ▼       ▼  ▼      ▼
┌─────┐ ┌─────┐ ┌─────┐           ┌─────┐ ┌─────┐
│ OS1 │ │ OS2 │ │ OS3 │           │ OS1 │ │ OS2 │  (few threads)
└─────┘ └─────┘ └─────┘           └─────┘ └─────┘
3 OS threads for 3 tasks           2 OS threads for 5+ tasks
```

---

## Self-Assessment

### Q1: Show that coroutines are cooperative (M:N scheduling) while threads are preemptive (1:1)

**Answer:**

This example runs three coroutines on a single thread, round-robin, alongside three threads running the same work preemptively. Notice that the coroutine output is perfectly deterministic - because nothing runs between `co_await` points - while the thread output order varies every time:

```cpp
#include <coroutine>
#include <iostream>
#include <thread>
#include <vector>
#include <string>
#include <chrono>

// === Minimal coroutine infrastructure ===
struct Task {
    struct promise_type {
        Task get_return_object() { return Task{std::coroutine_handle<promise_type>::from_promise(*this)}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~Task() { if (handle) handle.destroy(); }
    Task(Task&& o) : handle(o.handle) { o.handle = nullptr; }
    Task(std::coroutine_handle<promise_type> h) : handle(h) {}
};

// === Cooperative coroutine: explicit yield points ===
Task cooperative_task(const std::string& name) {
    std::cout << "[" << name << "] step 1 on thread "
              << std::this_thread::get_id() << "\n";
    co_await std::suspend_always{}; // <- explicit yield point

    std::cout << "[" << name << "] step 2 on thread "
              << std::this_thread::get_id() << "\n";
    co_await std::suspend_always{}; // <- explicit yield point

    std::cout << "[" << name << "] step 3 on thread "
              << std::this_thread::get_id() << "\n";
}

int main() {
    // === COROUTINES: cooperative, single-threaded interleaving ===
    std::cout << "=== Coroutines (cooperative, M:N) ===\n";
    std::cout << "Main thread: " << std::this_thread::get_id() << "\n";

    Task a = cooperative_task("A");
    Task b = cooperative_task("B");
    Task c = cooperative_task("C");

    // Manual round-robin scheduler on ONE thread
    // 3 coroutines multiplexed onto 1 OS thread (M:N where M=3, N=1)
    std::vector<std::coroutine_handle<>> handles = {
        a.handle, b.handle, c.handle
    };

    bool any_alive = true;
    while (any_alive) {
        any_alive = false;
        for (auto& h : handles) {
            if (h && !h.done()) {
                h.resume(); // coroutine runs until next co_await
                any_alive = true;
            }
        }
    }
    // Output (deterministic order!):
    // [A] step 1 on thread 140234567890
    // [B] step 1 on thread 140234567890  <- same thread
    // [C] step 1 on thread 140234567890  <- same thread
    // [A] step 2 on thread 140234567890
    // [B] step 2 on thread 140234567890
    // [C] step 2 on thread 140234567890
    // [A] step 3 on thread 140234567890
    // [B] step 3 on thread 140234567890
    // [C] step 3 on thread 140234567890

    // === THREADS: preemptive, OS-scheduled ===
    std::cout << "\n=== Threads (preemptive, 1:1) ===\n";
    auto thread_task = [](const std::string& name) {
        for (int i = 1; i <= 3; ++i) {
            std::cout << "[" << name << "] step " << i
                      << " on thread " << std::this_thread::get_id() << "\n";
            // No yield - OS can preempt at ANY point
        }
    };

    std::thread t1(thread_task, "X");
    std::thread t2(thread_task, "Y");
    std::thread t3(thread_task, "Z");
    t1.join(); t2.join(); t3.join();
    // Output (non-deterministic, interleaved randomly):
    // [X] step 1 on thread 140234111111
    // [Z] step 1 on thread 140234333333  <- different threads
    // [Y] step 1 on thread 140234222222
    // [X] step 2 ...  (order varies every run)
}
```

**Explanation:** Coroutines yield only at `co_await` - between yield points, code runs uninterrupted on a single thread. The scheduler is in user code, giving full control over execution order. Threads run on separate OS threads and can be preempted at any instruction - the output order is non-deterministic.

### Q2: Implement 1000 concurrent coroutine tasks and show lower overhead than 1000 threads

**Answer:**

This benchmark measures the real cost difference. Creating 1000 threads means 1000 system calls and somewhere around 8 GB of committed stack space. Coroutines are just heap allocations of a few hundred bytes each:

```cpp
#include <coroutine>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>
#include <atomic>

struct LightTask {
    struct promise_type {
        LightTask get_return_object() { return LightTask{std::coroutine_handle<promise_type>::from_promise(*this)}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
    ~LightTask() { if (handle) handle.destroy(); }
    LightTask(LightTask&& o) : handle(o.handle) { o.handle = nullptr; }
    LightTask(std::coroutine_handle<promise_type> h) : handle(h) {}
};

std::atomic<int> coro_counter{0};
std::atomic<int> thread_counter{0};

LightTask coro_work() {
    coro_counter.fetch_add(1, std::memory_order_relaxed);
    co_await std::suspend_always{};
    coro_counter.fetch_add(1, std::memory_order_relaxed);
    co_await std::suspend_always{};
    coro_counter.fetch_add(1, std::memory_order_relaxed);
}

int main() {
    constexpr int N = 1000;

    // === Benchmark: 1000 coroutines ===
    {
        auto start = std::chrono::steady_clock::now();

        std::vector<LightTask> tasks;
        tasks.reserve(N);
        for (int i = 0; i < N; ++i)
            tasks.push_back(coro_work());

        // Run all coroutines to completion (round-robin on 1 thread)
        bool any = true;
        while (any) {
            any = false;
            for (auto& t : tasks) {
                if (t.handle && !t.handle.done()) {
                    t.handle.resume();
                    any = true;
                }
            }
        }

        auto us = std::chrono::duration_cast<std::chrono::microseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "Coroutines: " << N << " tasks, "
                  << coro_counter.load() << " ops, "
                  << us << " us\n";
    }

    // === Benchmark: 1000 threads ===
    {
        auto start = std::chrono::steady_clock::now();

        std::vector<std::thread> threads;
        threads.reserve(N);
        for (int i = 0; i < N; ++i) {
            threads.emplace_back([] {
                thread_counter.fetch_add(1, std::memory_order_relaxed);
                thread_counter.fetch_add(1, std::memory_order_relaxed);
                thread_counter.fetch_add(1, std::memory_order_relaxed);
            });
        }
        for (auto& t : threads) t.join();

        auto us = std::chrono::duration_cast<std::chrono::microseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "Threads:    " << N << " tasks, "
                  << thread_counter.load() << " ops, "
                  << us << " us\n";
    }

    // Typical output:
    // Coroutines: 1000 tasks, 3000 ops, ~200 us
    // Threads:    1000 tasks, 3000 ops, ~50000 us (50 ms)
    //
    // Coroutines are ~100-250x FASTER to create and schedule
    // because:
    //   - No syscall (pthread_create) per task
    //   - No OS stack allocation (8 MB default) per task
    //   - No kernel context switches
    //   - Coroutine frame is ~100-200 bytes on the heap
}
```

**Explanation:** Creating 1000 threads requires 1000 `pthread_create` syscalls and allocates roughly 8 GB of stack space (8 MB times 1000). Coroutines allocate a small heap frame (~100-200 bytes each) and run cooperatively on the calling thread. The thread version is roughly 100x slower due to OS overhead alone.

### Q3: Explain when thread-per-connection is better vs coroutine-per-connection

**Answer:**

The decision comes down to what your tasks spend most of their time doing. If they are mostly waiting (for I/O, timers, network), coroutines let you fill that wait time with other work at near-zero overhead. If they are mostly computing, you need real OS threads to exploit multiple CPU cores - a coroutine scheduler on one thread cannot parallelize compute.

```cpp
When to use THREADS (1:1):
─────────────────────────
CPU-bound work (computation, encoding, physics)
  - Each thread fully utilizes a core; no benefit to yielding
Simple concurrency needs (< 100 connections)
  - Thread overhead is negligible at small scale
Blocking I/O with no async alternative
  - e.g., legacy database drivers that block
Real-time requirements
  - OS thread priorities and scheduling guarantees

When to use COROUTINES (M:N):
─────────────────────────────
I/O-bound work (network servers, file I/O)
  - Spend most time waiting; coroutines yield during waits
High concurrency (10,000+ connections)
  - Thread-per-connection exhausts OS resources
Event-driven architectures (chat servers, game loops)
  - Natural co_await at each event boundary
Pipeline/streaming processing
  - co_yield produces values lazily

Hybrid approach (common in production):
───────────────────────────────────────
Thread pool (N = CPU cores) + coroutines per thread
┌──────────────────────────────────────────┐
│   Thread 1          Thread 2             │
│  ┌─────────────┐  ┌─────────────┐       │
│  │ coro1 coro2 │  │ coro3 coro4 │       │
│  │ coro5 coro6 │  │ coro7 coro8 │ ...   │
│  └─────────────┘  └─────────────┘       │
│  Each thread runs 100s of coroutines     │
│  N threads = N CPU cores                 │
└──────────────────────────────────────────┘
```

**Concrete example:**

| Scenario | Best choice | Why |
| --- | --- | --- |
| HTTP server, 50K connections | Coroutines | 50K threads = ~400 GB stack space; impossible |
| Image encoding pipeline | Threads | CPU-bound; coroutines add overhead with no benefit |
| Game server, 1000 players | Hybrid | I/O (networking) via coroutines, physics via threads |
| CLI tool, 5 parallel downloads | Threads | Simple; overhead doesn't matter at this scale |
| Database proxy, 100K queries/s | Coroutines | I/O-bound, high connection count |

**Key insight:** Coroutines replace threads for *waiting*, not for *computing*. When a task is mostly waiting (I/O, timers, network), coroutines save enormous overhead. When a task is mostly computing, you need real OS threads to use CPU cores.

---

## Notes

- **C++20 coroutines are stackless:** they store only local variables in a heap-allocated frame, not a full call stack. This means coroutines cannot suspend from nested function calls (only from the coroutine body itself).
- **No built-in scheduler:** C++20 provides the coroutine mechanism but not a scheduler. Libraries like `cppcoro`, `libunifex`, or `asio` provide executors.
- **`co_await` is the yield point:** Code between `co_await` calls runs uninterrupted. No data races are possible within that region (on a single-threaded executor).
- **Thread pool + coroutines** is the production pattern: use `std::thread::hardware_concurrency()` threads, each running a coroutine scheduler.
- Compile with `-std=c++20 -O2 -pthread -fcoroutines` (GCC) or `-std=c++20` (Clang 14+/MSVC).
