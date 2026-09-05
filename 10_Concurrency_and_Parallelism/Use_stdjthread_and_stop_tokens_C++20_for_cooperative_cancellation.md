# Use std::jthread and stop tokens (C++20) for cooperative cancellation

**Category:** Concurrency and Parallelism  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/thread/jthread>  

---

## Topic Overview

`std::jthread` ("joining thread") is a drop-in replacement for `std::thread` with two improvements:

1. **Auto-join on destruction** - no more `std::terminate()` from forgetting to join
2. **Built-in stop token** - cooperative cancellation without manual flags

These two features together eliminate most of the boilerplate and footguns that come with raw `std::thread`.

### Comparison

```cpp
std::thread:                        std::jthread:
std::thread t(work);                 std::jthread jt(work);
// MUST call t.join() or             // auto-joins in destructor
// t.detach() before destruction     // auto-requests stop
// Forgetting = std::terminate()!    // No manual cleanup needed!

// Manual cancellation:              // Built-in cancellation:
std::atomic<bool> stop{false};       // jthread passes stop_token to callable
t = std::thread([&]{                 jt = std::jthread([](std::stop_token st){
  while(!stop) { work(); }             while(!st.stop_requested()) { work(); }
});                                  });
stop = true;                         // jt.request_stop() called automatically
t.join();                            // on destruction (or manually)
```

---

## Self-Assessment

### Q1: Replace a raw std::thread with std::jthread and show auto-join on destruction

**Answer:**

The reason `std::thread` requires you to call `join()` or `detach()` before destruction is that the destructor cannot safely make that choice for you - it does not know whether you intended for the thread to keep running. `jthread` solves this by always joining on destruction, which is the right answer in the vast majority of cases.

```cpp
#include <thread>
#include <iostream>
#include <chrono>
#include <stdexcept>

void demo_thread_danger() {
    std::cout << "=== std::thread dangers ===\n";

    // DANGER 1: Forgetting to join
    // {
    //     std::thread t([]{ std::cout << "work\n"; });
    //     // destructor called without join/detach -> std::terminate()!
    // }

    // DANGER 2: Exception skips join
    try {
        std::thread t([] {
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            std::cout << "  thread done\n";
        });
        // If this throws, t.join() is skipped -> std::terminate()!
        // throw std::runtime_error("oops");
        t.join(); // MUST remember this
    } catch (...) {
        // t is already destroyed -> terminate() was called
    }
}

void demo_jthread_safety() {
    std::cout << "\n=== std::jthread safety ===\n";

    // Safe: auto-joins on destruction
    {
        std::jthread jt([] {
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            std::cout << "  jthread done\n";
        });
        // jt goes out of scope here -> auto-joins. No terminate()!
    }
    std::cout << "  (after scope - jthread already joined)\n";

    // Safe even with exceptions
    try {
        std::jthread jt([] {
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            std::cout << "  jthread in try block\n";
        });
        throw std::runtime_error("oops");
        // jt destructor runs during stack unwinding -> auto-joins safely
    } catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << " (jthread was auto-joined)\n";
    }

    // Multiple jthreads - all auto-join in reverse order
    {
        std::jthread j1([]{std::cout << "  j1 done\n";});
        std::jthread j2([]{std::cout << "  j2 done\n";});
        std::jthread j3([]{std::cout << "  j3 done\n";});
        // Destruction order: j3, j2, j1 (reverse of declaration)
    }
}

int main() {
    demo_thread_danger();
    demo_jthread_safety();

    // Output:
    // === std::thread dangers ===
    //   thread done
    //
    // === std::jthread safety ===
    //   jthread done
    //   (after scope - jthread already joined)
    //   jthread in try block
    //   Caught: oops (jthread was auto-joined)
    //   j3 done
    //   j2 done
    //   j1 done
}
```

### Q2: Implement a cancellable worker loop using std::stop_token::stop_requested

**Answer:**

The stop token model is cooperative: the thread checks whether a stop has been requested and exits when it sees one. Nobody forcibly kills the thread - it finishes its current unit of work cleanly and then exits. This is almost always what you actually want, because forcible termination leaves shared state in an unknown condition.

```cpp
#include <thread>
#include <stop_token>
#include <iostream>
#include <chrono>
#include <vector>
#include <atomic>

// Worker accepts stop_token as FIRST parameter
// jthread automatically passes it
void sensor_reader(std::stop_token stoken, int sensor_id) {
    int reading = 0;
    while (!stoken.stop_requested()) {
        // Simulate sensor read
        ++reading;
        std::cout << "Sensor " << sensor_id << ": reading #" << reading << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    std::cout << "Sensor " << sensor_id << ": stopped after "
              << reading << " readings\n";
}

int main() {
    // === Single worker with manual stop ===
    {
        std::cout << "--- Manual stop ---\n";
        std::jthread worker(sensor_reader, 1);
        // worker callable receives stop_token automatically as first arg!

        std::this_thread::sleep_for(std::chrono::milliseconds(350));
        worker.request_stop(); // signal the worker to stop
        // worker auto-joins in destructor
    }

    // === Multiple workers stopped together ===
    {
        std::cout << "\n--- Shared stop source ---\n";
        std::stop_source shared_source;

        std::vector<std::jthread> workers;
        for (int i = 0; i < 3; ++i) {
            workers.emplace_back([token = shared_source.get_token(), i] {
                int count = 0;
                while (!token.stop_requested()) {
                    ++count;
                    std::this_thread::sleep_for(std::chrono::milliseconds(100));
                }
                std::cout << "Worker " << i << " did " << count << " iterations\n";
            });
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(250));
        shared_source.request_stop(); // stops ALL workers at once
        // workers auto-join when vector is destroyed
    }

    // === jthread destructor auto-requests stop ===
    {
        std::cout << "\n--- Auto stop on destruction ---\n";
        std::jthread worker([](std::stop_token st) {
            while (!st.stop_requested()) {
                std::cout << "Working...\n";
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
            }
            std::cout << "Stopped!\n";
        });

        std::this_thread::sleep_for(std::chrono::milliseconds(250));
        // worker goes out of scope:
        //   1. Destructor calls request_stop()
        //   2. Worker loop sees stop_requested() == true
        //   3. Destructor calls join()
    }

    // Output:
    // --- Manual stop ---
    // Sensor 1: reading #1
    // Sensor 1: reading #2
    // Sensor 1: reading #3
    // Sensor 1: stopped after 3 readings
    // ...
}
```

### Q3: Show how std::stop_callback can clean up resources when a stop is requested

**Answer:**

Here is where the stop token system becomes really powerful. The `stop_callback` lets you register cleanup logic that fires automatically when a stop is requested - even if the thread is currently blocked inside `cv.wait()` or a similar call. Without this mechanism, cancellation and blocking waits do not mix cleanly.

```cpp
#include <thread>
#include <stop_token>
#include <iostream>
#include <chrono>
#include <condition_variable>
#include <mutex>
#include <queue>

// stop_callback registers a function that is called when stop is requested.
// Use cases:
//   - Cancel blocking I/O operations
//   - Wake condition_variable waiters
//   - Release resources / close file handles
//   - Propagate cancellation to sub-operations

// === Example 1: Wake a blocked condition_variable ===
class CancellableQueue {
    std::queue<int> queue_;
    std::mutex mtx_;
    std::condition_variable cv_;

public:
    void push(int val) {
        {
            std::lock_guard lock(mtx_);
            queue_.push(val);
        }
        cv_.notify_one();
    }

    // Blocking pop that responds to cancellation
    std::optional<int> pop(std::stop_token stoken) {
        std::unique_lock lock(mtx_);

        // Register callback: when stop is requested, wake us up
        std::stop_callback callback(stoken, [this] {
            cv_.notify_all(); // wake blocked threads so they can check stop
        });

        cv_.wait(lock, [&] {
            return !queue_.empty() || stoken.stop_requested();
        });

        if (stoken.stop_requested())
            return std::nullopt; // cancelled

        int val = queue_.front();
        queue_.pop();
        return val;
    }
    // stop_callback is destroyed when pop() returns - automatically deregistered
};

// === Example 2: Chain cancellation (parent -> child) ===
void child_task(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        std::cout << "  Child working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    std::cout << "  Child cancelled\n";
}

void parent_task(std::stop_token parent_token) {
    std::stop_source child_source;

    // Propagate parent cancellation to child
    std::stop_callback propagate(parent_token, [&child_source] {
        std::cout << "  Propagating stop to child...\n";
        child_source.request_stop();
    });

    std::jthread child(child_task);
    // Connect child to child_source (not parent)
    // But via stop_callback, parent stop -> child stop

    std::this_thread::sleep_for(std::chrono::milliseconds(350));
    // If parent is stopped, propagate callback fires -> child stops
}

int main() {
    // === Cancellable queue ===
    std::cout << "--- Cancellable queue ---\n";
    {
        CancellableQueue q;

        std::jthread consumer([&q](std::stop_token st) {
            while (auto val = q.pop(st)) {
                std::cout << "Got: " << *val << "\n";
            }
            std::cout << "Consumer cancelled\n";
        });

        q.push(1);
        q.push(2);
        q.push(3);
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        // consumer destructor: request_stop() -> callback fires -> cv wakes -> pop returns nullopt
    }

    // === Chained cancellation ===
    std::cout << "\n--- Chained cancellation ---\n";
    {
        std::jthread parent(parent_task);
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
        parent.request_stop(); // parent stop -> callback -> child stop
    }

    // Output:
    // --- Cancellable queue ---
    // Got: 1
    // Got: 2
    // Got: 3
    // Consumer cancelled
    //
    // --- Chained cancellation ---
    //   Child working...
    //   Child working...
    //   Propagating stop to child...
    //   Child cancelled
}
```

`stop_callback` bridges the gap between stop tokens and other blocking primitives. Without it, a thread blocked on `cv.wait()` would never wake up to check `stop_requested()`. The callback calls `cv.notify_all()`, waking the thread so it can observe the stop request. This pattern - register a wake-up callback, then wait with a stop check in the predicate - is the standard recipe for making any blocking wait cancellable.

---

## Notes

- **`stop_callback` lifetime:** The callback is registered in the constructor and deregistered in the destructor. If stop was already requested, the callback runs **immediately in the constructor**.
- **Thread safety:** `stop_callback` is safe to construct and destroy from any thread. Multiple callbacks on the same token are all called when stop is requested.
- **`jthread` destructor sequence:** 1) `request_stop()` -> 2) callable checks `stop_requested()` -> 3) `join()`.
- **Migrating from `std::thread`:** Replace `std::thread` with `std::jthread`, replace `std::atomic<bool> stop` with `std::stop_token` parameter, replace `stop = true; t.join()` with nothing (destructor handles it).
- **`stop_source`/`stop_token`/`stop_callback`** can be used independently of `jthread` - they're general-purpose cancellation primitives.
- Compile with `-std=c++20 -O2 -pthread`.
