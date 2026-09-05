# Implement the Singleton pattern correctly in modern C++ (Meyers Singleton)

**Category:** OOP Design  
**Standard:** C++11  

---

## Topic Overview

The **Singleton** pattern ensures a class has exactly one instance with a global access point. In modern C++, Meyers' Singleton leverages the C++11 guarantee that local `static` variables are initialized exactly once, even under concurrent access (§6.7/4). This eliminates the need for manual double-checked locking, `std::call_once`, or pre-C++11 hacks. It's genuinely one of the cases where C++11 made something that was hard and error-prone completely trivial.

### Singleton Implementation Approaches

| Approach | Thread-Safe | Lazy | Destruction Order | Testability |
| --- | --- | --- | --- | --- |
| **Meyers' Singleton** (local static) | Yes (C++11+) | Yes | Reverse construction order | Poor without DI escape hatch |
| **Eager static member** | Yes (before main) | No | Static init order fiasco risk | Poor |
| `std::call_once` + `unique_ptr` | Yes | Yes | Must manage manually | Slightly better |
| **Double-checked locking** | If correct | Yes | Manual | Poor |
| **Dependency injection** (not singleton) | N/A | N/A | N/A | Excellent |

### Why Meyers' Singleton Works

```cpp
Thread A -> getInstance() -> static local init -> returns reference
Thread B -> getInstance() -> waits (compiler barrier) -> returns same reference
```

The compiler emits a guard variable and atomic check to ensure one-time initialization. On most platforms this compiles to a single atomic load on the fast path.

### When Singleton Is Appropriate

- **Hardware abstraction** - one GPU context, one serial port handle
- **Logging infrastructure** - single log sink coordinating output
- **Configuration registry** - read-once, used everywhere

### When to Avoid Singleton

- When you actually need **testability** - Singleton is global mutable state
- When **lifetime control** matters - destruction order of statics is fragile
- When **multiple instances** may be needed later (e.g., per-thread caches)

---

## Self-Assessment

### Q1: Implement Meyers' Singleton correctly and explain why pre-C++11 implementations were broken

Before C++11, there was no language-level guarantee that static local variable initialization was thread-safe. That forced developers into manual double-checked locking schemes, which are notoriously difficult to get right because the CPU and compiler are both allowed to reorder operations. C++11 fixed this by making it a language guarantee. Here's both the modern version and the broken old approach side-by-side, so you can see exactly what changed.

**Answer:**

```cpp
#include <iostream>
#include <string>
#include <mutex>

// === Modern Meyers' Singleton (C++11+) ===
class Logger {
public:
    static Logger& instance() {
        static Logger inst;  // Thread-safe since C++11
        return inst;
    }

    void log(const std::string& msg) {
        std::lock_guard lock(mtx_);  // Protect mutable state
        std::cout << "[LOG] " << msg << '\n';
    }

    // Delete copy/move to prevent duplication
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
    Logger(Logger&&) = delete;
    Logger& operator=(Logger&&) = delete;

private:
    Logger() { std::cout << "Logger constructed\n"; }
    ~Logger() { std::cout << "Logger destroyed\n"; }

    std::mutex mtx_;
};

// === Pre-C++11 BROKEN double-checked locking ===
// This is here for educational purposes - DO NOT USE
/*
class BrokenSingleton {
    static BrokenSingleton* ptr_;
    static std::mutex mtx_;
public:
    static BrokenSingleton* instance() {
        if (!ptr_) {                    // Race: ptr_ may be non-null
            std::lock_guard l(mtx_);    // but object not fully constructed
            if (!ptr_)
                ptr_ = new BrokenSingleton();  // Reordering possible!
        }
        return ptr_;
    }
};
// The compiler/CPU may reorder: allocate memory -> assign ptr_ -> construct.
// Thread B sees non-null ptr_ and uses an unconstructed object.
*/

int main() {
    Logger::instance().log("Application started");
    Logger::instance().log("Processing...");
    // Exactly one construction, one destruction
}
```

Key points about what changed between C++03 and C++11:

- Pre-C++11: there was no language guarantee on static local thread safety - two threads could race into the initialization and both try to construct the object.
- Pre-C++11 double-checked locking is broken because of memory model issues - the CPU is allowed to reorder the assignment to `ptr_` before the constructor runs, so another thread can see a non-null pointer to an unconstructed object.
- C++11 §6.7/4 guarantees that concurrent callers block until initialization completes - the language itself inserts the barrier for you.
- You still need a mutex for mutable operations on the singleton's *data* after it's constructed - the thread-safety guarantee only covers the initialization itself.

### Q2: What are the key trade-offs of Singleton, and how can you mitigate testability problems

**Answer:**

**Trade-offs:**

| Concern | Problem | Mitigation |
| --- | --- | --- |
| **Global state** | Hidden dependencies, hard to reason about | Document clearly; minimize mutable state |
| **Testability** | Can't substitute mock in unit tests | DI escape hatch or policy template |
| **Destruction order** | Static destruction order fiasco | Leak intentionally, or use `atexit` ordering |
| **Thread safety of data** | Instance creation is safe; data access is NOT | Protect mutable members with mutex |
| **Lifetime coupling** | Lives until program exit | Consider scoped singleton (shared_ptr) |

**DI Escape Hatch for testing:**

One practical way to get testability back is the Service Locator approach: the Singleton has a production implementation but also accepts an override pointer that tests can inject. This is less elegant than pure dependency injection but much more practical when you're dealing with existing code that uses Singletons everywhere.

```cpp
// Abstract interface for testability
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& msg) = 0;
};

class ProductionLogger : public ILogger {
public:
    void log(const std::string& msg) override {
        std::cout << "[PROD] " << msg << '\n';
    }
};

// Singleton accessor with override capability
class LoggerLocator {
public:
    static ILogger& get() {
        return override_ ? *override_ : default_instance();
    }

    // Test-only: inject a mock
    static void set(ILogger* mock) { override_ = mock; }
    static void reset() { override_ = nullptr; }

private:
    static ProductionLogger& default_instance() {
        static ProductionLogger inst;
        return inst;
    }
    inline static ILogger* override_ = nullptr;  // C++17 inline static
};

// In tests:
class MockLogger : public ILogger {
public:
    std::vector<std::string> messages;
    void log(const std::string& msg) override { messages.push_back(msg); }
};

void test_something() {
    MockLogger mock;
    LoggerLocator::set(&mock);

    // Code under test uses LoggerLocator::get().log(...)
    LoggerLocator::get().log("test message");
    assert(mock.messages.size() == 1);

    LoggerLocator::reset();
}
```

### Q3: Show how to handle the static destruction order fiasco and implement a "phoenix" singleton

The static destruction order fiasco is a real problem: if two singletons depend on each other and they're destroyed in the wrong order, the one that goes first might call the other during destruction - and that other object no longer exists. The compiler makes no guarantees about the order in which static objects across different translation units are destroyed.

**Answer:**

```cpp
#include <iostream>
#include <cstdlib>
#include <new>

// The problem: if Singleton A's destructor uses Singleton B,
// but B was destroyed first -> undefined behavior.

// === Solution 1: Leak intentionally (often the best choice) ===
class LeakingSingleton {
public:
    static LeakingSingleton& instance() {
        static LeakingSingleton* inst = new LeakingSingleton();
        // Never deleted - OS reclaims memory at process exit.
        // Safe: no destruction order issues.
        return *inst;
    }
private:
    LeakingSingleton() = default;
};

// === Solution 2: Phoenix Singleton (Alexandrescu) ===
// If destroyed, recreates itself on next access.
class PhoenixSingleton {
public:
    static PhoenixSingleton& instance() {
        if (!instance_) {
            if (destroyed_) {
                // Resurrect: placement-new into same storage
                new (&storage_) PhoenixSingleton();
                instance_ = reinterpret_cast<PhoenixSingleton*>(&storage_);
                destroyed_ = false;
                std::atexit(destroy);  // Re-register destruction
            } else {
                create();
            }
        }
        return *instance_;
    }

    void doWork() { std::cout << "PhoenixSingleton working\n"; }

private:
    PhoenixSingleton() { std::cout << "Phoenix constructed\n"; }
    ~PhoenixSingleton() {
        std::cout << "Phoenix destroyed\n";
        instance_ = nullptr;
        destroyed_ = true;
    }

    static void create() {
        new (&storage_) PhoenixSingleton();
        instance_ = reinterpret_cast<PhoenixSingleton*>(&storage_);
        std::atexit(destroy);
    }

    static void destroy() {
        instance_->~PhoenixSingleton();
    }

    inline static PhoenixSingleton* instance_ = nullptr;
    inline static bool destroyed_ = false;
    alignas(PhoenixSingleton) inline static char storage_[sizeof(PhoenixSingleton)]{};
};

// === Solution 3: Nifty Counter (used by <iostream>) ===
// Ensures construction before first use and destruction after last use
// by embedding a counter in every TU that includes the header.
// std::cout uses this pattern - that's why it works in static destructors.

// === Preferred modern alternative: just don't use Singleton ===
// Pass dependencies explicitly:
class Application {
    Logger& logger_;  // Injected, not globally fetched
public:
    explicit Application(Logger& logger) : logger_(logger) {}
    void run() { logger_.log("Running"); }
};

int main() {
    PhoenixSingleton::instance().doWork();
}
```

**Recommendation hierarchy:**

1. **Don't use Singleton** - pass dependencies explicitly
2. If you must: **Meyers' Singleton** with DI escape hatch for testing
3. If destruction order matters: **intentional leak** (simplest, safest)
4. If resurrection needed: **Phoenix Singleton** (complex, rarely justified)

---

## Notes

- Meyers' Singleton is thread-safe for *initialization* only - protect mutable state separately.
- The `static` local guarantee is in C++11 §6.7/4 (now §9.7/4 in C++23).
- `std::call_once` is an alternative but offers no advantages over Meyers' Singleton for most cases.
- Intentional leaking avoids destruction order fiasco and is used in production (e.g., Google's Abseil).
- Consider `std::shared_ptr`-based "scoped singleton" when you need deterministic lifetime.
- The Service Locator pattern (shown in Q2) is a pragmatic middle ground between pure DI and raw Singleton.
