# Understand Threading Model Differences and std::thread Portability

**Category:** Cross Platform Development  
**Standard:** C++11, C++20  
**Reference:** https://en.cppreference.com/w/cpp/thread/thread  

---

## Topic Overview

`std::thread` provides a portable threading abstraction since C++11, but it is a thin wrapper whose behavior depends on the underlying platform threading model. The two dominant models - Win32 threads and POSIX threads (pthreads) - differ in creation semantics, scheduling, naming, and cleanup guarantees. Understanding these differences is essential when writing high-performance, cross-platform concurrent code, because what works on Linux may silently misbehave or refuse to compile on Windows.

| Feature | Win32 Threads | POSIX Threads (pthreads) |
| --- | --- | --- |
| Creation | `CreateThread` / `_beginthreadex` | `pthread_create` |
| Join | `WaitForSingleObject` | `pthread_join` |
| Detach | `CloseHandle` (implicit) | `pthread_detach` |
| Thread naming | `SetThreadDescription` (Win10+) | `pthread_setname_np` (non-portable) |
| Affinity | `SetThreadAffinityMask` | `pthread_setaffinity_np` (Linux) |
| Priority | `SetThreadPriority` | `pthread_setschedparam` + `sched_param` |
| Cancel | No native cancel | `pthread_cancel` (async or deferred) |
| TLS | `__declspec(thread)`, `TlsAlloc` | `__thread`, `pthread_key_create` |
| Stack size | `CreateThread` parameter | `pthread_attr_setstacksize` |

C++20 added `std::jthread`, which is joinable-by-default and integrates cooperative cancellation via `std::stop_token`. This eliminates the most common portability hazard: forgetting to join or detach, which calls `std::terminate()`. However, `jthread` does not expose platform-specific features like naming or affinity, so you still need platform code for those.

Here is a side-by-side comparison of the two thread types to see what you gain with `jthread`:

```cpp
Thread lifecycle comparison:

std::thread          |   std::jthread (C++20)
---------------------+----------------------------
construct -> running  |   construct -> running
must join() / detach |   auto-joins in destructor
no cancellation      |   stop_token + stop_callback
```

Key portability pitfalls to keep in mind: `thread::hardware_concurrency()` may return 0 on exotic platforms; thread stack sizes default to 1-8 MB depending on the OS with no standard API to set them; and `thread_local` destructors are not called on detached threads on some implementations.

---

## Self-Assessment

### Q1: Write a portable thread wrapper that adds naming, priority, and affinity support across Windows and Linux

`std::thread` gives you portability for the basic lifecycle, but it deliberately omits platform-specific features. This wrapper fills that gap by running the platform-specific configuration inside the new thread itself (where it applies to the current thread), before calling the user-supplied function.

```cpp
#include <thread>
#include <string>
#include <cstdint>
#include <functional>
#include <iostream>
#include <optional>

#ifdef _WIN32
    #define WIN32_LEAN_AND_MEAN
    #include <windows.h>
#else
    #include <pthread.h>
    #include <sched.h>
#endif

enum class ThreadPriority { Low, Normal, High, Realtime };

struct ThreadConfig {
    std::string           name;
    ThreadPriority        priority = ThreadPriority::Normal;
    std::optional<std::uint64_t> affinity_mask;  // bitmask of CPU cores
};

class PlatformThread {
    std::thread thread_;
    ThreadConfig config_;

    static void apply_config(const ThreadConfig& cfg) {
        // Naming
        if (!cfg.name.empty()) {
            #ifdef _WIN32
            // SetThreadDescription (Windows 10 1607+)
            std::wstring wname(cfg.name.begin(), cfg.name.end());
            ::SetThreadDescription(::GetCurrentThread(), wname.c_str());
            #elif defined(__linux__)
            // Linux: max 15 chars + NUL
            pthread_setname_np(pthread_self(), cfg.name.substr(0, 15).c_str());
            #elif defined(__APPLE__)
            pthread_setname_np(cfg.name.substr(0, 63).c_str());
            #endif
        }

        // Priority
        #ifdef _WIN32
        int prio = THREAD_PRIORITY_NORMAL;
        switch (cfg.priority) {
            case ThreadPriority::Low:      prio = THREAD_PRIORITY_BELOW_NORMAL; break;
            case ThreadPriority::Normal:   prio = THREAD_PRIORITY_NORMAL; break;
            case ThreadPriority::High:     prio = THREAD_PRIORITY_ABOVE_NORMAL; break;
            case ThreadPriority::Realtime: prio = THREAD_PRIORITY_TIME_CRITICAL; break;
        }
        ::SetThreadPriority(::GetCurrentThread(), prio);
        #elif defined(__linux__)
        // SCHED_OTHER for normal, SCHED_FIFO for realtime (requires root)
        if (cfg.priority == ThreadPriority::Realtime) {
            sched_param sp{};
            sp.sched_priority = sched_get_priority_min(SCHED_FIFO);
            pthread_setschedparam(pthread_self(), SCHED_FIFO, &sp);
        }
        #endif

        // Affinity
        if (cfg.affinity_mask.has_value()) {
            #ifdef _WIN32
            ::SetThreadAffinityMask(::GetCurrentThread(),
                                    static_cast<DWORD_PTR>(*cfg.affinity_mask));
            #elif defined(__linux__)
            cpu_set_t cpuset;
            CPU_ZERO(&cpuset);
            auto mask = *cfg.affinity_mask;
            for (int i = 0; i < 64 && mask; ++i, mask >>= 1)
                if (mask & 1) CPU_SET(i, &cpuset);
            pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
            #endif
        }
    }

public:
    template <typename F, typename... Args>
    PlatformThread(ThreadConfig cfg, F&& f, Args&&... args)
        : config_(std::move(cfg))
    {
        thread_ = std::thread([this, fn = std::forward<F>(f),
                               ... captured = std::forward<Args>(args)]() mutable {
            apply_config(config_);
            fn(std::move(captured)...);
        });
    }

    void join() { if (thread_.joinable()) thread_.join(); }
    ~PlatformThread() { join(); }

    PlatformThread(const PlatformThread&) = delete;
    PlatformThread& operator=(const PlatformThread&) = delete;
};

int main() {
    PlatformThread worker(
        {.name = "compute-worker", .priority = ThreadPriority::High, .affinity_mask = 0x03},
        []() {
            std::cout << "Worker running on configured thread\n";
        }
    );
    // Auto-joins in destructor
    return 0;
}
```

The destructor calls `join()` unconditionally, so you never need to worry about forgetting it. This pattern mirrors what `std::jthread` does for the basic lifecycle, while adding the platform-specific configuration layer on top.

### Q2: Demonstrate `std::jthread` with cooperative cancellation and show how to integrate it with platform-specific cleanup

The key thing to understand about `std::jthread` cancellation is that it is cooperative, not preemptive. The running thread is never forcibly stopped - it must periodically check `stop_requested()` and exit voluntarily. This is much safer than `pthread_cancel` because the thread gets to clean up its own resources.

```cpp
#include <thread>
#include <stop_token>
#include <iostream>
#include <chrono>
#include <atomic>
#include <vector>

class WorkerPool {
    std::vector<std::jthread> workers_;
    std::atomic<int> tasks_completed_{0};

public:
    void start(int num_workers) {
        workers_.reserve(num_workers);

        for (int i = 0; i < num_workers; ++i) {
            workers_.emplace_back([this, i](std::stop_token stoken) {
                // Register a stop callback for platform-specific cleanup
                std::stop_callback cleanup(stoken, [i]() {
                    std::cout << "Worker " << i << ": stop requested, cleaning up\n";
                    // Platform-specific: flush buffers, release handles, etc.
                });

                while (!stoken.stop_requested()) {
                    // Simulate work
                    std::this_thread::sleep_for(std::chrono::milliseconds(100));
                    ++tasks_completed_;

                    if (tasks_completed_ % 10 == 0) {
                        std::cout << "Worker " << i << ": "
                                  << tasks_completed_.load() << " tasks done\n";
                    }
                }

                std::cout << "Worker " << i << ": exiting gracefully\n";
            });
        }
    }

    void stop() {
        for (auto& w : workers_) {
            w.request_stop();  // Cooperative cancellation
        }
        // jthread destructor auto-joins
    }

    int completed() const { return tasks_completed_.load(); }

    ~WorkerPool() { stop(); }
};

int main() {
    WorkerPool pool;
    pool.start(4);

    std::this_thread::sleep_for(std::chrono::seconds(1));

    std::cout << "Requesting shutdown...\n";
    pool.stop();
    std::cout << "Total tasks completed: " << pool.completed() << "\n";
    return 0;
}
```

Notice the `std::stop_callback` - it fires as soon as `request_stop()` is called, even if the thread happens to be sleeping at that moment. This lets you wake the thread or release a platform handle from the outside, without the thread having to poll at the exact right time.

### Q3: Show how to set thread stack size portably, handling the lack of a standard API

There is no standard way to set a thread's stack size through `std::thread`. If you need more than the default (which varies from 1 MB to 8 MB depending on the OS), you have to drop down to the platform threading API directly. This wrapper hides that behind a clean interface.

```cpp
#include <thread>
#include <functional>
#include <iostream>
#include <cstddef>
#include <memory>

#ifdef _WIN32
    #define WIN32_LEAN_AND_MEAN
    #include <windows.h>
#else
    #include <pthread.h>
#endif

// Portable thread creation with custom stack size
class StackSizedThread {
    std::size_t stack_size_;

    #ifdef _WIN32
    HANDLE handle_ = nullptr;
    #else
    pthread_t handle_{};
    #endif

    bool joinable_ = false;

    struct CallableBase {
        virtual ~CallableBase() = default;
        virtual void invoke() = 0;
    };

    template <typename F>
    struct CallableImpl : CallableBase {
        F func;
        explicit CallableImpl(F&& f) : func(std::move(f)) {}
        void invoke() override { func(); }
    };

    #ifdef _WIN32
    static DWORD WINAPI thread_proc(LPVOID arg) {
        auto* callable = static_cast<CallableBase*>(arg);
        callable->invoke();
        delete callable;
        return 0;
    }
    #else
    static void* thread_proc(void* arg) {
        auto* callable = static_cast<CallableBase*>(arg);
        callable->invoke();
        delete callable;
        return nullptr;
    }
    #endif

public:
    explicit StackSizedThread(std::size_t stack_bytes)
        : stack_size_(stack_bytes) {}

    template <typename F>
    void start(F&& func) {
        auto* callable = new CallableImpl<std::decay_t<F>>(std::forward<F>(func));

        #ifdef _WIN32
        handle_ = ::CreateThread(
            nullptr,
            stack_size_,         // dwStackSize
            thread_proc,
            callable,
            0,                   // run immediately
            nullptr
        );
        if (!handle_) {
            delete callable;
            throw std::runtime_error("CreateThread failed");
        }
        #else
        pthread_attr_t attr;
        pthread_attr_init(&attr);
        pthread_attr_setstacksize(&attr, stack_size_);

        int rc = pthread_create(&handle_, &attr, thread_proc, callable);
        pthread_attr_destroy(&attr);
        if (rc != 0) {
            delete callable;
            throw std::runtime_error("pthread_create failed");
        }
        #endif
        joinable_ = true;
    }

    void join() {
        if (!joinable_) return;
        #ifdef _WIN32
        ::WaitForSingleObject(handle_, INFINITE);
        ::CloseHandle(handle_);
        #else
        pthread_join(handle_, nullptr);
        #endif
        joinable_ = false;
    }

    ~StackSizedThread() { join(); }

    StackSizedThread(const StackSizedThread&) = delete;
    StackSizedThread& operator=(const StackSizedThread&) = delete;
};

int main() {
    constexpr std::size_t stack_2mb = 2 * 1024 * 1024;

    StackSizedThread t(stack_2mb);
    t.start([]() {
        // Deep recursion or large stack allocations safe here
        char large_buffer[1024 * 1024]; // 1 MB on stack
        large_buffer[0] = 'A';
        std::cout << "Running on thread with custom stack size\n";
        (void)large_buffer;
    });

    // Auto-joins in destructor
    return 0;
}
```

The `CallableBase` type-erasure trick is exactly what `std::thread` does internally - it lets `start()` accept any callable without the class itself being a template. The `delete callable` inside `thread_proc` mirrors the ownership model: the calling thread allocates, the new thread deallocates after invoking.

---

## Notes

- `std::jthread` should be the default choice for new C++20 code - it eliminates the "forgot to join" class of bugs entirely.
- Thread naming is purely diagnostic (debuggers, profilers, `htop`) - there is no standard API; each platform has its own non-portable extension.
- `std::thread::hardware_concurrency()` is a hint and may return 0. Always provide a fallback: `auto n = std::max(1u, std::thread::hardware_concurrency());`
- On Linux, `pthread_setaffinity_np` is GNU-specific. macOS has no per-thread affinity API - use `thread_policy_set` with `THREAD_AFFINITY_POLICY` instead.
- `thread_local` storage duration objects have their destructors called on thread exit for threads created with `std::thread` or `std::jthread`, but behavior is implementation-defined for detached threads.
- When wrapping platform threading APIs, always use `_beginthreadex` on Windows (not `CreateThread`) if C runtime functions will be used in the thread - `CreateThread` skips CRT per-thread initialization, which can corrupt the heap or produce wrong results from functions like `sprintf`.
