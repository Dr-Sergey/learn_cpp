# Implement a thread-safe singleton using std::call_once

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/call_once>  

---

## Topic Overview

The **singleton pattern** ensures only one instance of a class ever exists. In single-threaded code that is trivial to implement. In multithreaded code, it requires care: if ten threads all call `getInstance()` simultaneously before the instance exists, you need to guarantee that exactly one of them creates it, the others wait, and all of them get the same object. C++11 provides two clean solutions for this: `std::call_once` and function-local statics (Meyer's Singleton). Both are correct and both are safe.

Before C++11, people tried to solve this with "double-checked locking," which turned out to be broken in subtle ways. Understanding why it was broken helps explain why the C++11 solutions work.

### Three Approaches

| Approach | Thread-safe? | Lazy? | Exception-safe? | Complexity |
| --- | --- | --- | --- | --- |
| Pre-C++11 double-checked locking | Broken (UB) | Yes | No | Error-prone |
| `std::call_once` + `once_flag` | Yes | Yes | Yes (retries) | Moderate |
| Function-local static (Meyer's) | Yes (C++11) | Yes | Yes | Simplest |

### Core Example with call_once

Here is the fundamental pattern. The `once_flag` tracks whether initialization has happened, and `call_once` ensures the lambda runs exactly once even when called from many threads simultaneously:

```cpp
#include <mutex>
#include <memory>
#include <iostream>

class Database {
    Database() { std::cout << "Database initialized\n"; }
public:
    static Database& getInstance() {
        static std::once_flag flag;
        static std::unique_ptr<Database> instance;

        std::call_once(flag, [] {
            instance.reset(new Database());
        });

        return *instance;
    }

    void query(const char* sql) { std::cout << "Query: " << sql << "\n"; }
};

int main() {
    Database::getInstance().query("SELECT 1");
    Database::getInstance().query("SELECT 2");
    // Output:
    // Database initialized   <- only once!
    // Query: SELECT 1
    // Query: SELECT 2
}
```

No matter how many threads call `getInstance()` at the same time, "Database initialized" prints exactly once.

---

## Self-Assessment

### Q1: Use std::once_flag and std::call_once to initialize a global resource exactly once across threads

**Answer:**

The pattern scales naturally to any number of threads. The 10-thread test below puts the initialization under pressure - every thread races to be the first one in:

```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>
#include <memory>
#include <string>

class Logger {
    std::string filename_;

    Logger(const std::string& file) : filename_(file) {
        std::cout << "Logger created: " << filename_ << "\n";
    }

public:
    static Logger& getInstance() {
        static std::once_flag init_flag;
        static std::unique_ptr<Logger> instance;

        std::call_once(init_flag, [] {
            // This lambda runs EXACTLY ONCE, even with 100 threads
            instance = std::unique_ptr<Logger>(new Logger("app.log"));
        });

        return *instance;
    }

    void log(const std::string& msg) {
        std::cout << "[" << filename_ << "] " << msg << "\n";
    }
};

int main() {
    // Launch 10 threads all trying to get the instance simultaneously
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([i] {
            Logger::getInstance().log("Thread " + std::to_string(i));
        });
    }

    for (auto& t : threads) t.join();
    // Output:
    // Logger created: app.log    <- ONCE, regardless of thread count
    // [app.log] Thread 0
    // [app.log] Thread 3
    // ... (order varies)

    // HOW call_once works:
    // 1. First thread to call call_once runs the callable
    // 2. Other threads block until the callable completes
    // 3. If callable throws, the flag is NOT set - next thread retries
    // 4. After successful completion, all subsequent calls are no-ops (fast path)
}
```

`std::once_flag` tracks whether initialization has occurred. `std::call_once` checks the flag - if not set, it runs the callable and sets the flag atomically. All concurrent callers block until the first one finishes. The fast path (flag already set) is extremely cheap - just an atomic load.

### Q2: Explain why double-checked locking without call_once was broken before C++11

**Answer:**

This is worth understanding deeply, because the brokenness is not obvious at first glance. The double-checked locking pattern looks reasonable: check the pointer without a lock (cheap), only lock if it is null (rare), check again inside the lock (necessary). What could go wrong?

The problem is that `instance = new BrokenSingleton()` is not one step. It is three steps: allocate memory, construct the object, assign the pointer. The compiler and CPU are both allowed to reorder those steps as long as the observable result in a single thread is the same. In a multi-threaded context without memory ordering guarantees, another thread can see the pointer assigned (non-null) before the object is fully constructed. Before C++11, there was no language-level memory model at all, so there was no way to prevent this:

```cpp
#include <mutex>
#include <iostream>

// === BROKEN pre-C++11 double-checked locking ===
// (DO NOT USE - shown for educational purposes only)

class BrokenSingleton {
    static BrokenSingleton* instance; // raw pointer
    static std::mutex mtx;

public:
    static BrokenSingleton* getInstance() {
        if (instance == nullptr) {          // (1) First check (no lock)
            std::lock_guard lock(mtx);
            if (instance == nullptr) {      // (2) Second check (with lock)
                instance = new BrokenSingleton(); // (3) Construct
            }
        }
        return instance;
    }
};

// WHY THIS IS BROKEN (pre-C++11):
//
// Step (3) `instance = new BrokenSingleton()` does THREE things:
//   a. Allocate memory
//   b. Construct the object in that memory
//   c. Assign the pointer to `instance`
//
// The compiler/CPU can REORDER these to: a -> c -> b
// (assign pointer BEFORE construction completes)
//
// Thread 1:                    Thread 2:
//   allocate memory (a)
//   assign pointer (c)         check: instance != nullptr <- sees non-null!
//   [not yet constructed!]     return instance <- USES UNCONSTRUCTED OBJECT = UB!
//   construct object (b)
//
// Pre-C++11, there was NO memory model - no way to prevent this reordering.
//
// With C++11, this could be fixed with:
//   std::atomic<Singleton*> instance;
//   instance.store(ptr, std::memory_order_release);  // (3)
//   instance.load(std::memory_order_acquire);         // (1)
// But this is complex and error-prone.

int main() {
    // The C++11 CORRECT alternatives:

    // Option 1: std::call_once (explicit, clear)
    // Option 2: Meyer's Singleton (simplest):
    struct GoodSingleton {
        static GoodSingleton& getInstance() {
            static GoodSingleton instance; // thread-safe since C++11!
            return instance;
        }
    };

    // C++11 guarantees that function-local statics are initialized
    // exactly once, even when called from multiple threads.
    // The compiler inserts the equivalent of call_once internally.

    auto& s = GoodSingleton::getInstance();
    (void)s;
    std::cout << "Use Meyer's Singleton or std::call_once\n";
}
```

C++11's memory model and `std::call_once` / function-local static guarantees eliminate this class of problem entirely. You do not need to think about memory ordering yourself - the language guarantees it for you.

### Q3: Compare call_once with a function-local static singleton for performance and correctness

**Answer:**

Both approaches are equally correct in C++11. The performance is nearly identical on the fast path (already initialized). The differences are about flexibility and lifetime management:

```cpp
#include <mutex>
#include <iostream>
#include <chrono>
#include <thread>
#include <vector>

// === Approach 1: std::call_once ===
class CallOnceSingleton {
    static std::once_flag flag_;
    static CallOnceSingleton* instance_;

    CallOnceSingleton() = default;
public:
    static CallOnceSingleton& getInstance() {
        std::call_once(flag_, [] {
            instance_ = new CallOnceSingleton();
        });
        return *instance_;
    }
    void use() {}
};
std::once_flag CallOnceSingleton::flag_;
CallOnceSingleton* CallOnceSingleton::instance_ = nullptr;

// === Approach 2: Meyer's Singleton (function-local static) ===
class MeyersSingleton {
    MeyersSingleton() = default;
public:
    static MeyersSingleton& getInstance() {
        static MeyersSingleton instance; // thread-safe since C++11
        return instance;
    }
    void use() {}
};

int main() {
    constexpr int ITERS = 10'000'000;

    // Benchmark call_once
    auto t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < ITERS; ++i)
        CallOnceSingleton::getInstance().use();
    auto t2 = std::chrono::steady_clock::now();

    // Benchmark Meyer's
    auto t3 = std::chrono::steady_clock::now();
    for (int i = 0; i < ITERS; ++i)
        MeyersSingleton::getInstance().use();
    auto t4 = std::chrono::steady_clock::now();

    auto co_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();
    auto ms_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t4 - t3).count();

    std::cout << "call_once: " << co_ms << " ms\n";
    std::cout << "Meyer's:   " << ms_ms << " ms\n";
    // Both are very fast - the "already initialized" path is just an atomic load.
    // Typically within ~5% of each other.

    // COMPARISON:
    // +----------------+------------------+--------------------+
    // |                | std::call_once   | Meyer's Singleton  |
    // +----------------+------------------+--------------------+
    // | Lines of code  | More (flag+ptr)  | 1 line             |
    // | Thread safety  | Yes              | Yes (C++11)        |
    // | Exception      | Retries on throw | Retries on throw   |
    // | Destruction    | Manual (leak/at) | Automatic (atexit) |
    // | Init args      | Flexible         | Fixed at call site |
    // | Non-local init | Yes              | No (must be local) |
    // | Performance    | ~Equal           | ~Equal             |
    // +----------------+------------------+--------------------+

    // RECOMMENDATION:
    // Use Meyer's Singleton (function-local static) by default.
    // Use call_once when:
    //   - Initialization arguments come from runtime (not hardcoded)
    //   - You need to initialize non-local resources (global arrays, etc.)
    //   - The singleton must NOT be destroyed at program exit
}
```

Both approaches compile to an atomic load plus branch on the fast path. The difference is negligible in practice. Prefer Meyer's Singleton for simplicity - it is literally one line. Use `call_once` when you need runtime arguments or non-local initialization.

---

## Notes

- Exception behavior: If the callable passed to `call_once` throws, the flag remains unset. The next thread calling `call_once` will retry - giving you automatic retry semantics for transient failures without any extra machinery.
- `once_flag` is not copyable or movable. It must be a static or global variable.
- Destruction order: Meyer's Singleton is destroyed in reverse construction order during program exit via `atexit`. `call_once` with `new` has no automatic destruction - this is intentional for singletons that must outlive `main`.
- Performance: Both paths compile to an atomic load plus branch on the fast path. The difference is negligible.
- Compile with `-std=c++11 -pthread`.
