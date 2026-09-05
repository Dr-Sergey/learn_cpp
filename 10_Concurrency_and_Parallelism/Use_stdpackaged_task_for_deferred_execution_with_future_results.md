# Use std::packaged_task for deferred execution with future results

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/packaged_task>  

---

## Topic Overview

`std::packaged_task<R(Args...)>` wraps a callable and connects it to a `std::future<R>`. Unlike `std::async`, you control when and where the task executes. That extra degree of control is the whole point.

### How It Works

The lifecycle of a `packaged_task` is very deliberate: you create it, grab the future, and then at some later point - possibly on a completely different thread - you invoke it. Here is the four-step pattern:

```cpp
1. Create:     packaged_task<int(int)> task(compute);
2. Get future: auto fut = task.get_future();
3. Execute:    task(42);         // can be on any thread, at any time
4. Retrieve:   int r = fut.get(); // blocks until task completes
```

Steps 2 and 3 are deliberately separated. Between getting the future and calling the task, you can queue it, hand it off to a thread pool, delay it arbitrarily - whatever your architecture requires.

### async vs packaged_task vs promise

It's worth knowing where `packaged_task` sits on the spectrum of C++ futures. The three tools exist because different situations need different amounts of control:

```cpp
┌──────────────┬────────────────────────────────────────┐
│ std::async   │ Creates task + runs it immediately     │
│              │ (either on new thread or deferred)      │
│              │ Most convenient, least control          │
├──────────────┼────────────────────────────────────────┤
│ packaged_task│ Creates task, YOU decide when/where     │
│              │ to run it. Perfect for thread pools.    │
│              │ medium convenience, medium control      │
├──────────────┼────────────────────────────────────────┤
│ promise      │ YOU manually set the value/exception.   │
│              │ No callable - raw control.              │
│              │ Least convenient, most control          │
└──────────────┴────────────────────────────────────────┘
```

If `std::async` always starts the task for you (too eager), and `promise` requires you to manually produce the value (too low-level), `packaged_task` is the sweet spot for thread-pool style designs where you submit work and collect results later.

---

## Self-Assessment

### Q1: Wrap a callable in std::packaged_task and retrieve the result via its future after execution

This example shows three uses side by side: running on the current thread, running on a background thread, and wrapping a lambda. Notice that the task is always separate from the future - you can hold both independently.

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <functional>

int compute(int a, int b) {
    return a * a + b * b;
}

int main() {
    // === Basic: wrap a function ===
    std::packaged_task<int(int, int)> task(compute);
    auto fut = task.get_future();

    // Task hasn't run yet! We have full control.
    std::cout << "Task created, not yet executed\n";

    // Execute on current thread:
    task(3, 4); // calls compute(3, 4), stores result in shared state

    std::cout << "Result: " << fut.get() << "\n"; // 25

    // === Execute on another thread ===
    std::packaged_task<int(int, int)> task2(compute);
    auto fut2 = task2.get_future();

    // packaged_task is move-only (no copy)
    std::thread t(std::move(task2), 5, 12);

    std::cout << "Task2 running on thread...\n";
    std::cout << "Result2: " << fut2.get() << "\n"; // 169
    t.join();

    // === Wrap a lambda ===
    std::packaged_task<std::string()> task3([] {
        return std::string("Hello from packaged_task!");
    });
    auto fut3 = task3.get_future();
    task3();
    std::cout << fut3.get() << "\n";

    // Output:
    // Task created, not yet executed
    // Result: 25
    // Task2 running on thread...
    // Result2: 169
    // Hello from packaged_task!
}
```

The move-only nature of `packaged_task` is intentional: you can't accidentally copy a task and run it twice. When sending it to a thread, you have to `std::move` it in, which makes the ownership transfer explicit.

### Q2: Post a packaged_task to a thread pool and collect the future for later retrieval

This is where `packaged_task` really shines. The thread pool's `submit` method wraps your callable in a `packaged_task`, stores it in a queue, and returns the future to the caller. The workers then pick tasks off the queue and execute them - the future automatically fills when the task runs.

```cpp
#include <future>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <iostream>
#include <vector>

// Simple thread pool using packaged_task
class ThreadPool {
    std::vector<std::jthread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;

public:
    explicit ThreadPool(size_t n_threads) {
        for (size_t i = 0; i < n_threads; ++i) {
            workers_.emplace_back([this](std::stop_token) {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock lock(mtx_);
                        cv_.wait(lock, [&] { return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task();
                }
            });
        }
    }

    // Submit work, get a future back
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
        using R = decltype(f(args...));

        auto task = std::make_shared<std::packaged_task<R()>>(
            [f = std::forward<F>(f), ...args = std::forward<Args>(args)]() mutable {
                return f(args...);
            });

        auto fut = task->get_future();

        {
            std::lock_guard lock(mtx_);
            tasks_.push([task] { (*task)(); });
            // task is wrapped in shared_ptr because std::function requires copyable
            // but packaged_task is move-only
        }
        cv_.notify_one();
        return fut;
    }

    ~ThreadPool() {
        {
            std::lock_guard lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        // jthreads auto-join in destructor
    }
};

int main() {
    ThreadPool pool(4);

    // Submit tasks and collect futures
    std::vector<std::future<int>> futures;
    for (int i = 0; i < 10; ++i) {
        futures.push_back(pool.submit([](int x) {
            std::this_thread::sleep_for(std::chrono::milliseconds(20));
            return x * x;
        }, i));
    }

    // Collect results
    for (int i = 0; i < 10; ++i) {
        std::cout << i << "² = " << futures[i].get() << "\n";
    }

    // Output:
    // 0² = 0
    // 1² = 1
    // 2² = 4
    // 3² = 9
    // 4² = 16
    // 5² = 25
    // 6² = 36
    // 7² = 49
    // 8² = 64
    // 9² = 81
}
```

Notice the `shared_ptr` trick in `submit`: `packaged_task` is move-only, but `std::function` requires copyability. Wrapping the task in a `shared_ptr` sidesteps this - the lambda that captures the pointer is copyable, even though the task it points to is not.

### Q3: Explain the difference between packaged_task and std::async in terms of execution control

The comparison table below spells out the differences concisely. The key thing to internalize is that `std::async` is eager - it starts the work immediately - whereas `packaged_task` is passive until you explicitly invoke it.

```cpp
EXECUTION CONTROL COMPARISON
═════════════════════════════

                    std::async                  std::packaged_task
────────────────────────────────────────────────────────────────────
When it runs:       Immediately (async) or      When YOU invoke operator()
                    on get() (deferred)
Where it runs:      New thread or caller thread  Any thread you choose
Thread pool:        NO (creates new threads)     YES (queue + reuse threads)
Deferred execution: Only deferred policy         Natural - just delay the call
Exception:          Stored in future             Stored in future
Destructor blocks:  YES (async future joins)     NO
Move-only:          N/A (returns future)         YES (must std::move)
```

The code below shows the timing difference concretely. With `std::async`, the compute starts the moment you call it. With `packaged_task`, you can wait 200 ms before even deciding to run the task.

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <chrono>

using namespace std::chrono_literals;

int compute() {
    std::this_thread::sleep_for(100ms);
    return 42;
}

int main() {
    // === std::async: immediate execution, no control ===
    {
        auto start = std::chrono::steady_clock::now();
        auto fut = std::async(std::launch::async, compute);
        //                                        ^^^^^^^ runs NOW on new thread

        // Discarding the future BLOCKS! (destructor joins the thread)
        // std::async(std::launch::async, compute); // blocks here!

        auto result = fut.get();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start);
        std::cout << "async: " << result << " in " << ms.count() << "ms\n";
    }

    // === packaged_task: deferred execution, full control ===
    {
        auto start = std::chrono::steady_clock::now();
        std::packaged_task<int()> task(compute);
        auto fut = task.get_future();

        // Task NOT running yet - we control when it starts
        std::this_thread::sleep_for(200ms);

        // Run it now, on a specific thread
        std::thread t(std::move(task));
        t.join();

        auto result = fut.get();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start);
        std::cout << "packaged_task: " << result << " in " << ms.count() << "ms\n";
        // ~300ms (200ms delay + 100ms compute)
    }

    // === packaged_task: reset() to reuse ===
    {
        std::packaged_task<int()> task(compute);
        auto fut1 = task.get_future();
        task(); // first execution
        std::cout << "First: " << fut1.get() << "\n";

        task.reset(); // reset shared state - can get a new future
        auto fut2 = task.get_future();
        task(); // second execution
        std::cout << "Second: " << fut2.get() << "\n";
    }

    // Output:
    // async: 42 in 100ms
    // packaged_task: 42 in 300ms
    // First: 42
    // Second: 42
}
```

The `reset()` section at the bottom is worth remembering. After a task has run, you can call `reset()` to clear the shared state and get a fresh future - making it possible to run the same callable a second time through the same `packaged_task` object.

---

## Notes

- Move-only: `packaged_task` cannot be copied. Use `std::move` to pass into threads or queues. For `std::function` (which requires copyability), wrap in `shared_ptr`.
- `reset()` resets the shared state, allowing the task to be invoked again with a new future. The old future becomes broken (throws `broken_promise` if `.get()` not yet called).
- Exception handling: if the wrapped callable throws, the exception is stored in the shared state and rethrown on `fut.get()`.
- `void` return: `packaged_task<void(int)>` works for tasks with no return value. The future is `future<void>`.
- `packaged_task` is the standard building block for thread pools - `std::async` can't be queued or deferred.
- Compile with `-std=c++20 -O2 -pthread`.
