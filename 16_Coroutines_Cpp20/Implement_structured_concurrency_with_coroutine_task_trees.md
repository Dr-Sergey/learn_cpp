# Implement structured concurrency with coroutine task trees

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

### What Is Structured Concurrency

**Structured concurrency** ensures that concurrent tasks follow the same scoping rules as structured programming: a parent task does not complete until all its child tasks have completed (or been cancelled). Think of it as the concurrency equivalent of the block scope rule - just as a local variable can't outlive the function that owns it, a child task can't outlive the parent that spawned it.

Without this guarantee, you get three classic bugs that are very hard to track down:

- Fire-and-forget tasks that leak resources.
- Dangling references from parent scope to child tasks.
- Unhandled exceptions in detached tasks.

### The Problem with Unstructured Concurrency

Here is what unstructured concurrency looks like, and why it causes problems. The parent finishes before its child does, leaving the child task orphaned with no owner to propagate its exceptions or reclaim its resources:

```cpp
// Unstructured: parent returns before child completes
task<void> parent() {
    auto child = spawn(some_work());  // fire and forget
    co_return;  // parent done -- but child is still running!
    // Who owns child? Who catches its exceptions?
}
```

### Structured Concurrency with Task Trees

In a structured model, the parent **awaits** all children before it returns. No child can outlive its parent scope - the hierarchy enforces that automatically. Here is the contrast:

```cpp
task<void> parent() {
    // when_all waits for ALL children before parent continues
    co_await when_all(
        child_task_1(),
        child_task_2(),
        child_task_3()
    );
    // Here: all children are done, exceptions propagated
    co_return;
}
```

The `when_all` call is the key - the parent is suspended until every child has finished or been cancelled. This one mechanism prevents the entire class of orphan-task bugs.

### Implementing a Basic Task Nursery / Scope

A nursery (or scope) is the object that owns a group of child tasks and enforces the structured rule. Here is a simplified sketch of what one looks like internally:

```cpp
#include <coroutine>
#include <vector>
#include <exception>
#include <iostream>

// Forward declarations (simplified)
template<typename T> struct task;

struct task_scope {
    struct scope_promise;
    using handle_type = std::coroutine_handle<scope_promise>;

    std::vector<std::coroutine_handle<>> children_;
    std::exception_ptr first_error_;

    // Spawn a child task into this scope
    template<typename T>
    void spawn(task<T> child) {
        children_.push_back(child.release_handle());
    }

    // Wait for all children -- called by the parent coroutine
    auto join() {
        struct join_awaiter {
            task_scope& scope;

            bool await_ready() { return scope.children_.empty(); }

            void await_suspend(std::coroutine_handle<> parent) {
                // Resume each child, set continuation to parent
                for (auto& child : scope.children_) {
                    child.resume();
                }
            }

            void await_resume() {
                if (scope.first_error_)
                    std::rethrow_exception(scope.first_error_);
            }
        };
        return join_awaiter{*this};
    }
};
```

The reason this works is that `join()` returns an awaiter whose `await_suspend` holds onto the parent coroutine's handle. Once all children complete, they resume the parent - and only then does the parent continue past the `co_await scope.join()` line.

### Cancellation in Structured Concurrency

Real structured concurrency goes further than just waiting - it also handles partial failure. When one child throws, you don't want the other children to keep running indefinitely. The scope coordinates cancellation via stop tokens so everyone cleans up:

```cpp
task<void> robust_parent() {
    auto scope = make_task_scope();

    scope.spawn(download_file("a.txt"));
    scope.spawn(download_file("b.txt"));
    scope.spawn(download_file("c.txt"));

    try {
        co_await scope.join();
        // All downloads succeeded
    } catch (const std::exception& e) {
        // One child failed -- scope automatically cancelled others
        std::cerr << "Download failed: " << e.what() << "\n";
        // All children are guaranteed completed/cancelled here
    }
    // Scope destructor verifies all children are done
}
```

The key guarantee: even inside the `catch` block, all sibling tasks are already done. You never exit this function with a task still running in the background.

### Library Support

Most production C++ async libraries provide structured concurrency primitives out of the box. You usually don't implement a nursery from scratch:

| Library | Structured Concurrency Primitive |
| --- | --- |
| folly::coro | `AsyncScope`, `collectAll`, `collectAny` |
| libunifex | `when_all`, `let_value`, `async_scope` |
| stdexec | `when_all`, `async_scope` (P2300) |
| cppcoro | `when_all` (basic) |

### Example with cppcoro-style `when_all`

Here is the practical pattern you'll write most often - collecting a batch of tasks into a vector and running them all concurrently with `when_all`:

```cpp
task<void> process_batch(std::vector<std::string> urls) {
    // Structured: parent waits for ALL children
    std::vector<task<Response>> tasks;
    for (auto& url : urls) {
        tasks.push_back(fetch(url));
    }

    // when_all ensures all tasks complete before we continue
    auto results = co_await when_all(std::move(tasks));

    for (auto& response : results) {
        process(response);
    }
    // All tasks done -- no leaks, no dangling references
}
```

After the `co_await when_all(...)` line, every task is finished. You can safely walk `results` knowing the data is fully materialized.

---

## Self-Assessment

### Q1: Explain why structured concurrency prevents resource leaks

In structured concurrency, every spawned task is **owned by a scope** that guarantees completion before the scope exits. This is a strong guarantee, and it rules out an entire category of resource-management bugs in one shot:

1. **No orphaned tasks** - every task has a parent that waits for it.
2. **Exception propagation** - child exceptions are caught by the parent scope.
3. **Deterministic cleanup** - resources are released in reverse order of acquisition.
4. **No "fire and forget"** - you cannot accidentally ignore a task's result.

The reason this matters so much is that in unstructured async code, any `co_return` might leave background tasks still holding references to stack variables that have already been destroyed. Structured concurrency makes that class of bug structurally impossible.

```cpp
task<void> no_leak() {
    auto scope = make_scope();
    scope.spawn(open_connection());     // child 1
    scope.spawn(allocate_resource());   // child 2
    co_await scope.join();              // both done -- resources cleaned up
}
```

### Q2: Implement `when_all` for two tasks

Here is a simplified `when_all` for exactly two tasks of different types. Real library implementations generalize this to variadic packs, but the core idea is the same - start both tasks, then await each one in turn:

```cpp
template<typename T, typename U>
task<std::pair<T, U>> when_all(task<T> a, task<U> b) {
    // Start both tasks
    auto handle_a = a.start();
    auto handle_b = b.start();

    // Await both (simplified -- real impl uses continuations)
    T result_a = co_await handle_a;
    U result_b = co_await handle_b;

    co_return std::make_pair(std::move(result_a), std::move(result_b));
}

// Usage:
task<void> example() {
    auto [count, name] = co_await when_all(
        get_count(),     // task<int>
        get_name()       // task<std::string>
    );
    std::cout << name << ": " << count << "\n";
}
```

### Q3: How does cancellation work in a task tree

Picture a tree where parent spawned three children and one of them throws:

```cpp
Parent
+-- Child A (running)
+-- Child B (FAILED with exception)
+-- Child C (running)
```

When **Child B** fails, here is the exact sequence the structured concurrency scope carries out:

1. The scope catches the exception.
2. The scope sends a **cancellation request** (via stop tokens) to Child A and Child C.
3. Child A and Child C check the stop token at their next `co_await` and exit early.
4. The scope waits for A and C to acknowledge cancellation.
5. The scope re-throws Child B's exception to the parent.

The reason this design works is that stop tokens give every child a way to cooperatively check "should I keep going?" at each suspension point. No child is abruptly killed - each one gets a chance to clean up its own resources before the parent receives the exception.

---

## Notes

- Structured concurrency is a core principle of P2300 (`std::execution`), which is targeting C++26 as the standard async model for C++.
- Think of task scopes like lexical scopes: child tasks cannot outlive their parent scope, for exactly the same reason local variables can't outlive the function that owns them.
- Always prefer `when_all` over manual spawn-and-forget patterns. The small extra cost at the join point is nothing compared to the bugs it prevents.
- C++26's `std::execution` will provide a standard `async_scope` for structured concurrency, making this pattern part of the language rather than a library convention.
