# Identify Common UB in Multi-Threaded Code Beyond Data Races

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17, C++20, C++23  
**Reference:** [cppreference - Memory model](https://en.cppreference.com/w/cpp/language/memory_model)  

---

## Topic Overview

Data races are the most well-known form of concurrency UB, but they are far from the only one. The C++ memory model defines several additional categories of undefined behavior that arise in multi-threaded programs, even when you are using mutexes and atomics correctly. Many developers fall into these traps because their code looks structured and careful, yet the UB still lurks in the lifecycle of synchronization primitives themselves.

| UB Category | Description | Detection |
| --- | --- | --- |
| **Data race** | Concurrent unsynchronized access to non-atomic, at least one write | TSan |
| **Destroying locked mutex** | `std::mutex::~mutex()` while locked | Manual review / TSan |
| **Unlocking unowned mutex** | Thread unlocks `std::mutex` it doesn't own | Debug builds assert |
| **Deadlock via recursive lock** | Locking non-recursive mutex twice on same thread | Stalls, TSan can catch |
| **Signal handler + non-atomic** | Accessing non-atomic/non-`volatile sig_atomic_t` in signal handler | Static analysis |
| **`memory_order` misuse** | Using `memory_order_acquire` on a store, etc. | UBSan (partial) |
| **Calling non-thread-safe from signal** | `malloc`, `printf` in signal context | POSIX docs |
| **Destroying thread w/o join/detach** | `std::thread::~thread()` while joinable | Calls `std::terminate` |

Beyond data races, the most dangerous UB patterns involve **destroying synchronization primitives** while they are in use, **memory order misuse** that breaks the happens-before relationship, and **signal-unsafe function calls** in asynchronous contexts.

The diagrams below capture the valid and invalid lifecycle sequences for threads and mutexes. The invalid paths are where UB lives:

```cpp
Thread Lifecycle UB:

Thread Created --> Running --> Joinable --> join()/detach()  (OK)
                                   |
                                   +--> ~thread() while joinable  (std::terminate)

Mutex Lifecycle UB:

Mutex Created --> lock() --> unlock() --> ~mutex()  (OK)
                    |                         |
                    +-- lock() again (same    +-- ~mutex() while
                        thread, non-recursive)    locked = UB
                        = UB
```

These UB scenarios are particularly insidious because they may appear to work correctly for months or years, manifesting only under specific timing conditions or on specific hardware.

---

## Self-Assessment

### Q1: Identify all concurrency UB beyond data races

Each commented block below demonstrates a different kind of concurrency UB. Notice that none of them involve an obvious data race - these are the bugs you find after you have already been careful with your locks.

```cpp
#include <atomic>
#include <cstdio>
#include <mutex>
#include <thread>
#include <vector>

std::mutex mtx;

// --- UB 1: Destroying a locked mutex ---
void destroy_locked_mutex() {
    auto* m = new std::mutex();
    m->lock();
    // delete m;  // UB: destroying a locked mutex
    // The standard says: "The behavior is undefined if the mutex
    // is owned by any thread [...] when the destructor is called."
    m->unlock();
    delete m;  // OK: unlocked before destruction
}

// --- UB 2: Unlocking a mutex you don't own ---
void unlock_not_owned() {
    std::mutex m;
    // m.unlock();  // UB: calling unlock() without owning the mutex
    // Some implementations assert in debug mode, but it's UB per standard.
}

// --- UB 3: Recursive lock on non-recursive mutex ---
void recursive_lock_ub() {
    std::mutex m;
    m.lock();
    // m.lock();  // UB: second lock on non-recursive mutex by same thread
    // In practice: deadlock on most implementations, but UB per standard.
    m.unlock();
    // Fix: use std::recursive_mutex if you need recursive locking.
}

// --- UB 4: Destroying a joinable thread ---
void destroy_joinable_thread() {
    // std::thread t([]{ /* work */ });
    // // ~thread() while joinable -> std::terminate (not UB per se,
    // // but effectively crashes the program)
    // // Fix: t.join() or t.detach() before destruction.

    std::thread t([]{ std::printf("Worker done\n"); });
    t.join();  // OK: joined before destruction
}

// --- UB 5: Condition variable notify/wait on destroyed cv ---
void cv_lifetime_ub() {
    auto* cv = new std::condition_variable();
    // If a thread is waiting on *cv when we delete it -> UB
    // Always ensure no threads are waiting before destroying a cv.
    delete cv;  // OK only if no threads are waiting
}

// --- UB 6: Calling thread functions after std::quick_exit ---
// std::quick_exit does NOT call thread-local destructors or join threads.
// Any threads still running access destroyed resources -> UB.

int main() {
    destroy_locked_mutex();
    unlock_not_owned();
    recursive_lock_ub();
    destroy_joinable_thread();
    cv_lifetime_ub();
}
```

**Answer:** Six UB scenarios are shown. Key: destroying locked mutexes and condition variables with waiters are the most common real-world bugs. `std::jthread` (C++20) fixes the joinable-thread-destruction problem by auto-joining in its destructor.

---

### Q2: Demonstrate memory_order misuse and signal-handler UB

Memory ordering and signal handler safety are two topics that look simple but have sharp edges. The `consumer_broken` function below is especially important to understand: on x86 it often produces the right answer by accident because x86 has a very strong memory model, but on ARM it can fail.

```cpp
#include <atomic>
#include <csignal>
#include <cstdio>
#include <cstdlib>
#include <thread>

// --- memory_order misuse ---

std::atomic<int> data{0};
std::atomic<bool> ready{false};

void producer() {
    data.store(42, std::memory_order_relaxed);

    // CORRECT: release pairs with acquire
    ready.store(true, std::memory_order_release);
}

void consumer_correct() {
    while (!ready.load(std::memory_order_acquire))
        ;  // spin
    // Guaranteed to see data == 42 due to release-acquire pairing
    std::printf("Correct: data = %d\n", data.load(std::memory_order_relaxed));
}

void consumer_broken() {
    while (!ready.load(std::memory_order_relaxed))
        ;  // spin with relaxed - no acquire!
    // MAY NOT see data == 42!
    // Relaxed provides no ordering guarantee relative to other stores.
    // On x86 this often "works" (strong memory model), but on ARM it breaks.
    std::printf("Broken:  data = %d\n", data.load(std::memory_order_relaxed));
}

// --- Signal handler UB ---

// Only these are safe in signal handlers:
// - Reads/writes to volatile sig_atomic_t
// - Reads/writes to lock-free atomics
// - A very limited set of functions (signal, abort, _Exit, quick_exit)

volatile std::sig_atomic_t signal_flag = 0;
std::atomic<int> atomic_counter{0};

// SAFE signal handler
void safe_handler(int sig) {
    signal_flag = 1;                    // OK: volatile sig_atomic_t
    atomic_counter.store(1,             // OK: lock-free atomic
        std::memory_order_relaxed);
}

// UNSAFE signal handler
void unsafe_handler(int sig) {
    // std::printf("Signal %d caught\n", sig);   // UB: printf is not
    //                                           // async-signal-safe
    // std::string s = "hello";                  // UB: allocates memory
    //                                           // (malloc is not safe)

    // Even on POSIX, the list of async-signal-safe functions is limited.
    // See: man 7 signal-safety
    signal_flag = 1;  // This is the safe part
}

/*
Signal-Safety Table:

| Operation                  | Safe in Signal Handler? |
| --- | --- |
| volatile sig_atomic_t R/W  | Yes                     |
| Lock-free atomic R/W       | Yes (C++17+)            |
| printf, fprintf            | No                      |
| malloc, free, new, delete  | No                      |
| std::mutex::lock           | No                      |
| write() (POSIX)            | Yes                     |
| _Exit(), abort()           | Yes                     |
*/

void signal_demo() {
    std::signal(SIGINT, safe_handler);

    std::printf("Signal handler installed. Send SIGINT to test.\n");
    // In production, prefer signalfd (Linux) or dedicated signal thread.
}

int main() {
    // Memory order demo
    std::thread t1(producer);
    std::thread t2(consumer_correct);
    t1.join();
    t2.join();

    signal_demo();
}
```

The key mental model for memory orders: `release` on a store and `acquire` on a load form a synchronization pair. Everything the producer wrote before the `release` store is visible to the consumer after it observes the `acquire` load. Without `acquire` on the consumer side, the compiler and CPU are free to reorder the `data` read before the `ready` check - there is no guarantee at all.

**Answer:** Memory order misuse is technically not UB by itself (the operations are valid), but using `relaxed` where `acquire`/`release` is needed leads to data races on the non-atomic data, which IS UB. Signal handlers that call non-async-signal-safe functions (like `printf` or `malloc`) are UB. Only `volatile sig_atomic_t` and lock-free atomics are safe.

---

### Q3: Build a thread-safe component that avoids all concurrency UB

Here is a complete, UB-free bounded queue. Reading through the design decisions is as valuable as reading the code - every comment calls out which UB scenario is being prevented.

```cpp
#include <atomic>
#include <condition_variable>
#include <cstdio>
#include <functional>
#include <mutex>
#include <optional>
#include <queue>
#include <thread>
#include <vector>

// Thread-safe bounded queue that avoids all concurrency UB.
template <typename T, std::size_t MaxSize = 1024>
class SafeQueue {
public:
    // No UB: default construction of members is safe
    SafeQueue() = default;

    // Destructor: we MUST ensure no threads are waiting.
    // The design guarantees this by calling shutdown() + joining workers.
    ~SafeQueue() {
        shutdown();
    }

    // Non-copyable, non-movable (mutex/cv cannot be moved while in use)
    SafeQueue(const SafeQueue&) = delete;
    SafeQueue& operator=(const SafeQueue&) = delete;

    bool push(T value) {
        std::unique_lock lock(mutex_);
        // Wait until there's space or shutdown
        cv_not_full_.wait(lock, [this] {
            return queue_.size() < MaxSize || shutdown_;
        });
        if (shutdown_) return false;

        queue_.push(std::move(value));
        cv_not_empty_.notify_one();
        return true;
    }

    std::optional<T> pop() {
        std::unique_lock lock(mutex_);
        cv_not_empty_.wait(lock, [this] {
            return !queue_.empty() || shutdown_;
        });
        if (queue_.empty()) return std::nullopt;

        T value = std::move(queue_.front());
        queue_.pop();
        cv_not_full_.notify_one();
        return value;
    }

    void shutdown() {
        {
            std::lock_guard lock(mutex_);
            shutdown_ = true;
        }
        // Wake ALL waiting threads so they can check shutdown_ and exit.
        // If we don't do this, threads may wait forever on the cv,
        // and destroying the cv with waiters is UB.
        cv_not_empty_.notify_all();
        cv_not_full_.notify_all();
    }

private:
    std::queue<T>           queue_;
    std::mutex              mutex_;        // destroyed after all threads exit
    std::condition_variable cv_not_empty_;
    std::condition_variable cv_not_full_;
    bool                    shutdown_ = false;
};

void worker(SafeQueue<int>& q, int id) {
    while (auto item = q.pop()) {
        std::printf("Worker %d processed: %d\n", id, *item);
    }
    std::printf("Worker %d shutting down\n", id);
}

int main() {
    SafeQueue<int> queue;
    std::vector<std::thread> workers;

    // Start workers
    for (int i = 0; i < 3; ++i) {
        workers.emplace_back(worker, std::ref(queue), i);
    }

    // Produce items
    for (int i = 0; i < 10; ++i) {
        queue.push(i);
    }

    // Shutdown: wakes all waiters, preventing cv destruction UB
    queue.shutdown();

    // Join all threads: prevents joinable-thread-destruction terminate
    for (auto& w : workers) {
        w.join();
    }
    // Now ~SafeQueue() is safe: no waiters, mutex unlocked
}
```

The `shutdown()` method is the linchpin of the whole design. Without it, workers would block on `pop()` forever when work runs out, and when `SafeQueue` goes out of scope its destructor would call `shutdown()` - but by then the condition variables might have waiters, which is UB. By calling `shutdown()` explicitly and joining all threads before destruction, you close the gap.

**Answer:** The `SafeQueue` avoids all concurrency UB: `shutdown()` wakes all waiters before destruction (preventing cv-with-waiters UB), workers are joined before the queue is destroyed (preventing joinable-thread terminate), and all shared state is protected by the mutex (preventing data races). The destruction order guarantee (members destroyed in reverse declaration order) ensures the cv is destroyed before the mutex.

---

## Notes

- **`std::jthread` (C++20)** fixes the joinable-thread-destruction problem by requesting stop and joining in its destructor.
- **`std::stop_token`** (C++20) provides cooperative cancellation, replacing ad-hoc `shutdown_` booleans.
- ThreadSanitizer (TSan) detects data races, lock-order inversions, and some mutex misuse - but NOT memory-order bugs on x86 (which has a strong memory model).
- Testing on ARM or using `TSAN_OPTIONS=force_seq_cst_atomics=1` can expose relaxed-ordering bugs.
- Destroying synchronization primitives while in use is **statically detectable** with lifetime analysis tools (Clang's `-Wthread-safety`).
- In practice, the safest pattern is RAII for synchronization lifetime: the owning scope joins all threads before destroying shared state.
