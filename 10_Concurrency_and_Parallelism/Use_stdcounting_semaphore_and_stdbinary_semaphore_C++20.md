# Use std::counting_semaphore and std::binary_semaphore (C++20)

**Category:** Concurrency and Parallelism  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/thread/counting_semaphore>  

---

## Topic Overview

A **semaphore** is a counter-based synchronization primitive. `acquire()` decrements the counter (blocking if zero); `release()` increments it (waking a blocked thread). You can think of it as a permit system: a thread must obtain a permit to proceed, and permits are returned when the thread is done.

### Types

Here is how you declare the two kinds:

```cpp
#include <semaphore>

// counting_semaphore: counter can go up to LeastMaxValue
std::counting_semaphore<10> sem(3); // max>=10, initial count=3

// binary_semaphore: alias for counting_semaphore<1>
std::binary_semaphore bsem(0);      // initial count=0 (locked)
// Equivalent to: std::counting_semaphore<1> bsem(0);
```

### Semaphore vs Mutex

The most important thing to understand is that a semaphore has no ownership concept. Any thread can call `release()`, regardless of which thread called `acquire()`. That is exactly the opposite of a mutex. This makes semaphores the right tool for producer-consumer signaling, not for protecting shared data.

```cpp
                  Mutex                Semaphore
Ownership:        YES (same thread     NO (any thread can
                  must lock/unlock)    acquire/release)
Counter:          Binary (locked/      Integer (0..N)
                  unlocked)
Cross-thread:     NO                   YES (release from
                  (UB to unlock from   different thread)
                  another thread)
Use case:         Protect shared data  Limit concurrency,
                                       signal between threads
```

### API

| Method | Description |
| --- | --- |
| `sem.acquire()` | Decrement counter; block if counter == 0 |
| `sem.try_acquire()` | Try decrement; return false if counter == 0 |
| `sem.try_acquire_for(dur)` | Timed try |
| `sem.release(n=1)` | Increment counter by n; wake blocked threads |

---

## Self-Assessment

### Q1: Implement a resource pool using counting_semaphore to limit concurrent access

**Answer:**

The counting semaphore here acts as a gatekeeper: its initial count equals the number of available resources, and a thread must "spend" a permit to get one. The RAII `Lease` type ensures the permit is always returned.

```cpp
#include <semaphore>
#include <mutex>
#include <vector>
#include <thread>
#include <iostream>
#include <chrono>
#include <numeric>

// Connection pool: limit to N concurrent database connections
template<typename T>
class ResourcePool {
    std::counting_semaphore<100> sem_;  // blocks when all resources are taken
    std::mutex mtx_;                     // protects the pool vector
    std::vector<T> pool_;

public:
    explicit ResourcePool(std::vector<T> resources)
        : sem_(resources.size()), pool_(std::move(resources)) {}

    // RAII guard that returns resource on destruction
    class Lease {
        ResourcePool& pool_;
        T resource_;
    public:
        Lease(ResourcePool& p, T r) : pool_(p), resource_(std::move(r)) {}
        ~Lease() { pool_.return_resource(std::move(resource_)); }
        T& get() { return resource_; }
        Lease(const Lease&) = delete;
        Lease& operator=(const Lease&) = delete;
    };

    Lease acquire() {
        sem_.acquire(); // blocks if no resources available
        std::lock_guard lock(mtx_);
        T resource = std::move(pool_.back());
        pool_.pop_back();
        return Lease(*this, std::move(resource));
    }

private:
    void return_resource(T resource) {
        {
            std::lock_guard lock(mtx_);
            pool_.push_back(std::move(resource));
        }
        sem_.release(); // unblocks one waiting thread
    }
};

int main() {
    // Pool of 3 "connections" (represented as ints)
    ResourcePool<int> pool({1, 2, 3});
    std::atomic<int> max_concurrent{0};
    std::atomic<int> current{0};

    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&, i] {
            auto lease = pool.acquire();
            int c = current.fetch_add(1) + 1;
            int prev_max = max_concurrent.load();
            while (c > prev_max && !max_concurrent.compare_exchange_weak(prev_max, c));

            std::cout << "Thread " << i << " using connection "
                      << lease.get() << " (concurrent: " << c << ")\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(50));

            current.fetch_sub(1);
            // Lease destructor returns the connection
        });
    }
    for (auto& t : threads) t.join();

    std::cout << "Max concurrent: " << max_concurrent.load()
              << " (limit: 3)\n";

    // Output:
    // Thread 0 using connection 3 (concurrent: 1)
    // Thread 1 using connection 2 (concurrent: 2)
    // Thread 2 using connection 1 (concurrent: 3)
    // Thread 3 using connection 3 (concurrent: 3)  <- reused after Thread 0 finished
    // ...
    // Max concurrent: 3 (limit: 3)
}
```

The counting semaphore's initial count equals the number of resources. Each `acquire()` decrements the count; when it hits zero, subsequent threads block until a resource is returned via `release()`. The mutex only protects the pool vector (very short critical section) - the semaphore does the heavy lifting of blocking and waking threads.

### Q2: Show the difference between a semaphore and a mutex in terms of ownership semantics

**Answer:**

The ownership difference is not just a technicality - it determines what patterns are even possible. A mutex forces the locker to be the unlocker. A semaphore has no such restriction, which is exactly what you need for producer-consumer work.

```cpp
#include <semaphore>
#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>

int main() {
    // === MUTEX: Same-thread ownership ===
    {
        std::mutex mtx;

        std::thread locker([&] {
            mtx.lock();
            std::cout << "Mutex: locked by thread A\n";
            // Thread A MUST unlock - unlocking from thread B is UNDEFINED BEHAVIOR
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            mtx.unlock();
            std::cout << "Mutex: unlocked by thread A\n";
        });
        locker.join();

        // This is UB (don't do this):
        // std::thread([&]{ mtx.lock(); }).join();
        // std::thread([&]{ mtx.unlock(); }).join(); // UB: different thread unlocks!
    }

    // === SEMAPHORE: No ownership ===
    {
        std::binary_semaphore sem(0); // starts "locked" (count=0)

        // Thread A produces
        std::thread producer([&] {
            std::cout << "Sem: producer preparing data...\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
            sem.release(); // Thread A increments -> count becomes 1
            std::cout << "Sem: producer signaled (release from thread A)\n";
        });

        // Thread B consumes - legitimately acquires what Thread A released
        std::thread consumer([&] {
            std::cout << "Sem: consumer waiting...\n";
            sem.acquire(); // Thread B decrements -> this is FINE
            std::cout << "Sem: consumer got signal (acquire from thread B)\n";
        });

        producer.join();
        consumer.join();

        // KEY: release() in Thread A, acquire() in Thread B
        // This is the INTENDED use - impossible with mutex!
    }

    // === Practical example: ping-pong signaling ===
    {
        std::binary_semaphore ping(0), pong(0);

        std::thread t1([&] {
            for (int i = 0; i < 3; ++i) {
                ping.acquire();
                std::cout << "  Pong " << i << "\n";
                pong.release();
            }
        });

        for (int i = 0; i < 3; ++i) {
            std::cout << "Ping " << i << "\n";
            ping.release(); // signal t1
            pong.acquire(); // wait for t1
        }
        t1.join();
    }

    // Output:
    // Mutex: locked by thread A
    // Mutex: unlocked by thread A
    // Sem: producer preparing data...
    // Sem: consumer waiting...
    // Sem: producer signaled (release from thread A)
    // Sem: consumer got signal (acquire from thread B)
    // Ping 0
    //   Pong 0
    // Ping 1
    //   Pong 1
    // Ping 2
    //   Pong 2
}
```

A mutex is an **ownership** primitive - only the locking thread may unlock. A semaphore is a **signaling** primitive - any thread can release, enabling producer-consumer, ping-pong, and cross-thread notification patterns that are impossible with a mutex alone.

### Q3: Use binary_semaphore as a one-shot notification between threads

**Answer:**

A `binary_semaphore` initialized to 0 is a natural "not ready yet" gate. The consumer calls `acquire()` and blocks until the producer calls `release()`. Unlike `condition_variable`, there is no lost-notification problem here: if the producer releases before the consumer acquires, the count is simply 1, and the next `acquire()` returns immediately without blocking.

```cpp
#include <semaphore>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

// === Pattern 1: One producer, one consumer ===
void one_to_one() {
    std::binary_semaphore ready(0); // 0 = not signaled yet
    int result = 0;

    std::thread worker([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        result = 42; // produce data
        ready.release(); // signal: data is ready
    });

    ready.acquire(); // block until signaled
    std::cout << "Result: " << result << "\n"; // guaranteed 42

    worker.join();
}

// === Pattern 2: One producer, N consumers (start gate) ===
void start_gate() {
    constexpr int N = 5;
    // Each worker gets its own semaphore for the start signal
    std::vector<std::binary_semaphore> gates;
    for (int i = 0; i < N; ++i)
        gates.emplace_back(0); // all start at 0

    std::vector<std::thread> workers;
    for (int i = 0; i < N; ++i) {
        workers.emplace_back([&gates, i] {
            gates[i].acquire(); // wait for my start signal
            std::cout << "Worker " << i << " started\n";
        });
    }

    std::cout << "Releasing all workers...\n";
    for (int i = 0; i < N; ++i)
        gates[i].release(); // signal each worker

    for (auto& w : workers) w.join();
}

// === Pattern 3: N workers signal completion to main ===
void completion_fan_in() {
    constexpr int N = 4;
    std::counting_semaphore<4> done(0); // 0 = no completions yet

    std::vector<std::thread> workers;
    for (int i = 0; i < N; ++i) {
        workers.emplace_back([&done, i] {
            std::this_thread::sleep_for(std::chrono::milliseconds(10 * (i + 1)));
            std::cout << "Worker " << i << " done\n";
            done.release(); // signal completion
        });
    }

    // Wait for all N workers to complete
    for (int i = 0; i < N; ++i)
        done.acquire(); // blocks until a worker signals

    std::cout << "All workers completed\n";
    for (auto& w : workers) w.join();
}

int main() {
    std::cout << "--- One-to-one ---\n";
    one_to_one();

    std::cout << "\n--- Start gate ---\n";
    start_gate();

    std::cout << "\n--- Completion fan-in ---\n";
    completion_fan_in();

    // Output:
    // --- One-to-one ---
    // Result: 42
    //
    // --- Start gate ---
    // Releasing all workers...
    // Worker 0 started
    // Worker 1 started
    // Worker 2 started
    // Worker 3 started
    // Worker 4 started
    //
    // --- Completion fan-in ---
    // Worker 0 done
    // Worker 1 done
    // Worker 2 done
    // Worker 3 done
    // All workers completed
}
```

Binary semaphore initialized to 0 acts as a "gate" - the consumer blocks on `acquire()` until the producer calls `release()`. Unlike `condition_variable`, there's no predicate to manage, no mutex needed, and no lost-notification problem (release before acquire just sets count to 1).

---

## Notes

- **binary_semaphore(1) as a mutex substitute:** Initialized to 1, `acquire()` locks, `release()` unlocks. But unlike mutex, it has NO ownership - any thread can release.
- **Counting semaphore for rate limiting:** `counting_semaphore<N>` with initial count N allows at most N concurrent operations.
- **`release(n)`:** Increments count by `n` and wakes up to `n` blocked threads. Useful for batch release.
- **`try_acquire_for()`:** Returns `false` on timeout - useful for cancellation or deadline-based designs.
- **LeastMaxValue template parameter:** `counting_semaphore<100>` guarantees the max count is at least 100. The actual max may be higher (implementation-defined).
- **Performance:** Semaphores typically use futex/WaitOnAddress internally - similar performance to `condition_variable` but with simpler code.
- Compile with `-std=c++20 -O2 -pthread`.
