# Implement a thread-safe singleton using std::call_once (Part 2)

**Category:** Concurrency and Parallelism  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/call_once>  

---

## Topic Overview

This is a continuation of the `std::call_once` singleton topic, focusing on **advanced patterns**: parameterized initialization, exception recovery, and integration with dependency injection. If you have not read the fundamentals (Item #298) yet, start there.

### Advanced call_once Patterns

The two patterns below address situations where Meyer's Singleton is not quite enough. The first shows how to pass runtime arguments to a singleton constructor - the key detail is that the first caller's arguments win. The second shows that `call_once` has automatic retry-on-exception semantics built in:

```cpp
#include <mutex>
#include <memory>
#include <string>
#include <iostream>
#include <thread>
#include <vector>

// === Pattern 1: Parameterized singleton initialization ===
// (First caller's arguments "win" - subsequent calls use the same instance)

class ConfigManager {
    std::string config_path_;
    int verbosity_;

    ConfigManager(std::string path, int verbosity)
        : config_path_(std::move(path)), verbosity_(verbosity)
    {
        std::cout << "ConfigManager(" << config_path_ << ", " << verbosity_ << ")\n";
    }

public:
    static ConfigManager& getInstance(const std::string& path = "", int verbosity = 0) {
        static std::once_flag flag;
        static std::unique_ptr<ConfigManager> instance;

        std::call_once(flag, [&] {
            instance.reset(new ConfigManager(path, verbosity));
        });

        return *instance;
    }

    void print() const {
        std::cout << "Config: " << config_path_ << " (v=" << verbosity_ << ")\n";
    }
};

// === Pattern 2: Exception recovery with call_once ===
class NetworkClient {
    static std::once_flag flag_;
    static std::unique_ptr<NetworkClient> instance_;
    static int attempt_;

    NetworkClient(int attempt) {
        if (attempt < 3) throw std::runtime_error("Connection failed");
        std::cout << "Connected on attempt " << attempt << "\n";
    }

public:
    static NetworkClient& getInstance() {
        std::call_once(flag_, [] {
            // If this throws, the flag is NOT set - next caller retries!
            instance_.reset(new NetworkClient(attempt_++));
        });
        return *instance_;
    }
};
std::once_flag NetworkClient::flag_;
std::unique_ptr<NetworkClient> NetworkClient::instance_;
int NetworkClient::attempt_ = 1;

int main() {
    // Parameterized init - first caller sets the config
    auto& config = ConfigManager::getInstance("/etc/app.conf", 3);
    config.print();
    // Output: ConfigManager(/etc/app.conf, 3)
    //         Config: /etc/app.conf (v=3)

    // Second call - same instance, arguments ignored
    auto& config2 = ConfigManager::getInstance("other.conf", 0);
    config2.print();
    // Output: Config: /etc/app.conf (v=3) - first call's args persisted

    // Exception recovery
    for (int i = 0; i < 5; ++i) {
        try {
            auto& client = NetworkClient::getInstance();
            (void)client;
            break;
        } catch (const std::runtime_error& e) {
            std::cout << "Attempt failed: " << e.what() << "\n";
        }
    }
    // Output:
    // Attempt failed: Connection failed
    // Attempt failed: Connection failed
    // Connected on attempt 3
}
```

The exception recovery behavior is particularly useful: if your initialization code can fail transiently (a network connection, a file that has not appeared yet), `call_once` gives you automatic retry without any extra state tracking.

---

## Self-Assessment

### Q1: Use std::once_flag and std::call_once to initialize a global resource exactly once across threads

**Answer:**

This example uses a real file-backed log to show a resource that genuinely needs one-time initialization. Eight worker threads all try to get the logger simultaneously - only one will create it:

```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>
#include <fstream>
#include <string>

// Global resource: a shared log file
class SharedLog {
    std::ofstream file_;
    std::mutex write_mtx_; // protects writes (separate from init)

    static std::once_flag init_flag_;
    static SharedLog* instance_;

    SharedLog() {
        // Expensive initialization - only happens once
        file_.open("shared.log", std::ios::app);
        std::cout << "Log file opened (thread "
                  << std::this_thread::get_id() << ")\n";
    }

public:
    static SharedLog& get() {
        std::call_once(init_flag_, [] {
            instance_ = new SharedLog();
        });
        return *instance_;
    }

    void write(const std::string& msg) {
        std::lock_guard lock(write_mtx_); // thread-safe writes
        file_ << msg << "\n";
    }
};

std::once_flag SharedLog::init_flag_;
SharedLog* SharedLog::instance_ = nullptr;

int main() {
    std::vector<std::thread> workers;
    for (int i = 0; i < 8; ++i) {
        workers.emplace_back([i] {
            SharedLog::get().write("Worker " + std::to_string(i) + " started");
        });
    }
    for (auto& t : workers) t.join();
    // "Log file opened" printed exactly ONCE
}
```

### Q2: Explain why double-checked locking without call_once was broken before C++11

**Answer:** See the main topic file (Item #298) Q2 for the detailed explanation. The core issue: without a C++ memory model, the compiler and CPU could reorder the pointer assignment before object construction completed, causing other threads to see a non-null pointer to an unconstructed object - undefined behavior.

### Q3: Compare call_once with a function-local static singleton for performance and correctness

**Answer:**

Both approaches use the same underlying mechanism - an atomic guard variable plus blocking for concurrent initialization. The question is when each is the right tool to reach for:

```cpp
#include <iostream>

// Both approaches use the same underlying mechanism:
// an atomic guard variable + blocking for concurrent init.

// WHEN TO CHOOSE call_once:
// 1. You need to pass RUNTIME arguments to the constructor
//    (function-local statics use fixed args at the declaration site)
//
// 2. You want to initialize MULTIPLE related globals atomically:
//    std::call_once(flag, [] {
//        global_a = init_a();
//        global_b = init_b(global_a);  // depends on a
//    });
//
// 3. You want NO automatic destruction (intentional for some singletons)

// WHEN TO CHOOSE Meyer's Singleton:
// 1. Simple initialization with no runtime args
// 2. You want automatic cleanup at program exit
// 3. You want the simplest possible code

struct MeyersSingleton {
    static MeyersSingleton& get() {
        static MeyersSingleton instance; // one line = done
        return instance;
    }
};

int main() {
    MeyersSingleton::get();
    std::cout << "Prefer Meyer's Singleton for simplicity\n";
}
```

The rule of thumb: use Meyer's Singleton by default. Reach for `call_once` when you need runtime arguments, need to initialize multiple interdependent globals in one atomic block, or need to control object lifetime past program exit.

---

## Notes

- This file covers advanced patterns. See the main `call_once` file (Item #298) for fundamentals.
- Thread-safe destruction: Meyer's Singleton is destroyed at program exit via `atexit()`. `call_once` plus `new` leaks intentionally - use `std::unique_ptr` or `atexit` cleanup if you need the object destroyed.
- `once_flag` cannot be reset. Once the callable succeeds, the flag is permanently set. Design your initialization accordingly.
- Compile with `-std=c++11 -pthread`.
