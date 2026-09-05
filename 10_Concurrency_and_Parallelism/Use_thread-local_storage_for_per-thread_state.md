# Use thread-local storage for per-thread state

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/storage_duration>  

---

## Topic Overview

The `thread_local` keyword (C++11) gives each thread its own independent copy of a variable. This eliminates data races without locks and can dramatically improve performance for per-thread caches, counters, RNGs, and allocators. The key insight is that if no thread ever shares the variable, there is nothing to synchronize.

### How `thread_local` Works

Think of `thread_local` as a variable with a separate instance for every thread. Each instance is initialized independently and has its own lifetime tied to the thread that owns it.

```cpp
Main thread:             Thread A:              Thread B:
┌──────────────────┐    ┌──────────────────┐   ┌──────────────────┐
│ thread_local x=0 │    │ thread_local x=0 │   │ thread_local x=0 │
│ x = 10           │    │ x = 20           │   │ x = 30           │
│ print(x) -> 10   │    │ print(x) -> 20   │   │ print(x) -> 30   │
└──────────────────┘    └──────────────────┘   └──────────────────┘

Each thread has its own INDEPENDENT copy - no sharing, no races.
```

### Storage Duration Comparison

| Specifier | Lifetime | Scope | One Per |
| --- | --- | --- | --- |
| `auto` (default) | Block entry -> block exit | Block | Function call |
| `static` | Program start -> program end | File or block | Program |
| `thread_local` | Thread creation -> thread exit | File or block | **Thread** |
| `extern` | Program start -> program end | Cross-TU | Program |

### Declaration Syntax

`thread_local` can appear at namespace scope, as a static class member, or as a local-static inside a function. All three forms give each thread its own copy:

```cpp
// 1. Namespace scope - each thread gets its own copy
thread_local int tls_counter = 0;

// 2. Static member - each thread gets its own copy
class Logger {
    static thread_local std::string buffer;   // declaration
};
thread_local std::string Logger::buffer;       // definition

// 3. Local static - per-thread, initialized on first use per thread
void f() {
    thread_local int call_count = 0;  // each thread has its own
    ++call_count;
}

// 4. Combined with static/extern
static thread_local int x = 0;     // internal linkage, per-thread
extern thread_local int y;         // external linkage, per-thread
```

### Initialization Rules

| Type | When Initialized | Thread-Safe? |
| --- | --- | --- |
| Namespace-scope `thread_local` | Before first use in that thread (or at thread start, implementation-defined) | Yes - each thread initializes its own copy |
| Block-scope `thread_local` | First time control passes through the declaration **in that thread** | Yes - guaranteed by the standard |
| `static` (non-thread_local) | Once, by any thread, before first use | Yes (C++11 guarantees: "magic statics") |

### Key Difference from `static`

This is the most important thing to understand when you first encounter `thread_local`. A `static` variable is shared across all threads and needs a mutex. A `thread_local` variable is private to each thread and needs nothing.

```cpp
static int s_count = 0;          // ONE copy for ALL threads (needs mutex!)
thread_local int tl_count = 0;   // SEPARATE copy per thread (no mutex needed)
```

### Important Notes

- Destructors run when the thread exits (for non-trivial `thread_local` objects).
- On Windows, `thread_local` in DLLs loaded at runtime via `LoadLibrary` may have issues. Prefer static linking or be aware of `__declspec(thread)` limitations.
- Accessing `thread_local` is slightly slower than accessing a regular local variable (~1 extra indirection via TLS segment register on x86), but far faster than acquiring a mutex.
- `thread_local` state persists across tasks in `std::async` if the implementation reuses threads. This can cause subtle bugs - prefer explicit per-task state in that context.

---

## Self-Assessment

### Q1: Declare a `thread_local` RNG and show each thread has its own independently seeded instance

A random number generator is the textbook example for `thread_local`. A global `std::mt19937` shared across threads has a data race (undefined behavior). A mutex-protected global serializes all random number generation, killing parallelism. `thread_local` gives each thread its own independent engine with zero contention.

```cpp
#include <iostream>
#include <thread>
#include <random>
#include <vector>

// Each thread gets its own RNG, seeded with its thread ID + a base seed
thread_local std::mt19937 tl_rng;

void generate_random(int thread_id, int count) {
    // Seed with thread-unique value
    tl_rng.seed(42 + thread_id);

    std::uniform_int_distribution<int> dist(1, 100);

    std::cout << "Thread " << thread_id << ": ";
    for (int i = 0; i < count; ++i) {
        std::cout << dist(tl_rng) << " ";
    }
    std::cout << "\n";
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(generate_random, i, 5);
    }
    for (auto& t : threads) t.join();

    // Each thread produces a DIFFERENT sequence because they have
    // independent RNG instances with different seeds.
    // Running again produces the SAME sequences (deterministic per seed).
}
// Example output (order may vary between threads):
//   Thread 0: 16 92 35 67 44
//   Thread 1: 73 29 51 88 12
//   Thread 2: 40 63 17 95 26
//   Thread 3: 57 81 39 74 53
```

**Why `thread_local` is Perfect for RNGs:**

| Approach | Thread-Safe? | Performance | Deterministic? |
| --- | --- | --- | --- |
| Global RNG + mutex | Yes (slow) | Bad - serialized access | No (depends on scheduling) |
| `thread_local` RNG | Yes (no lock) | Fast - no contention | Yes per thread (if seeded deterministically) |
| Local RNG per call | Yes | Bad - construction overhead each call | Depends on seeding |

---

### Q2: Explain the initialization order of `thread_local` variables vs `static` variables

Initialization order is one of the trickier corners of C++. The good news is that `thread_local` follows the same rules as `static`, just scoped to each thread rather than the whole program. The bad news is that the across-translation-unit ordering problem (the "static initialization order fiasco") applies here too.

```cpp
Program startup:

  1. Zero-initialization of all static/thread_local with static storage
  2. Constant-initialization of constexpr-capable static/thread_local
  3. Dynamic initialization of namespace-scope static variables

     (order: within one TU = declaration order; across TUs = UNDEFINED)

Thread creation:

  4. Zero-initialization of thread_local variables for new thread
  5. Constant-initialization of constexpr-capable thread_local
  6. Dynamic initialization of thread_local variables

     (order within TU = declaration order; across TUs = UNDEFINED)
     (triggered: before first ODR-use in that thread, or at thread start)
```

The code below demonstrates the key behavioral difference between `static` and `thread_local` initialization - the static initializer runs once, while the thread_local initializer runs separately for each thread:

```cpp
#include <iostream>
#include <thread>

// Static: initialized once (before main, or on first use if block-scope)
static int s_init_order = [] {
    std::cout << "static init\n";
    return 1;
}();

// thread_local: initialized once PER THREAD
thread_local int tl_init_order = [] {
    std::cout << "thread_local init on thread "
              << std::this_thread::get_id() << "\n";
    return 2;
}();

void use_tls() {
    // Accessing tl_init_order triggers initialization for this thread
    std::cout << "tl value = " << tl_init_order << "\n";
}

int main() {
    std::cout << "--- main thread ---\n";
    use_tls();  // triggers TLS init for main thread

    std::cout << "--- spawning thread ---\n";
    std::thread t(use_tls);  // triggers TLS init for new thread
    t.join();

    std::cout << "--- spawning another thread ---\n";
    std::thread t2(use_tls);  // triggers ANOTHER init for this new thread
    t2.join();
}
// Expected output:
//   static init
//   --- main thread ---
//   thread_local init on thread <main_id>
//   tl value = 2
//   --- spawning thread ---
//   thread_local init on thread <thread1_id>
//   tl value = 2
//   --- spawning another thread ---
//   thread_local init on thread <thread2_id>
//   tl value = 2
```

**Critical Differences:**

| Aspect | `static` | `thread_local` |
| --- | --- | --- |
| How many times initialized? | **Once** for the entire program | **Once per thread** |
| Initialization trigger | Before first use (block-scope) or at startup | Before first use **in that thread** |
| Cross-TU order | Undefined ("static initialization order fiasco") | Undefined (same problem, per-thread) |
| Destructor call | At program exit (`atexit` order) | At **thread exit** |
| Block-scope thread-safety | C++11 guarantees single init ("magic statics") | Each thread initializes independently - no races |

**The "Static Init Order Fiasco" applies to `thread_local` too.** If two `thread_local` variables in different translation units depend on each other, you can't guarantee which is initialized first. The Construct-on-First-Use idiom solves this:

```cpp
// file_a.cpp
thread_local A a;  // may use b - but b might not be initialized yet

// file_b.cpp
thread_local B b;  // may use a - but a might not be initialized yet

// Solution: use the Construct-on-First-Use idiom
thread_local A& get_a() {
    thread_local A instance;
    return instance;  // initialized on first call per thread
}
```

---

### Q3: Show a performance win from replacing a mutex-protected global with `thread_local` accumulators

This is one of the most dramatic performance improvements you can make to certain classes of concurrent code. Instead of every thread fighting over one global counter with a mutex, each thread accumulates into its own private variable and only does a single atomic merge at the end.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <chrono>
#include <atomic>
#include <numeric>

static const int NUM_THREADS = 8;
static const int OPS_PER_THREAD = 10'000'000;

// ========== Approach 1: mutex-protected global ==========
std::mutex mtx;
long long global_sum = 0;

void mutex_accumulate() {
    for (int i = 0; i < OPS_PER_THREAD; ++i) {
        std::lock_guard lock(mtx);
        global_sum += i;
    }
}

// ========== Approach 2: thread_local + final merge ==========
thread_local long long tl_sum = 0;
std::atomic<long long> final_sum{0};

void tls_accumulate() {
    tl_sum = 0;  // reset for this thread
    for (int i = 0; i < OPS_PER_THREAD; ++i) {
        tl_sum += i;  // no lock needed - private to this thread!
    }
    // One atomic add at the end - 1 sync instead of 10M
    final_sum.fetch_add(tl_sum, std::memory_order_relaxed);
}

template <typename Fn>
long long benchmark(Fn fn, const char* label) {
    std::vector<std::thread> threads;
    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < NUM_THREADS; ++i)
        threads.emplace_back(fn);
    for (auto& t : threads) t.join();

    auto end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    std::cout << label << ": " << ms << " ms\n";
    return ms;
}

int main() {
    global_sum = 0;
    final_sum = 0;

    auto t_mutex = benchmark(mutex_accumulate, "mutex global    ");
    auto t_tls   = benchmark(tls_accumulate,   "thread_local    ");

    std::cout << "Mutex sum:  " << global_sum << "\n";
    std::cout << "TLS sum:    " << final_sum.load() << "\n";

    if (t_tls > 0) {
        std::cout << "Speedup: " << (double)t_mutex / t_tls << "x\n";
    }
}
// Typical output (8 cores):
//   mutex global    : 3200 ms
//   thread_local    : 45 ms
//   Mutex sum:  399999960000000
//   TLS sum:    399999960000000
//   Speedup: ~71x
//
// The mutex version serializes ALL 80M increments.
// The TLS version runs 8 threads in parallel with zero contention,
// then does ONE atomic add per thread at the end.
```

The diagram below shows why the speedup is so dramatic. The mutex version does 80 million lock/unlock operations in total, and they are all serialized. The TLS version does 80 million additions with zero synchronization, then 8 atomic merges at the end.

```cpp
Mutex approach:            thread_local approach:
┌────────────────────┐    ┌─────────────────────────────────┐
│ T1: lock add unlock│    │ T1: add add add ... (10M local) │
│ T2: lock add unlock│    │ T2: add add add ... (10M local) │
│ T3: lock add unlock│    │ T3: add add add ... (10M local) │
│ ...serial...       │    │ ...all parallel, zero locking...│
│                    │    │                                 │
│ 80M lock/unlock    │    │ 8 atomic adds at the end        │
│ operations         │    │                                 │
└────────────────────┘    └─────────────────────────────────┘
  Contention: 100%           Contention: ~0%
```

**When to Use This Pattern:**

- Accumulating statistics, counters, histograms
- Per-thread memory allocators (arena allocators)
- Per-thread caches (DNS, parsing, formatting buffers)
- Logger string buffers (avoid allocation per log call)

---

## Notes

- `thread_local` with `std::async`: if `std::async` reuses threads (thread pool), `thread_local` state persists from previous tasks - this can cause subtle bugs. Prefer explicit per-task state.
- Destruction order: `thread_local` objects are destroyed in reverse construction order when the thread exits. For the main thread, this happens during `exit()` (after `main()` returns).
- Cost: on x86-64, `thread_local` access uses `fs:` (Linux) or `gs:` (Windows) segment register - typically ~1-2 extra instructions vs a plain local.
- `thread_local` in shared libraries: on Linux, this works via `__tls_get_addr()` for dynamically loaded libraries - slightly slower than TLS in the main executable.
- Compile flags: `-pthread` is required on GCC/Clang for `thread_local` with non-trivial destructors.
- `errno` is `thread_local`: the C library's `errno` has been per-thread since the early days of pthreads.
