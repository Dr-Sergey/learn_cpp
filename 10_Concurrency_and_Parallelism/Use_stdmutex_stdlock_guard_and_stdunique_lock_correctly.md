# Use std::mutex, std::lock_guard, and std::unique_lock correctly

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/mutex>  

---

## Topic Overview

`std::mutex` protects shared data from concurrent access. RAII wrappers `lock_guard` and `unique_lock` ensure the mutex is always unlocked, even when exceptions are thrown.

### Lock Types Comparison

```cpp

┌──────────────────┬──────────────┬───────────────┬───────────────────┐
│                  │ lock_guard   │ unique_lock   │ scoped_lock(C++17)│
├──────────────────┼──────────────┼───────────────┼───────────────────┤
│ Locks on create  │ Yes          │ Configurable  │ Yes               │
│ Manual unlock    │ No           │ Yes           │ No                │
│ Deferred lock    │ No           │ Yes           │ No                │
│ Try-lock         │ No           │ Yes           │ No                │
│ Move             │ No           │ Yes           │ No                │
│ condition_variable│ No          │ Yes           │ No                │
│ Multiple mutexes │ No           │ No (use lock) │ Yes               │
│ Overhead         │ Zero         │ Flag + pointer│ Zero              │
└──────────────────┴──────────────┴───────────────┴───────────────────┘

```

---

## Self-Assessment

### Q1: Show a data race on a shared counter and fix it with std::mutex

**Answer:**

```cpp

#include <mutex>
#include <thread>
#include <vector>
#include <iostream>

// === BUG: Data race ===
void buggy_increment() {
    int counter = 0; // shared, unprotected
    constexpr int N = 100'000;

    std::vector<std::thread> threads;
    for (int t = 0; t < 4; ++t) {
        threads.emplace_back([&counter, N] {
            for (int i = 0; i < N; ++i) {
                ++counter; // DATA RACE: read-modify-write is NOT atomic
                // Step 1: read counter (e.g., 42)
                // Step 2: increment in register → 43
                // Step 3: write back 43
                // Another thread may read 42 between steps 1-3
                // and also write 43 → one increment LOST
            }
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "Buggy counter: " << counter
              << " (expected " << 4 * N << ")\n";
    // Typical output: Buggy counter: 312847 (expected 400000) — WRONG!
}

// === FIX: Protect with mutex ===
void fixed_increment() {
    int counter = 0;
    std::mutex mtx;
    constexpr int N = 100'000;

    std::vector<std::thread> threads;
    for (int t = 0; t < 4; ++t) {
        threads.emplace_back([&counter, &mtx, N] {
            for (int i = 0; i < N; ++i) {
                std::lock_guard<std::mutex> lock(mtx);
                ++counter; // SAFE: only one thread at a time
            } // lock released here automatically
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "Fixed counter: " << counter
              << " (expected " << 4 * N << ")\n";
    // Output: Fixed counter: 400000 (expected 400000) — CORRECT
}

int main() {
    buggy_increment();
    fixed_increment();
}

```

**What exactly is a data race?**  
Two threads concurrently access the same memory, at least one is a write, and there's no synchronization between them. Data races are **undefined behavior** in C++ — they don't just give wrong values, they make the entire program's behavior undefined.

### Q2: Explain why std::lock_guard is preferable to manual lock/unlock

**Answer:**

```cpp

#include <mutex>
#include <thread>
#include <iostream>
#include <stdexcept>
#include <vector>

std::mutex mtx;
std::vector<int> shared_data;

// === BAD: Manual lock/unlock ===
void manual_lock_danger(int value) {
    mtx.lock();

    shared_data.push_back(value);

    if (value < 0) {
        // BUG: If we throw, mtx.unlock() is NEVER called → DEADLOCK
        // Any other thread waiting on mtx will block forever
        throw std::runtime_error("negative value");
        // mtx is still locked!
    }

    if (value == 0) {
        return; // BUG: early return skips unlock → DEADLOCK
    }

    mtx.unlock(); // only reached on the happy path
}

// === GOOD: lock_guard (RAII) ===
void safe_lock(int value) {
    std::lock_guard<std::mutex> lock(mtx);
    // Or C++17: std::lock_guard lock(mtx);  (CTAD)

    shared_data.push_back(value);

    if (value < 0) {
        throw std::runtime_error("negative value");
        // lock_guard destructor runs → mtx is unlocked!
    }

    if (value == 0) {
        return; // lock_guard destructor runs → mtx is unlocked!
    }

    // Normal exit: lock_guard destructor runs → mtx is unlocked!
}

// Summary of why lock_guard is better:
// 1. Exception-safe:  destructor always unlocks
// 2. Early-return-safe: destructor always unlocks
// 3. Less code:       no forget-to-unlock bugs
// 4. Zero overhead:   same assembly as manual lock/unlock
// 5. Clear scope:     lock lifetime = scope lifetime

int main() {
    // lock_guard is always preferred over manual lock/unlock
    // The ONLY time you need unique_lock instead:
    //   - condition_variable (needs unlock/re-lock)
    //   - deferred locking
    //   - transferring lock ownership

    safe_lock(42);
    try { safe_lock(-1); } catch (...) {}
    safe_lock(0);

    for (int v : shared_data)
        std::cout << v << " ";
    std::cout << "\n";
    // Output: 42 -1 0
}

```

### Q3: Demonstrate std::unique_lock's deferred locking for use with condition variables

**Answer:**

```cpp

#include <mutex>
#include <condition_variable>
#include <thread>
#include <queue>
#include <iostream>
#include <chrono>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> work_queue;
bool done = false;

void producer() {
    for (int i = 1; i <= 5; ++i) {
        {
            std::lock_guard lock(mtx);
            work_queue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
        cv.notify_one();
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }
    {
        std::lock_guard lock(mtx);
        done = true;
    }
    cv.notify_one();
}

void consumer() {
    while (true) {
        // unique_lock is REQUIRED for condition_variable::wait()
        // because wait() must UNLOCK the mutex internally, then RE-LOCK it
        // lock_guard can't do this (it has no unlock method)
        std::unique_lock<std::mutex> lock(mtx);

        cv.wait(lock, [&] {
            return !work_queue.empty() || done;
        });
        // wait() does:
        //   1. Check predicate → if true, return (lock still held)
        //   2. Unlock mutex (via unique_lock)
        //   3. Block thread
        //   4. On wake: re-lock mutex (via unique_lock)
        //   5. Check predicate → if false, go to step 2
        //   6. Return (lock held, predicate is true)

        if (work_queue.empty() && done)
            break;

        int item = work_queue.front();
        work_queue.pop();
        lock.unlock(); // unique_lock allows manual early unlock
        // Process item without holding the lock (better throughput)
        std::cout << "Consumed: " << item << "\n";
    }
}

// === Other unique_lock features ===
void unique_lock_features() {
    std::mutex m1, m2;

    // Deferred locking: construct without locking
    std::unique_lock<std::mutex> lock1(m1, std::defer_lock);
    std::unique_lock<std::mutex> lock2(m2, std::defer_lock);
    // Both mutexes are NOT locked yet

    // Lock both atomically (deadlock-free)
    std::lock(lock1, lock2);
    // Both locked now, will be unlocked by destructors

    // Try-lock: non-blocking attempt
    std::unique_lock<std::mutex> trylock(m1, std::try_to_lock);
    if (trylock.owns_lock()) {
        // We got the lock
    } else {
        // Lock was held by someone else
    }

    // Adopt lock: take ownership of an already-locked mutex
    // m1.lock(); // locked elsewhere
    // std::unique_lock<std::mutex> adopted(m1, std::adopt_lock);
    // adopted destructor will unlock m1
}

int main() {
    std::thread p(producer);
    std::thread c(consumer);
    p.join();
    c.join();

    // Output:
    // Produced: 1
    // Consumed: 1
    // Produced: 2
    // Consumed: 2
    // Produced: 3
    // Consumed: 3
    // Produced: 4
    // Consumed: 4
    // Produced: 5
    // Consumed: 5
}

```

---

## Notes

- **Rule of thumb:** Use `lock_guard` by default. Use `unique_lock` only when you need its extra features (condition_variable, deferred lock, try-lock, manual unlock).
- **C++17 `scoped_lock`:** Locks multiple mutexes at once without deadlock. `std::scoped_lock lock(m1, m2, m3);` replaces the `std::lock()` + `std::lock_guard` pattern.
- **Recursive mutex:** `std::recursive_mutex` allows the same thread to lock multiple times. Generally a design smell — refactor instead.
- **Timed mutex:** `std::timed_mutex` supports `try_lock_for()` and `try_lock_until()` for deadline-based locking.
- **Minimize critical section:** Lock for the shortest time possible. Move computation and I/O outside the lock.
- Compile with `-std=c++17 -O2 -pthread`.
