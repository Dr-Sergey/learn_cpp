# Use the async_scope pattern for structured concurrency in async code

**Category:** Design Patterns Modern Takes  
**Standard:** Not version-specific  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

**Structured concurrency** guarantees that all spawned async work completes before the parent scope exits - no dangling tasks, no fire-and-forget leaks. The `async_scope` pattern implements this by collecting spawned tasks and joining them at scope exit. This mirrors C++20's `std::jthread` automatic-join for threads but applies to async task graphs.

The core problem with unstructured concurrency is lifetime. When you `std::async` a task and drop the future, the task may still be running and holding references to local variables - variables that may be destroyed before the task finishes. The `async_scope` pattern makes lifetime impossible to get wrong: tasks cannot outlive the scope that created them.

### Unstructured vs Structured Concurrency

Here's the problem in miniature. With `std::async`, it's easy to accidentally let a task outlive its creating scope. With `async_scope`, the destructor takes care of the join - you simply cannot forget it.

```cpp
Unstructured (std::async):           Structured (async_scope):
  auto f1 = std::async(task1);        async_scope scope;
  auto f2 = std::async(task2);        scope.spawn(task1);
  // What if we forget f1.get()?      scope.spawn(task2);
  // Task may outlive caller!         // scope destructor waits for ALL tasks
  return f2.get();                    co_await scope.join();
                                      // Guaranteed: all tasks done here
```

---

## Self-Assessment

### Q1: Use async_scope to ensure all spawned async tasks complete before the scope is destroyed

The `AsyncScope` class below is a simple but complete implementation. `spawn` fires a task and stores its future. The destructor calls `join`, which blocks until every stored future is done. The mutex protects the futures vector from concurrent access if `spawn` is called from multiple threads.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <future>
#include <functional>
#include <mutex>

// Simple async_scope implementation
class AsyncScope {
    std::vector<std::future<void>> tasks_;
    std::mutex mtx_;

public:
    // Spawn a task into the scope
    template<typename F>
    void spawn(F&& func) {
        std::lock_guard lock(mtx_);
        tasks_.push_back(std::async(std::launch::async, std::forward<F>(func)));
    }

    // Wait for ALL spawned tasks to complete
    void join() {
        std::vector<std::future<void>> to_join;
        {
            std::lock_guard lock(mtx_);
            to_join = std::move(tasks_);
        }
        for (auto& f : to_join) {
            f.get();  // Blocks until task completes
        }
    }

    // Destructor guarantees all tasks are joined
    ~AsyncScope() {
        join();  // Structured: NEVER let tasks outlive the scope
    }

    size_t pending() {
        std::lock_guard lock(mtx_);
        return tasks_.size();
    }
};

int main() {
    std::mutex print_mtx;

    {
        AsyncScope scope;

        scope.spawn([&] {
            std::lock_guard lock(print_mtx);
            std::cout << "Task 1 running on thread "
                      << std::this_thread::get_id() << '\n';
        });

        scope.spawn([&] {
            std::lock_guard lock(print_mtx);
            std::cout << "Task 2 running on thread "
                      << std::this_thread::get_id() << '\n';
        });

        scope.spawn([&] {
            std::lock_guard lock(print_mtx);
            std::cout << "Task 3 running on thread "
                      << std::this_thread::get_id() << '\n';
        });

        // scope destructor waits for all 3 tasks
    }
    std::cout << "All tasks guaranteed complete here\n";
}
```

The closing brace of the block is all you need. When `scope` goes out of scope, its destructor runs `join()`, and that blocks until all three tasks print their messages. There is no way to reach the final `std::cout` line before all tasks are done.

### Q2: Show a use case: spawning N parallel async tasks and waiting for all results

A common pattern is splitting expensive work into chunks, running them in parallel, and collecting the results. `ResultScope<T>` is the result-collecting variant of `AsyncScope` - it stores `future<T>` instead of `future<void>` and returns all results when you call `join_all`.

**Answer:**

```cpp
#include <iostream>
#include <vector>
#include <future>
#include <numeric>
#include <cmath>
#include <mutex>

// Result-collecting async scope
template<typename T>
class ResultScope {
    std::vector<std::future<T>> futures_;
public:
    template<typename F>
    void spawn(F&& func) {
        futures_.push_back(std::async(std::launch::async, std::forward<F>(func)));
    }

    // Collect all results (blocks until all complete)
    std::vector<T> join_all() {
        std::vector<T> results;
        results.reserve(futures_.size());
        for (auto& f : futures_) {
            results.push_back(f.get());
        }
        futures_.clear();
        return results;
    }
};

// Expensive computation
double compute_chunk(int start, int count) {
    double sum = 0;
    for (int i = start; i < start + count; ++i) {
        sum += std::sqrt(static_cast<double>(i));
    }
    return sum;
}

int main() {
    constexpr int TOTAL = 10'000'000;
    constexpr int N_TASKS = 8;
    constexpr int CHUNK = TOTAL / N_TASKS;

    ResultScope<double> scope;

    // Spawn N parallel tasks
    for (int i = 0; i < N_TASKS; ++i) {
        int start = i * CHUNK;
        scope.spawn([start, CHUNK] {
            return compute_chunk(start, CHUNK);
        });
    }

    // Wait for all and reduce
    auto results = scope.join_all();
    double total = std::accumulate(results.begin(), results.end(), 0.0);
    std::cout << "Sum of sqrt(0.." << TOTAL << ") = " << total << '\n';
    std::cout << "Computed in " << N_TASKS << " parallel chunks\n";
}
```

`join_all()` calls `.get()` on each future in order, which means it collects results in spawn order regardless of which task finishes first. All eight chunks run in parallel while `join_all()` waits, then the main thread reduces the partial sums.

### Q3: Compare async_scope with std::async futures and explain the structured lifetime guarantee

This example puts the problem and solution side by side. The `unstructured_example` shows the subtle gotcha with raw `std::async` futures: a `std::async` future that is not `.get()`-ed will block in its destructor - which is surprising, but at least safe. The real danger is `std::packaged_task` or detached threads, where the task truly continues running after the future is destroyed.

**Answer:**

```cpp
#include <iostream>
#include <future>
#include <vector>
#include <thread>
#include <chrono>

// PROBLEM: Unstructured std::async
void unstructured_example() {
    std::cout << "=== Unstructured (std::async) ===\n";

    {
        // Danger: forgetting to call .get() on the future
        auto f = std::async(std::launch::async, [] {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            std::cout << "  Task finished\n";
            return 42;
        });
        // If we forget f.get() here, the future's destructor BLOCKS
        // This is a subtle gotcha - std::async futures block on destruction
        // But std::packaged_task futures do NOT block - task may outlive scope!
    }
    // f.get() called implicitly by destructor - but this is implementation-specific
}

// SOLUTION: Structured concurrency
class AsyncScope {
    std::vector<std::future<void>> tasks_;
public:
    template<typename F>
    void spawn(F&& f) {
        tasks_.push_back(std::async(std::launch::async, std::forward<F>(f)));
    }
    void join() {
        for (auto& f : tasks_) f.get();
        tasks_.clear();
    }
    ~AsyncScope() { join(); }
};

void structured_example() {
    std::cout << "=== Structured (async_scope) ===\n";

    {
        AsyncScope scope;
        scope.spawn([] { std::cout << "  Task A done\n"; });
        scope.spawn([] { std::cout << "  Task B done\n"; });
        scope.spawn([] { std::cout << "  Task C done\n"; });
        // Destructor joins ALL tasks - no leaks possible
    }
    std::cout << "  All tasks guaranteed done\n";
}

int main() {
    unstructured_example();
    structured_example();

    /*
    Comparison:

    std::async + future:
    - Easy to forget .get() -> potential resource leak
    - std::async futures block in destructor (surprising behavior)
    - No central collection point for multiple tasks
    - Each task needs a separate future variable

    async_scope:
    - Destructor guarantees ALL tasks complete (structured lifetime)
    - Single object manages N tasks
    - Impossible to forget join - destructor handles it
    - Composable: scopes can nest (child scope joins before parent)
    - Foundation for P2300 std::execution proposal

    The key guarantee:
      "No task outlives its creating scope"
      This eliminates dangling references, use-after-free in callbacks,
      and fire-and-forget resource leaks.
    */
}
```

The key insight in the comment is worth reading twice: "No task outlives its creating scope." When that guarantee holds, you can safely capture local variables by reference in your tasks, just like you would in a for loop. Without it, every lambda capture is a potential use-after-free waiting to happen.

---

## Notes

- `async_scope` is not yet in the standard but is part of the P2300 (`std::execution`) proposal, which is targeting a future C++ standard.
- The NVIDIA stdexec library provides `exec::async_scope` for production use today.
- `std::jthread` provides structured concurrency for threads; `async_scope` extends this idea to async task graphs.
- Always prefer structured concurrency - it makes async code as safe as scoped variables, and reasoning about lifetimes becomes straightforward.
