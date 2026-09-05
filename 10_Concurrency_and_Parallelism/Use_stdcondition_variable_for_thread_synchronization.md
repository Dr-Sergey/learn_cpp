# Use std::condition_variable for thread synchronization

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/condition_variable>  

---

## Topic Overview

`std::condition_variable` allows one thread to **wait** until another thread signals that a condition is satisfied. It always works together with a `std::mutex` and a **predicate** (a boolean condition). The mutex is not optional decoration - it is a fundamental part of how `condition_variable` prevents races between checking the condition and going to sleep.

### Core Pattern

Here is the canonical producer-consumer shape. The consumer holds the lock, surrenders it while it sleeps, and the lock is automatically reacquired when the thread wakes up:

```cpp
Producer:                          Consumer:
{                                  {
  lock_guard lk(mtx);               unique_lock lk(mtx);
  queue.push(item);                  cv.wait(lk, [&]{ return !queue.empty(); });
}                                    // mutex is locked, queue is non-empty
cv.notify_one();                     item = queue.front(); queue.pop();
                                   }
```

### API Summary

| Method | Description |
| --- | --- |
| `cv.wait(lock)` | Unlocks mutex, blocks, re-locks on wake |
| `cv.wait(lock, pred)` | Loops: while(!pred()) wait(lock); - handles spurious wakeups |
| `cv.wait_for(lock, dur, pred)` | Timed wait, returns false on timeout |
| `cv.wait_until(lock, tp, pred)` | Wait until timepoint |
| `cv.notify_one()` | Wake one waiting thread |
| `cv.notify_all()` | Wake all waiting threads |

---

## Self-Assessment

### Q1: Implement a thread-safe bounded queue using condition variables

**Answer:**

Two condition variables - one for "not full" and one for "not empty" - create natural backpressure. The producer cannot race ahead of the consumer indefinitely, and the consumer cannot outrun the producer.

```cpp
#include <condition_variable>
#include <mutex>
#include <queue>
#include <thread>
#include <iostream>
#include <vector>
#include <optional>

template<typename T>
class BoundedQueue {
    std::queue<T> queue_;
    std::mutex mtx_;
    std::condition_variable not_full_;   // signaled when an item is removed
    std::condition_variable not_empty_;  // signaled when an item is added
    size_t capacity_;

public:
    explicit BoundedQueue(size_t cap) : capacity_(cap) {}

    void push(T item) {
        std::unique_lock lock(mtx_);
        // Block until there's room
        not_full_.wait(lock, [&] { return queue_.size() < capacity_; });
        queue_.push(std::move(item));
        not_empty_.notify_one(); // wake one consumer
    }

    T pop() {
        std::unique_lock lock(mtx_);
        // Block until there's an item
        not_empty_.wait(lock, [&] { return !queue_.empty(); });
        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one(); // wake one producer
        return item;
    }

    // Non-blocking try_pop with timeout
    std::optional<T> try_pop(std::chrono::milliseconds timeout) {
        std::unique_lock lock(mtx_);
        if (!not_empty_.wait_for(lock, timeout,
                [&] { return !queue_.empty(); })) {
            return std::nullopt; // timed out
        }
        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();
        return item;
    }
};

int main() {
    BoundedQueue<int> q(3); // capacity = 3
    std::atomic<bool> done{false};

    // Producer: pushes 10 items
    std::thread producer([&] {
        for (int i = 1; i <= 10; ++i) {
            q.push(i);
            std::cout << "Produced: " << i << "\n";
        }
        done.store(true);
    });

    // Consumer: pops items
    std::thread consumer([&] {
        while (true) {
            auto item = q.try_pop(std::chrono::milliseconds(200));
            if (item) {
                std::cout << "Consumed: " << *item << "\n";
            } else if (done.load()) {
                break; // producer is done and queue empty
            }
        }
    });

    producer.join();
    consumer.join();

    // Output (interleaved):
    // Produced: 1
    // Produced: 2
    // Produced: 3
    // Consumed: 1
    // Produced: 4
    // Consumed: 2
    // ... (producer blocks at capacity 3, resumes when consumer pops)
}
```

Two condition variables create **backpressure** - the producer blocks when the queue is full, and the consumer blocks when it's empty. The bounded queue naturally throttles producers to match consumer speed.

### Q2: Explain the spurious wakeup problem and how the predicate overload of wait() solves it

**Answer:**

This is one of the most important things to understand about condition variables. The OS is allowed to wake a thread from `wait()` for reasons completely unrelated to your `notify_one()` call. This is not a bug - it is an intentional relaxation that allows more efficient implementations. The implication is that you must never assume the condition is true just because you woke up.

```cpp
#include <condition_variable>
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
bool data_ready = false;

// === BUG: wait() without predicate ===
void consumer_buggy() {
    std::unique_lock lock(mtx);
    cv.wait(lock);
    // ^^^ PROBLEM: wait() can wake up SPURIOUSLY
    // The OS or implementation may wake this thread even though
    // nobody called notify_one/notify_all.
    //
    // After a spurious wake: data_ready is still false,
    // but we proceed as if data is ready -> BUG!
    std::cout << "data_ready = " << data_ready << "\n"; // might print 0!
}

// === FIX 1: Manual loop (old style) ===
void consumer_manual_loop() {
    std::unique_lock lock(mtx);
    while (!data_ready) { // re-check after every wakeup
        cv.wait(lock);
    }
    // Guaranteed: data_ready == true
}

// === FIX 2: Predicate overload (recommended) ===
void consumer_predicate() {
    std::unique_lock lock(mtx);
    cv.wait(lock, [] { return data_ready; });
    // ^^^ Equivalent to: while(!data_ready) cv.wait(lock);
    //
    // The predicate is checked:
    //   1. BEFORE the first wait (early exit if already true)
    //   2. After EVERY wakeup (spurious or real)
    //
    // Thread only proceeds when predicate returns true
    std::cout << "data_ready = " << data_ready << "\n"; // always 1
}

void producer() {
    {
        std::lock_guard lock(mtx);
        data_ready = true;
    }
    cv.notify_one();
}

int main() {
    data_ready = false;
    std::thread p(producer);
    std::thread c(consumer_predicate);
    p.join();
    c.join();
    // Output: data_ready = 1

    // WHY do spurious wakeups exist?
    // 1. On Linux, futex() can return EINTR (interrupted by signal)
    // 2. On multi-core, a thread may be woken by a notify meant
    //    for another thread waiting on the same CV
    // 3. The OS scheduler may decide to run the thread
    // 4. The implementation may trade rare spurious wakes for
    //    faster normal-path performance
}
```

The predicate overload is not just a convenience - it is the correct way to use `wait()`. The `while(!pred()) wait(lock)` loop is the only safe pattern, and the two-argument form gives you that pattern in one readable call.

### Q3: Show a lost notification caused by notifying before the waiter enters wait()

**Answer:**

Here is another subtlety that trips people up: `condition_variable` does not buffer notifications. If you call `notify_one()` and nobody is currently inside `wait()`, the notification is simply lost. The fix is to always pair the notification with a state change that the waiter can observe before entering `wait()`.

```cpp
#include <condition_variable>
#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>

// === THE BUG: Lost notification ===
// If notify_one is called BEFORE the consumer calls wait(),
// the notification is lost - there's no "pending notification" buffer.

void demo_lost_notification() {
    std::mutex mtx;
    std::condition_variable cv;
    bool ready = false; // This flag is CRITICAL

    // === BUG VERSION (no flag check) ===
    // Producer notifies immediately, consumer waits later -> DEADLOCK!
    /*
    std::thread producer([&] {
        // Notify immediately - consumer hasn't called wait() yet
        cv.notify_one(); // LOST! Nobody is waiting
    });

    std::thread consumer([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        std::unique_lock lock(mtx);
        cv.wait(lock); // DEADLOCK: notification already happened and was lost
    });
    */

    // === FIXED VERSION (with predicate) ===
    std::thread producer([&] {
        // Notify immediately
        {
            std::lock_guard lock(mtx);
            ready = true; // set the flag BEFORE notifying
        }
        cv.notify_one(); // even if lost, the flag persists
    });

    std::thread consumer([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        std::unique_lock lock(mtx);
        cv.wait(lock, [&] { return ready; });
        // ^^^ First checks ready -> it's true -> doesn't even call wait()!
        // The predicate saves us from the lost notification
        std::cout << "Consumer: got notification (ready=" << ready << ")\n";
    });

    producer.join();
    consumer.join();
}

// === RULE: Always protect condition_variable with a predicate ===
//
// The predicate (bool flag / queue size / etc.) is the REAL synchronization.
// The condition_variable is just an OPTIMIZATION to avoid busy-waiting.
//
//   WRONG: cv.wait(lock);                       // no predicate -> lost notifications, spurious wakes
//   RIGHT: cv.wait(lock, [&]{ return ready; }); // predicate handles all edge cases

// === Must the mutex be locked when calling notify? ===
//
// Technically NO - notify_one/all can be called without the mutex.
// But the FLAG must be set under the mutex:
//
//   std::lock_guard lock(mtx);
//   ready = true;          // SET FLAG under lock
//   }                      // UNLOCK
//   cv.notify_one();       // Can be outside the lock (often more efficient)
//
// Why more efficient? If notify is inside the lock, the woken thread
// immediately tries to lock the mutex... which is still held -> blocks again.

int main() {
    demo_lost_notification();
    // Output: Consumer: got notification (ready=1)
}
```

**Key takeaways:**

1. `notify_one/all` does NOT buffer - if no thread is waiting, the notification is lost.
2. **Always use a predicate** to guard against both spurious wakeups and lost notifications.
3. The predicate flag must be set under the mutex lock to prevent a TOCTOU race.
4. `notify_one/all` can safely be called outside the mutex lock for better performance.

---

## Notes

- **`unique_lock` vs `lock_guard`:** `wait()` requires `unique_lock` because it needs to unlock/re-lock the mutex internally. `lock_guard` can't do this.
- **`condition_variable` vs `condition_variable_any`:** `condition_variable` works only with `std::mutex`. `condition_variable_any` works with any lockable type (e.g., `shared_mutex`), but is slower.
- **`notify_one` vs `notify_all`:** Use `notify_one` when only one consumer can act on the condition change. Use `notify_all` when multiple consumers might be able to proceed (or when the predicate differs per consumer).
- **Thundering herd:** `notify_all` wakes all waiters; all but one will re-check the predicate, find it false, and go back to sleep. This wastes CPU for many waiters.
- **C++20 alternative:** `std::atomic::wait/notify` is simpler for flag-based signaling (no mutex needed).
- Compile with `-std=c++11 -O2 -pthread`.
