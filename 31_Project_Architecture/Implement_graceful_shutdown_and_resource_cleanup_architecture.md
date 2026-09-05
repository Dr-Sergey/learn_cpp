# Implement graceful shutdown and resource cleanup architecture

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Graceful shutdown** means your application finishes what it is doing, saves any important state, releases all resources, and exits cleanly - instead of being abruptly killed and potentially leaving corrupt data, leaked file handles, or half-written records behind. In C++ this requires handling OS signals properly, shutting down subsystems in the right order, draining any work queues before stopping threads, and flushing persistent state to disk. Get this wrong and you get data corruption, resource leaks, and hung processes that operations teams have to `kill -9`.

The challenges are not trivial, and they interact in subtle ways. The table below shows the main ones:

### Shutdown Challenges

| Challenge | Risk | Solution |
| --- | --- | --- |
| Signal handling | UB in signal handlers | `sig_atomic_t` flag + event loop check |
| Thread joining | Deadlock on shutdown | Stop tokens, poison messages |
| Resource ordering | Use-after-free | Reverse initialization order |
| Pending I/O | Data loss | Drain queues before stop |
| Static destruction | Order undefined | Explicit cleanup before `main` returns |

The resource ordering row is the one that trips people up most often. If you start subsystem A first and it depends on subsystem B, then on shutdown you must stop A before B. Otherwise A keeps calling into B after B is gone, which is a use-after-free. The rule is simply: shut down in the reverse order you started.

---

## Self-Assessment

### Q1: Implement signal-safe graceful shutdown

Signal handlers are severely restricted. You cannot allocate memory, call `printf`, throw exceptions, or do almost anything interesting inside one. The only safe thing to do is set a flag. The atomic flag approach shown here is the standard pattern: the handler writes to the flag, and the main loop polls it.

**Answer:**

```cpp
#include <csignal>
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>
#include <functional>

// === Signal-safe shutdown flag ===
std::atomic<bool> g_shutdown_requested{false};

// Signal handler: only set atomic flag (async-signal-safe)
void signal_handler(int signum) {
    g_shutdown_requested.store(true, std::memory_order_relaxed);
}

// === Shutdown coordinator ===
class ShutdownCoordinator {
public:
    // Register cleanup functions (called in reverse order)
    void on_shutdown(std::string name, std::function<void()> cleanup) {
        cleanups_.push_back({std::move(name), std::move(cleanup)});
    }

    void install_signal_handlers() {
        struct sigaction sa{};
        sa.sa_handler = signal_handler;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = 0;
        sigaction(SIGINT, &sa, nullptr);
        sigaction(SIGTERM, &sa, nullptr);
    }

    bool shutdown_requested() const {
        return g_shutdown_requested.load(std::memory_order_relaxed);
    }

    void execute_shutdown() {
        std::cout << "Shutdown initiated...\n";
        // Execute in reverse registration order (LIFO)
        for (auto it = cleanups_.rbegin(); it != cleanups_.rend(); ++it) {
            std::cout << "  Cleaning up: " << it->name << "\n";
            try {
                it->cleanup();
            } catch (const std::exception& e) {
                std::cerr << "  Error during " << it->name
                          << ": " << e.what() << "\n";
            }
        }
        std::cout << "Shutdown complete.\n";
    }

private:
    struct Entry {
        std::string name;
        std::function<void()> cleanup;
    };
    std::vector<Entry> cleanups_;
};
```

The `execute_shutdown` iterates in reverse with `rbegin`/`rend`, which enforces LIFO (last-in, first-out) cleanup order. The try/catch around each cleanup means one failing subsystem does not prevent the others from being cleaned up - you always want to attempt every cleanup step even if one goes wrong.

### Q2: Implement subsystem lifecycle with ordered shutdown

The two-phase shutdown design shown here is important enough to understand deeply. Phase 1 calls `request_stop()` on all subsystems - this is non-blocking, it just signals them to stop. Phase 2 calls `wait_stopped()` - this blocks until each subsystem has actually finished. Separating these phases lets all subsystems start their shutdown work concurrently during phase 1, rather than waiting for each one to fully finish before telling the next one to stop.

**Answer:**

```cpp
// === Subsystem interface ===
class ISubsystem {
public:
    virtual ~ISubsystem() = default;
    virtual const char* name() const = 0;
    virtual void start() = 0;
    virtual void request_stop() = 0;  // Non-blocking: signal to stop
    virtual void wait_stopped() = 0;  // Blocking: wait for completion
};

// === Application with ordered lifecycle ===
class Application {
public:
    void add_subsystem(std::unique_ptr<ISubsystem> sub) {
        subsystems_.push_back(std::move(sub));
    }

    void start() {
        coordinator_.install_signal_handlers();

        // Start in order
        for (auto& sub : subsystems_) {
            std::cout << "Starting: " << sub->name() << "\n";
            sub->start();
        }
    }

    void run() {
        // Main loop: wait for shutdown signal
        while (!coordinator_.shutdown_requested()) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }

    void shutdown() {
        // Phase 1: Request all to stop (non-blocking)
        for (auto it = subsystems_.rbegin(); it != subsystems_.rend(); ++it) {
            std::cout << "Requesting stop: " << (*it)->name() << "\n";
            (*it)->request_stop();
        }

        // Phase 2: Wait for each to finish (blocking, reverse order)
        for (auto it = subsystems_.rbegin(); it != subsystems_.rend(); ++it) {
            std::cout << "Waiting: " << (*it)->name() << "\n";
            (*it)->wait_stopped();
            std::cout << "Stopped: " << (*it)->name() << "\n";
        }
    }

private:
    ShutdownCoordinator coordinator_;
    std::vector<std::unique_ptr<ISubsystem>> subsystems_;
};

// === Example subsystem: worker threads ===
class WorkerPool : public ISubsystem {
public:
    const char* name() const override { return "WorkerPool"; }

    void start() override {
        for (int i = 0; i < 4; ++i) {
            workers_.emplace_back([this] {
                while (!stop_.load()) {
                    // Process work...
                    std::this_thread::sleep_for(
                        std::chrono::milliseconds(10));
                }
            });
        }
    }

    void request_stop() override {
        stop_.store(true);
    }

    void wait_stopped() override {
        for (auto& w : workers_) w.join();
        workers_.clear();
    }

private:
    std::vector<std::jthread> workers_;
    std::atomic<bool> stop_{false};
};

// === main ===
int main() {
    Application app;
    app.add_subsystem(std::make_unique<DatabaseSubsystem>());
    app.add_subsystem(std::make_unique<WorkerPool>());
    app.add_subsystem(std::make_unique<HttpServer>());

    app.start();    // DB -> Workers -> HTTP
    app.run();      // Wait for SIGINT/SIGTERM
    app.shutdown(); // HTTP -> Workers -> DB (reverse order)
}
```

Notice the comment in `main`: start order is DB, then Workers, then HTTP. Shutdown order is the exact reverse - HTTP first, then Workers, then DB. That ordering is not arbitrary: the HTTP server depends on the workers, and the workers depend on the database. Shutting down in the opposite order means nothing tries to use a dependency after it has been torn down.

### Q3: Handle shutdown with C++20 stop_token

C++20's `std::jthread` and `std::stop_token` are purpose-built for exactly this kind of cooperative cancellation, and they eliminate a lot of the manual atomic flag management. A `jthread` automatically receives a `stop_token` and its destructor calls `request_stop()` and then joins - so just clearing the `workers_` vector handles both phases at once.

**Answer:**

```cpp
#include <stop_token>
#include <thread>

// === jthread with stop_token for cooperative cancellation ===
class ModernWorkerPool {
public:
    void start() {
        for (int i = 0; i < 4; ++i) {
            // jthread passes stop_token automatically
            workers_.emplace_back([this](std::stop_token st) {
                while (!st.stop_requested()) {
                    // Check stop_token in inner loops too
                    process_batch(st);
                }
            });
        }
    }

    void request_stop() {
        // jthread::request_stop() signals all workers
        for (auto& w : workers_)
            w.request_stop();
    }

    void wait_stopped() {
        workers_.clear();  // jthread destructor joins
    }

private:
    void process_batch(std::stop_token st) {
        for (int i = 0; i < 1000 && !st.stop_requested(); ++i) {
            // Process item...
        }
    }

    std::vector<std::jthread> workers_;
};

// === Drain a message queue on shutdown ===
template<typename T>
class DrainableQueue {
public:
    void push(T item) {
        std::lock_guard lock(mutex_);
        if (closed_) return;  // Reject new items during shutdown
        queue_.push(std::move(item));
        cv_.notify_one();
    }

    // Close the queue: no new items, drain existing
    void close() {
        std::lock_guard lock(mutex_);
        closed_ = true;
        cv_.notify_all();
    }

    // Returns nullopt when queue is closed AND empty
    std::optional<T> pop() {
        std::unique_lock lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty() || closed_; });
        if (queue_.empty()) return std::nullopt;
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }

    size_t size() const {
        std::lock_guard lock(mutex_);
        return queue_.size();
    }

private:
    std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool closed_ = false;
};
```

The `DrainableQueue` is an important design. When you call `close()`, it sets a flag that rejects new pushes but still lets existing items be popped. Workers keep calling `pop()` and processing items until the queue is empty, at which point `pop()` returns `std::nullopt` and the worker exits cleanly. That is how you guarantee no in-flight work is lost during shutdown.

---

## Notes

- Signal handlers must be async-signal-safe: only set an atomic flag, nothing else.
- Shutdown order = reverse of startup: stops dependents before their dependencies.
- Two-phase shutdown: `request_stop()` (fast, non-blocking) then `wait_stopped()` (blocking).
- C++20 `std::stop_token` / `std::jthread` simplifies cooperative cancellation.
- Drain queues before stopping workers - otherwise pending work is lost.
- Reject new work during shutdown: mark queues as closed.
- Avoid `std::atexit` for complex cleanup - explicit shutdown in main() is more controllable.
