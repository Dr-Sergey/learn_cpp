# Use std::thread correctly and always join or detach

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/thread>  

---

## Topic Overview

`std::thread` is the fundamental threading primitive in C++. Every `std::thread` object that represents an active thread **must** be either joined or detached before its destructor runs - failing to do so calls `std::terminate()` and crashes the program. This is not a subtle UB situation you might get away with; it is an immediate, hard termination.

### Thread Lifecycle

The lifecycle diagram below shows the two legal paths. Every joinable thread must go down one of them before the destructor fires.

```cpp
┌──────────────┐
│  std::thread  │  t(func, args...);
│  constructed  │
└──────┬───────┘
       │
       ▼  t.joinable() == true
       │
   ┌───┴────────────────────────┐
   │                            │
   ▼                            ▼
t.join()                   t.detach()
   │                            │
   │ blocks caller              │ thread runs independently
   │ until thread finishes      │ no longer associated with t
   │                            │
   ▼                            ▼
t.joinable() == false      t.joinable() == false
   │                            │
   ▼                            ▼
~thread() OK               ~thread() OK


If NEITHER join() nor detach() is called:
   ~thread() with joinable()==true  ->  std::terminate()  ->  CRASH
```

### Key Rules

Here is what you need to keep in mind when working with `std::thread`:

| Rule | Detail |
| --- | --- |
| **Always join or detach** | A joinable thread's destructor calls `std::terminate()` |
| **join() blocks** | Caller waits until the thread function returns |
| **detach() disconnects** | Thread runs independently; no way to wait for it later |
| **Move-only** | `std::thread` cannot be copied, only moved |
| **Arguments are copied** | By default, arguments are **copied** into the thread; use `std::ref()` to pass by reference |
| **Exceptions terminate** | An uncaught exception in a thread calls `std::terminate()` |
| **C++20: `std::jthread`** | Automatically joins on destruction + supports cooperative cancellation |

### Passing Arguments

Arguments to thread functions deserve special attention. The reason `std::ref` is required for pass-by-reference is that `std::thread` stores all arguments as copies internally - if you just write `std::thread t(func, x)`, `x` is always copied regardless of what `func`'s parameter type says. Using `std::ref` wraps the argument in a reference wrapper that survives the copy.

```cpp
#include <thread>
#include <iostream>
#include <string>
#include <functional>  // std::ref

void greet(const std::string& name) {
    std::cout << "Hello, " << name << "\n";
}

void increment(int& x) { ++x; }

int main() {
    // 1. Pass by value (safe - string is copied into the thread)
    std::string name = "Alice";
    std::thread t1(greet, name);
    t1.join();

    // 2. Pass by reference - MUST use std::ref
    int counter = 0;
    std::thread t2(increment, std::ref(counter));
    t2.join();
    std::cout << counter << "\n";  // 1

    // 3. Move into thread (transfers ownership)
    std::string data = "important";
    std::thread t3([d = std::move(data)] {
        std::cout << d << "\n";
    });
    t3.join();
    // data is now empty - moved into the thread
}
```

### `std::jthread` (C++20) - The Safer Alternative

If you are on C++20, prefer `std::jthread` over raw `std::thread`. It auto-joins on destruction, which makes forgetting to join a non-issue, and it supports cooperative cancellation via `std::stop_token`.

```cpp
#include <thread>
#include <iostream>

void worker(std::stop_token st) {
    while (!st.stop_requested()) {
        // do work...
    }
    std::cout << "Worker stopped\n";
}

int main() {
    std::jthread jt(worker);
    // No need for join() - auto-joins on destruction
    // Also supports cooperative cancellation via stop_token
}   // jt destructor: requests stop, then joins
```

### Important Notes

- Never access shared data from multiple threads without synchronization.
- Detached threads outlive their creator - ensure they don't access dangling references.
- Lambda captures by reference are dangerous if the thread outlives the captured variables.
- Always test with ThreadSanitizer (`-fsanitize=thread`) to detect data races.

---

## Self-Assessment

### Q1: Show that failing to join a joinable thread before destruction calls `std::terminate`

The bad example below is deliberately left commented out because running it terminates the program immediately. The good examples show the two correct patterns and the exception-safe variant, which is the one people most often forget about.

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void work() {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::cout << "Work done\n";
}

// BAD - this WILL call std::terminate
void bad_example() {
    std::thread t(work);
    // t goes out of scope while joinable -> std::terminate()!
}

// GOOD - always join
void good_join() {
    std::thread t(work);
    t.join();  // blocks until work() returns
    std::cout << "Joined successfully\n";
}

// GOOD - detach if you truly want fire-and-forget
void good_detach() {
    std::thread t(work);
    t.detach();  // thread runs independently
    // WARNING: cannot wait for it or check if it's done
}

// GOOD - exception-safe with try/catch
void exception_safe() {
    std::thread t(work);
    try {
        // do something that might throw
        throw std::runtime_error("oops");
    } catch (...) {
        t.join();  // MUST join even in error path
        throw;     // re-throw after joining
    }
    t.join();
}

int main() {
    good_join();
    good_detach();
    std::this_thread::sleep_for(std::chrono::milliseconds(200));  // let detached thread finish

    // Uncomment to see std::terminate:
    // bad_example();

    std::cout << "All done\n";
}
// Expected output:
//   Work done
//   Joined successfully
//   Work done     (from detached thread)
//   All done
```

Why does the destructor terminate instead of silently joining or detaching? This was a deliberate design choice. Silently joining could hide bugs - imagine an accidental join that blocks indefinitely. Silently detaching could create dangling reference problems. Neither is safe as a default, so the standard makes both explicit.

---

### Q2: Implement a thread guard RAII wrapper that joins on destruction

RAII is the right answer to the "always join" requirement. The code below shows two approaches: `ThreadGuard` is non-owning (it wraps a reference to an existing thread), and `ScopedThread` is owning (it takes the thread by move). C++20's `std::jthread` supersedes both, but it's useful to know how to build this yourself.

```cpp
#include <iostream>
#include <thread>
#include <stdexcept>

// RAII wrapper: guarantees join on destruction
class ThreadGuard {
    std::thread& t_;
public:
    explicit ThreadGuard(std::thread& t) : t_(t) {}

    ~ThreadGuard() {
        if (t_.joinable()) {
            t_.join();
        }
    }

    // Non-copyable, non-movable
    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;
};

// Better alternative: owning wrapper (takes ownership via move)
class ScopedThread {
    std::thread t_;
public:
    explicit ScopedThread(std::thread t) : t_(std::move(t)) {
        if (!t_.joinable()) {
            throw std::logic_error("No thread to guard");
        }
    }

    ~ScopedThread() {
        t_.join();  // always joinable - checked in constructor
    }

    ScopedThread(ScopedThread&&) = default;
    ScopedThread& operator=(ScopedThread&&) = default;
    ScopedThread(const ScopedThread&) = delete;
    ScopedThread& operator=(const ScopedThread&) = delete;
};

void do_work(int id) {
    std::cout << "Thread " << id << " working\n";
}

int main() {
    // Example 1: ThreadGuard (non-owning)
    {
        std::thread t(do_work, 1);
        ThreadGuard guard(t);
        // Even if an exception is thrown here, guard joins t
        throw std::runtime_error("test");  // guard still joins!
    }   // guard destructor joins t

    // Example 2: ScopedThread (owning)
    {
        ScopedThread st(std::thread(do_work, 2));
        // st owns the thread and joins on destruction
    }

    // Example 3: C++20 jthread - built-in RAII
    {
        std::jthread jt(do_work, 3);
        // auto-joins on destruction - no wrapper needed
    }

    std::cout << "All threads joined safely\n";
}
// Expected output:
//   Thread 1 working
//   Thread 2 working
//   Thread 3 working
//   All threads joined safely
```

**Comparison of Approaches:**

| Approach | Ownership | Pros | Cons |
| --- | --- | --- | --- |
| `ThreadGuard` (reference) | Non-owning | Simple, works with existing threads | Thread must outlive guard |
| `ScopedThread` (move) | Owning | Safer - no dangling reference | Pre-C++20 only |
| `std::jthread` (C++20) | Owning | Built-in, supports cancellation | Requires C++20 |

---

### Q3: Explain the difference between `join` and `detach` and when each is appropriate

The table below captures the practical differences. The core rule of thumb is: if your thread touches anything outside of itself (local variables it captured, shared state, output parameters), you should join. Detach is only safe when the thread is truly self-contained.

| Aspect | `join()` | `detach()` |
| --- | --- | --- |
| **Blocks caller?** | Yes - caller waits until thread completes | No - returns immediately |
| **Thread still tracked?** | No - `joinable()` becomes false | No - `joinable()` becomes false |
| **Can access result?** | Yes (via shared data, future, etc.) | Difficult - no synchronization point |
| **Resource cleanup** | Guaranteed - thread resources freed on join | Eventual - freed when thread finishes |
| **Reference safety** | Safe - caller waits, so captured refs valid | Dangerous - thread may outlive captured refs |
| **Exception safety** | Must still join on error paths | Once detached, no further responsibility |

**When to Use `join()`:**

```cpp
// 1. You need the thread's result
std::vector<int> results(8);
std::vector<std::thread> workers;
for (int i = 0; i < 8; ++i) {
    workers.emplace_back([&results, i] {
        results[i] = compute(i);  // each writes its own slot
    });
}
for (auto& w : workers) w.join();
// Now safe to read results[]

// 2. Orderly shutdown - must wait for threads to finish
// 3. Thread accesses local variables - must join before they go out of scope
```

**When to Use `detach()`:**

```cpp
// 1. Fire-and-forget background task (rare, be careful)
void log_async(std::string msg) {
    std::thread([msg = std::move(msg)] {
        write_to_file(msg);  // self-contained, no dangling refs
    }).detach();
}

// 2. Daemon-style threads that run for program lifetime
//    (even then, jthread + stop_token is usually better)
```

**Danger of Detach with Dangling References:**

```cpp
void DANGEROUS() {
    int local = 42;
    std::thread t([&local] {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << local << "\n";  // ERROR: local is destroyed!
    });
    t.detach();
}   // local destroyed here, but thread still running
```

**Best Practice Decision Tree:**

```cpp
Need to wait for thread's result?
  └─ YES -> join()
  └─ NO  -> Thread accesses any external references?
             └─ YES -> join() (must keep refs alive)
             └─ NO  -> Thread is truly self-contained?
                        └─ YES -> detach() is acceptable
                        └─ NO  -> join()

Best of all: use std::jthread (C++20) - auto-joins, supports cancellation.
```

---

## Notes

- `std::thread::hardware_concurrency()` returns the number of hardware threads (cores x hyperthreads). Returns 0 if unknown.
- Thread IDs: `std::this_thread::get_id()` and `t.get_id()` are useful for debugging and logging.
- `std::thread` with member functions: use `std::thread(&Class::method, &obj, args...)`.
- Moved-from thread: after `std::thread t2 = std::move(t1)`, `t1` is no longer joinable.
- Compile flags: `-pthread` on GCC/Clang; MSVC handles it automatically.
- Maximum threads: creating too many threads (thousands) wastes memory (~1-8 MB stack each). Use thread pools for many tasks.
