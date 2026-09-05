# Handle Real-Time Scheduling and Priority Inversion in C++

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** https://en.wikipedia.org/wiki/Priority_inversion  

---

## Topic Overview

Real-time systems require not just correct results but correct timing. A missed deadline in a hard real-time system (flight controller, ABS braking) is a system failure. The scheduler - whether a bare-metal superloop, a cooperative round-robin, or a preemptive RTOS (FreeRTOS, Zephyr, RTEMS) - determines which task runs when, and its interaction with C++ resource management patterns (RAII mutexes, shared state) creates the priority inversion problem.

Priority inversion occurs when a high-priority task is blocked waiting for a resource held by a low-priority task, while a medium-priority task preempts the low-priority task - effectively running medium before high. The classic example is the Mars Pathfinder bug (1997), where priority inversion caused the system watchdog to fire repeatedly. The standard solutions are priority inheritance (the low-priority task temporarily runs at the high-priority task's level while holding the lock) and priority ceiling (the mutex itself has a priority, and any task acquiring it elevates to that ceiling).

| Scheduling Algorithm     | Analysis Method           | Optimal For                        |
| --- | --- | --- |
| Rate-Monotonic (RM)      | RM utilization bound      | Fixed-priority, periodic tasks     |
| Earliest Deadline First  | Processor demand analysis | Dynamic priority, utilization <= 1 |
| Deadline Monotonic (DM)  | Response time analysis    | Fixed-priority, D <= T             |
| Round-Robin              | None (best-effort)        | Non-real-time, fairness            |

The timeline diagrams below show the problem and fix concretely. Without priority inheritance, Medium runs freely while High is stuck waiting - the effective priority order is inverted. With inheritance, Low temporarily borrows High's priority, finishes quickly, and releases the lock before Medium ever gets to run.

```text
  Priority Inversion Timeline:

  High --+ ####......................#####  (blocked waiting for mutex)
  Med  --+      @@@@@@@@         (preempts Low, unrelated work)
  Low  --+ @@@@.............@@@@#    (holds mutex, preempted by Med)
         +----------------------------> time

  # = executing  @ = executing (causing inversion)  . = blocked/preempted

  With Priority Inheritance:
  High --+ ####..##############################   (blocked briefly, runs after Low releases)
  Med  --+            @@@@@@@@   (cannot preempt inherited-priority Low)
  Low  --+ @@@@####             (inherits High priority, finishes fast)
         +---------------------------> time
```

Rate-Monotonic Analysis (RMA) provides a sufficient schedulability test: for N periodic tasks, the system is schedulable if total utilization U = sum(Ci/Ti) <= N(2^(1/N) - 1), which converges to ln(2) approximately 0.693 as N approaches infinity.

---

## Self-Assessment

### Q1: How do you implement a priority inheritance mutex in C++ that integrates with an RTOS

The key insight is that the mutex needs to know which task owns it and what that task's original priority was, so it can restore the original priority when the lock is released. The implementation below keeps it simple with a single waiter for clarity - production RTOS implementations handle a full wait queue, but the priority inheritance logic is the same.

```cpp
#include <cstdint>
#include <cstddef>

// Minimal RTOS abstraction layer (maps to FreeRTOS/Zephyr/RTEMS)
namespace rtos {
    using TaskHandle  = void*;
    using Priority    = std::uint32_t;

    // Platform-specific RTOS calls (implemented per target)
    extern TaskHandle  get_current_task() noexcept;
    extern Priority    get_task_priority(TaskHandle) noexcept;
    extern void        set_task_priority(TaskHandle, Priority) noexcept;
    extern void        enter_critical() noexcept;  // disable interrupts
    extern void        exit_critical() noexcept;   // restore interrupts
    extern void        yield() noexcept;
    extern void        suspend_task(TaskHandle) noexcept;
    extern void        resume_task(TaskHandle) noexcept;
}

// Priority Inheritance Mutex
// When a high-priority task blocks on this mutex, the owner's priority
// is temporarily raised to prevent inversion.
class PriorityInheritanceMutex {
    rtos::TaskHandle owner_         = nullptr;
    rtos::Priority   original_prio_ = 0;
    rtos::TaskHandle waiter_        = nullptr;  // simplified: single waiter
    bool             locked_        = false;

public:
    bool try_lock() noexcept {
        rtos::enter_critical();

        if (!locked_) {
            locked_ = true;
            owner_ = rtos::get_current_task();
            original_prio_ = rtos::get_task_priority(owner_);
            rtos::exit_critical();
            return true;
        }

        rtos::exit_critical();
        return false;
    }

    void lock() noexcept {
        rtos::enter_critical();

        if (!locked_) {
            locked_ = true;
            owner_ = rtos::get_current_task();
            original_prio_ = rtos::get_task_priority(owner_);
            rtos::exit_critical();
            return;
        }

        // Priority inheritance: boost owner if current task has higher priority
        auto current = rtos::get_current_task();
        auto current_prio = rtos::get_task_priority(current);
        auto owner_prio = rtos::get_task_priority(owner_);

        if (current_prio > owner_prio) {
            // Boost owner to our priority (higher number = higher priority)
            rtos::set_task_priority(owner_, current_prio);
        }

        waiter_ = current;
        rtos::exit_critical();

        // Block until owner releases (RTOS-specific wait mechanism)
        rtos::suspend_task(current);
    }

    void unlock() noexcept {
        rtos::enter_critical();

        // Restore original priority before releasing
        rtos::set_task_priority(owner_, original_prio_);

        locked_ = false;
        auto pending = waiter_;
        waiter_ = nullptr;
        owner_ = nullptr;

        rtos::exit_critical();

        // Wake the blocked waiter, if any
        if (pending) {
            rtos::resume_task(pending);
        }
    }
};

// RAII lock guard - ensures unlock on scope exit even with early return
class PiLockGuard {
    PriorityInheritanceMutex& mtx_;
public:
    explicit PiLockGuard(PriorityInheritanceMutex& m) noexcept : mtx_(m) {
        mtx_.lock();
    }
    ~PiLockGuard() noexcept { mtx_.unlock(); }

    PiLockGuard(const PiLockGuard&) = delete;
    PiLockGuard& operator=(const PiLockGuard&) = delete;
};

// Usage in task code
PriorityInheritanceMutex shared_sensor_mutex;
std::int32_t shared_sensor_value = 0;

void high_priority_controller_task() {
    PiLockGuard lock(shared_sensor_mutex);
    // Access shared_sensor_value with inversion protection
    std::int32_t val = shared_sensor_value;
    (void)val;
}
```

The reason `unlock` restores the priority before waking the waiter is subtle: if you wake the waiter first and it immediately preempts the current task, the priority restore might never happen. Restoring first ensures a clean handoff.

### Q2: How do you implement Rate-Monotonic scheduling analysis as a compile-time check in C++

Rate-Monotonic gives you a fast sufficient test (if utilization is below the bound, you're definitely schedulable) and an exact test (response-time analysis). The compile-time version below uses fixed-point integer arithmetic to avoid floating point in `constexpr` context, and exposes both tests as `static_assert` guards. If you add a new task or increase a WCET and the budget is exceeded, the build breaks.

```cpp
#include <cstdint>
#include <cstddef>
#include <array>

// Rate-Monotonic schedulability test:
// Sufficient condition: U = sum(Ci/Ti) <= N(2^(1/N) - 1)
// Exact test: response time analysis

struct TaskSpec {
    const char* name;
    std::uint32_t period_us;       // Ti: period in microseconds
    std::uint32_t wcet_us;         // Ci: worst-case execution time
    std::uint32_t deadline_us;     // Di: relative deadline (Di <= Ti for RM)
    std::uint32_t priority;        // Assigned by RM: shorter period = higher prio
};

// Fixed-point utilization to avoid floating point at compile time
// Represents utilization as parts-per-million for integer arithmetic
constexpr std::uint64_t utilization_ppm(std::uint32_t wcet, std::uint32_t period) {
    return (static_cast<std::uint64_t>(wcet) * 1'000'000u) / period;
}

// RM utilization bound: N(2^(1/N) - 1) x 1,000,000
// Precomputed for N=1..8 (converges to ln(2) approximately 693,147 ppm)
constexpr std::uint64_t rm_bound_ppm[] = {
    1'000'000,  // N=1: 100%
      828'427,  // N=2: 82.8%
      779'763,  // N=3: 78.0%
      756'828,  // N=4: 75.7%
      743'491,  // N=5: 74.3%
      734'772,  // N=6: 73.5%
      728'627,  // N=7: 72.9%
      724'062,  // N=8: 72.4%
};

// Compile-time schedulability check
template <std::size_t N>
constexpr bool is_rm_schedulable(const std::array<TaskSpec, N>& tasks) {
    static_assert(N >= 1 && N <= 8, "Supported for 1-8 tasks");

    std::uint64_t total_util = 0;
    for (std::size_t i = 0; i < N; ++i) {
        total_util += utilization_ppm(tasks[i].wcet_us, tasks[i].period_us);
    }

    return total_util <= rm_bound_ppm[N - 1];
}

// Response-time analysis (exact test, iterative)
// Ri = Ci + sum_j_in_hp(i) ceil(Ri/Tj) x Cj
// Converges when Ri stabilizes; task is schedulable if Ri <= Di
template <std::size_t N>
constexpr bool response_time_check(const std::array<TaskSpec, N>& tasks) {
    // Tasks must be sorted by priority (shortest period first for RM)
    for (std::size_t i = 0; i < N; ++i) {
        std::uint64_t r = tasks[i].wcet_us;

        for (int iter = 0; iter < 100; ++iter) {
            std::uint64_t interference = 0;
            for (std::size_t j = 0; j < i; ++j) {
                // Ceiling division: ceil(R/Tj)
                std::uint64_t preemptions = (r + tasks[j].period_us - 1)
                                          / tasks[j].period_us;
                interference += preemptions * tasks[j].wcet_us;
            }

            std::uint64_t r_new = tasks[i].wcet_us + interference;
            if (r_new == r) break;    // converged
            if (r_new > tasks[i].deadline_us) return false;  // missed deadline
            r = r_new;
        }
    }
    return true;
}

// System task set definition
constexpr std::array<TaskSpec, 4> system_tasks = {{
    {"imu_read",    1'000,   150,  1'000, 4},  // 1 kHz, 150 us WCET
    {"pid_control", 2'000,   300,  2'000, 3},  // 500 Hz, 300 us WCET
    {"can_tx",      5'000,   200,  5'000, 2},  // 200 Hz, 200 us WCET
    {"telemetry",  10'000,   500, 10'000, 1},  // 100 Hz, 500 us WCET
}};

// Utilization: 150/1000 + 300/2000 + 200/5000 + 500/10000
//            = 0.15 + 0.15 + 0.04 + 0.05 = 0.39 (39%)
// RM bound for N=4: 75.7% -> SCHEDULABLE

static_assert(is_rm_schedulable(system_tasks),
              "Task set fails RM utilization bound!");
static_assert(response_time_check(system_tasks),
              "Task set fails exact response time analysis!");
```

The utilization bound is a quick sanity check - if you're at 39% with four tasks you have plenty of headroom. The response-time check catches cases where the bound says "might be fine" but the actual worst-case response time of a low-priority task exceeds its deadline due to interference from higher-priority tasks.

### Q3: How do you implement a lock-free shared data pattern to avoid priority inversion entirely

The cleanest way to avoid priority inversion is to not use mutexes at all on the shared data path. A double buffer lets the writer (typically an ISR or high-priority task) always write to the inactive buffer, then atomically swap the active index. The reader always reads from the active buffer. There is no lock, so there is no inversion possible - the writer never blocks the reader and the reader never blocks the writer.

```cpp
#include <cstdint>
#include <atomic>
#include <array>

// Lock-free double buffer: writer and reader never block each other
// No mutex -> no priority inversion possible
// Suitable for single-writer/single-reader (SW-SR) sensor data sharing

template <typename T>
class DoubleBuffer {
    static_assert(std::is_trivially_copyable_v<T>,
                  "Must be trivially copyable for lock-free operation");

    std::array<T, 2> buffers_{};
    std::atomic<std::uint8_t> active_{0};  // index of the readable buffer

public:
    // Writer: always writes to the inactive buffer, then swaps
    void write(const T& data) noexcept {
        std::uint8_t inactive = 1u - active_.load(std::memory_order_relaxed);
        buffers_[inactive] = data;
        active_.store(inactive, std::memory_order_release);
    }

    // Reader: always reads from the active buffer (latest complete write)
    T read() const noexcept {
        std::uint8_t idx = active_.load(std::memory_order_acquire);
        return buffers_[idx];
    }
};

// Application: IMU data shared between ISR (writer) and control task (reader)
struct ImuData {
    std::int16_t accel_x, accel_y, accel_z;
    std::int16_t gyro_x,  gyro_y,  gyro_z;
    std::uint32_t timestamp_us;
};

DoubleBuffer<ImuData> imu_buffer;

// Called from DMA complete ISR - highest priority, must not block
extern "C" void DMA_ISR() {
    ImuData sample{};
    // Read from SPI DMA buffer (hardware-specific)
    // sample = read_from_dma_buffer();
    sample.timestamp_us = 0; // read from timer
    imu_buffer.write(sample);
}

// Called from control task (lower priority than ISR, higher than telemetry)
void control_loop() {
    ImuData latest = imu_buffer.read();  // never blocks, never inverts
    // Process latest IMU data for PID control
    (void)latest;
}

// Triple buffer variant for higher update rates (writer never waits)
template <typename T>
class TripleBuffer {
    std::array<T, 3> buffers_{};
    // Pack write_idx and read_idx into a single atomic for CAS
    struct State {
        std::uint8_t write_idx : 2;
        std::uint8_t ready_idx : 2;
        std::uint8_t read_idx  : 2;
    };
    std::atomic<std::uint16_t> state_{0x0102};  // w=0, ready=1, r=2

public:
    void write(const T& data) noexcept {
        auto s = state_.load(std::memory_order_relaxed);
        std::uint8_t w = s & 0x3;
        buffers_[w] = data;

        // Swap write and ready indices
        std::uint16_t expected = s;
        std::uint16_t desired;
        do {
            expected = state_.load(std::memory_order_relaxed);
            w = expected & 0x3;
            std::uint8_t ready = (expected >> 2) & 0x3;
            std::uint8_t r = (expected >> 4) & 0x3;
            desired = ready | (static_cast<std::uint16_t>(w) << 2) |
                      (static_cast<std::uint16_t>(r) << 4);
            buffers_[w] = data;
        } while (!state_.compare_exchange_weak(
            expected, desired,
            std::memory_order_release, std::memory_order_relaxed));
    }

    T read() noexcept {
        auto s = state_.load(std::memory_order_acquire);
        std::uint8_t ready = (s >> 2) & 0x3;
        std::uint8_t r = (s >> 4) & 0x3;
        // Swap read and ready
        std::uint16_t expected, desired;
        do {
            expected = state_.load(std::memory_order_relaxed);
            std::uint8_t w2 = expected & 0x3;
            ready = (expected >> 2) & 0x3;
            r = (expected >> 4) & 0x3;
            desired = w2 | (static_cast<std::uint16_t>(r) << 2) |
                      (static_cast<std::uint16_t>(ready) << 4);
        } while (!state_.compare_exchange_weak(
            expected, desired,
            std::memory_order_acq_rel, std::memory_order_relaxed));
        return buffers_[ready];
    }
};
```

The table below summarizes how the main synchronization strategies compare on the key real-time dimensions:

| Pattern            | Priority Inversion? | Latency  | Complexity |
|--------------------|---------------------|----------|------------|
| Mutex              | Yes (without PI)    | Variable | Low        |
| PI Mutex           | Bounded             | Variable | Medium     |
| Priority Ceiling   | None (immediate)    | Fixed    | Medium     |
| Double Buffer      | None                | O(1)     | Low        |
| Triple Buffer      | None                | O(1)     | Medium     |
| Lock-free queue    | None                | O(1)     | High       |

---

## Notes

- Priority inheritance solves unbounded inversion but adds overhead (priority tracking, context switches); priority ceiling is simpler if you know all accessor priorities at design time.
- FreeRTOS mutexes (`xSemaphoreCreateMutex`) implement priority inheritance by default; binary semaphores do NOT - never use binary semaphores for mutual exclusion.
- Rate-Monotonic is optimal among fixed-priority schedulers for periodic tasks where deadline = period; if D < T, use Deadline-Monotonic.
- The RM utilization bound is sufficient but not necessary - a task set can be schedulable even above the bound; use exact response-time analysis for tight systems.
- Lock-free double/triple buffers are ideal for ISR-to-task communication but only work for single-writer/single-reader; multi-writer requires lock-free queues.
- `std::atomic` on Cortex-M0 (no LDREX/STREX) may fall back to disabling interrupts - check `is_always_lock_free` and prefer interrupt masking explicitly.
- Jitter (variation in task start time) is as important as worst-case latency; analyze both in hard real-time systems.
