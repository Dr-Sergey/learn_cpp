# Design thread pool architectures for concurrent task processing

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

A **thread pool** manages a fixed number of worker threads that execute tasks from a shared queue. This avoids the overhead of thread creation/destruction per task and bounds concurrency. Modern C++ thread pools range from simple work queues to work-stealing designs optimized for different workload patterns.

The core intuition: creating a `std::thread` is expensive (OS call, stack allocation, scheduler registration). If you're processing thousands of short tasks, spawning a thread per task would spend more time on thread management than actual work. A thread pool amortizes that cost by keeping N threads alive and handing them tasks to execute as they become available.

### Thread Pool Design Variants

| Design | Best For | Overhead | Scalability |
| --- | --- | --- | --- |
| **Single queue** | Simple tasks, I/O | Mutex contention | Moderate |
| **Work-stealing** | Uneven CPU tasks | Complex, low contention | Excellent |
| **Per-thread queues** | Affinity-bound tasks | Minimal contention | Good |
| **Priority queue** | Mixed urgency tasks | Priority management | Moderate |

---

## Self-Assessment

### Q1: Implement a production-quality thread pool with futures

This implementation uses a single shared queue protected by a mutex and a condition variable. Workers sleep on the condition variable when the queue is empty and wake up when a task is pushed. The `submit()` method returns a `std::future<ReturnType>` so the caller can optionally wait for the result and retrieve it (or catch any exception the task threw).

The `packaged_task` + `shared_ptr` pattern is the standard trick for erasing the return type: `std::function<void()>` in the queue stores a lambda that calls the `packaged_task`, and the future is detached from it before we move everything into the queue.

**Answer:**

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>
#include <future>
#include <vector>
#include <atomic>

class ThreadPool {
public:
    explicit ThreadPool(size_t threads = std::thread::hardware_concurrency()) {
        for (size_t i = 0; i < threads; ++i) {
            workers_.emplace_back([this] { worker_loop(); });
        }
    }

    // Submit task and get future for result
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using ReturnType = std::invoke_result_t<F, Args...>;

        auto task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );

        auto future = task->get_future();

        {
            std::lock_guard lock(mutex_);
            if (stopped_)
                throw std::runtime_error("Pool is stopped");
            tasks_.push([task]() { (*task)(); });
        }
        cv_.notify_one();

        return future;
    }

    // Wait for all tasks to complete
    void wait_idle() {
        std::unique_lock lock(mutex_);
        idle_cv_.wait(lock, [this] {
            return tasks_.empty() && active_tasks_ == 0;
        });
    }

    size_t thread_count() const { return workers_.size(); }
    size_t pending_count() {
        std::lock_guard lock(mutex_);
        return tasks_.size();
    }

    ~ThreadPool() {
        {
            std::lock_guard lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
        for (auto& w : workers_) w.join();
    }

private:
    void worker_loop() {
        while (true) {
            std::function<void()> task;
            {
                std::unique_lock lock(mutex_);
                cv_.wait(lock, [this] {
                    return stopped_ || !tasks_.empty();
                });
                if (stopped_ && tasks_.empty()) return;
                task = std::move(tasks_.front());
                tasks_.pop();
                ++active_tasks_;
            }

            task();

            {
                std::lock_guard lock(mutex_);
                --active_tasks_;
                if (tasks_.empty() && active_tasks_ == 0)
                    idle_cv_.notify_all();
            }
        }
    }

    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mutex_;
    std::condition_variable cv_;
    std::condition_variable idle_cv_;
    int active_tasks_ = 0;
    bool stopped_ = false;
};

// === Usage ===
void example() {
    ThreadPool pool(4);

    // Submit tasks, get futures
    auto f1 = pool.submit([] { return 42; });
    auto f2 = pool.submit([](int a, int b) { return a + b; }, 10, 20);
    auto f3 = pool.submit([] {
        std::vector<int> v(1'000'000);
        std::iota(v.begin(), v.end(), 0);
        return std::accumulate(v.begin(), v.end(), 0LL);
    });

    std::cout << f1.get() << "\n";  // 42
    std::cout << f2.get() << "\n";  // 30
    std::cout << f3.get() << "\n";  // 499999500000
}
```

The destructor sets `stopped_ = true`, then `notify_all()` wakes all sleeping workers, and each one sees the stop condition and exits cleanly. This ensures no tasks are dropped silently - the workers drain pending tasks first (note: `stopped_ && tasks_.empty()` means a worker exits only when both the stop flag is set *and* the queue is empty).

### Q2: Implement work-stealing for better load balancing

The limitation of a single shared queue is mutex contention: every push and pop takes the lock. With many threads, that becomes a bottleneck. Work-stealing solves this by giving each worker its own queue. A worker pushes new tasks onto its own queue (no contention) and pops from the back (LIFO, which is cache-friendly because recently pushed tasks are still warm in cache). If a worker's queue is empty, it "steals" from the *front* of a randomly chosen sibling's queue (FIFO, which tends to steal larger/older tasks - the kind most worth stealing).

**Answer:**

```cpp
#include <deque>
#include <random>

// === Lock-free-ish work-stealing queue (simplified) ===
class WorkStealingQueue {
public:
    void push(std::function<void()> task) {
        std::lock_guard lock(mutex_);
        deque_.push_back(std::move(task));
    }

    // Owner pops from back (LIFO - cache-friendly)
    bool pop(std::function<void()>& task) {
        std::lock_guard lock(mutex_);
        if (deque_.empty()) return false;
        task = std::move(deque_.back());
        deque_.pop_back();
        return true;
    }

    // Thieves steal from front (FIFO - older/larger tasks)
    bool steal(std::function<void()>& task) {
        std::lock_guard lock(mutex_);
        if (deque_.empty()) return false;
        task = std::move(deque_.front());
        deque_.pop_front();
        return true;
    }

    bool empty() {
        std::lock_guard lock(mutex_);
        return deque_.empty();
    }

private:
    std::deque<std::function<void()>> deque_;
    std::mutex mutex_;
};

class WorkStealingPool {
public:
    explicit WorkStealingPool(size_t n = std::thread::hardware_concurrency())
        : queues_(n), stopped_(false) {
        for (size_t i = 0; i < n; ++i) {
            workers_.emplace_back([this, i] { worker_loop(i); });
        }
    }

    void submit(std::function<void()> task) {
        // Round-robin distribution
        auto idx = next_queue_++ % queues_.size();
        queues_[idx].push(std::move(task));
        cv_.notify_one();
    }

    ~WorkStealingPool() {
        stopped_ = true;
        cv_.notify_all();
        for (auto& w : workers_) w.join();
    }

private:
    void worker_loop(size_t my_idx) {
        while (!stopped_) {
            std::function<void()> task;

            // Try own queue first
            if (queues_[my_idx].pop(task)) {
                task();
                continue;
            }

            // Try stealing from others
            bool stolen = false;
            for (size_t i = 0; i < queues_.size(); ++i) {
                if (i == my_idx) continue;
                if (queues_[i].steal(task)) {
                    task();
                    stolen = true;
                    break;
                }
            }

            if (!stolen) {
                // No work anywhere - wait
                std::unique_lock lock(wait_mutex_);
                cv_.wait_for(lock, std::chrono::milliseconds(1));
            }
        }
    }

    std::vector<WorkStealingQueue> queues_;
    std::vector<std::jthread> workers_;
    std::atomic<bool> stopped_;
    std::atomic<size_t> next_queue_{0};
    std::mutex wait_mutex_;
    std::condition_variable cv_;
};
```

The reason this is often described as "lock-free-ish" rather than truly lock-free is that this simplified version still uses a mutex per queue. In a production work-stealing implementation (like those in Intel TBB or the Tokio runtime) the queue uses atomic operations only, which is harder to implement correctly but eliminates all lock overhead on the hot path.

### Q3: Implement parallel for_each and parallel_reduce

Once you have a thread pool, you can build parallel algorithms on top of it. The idea is to divide the input range into chunks, submit one task per chunk, and wait for all futures to complete. The `parallel_reduce` collects partial results from each chunk and combines them in the main thread.

The `chunk_size` formula `total / (thread_count * 4)` produces about four chunks per thread. That's intentional over-decomposition: if one chunk takes longer than the others, idle threads can start the next chunk. Exactly one chunk per thread would leave idle capacity if the work is uneven.

**Answer:**

```cpp
// === Parallel algorithms on top of thread pool ===
template<typename Iter, typename Func>
void parallel_for_each(ThreadPool& pool, Iter begin, Iter end, Func fn,
                        size_t chunk_size = 0) {
    auto total = std::distance(begin, end);
    if (chunk_size == 0)
        chunk_size = std::max(size_t(1),
            static_cast<size_t>(total) / (pool.thread_count() * 4));

    std::vector<std::future<void>> futures;
    for (auto it = begin; it < end; it += chunk_size) {
        auto chunk_end = std::min(it + chunk_size, end);
        futures.push_back(pool.submit([it, chunk_end, &fn] {
            for (auto i = it; i < chunk_end; ++i)
                fn(*i);
        }));
    }
    for (auto& f : futures) f.get();
}

template<typename Iter, typename T, typename ReduceOp>
T parallel_reduce(ThreadPool& pool, Iter begin, Iter end,
                   T init, ReduceOp op) {
    auto total = std::distance(begin, end);
    size_t chunk_size = std::max(size_t(1),
        static_cast<size_t>(total) / (pool.thread_count() * 4));

    std::vector<std::future<T>> futures;
    for (auto it = begin; it < end; it += chunk_size) {
        auto chunk_end = std::min(it + chunk_size, end);
        futures.push_back(pool.submit([it, chunk_end, op, init] {
            T local = init;
            for (auto i = it; i < chunk_end; ++i)
                local = op(local, *i);
            return local;
        }));
    }

    T result = init;
    for (auto& f : futures)
        result = op(result, f.get());
    return result;
}

// Usage:
// std::vector<double> data(10'000'000);
// parallel_for_each(pool, data.begin(), data.end(),
//     [](double& x) { x = std::sin(x); });
//
// double sum = parallel_reduce(pool, data.begin(), data.end(),
//     0.0, std::plus<>{});
```

The `parallel_reduce` result is only correct if `op` is associative - addition and multiplication are fine, but operations that depend on order are not. The partial results are combined in the order futures complete, which may differ from the original element order.

---

## Notes

- **Default thread count** = `std::thread::hardware_concurrency()`, adjust for I/O-bound workloads.
- Single-queue pools: simple but contended under high load; sufficient for most applications.
- Work-stealing: excellent for uneven tasks, minimal idle time, more complex to implement.
- Use `std::future` for tasks that produce results; fire-and-forget for side effects.
- `wait_idle()` is essential for testing and batch processing.
- Pool destructor must drain pending tasks or cancel them - decide which semantic you need.
- C++23 `std::execution` / sender-receiver will eventually standardize thread pool semantics.
