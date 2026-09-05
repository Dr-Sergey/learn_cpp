# Implement co_await for a custom awaitable type

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

Any type can be made **awaitable** by implementing three methods: `await_ready()`, `await_suspend()`, and `await_resume()`. The compiler uses these to control suspension and resumption at each `co_await` expression. You don't need to inherit from any interface - just provide those three methods, and `co_await yourType` will work.

### Awaitable Protocol

Here is the exact flow the compiler follows each time it hits a `co_await` expression. Step through this carefully the first time, because once it clicks it makes everything about custom awaitables obvious:

```cpp
co_await expr;

1. await_ready()  -- true -> skip suspension, go to step 3

       |
       false
       |

2. await_suspend(handle)

       |
       +- void: always suspend
       +- bool: suspend if true, resume if false
       +- coroutine_handle<>: symmetric transfer to that handle
       |
       [SUSPENDED -- wait for resume]
       |

3. await_resume()  -> result of co_await expression
```

Step 1 is a fast-path check - if the result is already available, skip the whole suspension machinery. Step 2 is where you hand off control to whatever async mechanism you're using (a thread pool, an I/O event loop, a timer, etc.). Step 3 is what the `co_await` expression evaluates to - the return value of `await_resume()` becomes the value assigned to the variable on the left of `=`.

### await_suspend Return Types

The return type of `await_suspend` is more flexible than people expect. Here is what each option does:

| Return type | Behavior |
| --- | --- |
| `void` | Always suspends |
| `bool` | Suspends if `true`, resumes immediately if `false` |
| `coroutine_handle<>` | Symmetric transfer to the returned handle |

The `bool` form is handy for "try-and-maybe-suspend" patterns - think of a mutex that might be uncontested. The `coroutine_handle<>` form (symmetric transfer) is an advanced technique that avoids stack growth when chaining coroutines.

---

## Self-Assessment

### Q1: Write an awaitable with `await_ready`, `await_suspend`, and `await_resume`

This is the simplest possible end-to-end example - a simulated network fetch. Every step of the awaitable protocol is printed so you can trace exactly when each method fires:

```cpp
#include <coroutine>
#include <iostream>
#include <string>

// Simple task for demonstration
struct Task {
    struct promise_type {
        Task get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
};

// Custom awaitable: simulates fetching a value
struct FetchAwaitable {
    std::string url;
    std::string result;  // will hold the "fetched" data

    bool await_ready() const noexcept {
        std::cout << "  await_ready: checking cache for " << url << '\n';
        return false;  // not ready, must suspend
    }

    void await_suspend(std::coroutine_handle<> handle) {
        std::cout << "  await_suspend: starting fetch for " << url << '\n';
        // Simulate async operation (in reality, dispatch to IO)
        result = "Data from " + url;
        // Resume immediately for this demo
        handle.resume();
    }

    std::string await_resume() {
        std::cout << "  await_resume: returning result\n";
        return std::move(result);  // this becomes the co_await result
    }
};

Task demo() {
    std::cout << "Before fetch\n";
    std::string data = co_await FetchAwaitable{"https://example.com"};
    std::cout << "Got: " << data << '\n';
}

int main() {
    demo();
}
// Expected output:
// Before fetch
//   await_ready: checking cache for https://example.com
//   await_suspend: starting fetch for https://example.com
//   await_resume: returning result
// Got: Data from https://example.com
```

Notice how the flow matches the protocol diagram exactly: `await_ready` returns `false`, so `await_suspend` runs, which resumes the coroutine, which triggers `await_resume`, whose return value becomes the string assigned to `data`. The three methods form one complete, composable unit.

### Q2: Show how `await_ready` returning `true` avoids suspension entirely

This example adds a cache layer. When the key is already cached, `await_ready` returns `true` and the entire suspension mechanism - `await_suspend`, the context switch, the resume - is skipped. You pay nothing for the fast path:

```cpp
#include <coroutine>
#include <iostream>
#include <optional>
#include <string>
#include <unordered_map>

struct Task {
    struct promise_type {
        Task get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
};

// Simulated cache
std::unordered_map<std::string, std::string> cache = {
    {"key1", "cached_value_1"},
    {"key3", "cached_value_3"},
};

// Awaitable that checks cache first
struct CachedFetch {
    std::string key;
    mutable std::optional<std::string> cached_result;

    bool await_ready() const noexcept {
        auto it = cache.find(key);
        if (it != cache.end()) {
            cached_result = it->second;
            std::cout << "  [" << key << "] Cache HIT - no suspension\n";
            return true;   // ready! skip await_suspend entirely
        }
        std::cout << "  [" << key << "] Cache MISS - must suspend\n";
        return false;      // not ready, will suspend
    }

    void await_suspend(std::coroutine_handle<> h) {
        std::cout << "  [" << key << "] Fetching...\n";
        cached_result = "fetched_" + key;  // simulate fetch
        h.resume();
    }

    std::string await_resume() {
        return std::move(*cached_result);
    }
};

Task demo() {
    auto v1 = co_await CachedFetch{"key1"};  // cache hit - no suspension
    std::cout << "v1 = " << v1 << '\n';

    auto v2 = co_await CachedFetch{"key2"};  // cache miss - suspends
    std::cout << "v2 = " << v2 << '\n';

    auto v3 = co_await CachedFetch{"key3"};  // cache hit
    std::cout << "v3 = " << v3 << '\n';
}

int main() {
    demo();
}
// Expected output:
//   [key1] Cache HIT - no suspension
// v1 = cached_value_1
//   [key2] Cache MISS - must suspend
//   [key2] Fetching...
// v2 = fetched_key2
//   [key3] Cache HIT - no suspension
// v3 = cached_value_3
```

When `await_ready()` returns `true`, `await_suspend()` is **never called**. The coroutine proceeds directly to `await_resume()` without suspending at all. This pattern is essential for caching: if the result is already there, don't pay the cost of suspension overhead.

### Q3: Implement a thread-hopping awaitable that resumes the coroutine on a different thread

This is where awaitables start doing genuinely interesting work. The trick is that `await_suspend` receives the coroutine's handle, and whoever calls `handle.resume()` is whoever the coroutine continues on. Hand that handle to a new thread, and the coroutine "hops":

```cpp
#include <coroutine>
#include <iostream>
#include <thread>

struct Task {
    struct promise_type {
        Task get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
};

// Awaitable that resumes the coroutine on a new thread
struct ResumeOnNewThread {
    bool await_ready() const noexcept { return false; }  // always suspend

    void await_suspend(std::coroutine_handle<> handle) const {
        // Launch a new thread and resume the coroutine there
        std::thread([handle]() {
            handle.resume();  // coroutine continues on THIS thread
        }).detach();
    }

    void await_resume() const noexcept {}  // no result
};

// Awaitable that resumes on a specific thread via a callback
struct ResumeOnThread {
    std::thread::id target_thread;  // not used in this demo
    std::thread* worker = nullptr;

    bool await_ready() const noexcept { return false; }

    void await_suspend(std::coroutine_handle<> handle) {
        // In a real implementation, post to the target thread's event queue
        worker = new std::thread([handle]() {
            handle.resume();
        });
    }

    void await_resume() {
        if (worker) { worker->join(); delete worker; worker = nullptr; }
    }
};

Task demo() {
    std::cout << "Starting on thread: " << std::this_thread::get_id() << '\n';

    co_await ResumeOnNewThread{};  // hop to a different thread

    std::cout << "Now on thread:     " << std::this_thread::get_id() << '\n';

    co_await ResumeOnNewThread{};  // hop again

    std::cout << "Now on thread:     " << std::this_thread::get_id() << '\n';
}

int main() {
    demo();
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
}
// Expected output (thread IDs will differ):
// Starting on thread: 1
// Now on thread:     2
// Now on thread:     3
```

`await_suspend` receives the handle and launches a new thread. That thread calls `handle.resume()`, so the coroutine picks up exactly where it left off - but now it's running on a different thread. Each `co_await ResumeOnNewThread{}` hops the execution context. In production code you'd use an executor or thread pool rather than raw `std::thread`, but the underlying mechanism is identical.

---

## Notes

- The compiler looks for `await_ready`/`await_suspend`/`await_resume` by name - no base class or interface needed.
- You can also make a type awaitable by providing `operator co_await()` as a member or free function, which is useful when you want to keep the awaitable protocol separate from the type itself.
- `await_suspend` returning `bool` is useful for "try to lock" patterns: suspend only if the lock isn't available. This avoids unnecessary context switches when the resource is free.
- **Thread safety:** be careful with `handle.resume()` - the coroutine must not be resumed concurrently from two places at once, as that is undefined behavior.
