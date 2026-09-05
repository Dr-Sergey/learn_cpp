# Use std::async with a thread pool pattern for task parallelism

**Category:** Concurrency and Parallelism  
**Standard:** C++11, C++20  
**Reference:** <https://en.cppreference.com/w/cpp/thread/async>  

---

## Topic Overview

`std::async` is the simplest way to launch parallel tasks in C++. It returns a `std::future` that holds the result. However, `std::async` has surprising behaviors (especially with `launch::deferred`) and doesn't guarantee a thread pool - it may create a new thread per call. That's fine for a handful of tasks, but if you're submitting hundreds of short tasks, the thread-creation overhead adds up quickly.

### std::async Launch Policies

There are three launch modes and the default is not what most people expect:

```cpp
std::launch::async      -> runs in a NEW thread (guaranteed)
std::launch::deferred   -> runs when future.get() is called (lazy, same thread)
std::launch::async |    -> implementation chooses (default)
std::launch::deferred
```

The default policy (`async | deferred`) lets the implementation decide. In practice this means you can't be sure you're getting parallelism unless you explicitly ask for `launch::async`.

### async vs Thread Pool

| Feature | `std::async` | Custom Thread Pool |
| --- | --- | --- |
| Thread reuse | No (creates new threads) | Yes (fixed pool) |
| Thread count control | No | Yes (N = cpu cores) |
| Task queuing | No | Yes (FIFO/priority) |
| Overhead per task | High (thread creation) | Low (enqueue) |
| Simplicity | Very simple | Moderate |
| Standard | C++11 | Manual or library |

---

## Self-Assessment

### Q1: Submit N tasks with std::async and collect futures in a vector for batch result retrieval

**Answer:**

There's a critical gotcha here that catches almost everyone: you must store the futures in a container *before* calling `get()` on any of them. If you don't store a future, its destructor runs immediately and blocks until the task finishes - making everything sequential. The comments explain this:

```cpp
#include <future>
#include <vector>
#include <iostream>
#include <chrono>
#include <cmath>
#include <numeric>

// Simulate a CPU-intensive computation
double heavy_computation(int id) {
    double result = 0.0;
    for (int i = 0; i < 1'000'000; ++i)
        result += std::sin(i * 0.001 + id);
    return result;
}

int main() {
    constexpr int N = 8;
    auto start = std::chrono::steady_clock::now();

    // Submit N tasks - each returns a future
    std::vector<std::future<double>> futures;
    for (int i = 0; i < N; ++i) {
        futures.push_back(
            std::async(std::launch::async, heavy_computation, i)
            //               ^^^^^
            // force async: guarantee each runs in a separate thread
        );
    }

    // Collect results - future.get() blocks until result is ready
    std::vector<double> results;
    for (auto& f : futures) {
        results.push_back(f.get()); // blocks if not yet computed
    }

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    // Print results
    double total = std::accumulate(results.begin(), results.end(), 0.0);
    std::cout << "N=" << N << " tasks completed in " << ms << " ms\n";
    std::cout << "Total: " << total << "\n";

    // === Comparison: sequential execution ===
    auto start_seq = std::chrono::steady_clock::now();
    double seq_total = 0;
    for (int i = 0; i < N; ++i)
        seq_total += heavy_computation(i);
    auto ms_seq = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start_seq).count();
    std::cout << "Sequential: " << ms_seq << " ms\n";

    // Output (8-core machine):
    // N=8 tasks completed in 15 ms      <- parallel
    // Sequential: 100 ms                <- serial
    // ~6-7x speedup with 8 threads

    // IMPORTANT: the future returned by std::async has special behavior.
    // Its destructor blocks until the task completes!
    // So this is SEQUENTIAL (not parallel):
    //   for (int i = 0; i < N; ++i)
    //     std::async(launch::async, f, i);  // future destroyed immediately -> blocks!
    //
    // You MUST store futures in a container to get parallelism.
}
```

**Explanation:** Store all futures in a vector *before* calling `get()` on any of them. This ensures all tasks run in parallel. If you don't store the futures, the temporary `std::future` destructor blocks until completion - making execution sequential.

### Q2: Show the problem with std::launch::deferred when combined with future::get on a single thread

**Answer:**

`launch::deferred` is useful for lazy evaluation, but it's a trap when you think you're getting parallelism. Watch the thread IDs in the output - all three deferred tasks run on the main thread:

```cpp
#include <future>
#include <iostream>
#include <chrono>
#include <thread>

int slow_task(int id) {
    // Simulate 500ms of work
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    std::cout << "Task " << id << " completed on thread "
              << std::this_thread::get_id() << "\n";
    return id * 10;
}

int main() {
    std::cout << "Main thread: " << std::this_thread::get_id() << "\n\n";

    // === DEFERRED: runs on get(), NOT in parallel ===
    {
        auto start = std::chrono::steady_clock::now();

        auto f1 = std::async(std::launch::deferred, slow_task, 1);
        auto f2 = std::async(std::launch::deferred, slow_task, 2);
        auto f3 = std::async(std::launch::deferred, slow_task, 3);

        // Each get() runs the task on the CALLING thread
        int r1 = f1.get(); // runs task 1 HERE (500ms)
        int r2 = f2.get(); // runs task 2 HERE (500ms)
        int r3 = f3.get(); // runs task 3 HERE (500ms)

        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "Deferred: " << ms << " ms (sequential!)\n\n";
        // Output:
        // Task 1 completed on thread 140234567890 <- main thread!
        // Task 2 completed on thread 140234567890 <- main thread!
        // Task 3 completed on thread 140234567890 <- main thread!
        // Deferred: ~1500 ms (3 x 500ms, sequential)
    }

    // === ASYNC: runs in parallel ===
    {
        auto start = std::chrono::steady_clock::now();

        auto f1 = std::async(std::launch::async, slow_task, 1);
        auto f2 = std::async(std::launch::async, slow_task, 2);
        auto f3 = std::async(std::launch::async, slow_task, 3);

        f1.get(); f2.get(); f3.get();

        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "Async: " << ms << " ms (parallel!)\n\n";
        // Output:
        // Task 1 completed on thread 140234111111 <- new thread
        // Task 2 completed on thread 140234222222 <- different thread
        // Task 3 completed on thread 140234333333 <- different thread
        // Async: ~500 ms (all run in parallel)
    }

    // === DANGER: default policy might choose deferred ===
    {
        auto f = std::async(slow_task, 1); // default = async|deferred
        // Implementation MAY run this deferred if it decides to!
        //
        // Worse: you can't reliably check if it's running yet:
        //   f.wait_for(0s) == future_status::deferred  // might be true!
        //
        // RULE: Always specify std::launch::async explicitly
        // if you need parallel execution.
    }
}
```

**Explanation:** `launch::deferred` doesn't launch a thread - it stores the callable and runs it lazily when `get()` or `wait()` is called. All tasks execute on the calling thread, sequentially. This defeats the purpose of async execution. Always use `launch::async` for true parallelism.

### Q3: Implement a simple thread pool using jthread + queue to replace std::async

**Answer:**

For many short tasks, a thread pool beats `std::async` because it avoids the overhead of creating and destroying threads for each task. The pool creates `N` workers once at startup and reuses them for every submitted task. The `submit()` method returns a `std::future<T>` just like `std::async`, so the calling code barely needs to change:

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>
#include <future>
#include <vector>
#include <iostream>
#include <chrono>
#include <stop_token>

class ThreadPool {
    std::vector<std::jthread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool shutdown_ = false;

public:
    explicit ThreadPool(size_t num_threads = std::thread::hardware_concurrency()) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this](std::stop_token token) {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock lock(mtx_);
                        cv_.wait(lock, [&] {
                            return shutdown_ || !tasks_.empty();
                        });
                        if (shutdown_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task(); // execute outside the lock
                }
            });
        }
    }

    ~ThreadPool() {
        {
            std::lock_guard lock(mtx_);
            shutdown_ = true;
        }
        cv_.notify_all();
        // jthread destructor calls request_stop() + join()
    }

    // Submit a task and get a future for the result
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args) -> std::future<std::invoke_result_t<F, Args...>> {
        using return_type = std::invoke_result_t<F, Args...>;

        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );

        std::future<return_type> result = task->get_future();
        {
            std::lock_guard lock(mtx_);
            tasks_.emplace([task] { (*task)(); });
        }
        cv_.notify_one();
        return result;
    }
};

// === Benchmark: thread pool vs std::async ===
double compute(int id) {
    double result = 0;
    for (int i = 0; i < 100'000; ++i)
        result += std::sin(i * 0.001 + id);
    return result;
}

int main() {
    constexpr int TASKS = 100;

    // === std::async (creates 100 threads!) ===
    {
        auto start = std::chrono::steady_clock::now();
        std::vector<std::future<double>> futures;
        for (int i = 0; i < TASKS; ++i)
            futures.push_back(std::async(std::launch::async, compute, i));
        for (auto& f : futures) f.get();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "std::async (" << TASKS << " threads): " << ms << " ms\n";
    }

    // === Thread pool (reuses N threads) ===
    {
        ThreadPool pool(std::thread::hardware_concurrency());
        auto start = std::chrono::steady_clock::now();
        std::vector<std::future<double>> futures;
        for (int i = 0; i < TASKS; ++i)
            futures.push_back(pool.submit(compute, i));
        for (auto& f : futures) f.get();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "Thread pool (" << std::thread::hardware_concurrency()
                  << " threads): " << ms << " ms\n";
    }

    // Typical output (8-core machine, 100 tasks):
    // std::async (100 threads): 250 ms  <- thread creation overhead
    // Thread pool (8 threads):  120 ms  <- reuses threads, better locality
}
```

**Explanation:** The thread pool creates N worker threads once and reuses them for all tasks. `submit()` returns a `std::future<T>` just like `std::async`, but without creating a new thread each time. For many small tasks, a thread pool is significantly faster than `std::async` due to eliminated thread creation/destruction overhead.

---

## Notes

- **`std::async` future destructor blocks!** The future returned by `async` (unlike promise-created futures) blocks in its destructor until the task completes. This is a common source of accidental sequential execution.
- **Default launch policy** (`async | deferred`) lets the implementation choose - this means your code might silently become sequential. Always specify `launch::async`.
- **Thread pool libraries:** For production use, consider `BS::thread_pool`, Intel TBB, or `std::execution` (C++17 parallel algorithms).
- **`std::packaged_task`** wraps any callable with a future - useful as the building block for custom thread pools.
- **C++23/26 `std::execution`** will provide standard executors and thread pools, eventually replacing hand-rolled solutions.
- Compile with `-std=c++20 -O2 -pthread`.
