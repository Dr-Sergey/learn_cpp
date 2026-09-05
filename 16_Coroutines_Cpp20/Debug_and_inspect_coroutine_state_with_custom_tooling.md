# Debug and inspect coroutine state with custom tooling

**Category:** Coroutines Cpp20  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/coroutine/coroutine_handle>  

---

## Topic Overview

### Why Coroutine Debugging Is Hard

Coroutine frames are heap-allocated, opaque objects. When a coroutine suspends, debugging becomes genuinely difficult in ways that don't apply to normal function calls:

- The stack frame disappears - there is no call stack to inspect.
- The compiler-generated coroutine frame stores local variables in an implementation-defined layout.
- Standard debuggers (GDB, LLDB, MSVC) have limited native coroutine support.

The reason this is more painful than ordinary multithreaded debugging is that coroutines look synchronous in source code but behave asynchronously at runtime. A suspended coroutine has no thread, no stack, and no obvious location in memory that a standard debugger can follow. The solution is to build your own observability into the coroutine type itself.

### Inspecting Coroutine State Programmatically

`std::coroutine_handle<>` provides basic inspection. The `address()` method gives you the raw pointer to the coroutine frame, and `done()` tells you whether the coroutine has reached `final_suspend`. That's enough to start building diagnostic tools. Here is a simple utility that prints what it can about any coroutine handle:

```cpp
#include <coroutine>
#include <iostream>

template<typename Promise>
void inspect_coroutine(std::coroutine_handle<Promise> h) {
    std::cout << "Address:  " << h.address() << "\n";
    std::cout << "Done:     " << std::boolalpha << h.done() << "\n";

    if constexpr (requires { h.promise(); }) {
        std::cout << "Promise:  " << &h.promise() << "\n";
    }
}
```

### Building a Coroutine Registry

If you want to track all live coroutines - for a dashboard, for debugging a hang, or for finding leaks - a global registry keyed on the handle's address is a natural solution. Each coroutine registers itself when it starts and unregisters when it completes. Because coroutines can be resumed from different threads, the registry uses a mutex:

```cpp
#include <coroutine>
#include <unordered_map>
#include <string>
#include <mutex>
#include <iostream>

struct CoroutineInfo {
    std::string name;
    std::string file;
    int line;
    bool suspended;
};

class CoroutineRegistry {
    std::mutex mtx_;
    std::unordered_map<void*, CoroutineInfo> coroutines_;

public:
    static CoroutineRegistry& instance() {
        static CoroutineRegistry reg;
        return reg;
    }

    void register_coroutine(void* addr, CoroutineInfo info) {
        std::lock_guard lock(mtx_);
        coroutines_[addr] = std::move(info);
    }

    void mark_suspended(void* addr) {
        std::lock_guard lock(mtx_);
        if (auto it = coroutines_.find(addr); it != coroutines_.end())
            it->second.suspended = true;
    }

    void mark_resumed(void* addr) {
        std::lock_guard lock(mtx_);
        if (auto it = coroutines_.find(addr); it != coroutines_.end())
            it->second.suspended = false;
    }

    void unregister(void* addr) {
        std::lock_guard lock(mtx_);
        coroutines_.erase(addr);
    }

    void dump() {
        std::lock_guard lock(mtx_);
        std::cout << "=== Live Coroutines ===\n";
        for (auto& [addr, info] : coroutines_) {
            std::cout << "  " << addr << " [" << info.name << "] "
                      << (info.suspended ? "SUSPENDED" : "RUNNING")
                      << " at " << info.file << ":" << info.line << "\n";
        }
        std::cout << "=== Total: " << coroutines_.size() << " ===\n";
    }
};
```

### Instrumenting the Promise Type

The cleanest place to hook into the registry is the promise type itself. `get_return_object()` is called right when the coroutine starts, so it is the right place to register. `final_suspend` is the right place to unregister. By using `std::source_location`, you can capture the filename and line number of the coroutine's declaration automatically - no manual tagging required:

```cpp
#include <source_location>

template<typename T>
struct tracked_promise {
    T value_;
    std::source_location loc_;

    auto get_return_object() {
        auto h = std::coroutine_handle<tracked_promise>::from_promise(*this);
        CoroutineRegistry::instance().register_coroutine(
            h.address(),
            {.name = loc_.function_name(),
             .file = loc_.file_name(),
             .line = static_cast<int>(loc_.line()),
             .suspended = false});
        return tracked_task<T>{h};
    }

    auto initial_suspend() {
        return suspend_and_track{std::coroutine_handle<tracked_promise>::from_promise(*this)};
    }

    void return_value(T v) { value_ = std::move(v); }
    void unhandled_exception() { std::terminate(); }

    auto final_suspend() noexcept {
        CoroutineRegistry::instance().unregister(
            std::coroutine_handle<tracked_promise>::from_promise(*this).address());
        return std::suspend_always{};
    }
};
```

### GDB/LLDB Tips for Coroutines

When you do need to drop to the debugger, these flags and commands help. Note that `-fcoroutines-debug` (Clang) is particularly valuable because it preserves local variable names inside the coroutine frame, which are otherwise lost after compilation:

```bash
# GDB: inspect coroutine frame memory
(gdb) p/x *(char(*)[128])coroutine_handle.address()

# LLDB: show coroutine handle
(lldb) p coroutine_handle.__handle_

# Clang: compile with -fcoroutines-debug to preserve variable names in frames
# GCC: -gdwarf-5 for improved coroutine debug info

# Print all threads and look for coroutine resume functions
(gdb) info threads
(gdb) thread apply all bt
```

### Logging Awaiter for Debugging Suspension Points

If you want to trace exactly when a coroutine suspends and resumes at a particular `co_await` point, the cleanest approach is a wrapper awaiter that adds logging around the inner awaitable. You drop this in when debugging and remove it for production. Notice that it works with any awaitable - the template parameter `Awaitable` is whatever the original `co_await` expression produced:

```cpp
template<typename Awaitable>
struct logging_awaiter {
    Awaitable inner_;
    const char* label_;

    bool await_ready() {
        bool ready = inner_.await_ready();
        std::cout << "[await:" << label_ << "] ready=" << ready << "\n";
        return ready;
    }

    auto await_suspend(std::coroutine_handle<> h) {
        std::cout << "[await:" << label_ << "] suspending " << h.address() << "\n";
        CoroutineRegistry::instance().mark_suspended(h.address());
        return inner_.await_suspend(h);
    }

    auto await_resume() {
        std::cout << "[await:" << label_ << "] resuming\n";
        return inner_.await_resume();
    }
};

// Usage: co_await logging_awaiter{some_op(), "db_query"};
```

---

## Self-Assessment

### Q1: Implement a simple coroutine registry and show its dump output

Here the registry itself is defined in the Topic Overview above. This example just exercises it directly by simulating three coroutines in different states, then calling `dump()` to see the output:

```cpp
// Using the CoroutineRegistry class from above
int main() {
    auto& reg = CoroutineRegistry::instance();

    // Simulate registering coroutines
    reg.register_coroutine((void*)0x1000,
        {"fetch_data", "server.cpp", 42, true});
    reg.register_coroutine((void*)0x2000,
        {"process_request", "handler.cpp", 87, false});
    reg.register_coroutine((void*)0x3000,
        {"send_response", "handler.cpp", 120, true});

    reg.dump();
    // Output:
    // === Live Coroutines ===
    //   0x1000 [fetch_data] SUSPENDED at server.cpp:42
    //   0x2000 [process_request] RUNNING at handler.cpp:87
    //   0x3000 [send_response] SUSPENDED at handler.cpp:120
    // === Total: 3 ===
}
```

### Q2: Write a logging awaiter that traces suspension and resumption

See the `logging_awaiter` template in the Topic Overview above. The output you'd see wrapping a `db_query` operation looks like this - one line when it checks readiness, one when it actually suspends, one when it resumes:

```text
[await:db_query] ready=false
[await:db_query] suspending 0x55a8b3c0
[await:db_query] resuming
```

### Q3: What compiler flags improve coroutine debugging

| Compiler | Flag | Effect |
| --- | --- | --- |
| Clang | `-fcoroutine-debug` | Preserves variable names in coroutine frames |
| GCC | `-gdwarf-5` | Improved debug info for coroutine state |
| MSVC | `/Zi /Od` | Full debug info, no optimization (preserves frame layout) |
| All | `-fno-elide-constructors` | Prevents HALO (heap allocation elision) for visibility |

---

## Notes

- Native debugger support for coroutines is rapidly improving - check your debugger version.
- In production, use a coroutine registry with minimal overhead (atomic counters, not maps).
- The `std::source_location` integration makes it easy to track where coroutines were created.
- Consider building a web dashboard for the coroutine registry in long-running server applications.
