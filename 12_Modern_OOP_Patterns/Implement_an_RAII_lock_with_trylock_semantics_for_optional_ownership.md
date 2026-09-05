# Implement an RAII lock with trylock semantics for optional ownership

**Category:** Modern OOP Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/thread/unique_lock>  

---

## Topic Overview

A standard `lock_guard` blocks until the mutex is acquired. In many scenarios you need **non-blocking** or **timed** lock acquisition - polling a shared resource, avoiding priority inversion, or preventing deadlocks. The standard library provides several mechanisms:

| Mechanism | Header | Blocking? | RAII? |
| --- | --- | --- | --- |
| `lock_guard<M>` | `<mutex>` | Yes - blocks | Yes |
| `unique_lock<M>(m, try_to_lock)` | `<mutex>` | No - returns immediately | Yes |
| `unique_lock<M>(m, defer_lock)` | `<mutex>` | Deferred - lock later | Yes |
| `m.try_lock()` | `<mutex>` | No | No - manual unlock |
| `unique_lock<M>::try_lock_for(dur)` | `<mutex>` | Timed | Yes |
| `scoped_lock<M1, M2, ...>` | `<mutex>` | Yes - deadlock-free | Yes |

The important distinction is that `try_lock`-based locking gives you a guard that *might not own the mutex*. The guard tracks whether it owns the lock and only unlocks on destruction if it does. That's optional ownership.

### RAII Try-Lock Flow

Here's what happens in both cases - the key insight is that the destructor behaves differently depending on whether the lock was acquired:

```cpp
try_to_lock                          try_lock_for(100ms)
    |                                        |
    v                                        v
mutex.try_lock()                   mutex.try_lock_for(100ms)
    |                                        |
 +--+--+                                +---+---+
 |     |                                |       |
true  false                           true    false
 |     |                                |       |
 v     v                                v       v
owns  !owns                           owns   !owns
lock  lock                            lock    lock
 |     |                                |       |
 +--+--+                                +---+---+
    v                                        v
~unique_lock()                       ~unique_lock()
unlock if owns_lock()                unlock if owns_lock()
```

---

## Self-Assessment

### Q1: Write a TryLockGuard that attempts to acquire a mutex and exposes a bool for success

The design here mirrors `unique_lock` but is simpler - the constructor does exactly one `try_lock` attempt, stores the result, and the destructor only unlocks if that attempt succeeded.

**Solution - Custom TryLockGuard:**

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

// Custom RAII guard with try-lock semantics
template <typename Mutex>
class TryLockGuard {
    Mutex& mutex_;
    bool   locked_;

public:
    explicit TryLockGuard(Mutex& m) : mutex_(m), locked_(m.try_lock()) {}

    ~TryLockGuard() {
        if (locked_) mutex_.unlock();
    }

    // Non-copyable, non-movable
    TryLockGuard(const TryLockGuard&) = delete;
    TryLockGuard& operator=(const TryLockGuard&) = delete;

    // Query ownership
    [[nodiscard]] bool owns_lock() const noexcept { return locked_; }
    explicit operator bool() const noexcept { return locked_; }
};

std::mutex mtx;
int shared_counter = 0;

void worker(int id) {
    for (int i = 0; i < 5; ++i) {
        TryLockGuard<std::mutex> guard(mtx);
        if (guard) {
            ++shared_counter;
            std::cout << "Thread " << id << " acquired lock, counter="
                      << shared_counter << "\n";
            // Lock released automatically at end of scope
        } else {
            std::cout << "Thread " << id << " could NOT acquire lock, skipping\n";
        }
    }
}

int main() {
    std::vector<std::jthread> threads;
    for (int i = 0; i < 3; ++i)
        threads.emplace_back(worker, i);

    // jthreads auto-join
    threads.clear();
    std::cout << "Final counter: " << shared_counter << "\n";
}
// Example output (non-deterministic):
//   Thread 0 acquired lock, counter=1
//   Thread 0 acquired lock, counter=2
//   Thread 1 could NOT acquire lock, skipping
//   Thread 2 acquired lock, counter=3
//   ...
//   Final counter: <some value <= 15>
```

Notice that `explicit operator bool` lets you write the idiomatic `if (guard)` check. The `explicit` means it won't silently convert to bool in unintended contexts, but it works cleanly in `if` conditions.

**Key design decisions:**

- Constructor calls `try_lock()` - never blocks
- Destructor calls `unlock()` only if `locked_ == true`
- Non-copyable/non-movable prevents double-unlock bugs
- `explicit operator bool` for idiomatic `if (guard)` usage

---

### Q2: Use `std::unique_lock` with `std::try_to_lock` to implement non-blocking lock acquisition

Rather than rolling your own guard, `std::unique_lock` handles all of this with tag dispatch. Pass `std::try_to_lock` at construction and you get exactly the same optional-ownership semantics.

**Solution:**

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <chrono>
#include <vector>

std::mutex data_mutex;
int shared_data = 0;

void try_update(int id) {
    // try_to_lock: attempt ONCE, don't block
    std::unique_lock<std::mutex> lock(data_mutex, std::try_to_lock);

    if (lock.owns_lock()) {
        // We own the lock - safe to modify shared state
        ++shared_data;
        std::cout << "[Thread " << id << "] Updated data to " << shared_data << "\n";
        // Simulate long critical section
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    } else {
        // Lock not acquired - do fallback work
        std::cout << "[Thread " << id << "] Lock busy, doing other work...\n";
    }
    // unique_lock destructor: unlocks ONLY if owns_lock() == true
}

void polling_pattern(int id) {
    // Retry pattern: try until success
    std::unique_lock<std::mutex> lock(data_mutex, std::defer_lock);

    int attempts = 0;
    while (!lock.try_lock()) {
        ++attempts;
        // Do useful work while waiting (non-blocking)
        std::cout << "[Thread " << id << "] Attempt " << attempts << " failed\n";
        std::this_thread::yield();  // give other threads a chance
    }

    ++shared_data;
    std::cout << "[Thread " << id << "] Got lock after " << attempts
              << " retries, data=" << shared_data << "\n";
}

int main() {
    std::cout << "=== try_to_lock pattern ===\n";
    {
        std::vector<std::jthread> threads;
        for (int i = 0; i < 4; ++i)
            threads.emplace_back(try_update, i);
    }

    std::cout << "\n=== Polling with defer_lock ===\n";
    shared_data = 0;
    {
        std::vector<std::jthread> threads;
        for (int i = 0; i < 3; ++i)
            threads.emplace_back(polling_pattern, i);
    }

    std::cout << "\nFinal data: " << shared_data << "\n";
}
```

**`unique_lock` tag dispatch overview:**

| Tag | Effect | `owns_lock()` |
| --- | --- | --- |
| *(none)* | Blocks until acquired | `true` |
| `std::try_to_lock` | Single non-blocking attempt | `true` or `false` |
| `std::defer_lock` | Don't lock yet | `false` (lock later) |
| `std::adopt_lock` | Already locked externally | `true` |

---

### Q3: Show the timed variant using `try_lock_for` and its use in deadlock avoidance strategies

Classic deadlock happens when two threads each hold one lock and are waiting for the other. Timed locking breaks out of that situation by giving up after a timeout, releasing the lock already held, and retrying from scratch.

**Solution - Timed Locking for Deadlock Avoidance:**

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <chrono>
using namespace std::chrono_literals;

// timed_mutex supports try_lock_for / try_lock_until
std::timed_mutex mutex_a;
std::timed_mutex mutex_b;

// Deadlock-prone WITHOUT timed locking:
//   Thread 1: lock(A) -> lock(B)
//   Thread 2: lock(B) -> lock(A)  <- classic deadlock!

// Deadlock avoidance with try_lock_for:
void safe_transfer(int id, std::timed_mutex& first, std::timed_mutex& second) {
    while (true) {
        // Step 1: Lock first mutex with timeout
        std::unique_lock lock1(first, std::defer_lock);
        if (!lock1.try_lock_for(100ms)) {
            std::cout << "[" << id << "] Timeout on first mutex, retrying...\n";
            continue;
        }

        // Step 2: Try second mutex with timeout
        std::unique_lock lock2(second, std::defer_lock);
        if (!lock2.try_lock_for(100ms)) {
            // Release first lock and retry both
            lock1.unlock();
            std::cout << "[" << id << "] Timeout on second mutex, backing off...\n";
            std::this_thread::sleep_for(10ms);  // back-off to reduce contention
            continue;
        }

        // Both locks acquired - perform operation
        std::cout << "[" << id << "] Both locks acquired, performing transfer\n";
        break;
    }
    // Both unique_locks go out of scope -> automatic unlock
}

void thread1_work() { safe_transfer(1, mutex_a, mutex_b); }
void thread2_work() { safe_transfer(2, mutex_b, mutex_a); }  // opposite order!

int main() {
    std::cout << "=== Deadlock avoidance with try_lock_for ===\n";
    std::jthread t1(thread1_work);
    std::jthread t2(thread2_work);
}
// Expected: both threads eventually succeed (no deadlock)
// Sample output:
//   [1] Both locks acquired, performing transfer
//   [2] Timeout on second mutex, backing off...
//   [2] Both locks acquired, performing transfer
```

The `lock1.unlock()` call before backing off is critical. If you hold the first lock while giving up on the second, you're essentially recreating the deadlock condition for the next attempt. Release everything and start fresh.

**Comparison of deadlock avoidance strategies:**

| Strategy | Mechanism | Pros | Cons |
| --- | --- | --- | --- |
| `std::lock(a, b)` | Lock all at once (deadlock-free) | Simple, guaranteed | Blocks indefinitely |
| `std::scoped_lock<A,B>` | Same + RAII | Simplest API | Blocks indefinitely |
| `try_lock_for` + back-off | Timed retry | Livelock-resistant with back-off | More complex code |
| Lock ordering | Always lock in same global order | Zero overhead | Hard to enforce globally |

```cpp
// Preferred: std::scoped_lock when you can lock both at once
void simplest_approach() {
    std::scoped_lock both(mutex_a, mutex_b);  // deadlock-free, RAII
    // ... use both resources ...
}
```

When you can acquire all needed locks at once, `std::scoped_lock` is the cleanest answer. Reach for timed locking only when you need to do real work in the gaps between attempts.

---

## Notes

- **Prefer `std::scoped_lock`** for multi-mutex locking when you can acquire all at once - it uses a deadlock-avoidance algorithm internally.
- **`unique_lock` is heavier** than `lock_guard` (stores a `bool owns_` flag + supports deferred/timed locking). Use `lock_guard` when you don't need the extra features.
- **`try_lock()` is not fair:** On most platforms, repeated try-lock loops can starve other threads. Add back-off (`yield()` or `sleep_for()`) to mitigate.
- **`std::timed_mutex`** has higher overhead than `std::mutex` - use it only when you need `try_lock_for`/`try_lock_until`.
- **`std::shared_timed_mutex`** (C++14) supports `try_lock_shared_for()` for timed reader locks.
- **`std::unique_lock::release()`** transfers ownership *without* unlocking - useful when passing a locked mutex to another function that will manage it.
