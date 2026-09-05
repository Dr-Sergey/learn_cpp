# Configure Real-Time Scheduling (SCHED_FIFO) for C++ Threads

**Category:** Low Latency and Real Time  
**Standard:** C++17 (Linux-specific)  
**Reference:** [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html), [RT Linux Wiki](https://wiki.linuxfoundation.org/realtime/start)  

---

## Topic Overview

Linux provides two real-time scheduling policies: **SCHED_FIFO** and **SCHED_RR**. Under the default `SCHED_OTHER` (CFS), threads receive time slices proportional to their nice value and are preempted after their quantum. Under SCHED_FIFO, a thread runs until it voluntarily yields or is preempted by a **higher-priority** RT thread - there is no timeslicing. Under SCHED_RR, RT threads at the same priority are round-robin scheduled with a configurable time quantum.

The reason this matters for low latency is simple: under the default CFS scheduler, the kernel can preempt your thread at any moment to run something else. Your thread might have been one instruction away from finishing a critical operation when the scheduler decided another thread deserved a turn. That preemption adds latency, and it adds it unpredictably. With SCHED_FIFO, the scheduler only touches your thread when something more important needs to run - and you control what "more important" means through the priority number.

RT priorities range from 1 (lowest) to 99 (highest). A SCHED_FIFO thread at priority 80 will preempt any thread at priority 79 or below, regardless of the lower thread's state. This provides hard determinism: the highest-priority ready thread always runs immediately. However, a runaway RT thread at priority 99 can **starve the entire system**, including the kernel's own threads. That's not hypothetical - a bug in your RT thread's loop condition can make your machine completely unresponsive. Treat high RT priorities with respect.

The **RT throttle** (`/proc/sys/kernel/sched_rt_runtime_us`) limits RT threads to 950ms per 1000ms by default, reserving 50ms for non-RT tasks. For production RT systems, this is often set to -1 (disabled), but only with extreme care. Combine RT scheduling with CPU isolation (`isolcpus=`, `nohz_full=`) for maximum determinism.

| Policy | Priority Range | Timeslicing | Preemption | Use Case |
| --- | --- | --- | --- | --- |
| `SCHED_OTHER` (CFS) | nice: -20 to 19 | Yes (quantum) | By scheduler | General purpose |
| `SCHED_FIFO` | 1-99 | No | By higher prio only | Hard real-time |
| `SCHED_RR` | 1-99 | Yes (RT quantum) | By higher/equal prio | Soft real-time |
| `SCHED_DEADLINE` | - | Deadline-based | EDF scheduling | Periodic tasks |

Here's a practical layering of priorities for a trading system. Higher-numbered entries run first; lower-priority threads only get CPU time when all higher-priority threads are blocked or idle:

```cpp
PRIORITY PREEMPTION:
┌─────────────────────────────────────────────┐
│ Priority 99: Watchdog (SCHED_FIFO)          │ ← runs first
├─────────────────────────────────────────────┤
│ Priority 80: Market data handler (FIFO)     │ ← preempts all below
├─────────────────────────────────────────────┤
│ Priority 60: Order matching (FIFO)          │
├─────────────────────────────────────────────┤
│ Priority 40: Risk engine (FIFO)             │
├─────────────────────────────────────────────┤
│ SCHED_OTHER: Logging, monitoring            │ ← runs when RT idle
└─────────────────────────────────────────────┘
```

---

## Self-Assessment

### Q1: Create a `RealtimeThread` class that configures SCHED_FIFO, sets priority, pins to a core, and locks memory - the full RT thread setup

Setting up a proper RT thread involves several independent steps: setting the scheduler policy, setting the priority, pinning to a CPU core, and locking memory pages. Doing them piecemeal is error-prone. This class bundles everything into one creation call.

<!-- compile: needs POSIX header `pthread.h` -->
```cpp
#include <pthread.h>
#include <sched.h>
#include <sys/mman.h>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <cerrno>
#include <functional>
#include <stdexcept>
#include <thread>

class RealtimeThread {
    pthread_t thread_{};
    bool running_ = false;

    struct Config {
        int core_id;
        int rt_priority;       // 1-99
        int sched_policy;      // SCHED_FIFO or SCHED_RR
        std::size_t stack_size;
    };

    static void* thread_entry(void* arg) {
        auto* fn = static_cast<std::function<void()>*>(arg);
        (*fn)();
        delete fn;
        return nullptr;
    }

public:
    static RealtimeThread create(Config cfg, std::function<void()> work) {
        RealtimeThread rt;

        pthread_attr_t attr;
        pthread_attr_init(&attr);

        // Stack size
        if (cfg.stack_size > 0) {
            pthread_attr_setstacksize(&attr, cfg.stack_size);
        }

        // Scheduling policy and priority
        pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
        pthread_attr_setschedpolicy(&attr, cfg.sched_policy);

        struct sched_param param{};
        param.sched_priority = cfg.rt_priority;
        pthread_attr_setschedparam(&attr, &param);

        // Create thread
        auto* fn = new std::function<void()>(std::move(work));
        int rc = pthread_create(&rt.thread_, &attr, thread_entry, fn);
        pthread_attr_destroy(&attr);

        if (rc != 0) {
            delete fn;
            std::fprintf(stderr, "pthread_create failed: %s (need CAP_SYS_NICE)\n",
                         strerror(rc));
            throw std::runtime_error("Failed to create RT thread");
        }

        // Pin to core
        cpu_set_t cpuset;
        CPU_ZERO(&cpuset);
        CPU_SET(cfg.core_id, &cpuset);
        rc = pthread_setaffinity_np(rt.thread_, sizeof(cpu_set_t), &cpuset);
        if (rc != 0) {
            std::fprintf(stderr, "Warning: affinity set failed: %s\n",
                         strerror(rc));
        }

        rt.running_ = true;
        return rt;
    }

    void join() {
        if (running_) {
            pthread_join(thread_, nullptr);
            running_ = false;
        }
    }

    // Query actual scheduling
    void print_info() const {
        int policy;
        struct sched_param param;
        pthread_getschedparam(thread_, &policy, &param);
        const char* name = (policy == SCHED_FIFO) ? "SCHED_FIFO" :
                           (policy == SCHED_RR)   ? "SCHED_RR" : "SCHED_OTHER";
        std::printf("Thread policy: %s, priority: %d\n", name, param.sched_priority);
    }

    ~RealtimeThread() { join(); }

    RealtimeThread() = default;
    RealtimeThread(RealtimeThread&& o) noexcept
        : thread_(o.thread_), running_(o.running_) { o.running_ = false; }
    RealtimeThread& operator=(RealtimeThread&&) = delete;
};

int main() {
    // Lock all memory to prevent page faults
    if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
        std::fprintf(stderr, "mlockall: %s\n", strerror(errno));
    }

    auto rt = RealtimeThread::create(
        {.core_id = 2, .rt_priority = 80,
         .sched_policy = SCHED_FIFO, .stack_size = 1024 * 1024},
        [] {
            std::printf("RT thread running on core 2 at SCHED_FIFO:80\n");
            // Real-time work loop
            volatile int sum = 0;
            for (int i = 0; i < 10'000'000; ++i) sum += i;
            std::printf("RT work completed\n");
        }
    );

    rt.print_info();
    rt.join();
}
```

Notice `PTHREAD_EXPLICIT_SCHED` - without it, the child thread inherits the parent's scheduling parameters, and your RT priority setting is silently ignored. That's a common source of confusion when first setting up RT threads.

### Q2: Implement a priority inversion detector that monitors mutex hold times and detects when a low-priority thread blocks a high-priority one

Priority inversion is a subtle failure mode. A low-priority thread holds a mutex. A high-priority thread needs that mutex. The high-priority thread blocks, now effectively running at the low priority. Meanwhile, a medium-priority thread can preempt the low-priority lock holder because it has higher priority - and the low-priority thread never gets to release the mutex. The `PTHREAD_PRIO_INHERIT` protocol addresses this by temporarily boosting the lock holder's priority.

<!-- compile: needs POSIX header `pthread.h` -->
```cpp
#include <pthread.h>
#include <sched.h>
#include <cstdio>
#include <atomic>
#include <chrono>
#include <thread>
#include <cerrno>
#include <cstring>

// Use PTHREAD_PRIO_INHERIT to enable priority inheritance protocol
class PriorityInheritMutex {
    pthread_mutex_t mtx_;
    std::atomic<int> holder_priority_{-1};
    std::chrono::steady_clock::time_point lock_time_;

public:
    PriorityInheritMutex() {
        pthread_mutexattr_t attr;
        pthread_mutexattr_init(&attr);
        // Priority inheritance: if high-prio thread blocks on this mutex,
        // the holder's priority is temporarily boosted
        pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
        pthread_mutex_init(&mtx_, &attr);
        pthread_mutexattr_destroy(&attr);
    }

    void lock() {
        pthread_mutex_lock(&mtx_);
        lock_time_ = std::chrono::steady_clock::now();
        struct sched_param param;
        int policy;
        pthread_getschedparam(pthread_self(), &policy, &param);
        holder_priority_.store(param.sched_priority);
    }

    void unlock() {
        auto held = std::chrono::steady_clock::now() - lock_time_;
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(held).count();
        if (us > 100) {
            std::printf("WARNING: mutex held for %lld µs by prio %d\n",
                        (long long)us, holder_priority_.load());
        }
        holder_priority_.store(-1);
        pthread_mutex_unlock(&mtx_);
    }

    ~PriorityInheritMutex() { pthread_mutex_destroy(&mtx_); }
};

void set_rt_priority(int priority) {
    struct sched_param param{};
    param.sched_priority = priority;
    if (pthread_setschedparam(pthread_self(), SCHED_FIFO, &param) != 0) {
        std::fprintf(stderr, "setschedparam: %s\n", strerror(errno));
    }
}

PriorityInheritMutex shared_mutex;
std::atomic<bool> running{true};

void low_priority_work() {
    set_rt_priority(10);
    shared_mutex.lock();
    std::printf("[LOW  prio 10] Holding mutex, doing slow work...\n");
    // Simulate long computation while holding the lock
    volatile int x = 0;
    for (int i = 0; i < 50'000'000; ++i) x += i;
    shared_mutex.unlock();
    std::printf("[LOW  prio 10] Released mutex\n");
}

void high_priority_work() {
    set_rt_priority(80);
    // Small delay to let low-prio thread acquire lock first
    std::this_thread::sleep_for(std::chrono::milliseconds(10));

    std::printf("[HIGH prio 80] Trying to acquire mutex...\n");
    auto t0 = std::chrono::steady_clock::now();
    shared_mutex.lock();
    auto t1 = std::chrono::steady_clock::now();
    auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
    std::printf("[HIGH prio 80] Acquired mutex after %lld µs wait\n",
                (long long)us);
    shared_mutex.unlock();
}

int main() {
    std::thread low(low_priority_work);
    std::thread high(high_priority_work);
    low.join();
    high.join();
    std::printf("Priority inheritance prevented unbounded inversion.\n");
}
```

Without `PTHREAD_PRIO_INHERIT`, the high-priority thread could wait for a very long time if medium-priority threads keep getting scheduled ahead of the low-priority lock holder. With it, the kernel temporarily promotes the lock holder to the waiter's priority so it can finish and release the lock promptly.

### Q3: Combine `SCHED_FIFO`, `isolcpus` core isolation, and RT throttle configuration to create a minimal-jitter execution environment

This example measures actual jitter - the deviation of a periodic loop from its intended wake time. The jitter numbers tell you whether your RT setup is actually working. Before isolation, you'll typically see p99.9 jitter in the tens of microseconds. With proper isolation and RT scheduling, you can drive it down to the low hundreds of nanoseconds.

<!-- compile: needs POSIX header `pthread.h` -->
```cpp
#include <pthread.h>
#include <sched.h>
#include <sys/mman.h>
#include <cstdio>
#include <cstring>
#include <cerrno>
#include <chrono>
#include <fstream>
#include <string>
#include <vector>
#include <numeric>
#include <algorithm>

// Read current RT throttle settings
void print_rt_throttle() {
    auto read_file = [](const char* path) -> std::string {
        std::ifstream f(path);
        std::string val;
        if (f >> val) return val;
        return "N/A";
    };

    std::printf("RT Throttle Settings:\n");
    std::printf("  sched_rt_runtime_us: %s\n",
                read_file("/proc/sys/kernel/sched_rt_runtime_us").c_str());
    std::printf("  sched_rt_period_us:  %s\n",
                read_file("/proc/sys/kernel/sched_rt_period_us").c_str());
}

// Check which CPUs are isolated
void print_isolated_cpus() {
    std::ifstream f("/sys/devices/system/cpu/isolated");
    std::string cpus;
    if (std::getline(f, cpus) && !cpus.empty()) {
        std::printf("Isolated CPUs: %s\n", cpus.c_str());
    } else {
        std::printf("Isolated CPUs: none (add isolcpus= to kernel cmdline)\n");
    }
}

// Jitter measurement loop
void measure_jitter(int core_id, int priority, int iterations) {
    // Pin and set SCHED_FIFO
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);

    struct sched_param param{};
    param.sched_priority = priority;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);

    // Measurement
    std::vector<int64_t> deltas;
    deltas.reserve(iterations);
    constexpr std::chrono::microseconds target_period{10};  // 100 kHz loop

    auto next_wake = std::chrono::steady_clock::now();
    for (int i = 0; i < iterations; ++i) {
        next_wake += target_period;
        auto now = std::chrono::steady_clock::now();
        // Busy-wait until target time
        while (std::chrono::steady_clock::now() < next_wake) {}

        auto actual = std::chrono::steady_clock::now();
        auto jitter = std::chrono::duration_cast<std::chrono::nanoseconds>(
            actual - next_wake).count();
        deltas.push_back(jitter);
    }

    // Report
    std::sort(deltas.begin(), deltas.end());
    double avg = std::accumulate(deltas.begin(), deltas.end(), 0.0) / iterations;

    std::printf("\nJitter Report (core %d, SCHED_FIFO:%d, %d samples):\n",
                core_id, priority, iterations);
    std::printf("┌────────────┬──────────┐\n");
    std::printf("│ Metric     │ Value    │\n");
    std::printf("├────────────┼──────────┤\n");
    std::printf("│ Avg        │ %4.0f ns  │\n", avg);
    std::printf("│ p50        │ %4lld ns  │\n", (long long)deltas[iterations / 2]);
    std::printf("│ p99        │ %4lld ns  │\n", (long long)deltas[iterations * 99 / 100]);
    std::printf("│ p99.9      │ %4lld ns  │\n", (long long)deltas[iterations * 999 / 1000]);
    std::printf("│ Max        │ %4lld ns  │\n", (long long)deltas.back());
    std::printf("└────────────┴──────────┘\n");
}

int main() {
    mlockall(MCL_CURRENT | MCL_FUTURE);

    print_rt_throttle();
    print_isolated_cpus();

    // Run jitter test on core 2 at priority 80
    measure_jitter(2, 80, 100'000);

    munlockall();
}
```

The max jitter value is often more informative than p99. A single large spike can cause a missed deadline even if the average looks great. If you're seeing unexpected spikes, common culprits are: IRQs landing on your isolated core, RCU callbacks, or NMI handlers.

---

## Notes

- **`CAP_SYS_NICE`** capability is needed to set RT scheduling; `CAP_IPC_LOCK` for `mlockall`. Use `setcap` or run as root.
- **RT throttle**: Default 950ms/1000ms. Set to `-1` in `/proc/sys/kernel/sched_rt_runtime_us` for production RT but only with watchdog support.
- **`SCHED_DEADLINE`** (Linux 3.14+) uses Earliest Deadline First (EDF) scheduling - ideal for periodic tasks with known deadlines, avoids manual priority assignment.
- **Priority inversion**: Always use `PTHREAD_PRIO_INHERIT` on mutexes shared between RT threads of different priorities.
- Kernel boot parameters for full RT: `isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3 irqaffinity=0,1`.
- **PREEMPT_RT** kernel patch provides full preemptibility - essential for sub-10µs worst-case latency.
