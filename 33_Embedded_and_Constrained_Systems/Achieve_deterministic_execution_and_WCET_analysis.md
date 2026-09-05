# Achieve Deterministic Execution and WCET Analysis

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** [AbsInt aiT WCET Analyzer](https://www.absint.com/ait/), [ARM Cortex-M DWT](https://developer.arm.com/documentation/ddi0403/latest/)  

---

## Topic Overview

Worst-Case Execution Time (WCET) analysis is the cornerstone of hard real-time systems. A task that *usually* completes in 50 µs but occasionally spikes to 2 ms is unacceptable when a missed deadline can cause physical harm. WCET analysis determines an upper bound on execution time through static analysis (abstract interpretation of the binary against a processor timing model, as done by aiT or OTAWA), measurement-based approaches (instrumented runs capturing high-water marks, as done by RapiTime), or hybrid methods combining both. The goal is a safe, tight bound: safe enough that no execution ever exceeds it, tight enough that you don't waste schedulability margin.

Deterministic execution in C++ requires disciplining the language. Dynamic memory allocation (`new`/`malloc`) introduces non-deterministic latency due to heap fragmentation and allocator locking. Exceptions add hidden control flow and stack unwinding that defeats static WCET tools. Virtual dispatch through vtables introduces indirect branches that complicate pipeline and cache analysis. In safety-critical domains (DO-178C, ISO 26262, IEC 61508), these features are typically banned or heavily restricted in the hot path. MISRA C++ and AUTOSAR C++14 guidelines encode many of these restrictions.

On ARM Cortex-M processors, the Data Watchpoint and Trace (DWT) unit provides a cycle counter (`DWT->CYCCNT`) that enables precise, low-overhead timing measurement at the hardware level. This is the primary tool for measurement-based WCET estimation on embedded targets. You instrument critical sections, collect cycle counts across representative and stress inputs, and apply statistical margins. Combined with static analysis to confirm no unbounded loops or recursion exist, this yields production-grade WCET estimates.

The practical discipline is: allocate everything statically or from pools, replace virtual dispatch with CRTP or `std::variant` dispatch, avoid exceptions in real-time paths (use error codes or `std::expected`), eliminate unbounded loops, and prove every code path has a known maximum iteration count. The compiler must also cooperate - link-time optimization and `-fno-exceptions` help tools reason about the final binary.

---

## Self-Assessment

### Q1: How do you build a fixed-capacity container that provides deterministic allocation for real-time code

The key insight here is that `std::vector` is off the table in hard real-time code because it calls `new` internally. Instead, you store everything in a fixed-size array embedded directly in the object. The container below never touches the heap - `push_back` either succeeds and returns `true`, or tells you it's full and returns `false`. WCET tools love this because the maximum number of iterations through any loop is provably bounded.

```cpp
#include <cstddef>
#include <cstdint>
#include <new>
#include <type_traits>
#include <algorithm>

// A stack-allocated vector with compile-time capacity.
// Zero heap allocation, O(1) push/pop, fully deterministic.
template <typename T, std::size_t Capacity>
class StaticVector {
    static_assert(Capacity > 0, "Capacity must be positive");

    alignas(T) std::byte storage_[sizeof(T) * Capacity];
    std::size_t size_ = 0;

    T* data() noexcept {
        return std::launder(reinterpret_cast<T*>(storage_));
    }
    const T* data() const noexcept {
        return std::launder(reinterpret_cast<const T*>(storage_));
    }

public:
    StaticVector() = default;

    ~StaticVector() {
        for (std::size_t i = 0; i < size_; ++i)
            data()[i].~T();
    }

    // Returns false if full - no exceptions, no UB.
    [[nodiscard]] bool push_back(const T& value) noexcept(
        std::is_nothrow_copy_constructible_v<T>)
    {
        if (size_ >= Capacity) return false;
        ::new (static_cast<void*>(&storage_[sizeof(T) * size_])) T(value);
        ++size_;
        return true;
    }

    [[nodiscard]] bool push_back(T&& value) noexcept(
        std::is_nothrow_move_constructible_v<T>)
    {
        if (size_ >= Capacity) return false;
        ::new (static_cast<void*>(&storage_[sizeof(T) * size_])) T(std::move(value));
        ++size_;
        return true;
    }

    void pop_back() noexcept {
        if (size_ > 0) {
            --size_;
            data()[size_].~T();
        }
    }

    T& operator[](std::size_t i) noexcept { return data()[i]; }
    const T& operator[](std::size_t i) const noexcept { return data()[i]; }

    [[nodiscard]] std::size_t size() const noexcept { return size_; }
    [[nodiscard]] bool empty() const noexcept { return size_ == 0; }
    [[nodiscard]] constexpr std::size_t capacity() const noexcept { return Capacity; }

    T* begin() noexcept { return data(); }
    T* end() noexcept { return data() + size_; }
    const T* begin() const noexcept { return data(); }
    const T* end() const noexcept { return data() + size_; }
};

// Usage in a hard real-time control loop:
struct SensorReading { uint32_t timestamp; float value; };

void process_cycle(StaticVector<SensorReading, 64>& readings) {
    // Bounded iteration - WCET tools can determine max 64 iterations
    float sum = 0.0f;
    for (const auto& r : readings) {
        sum += r.value;
    }
    float avg = readings.empty() ? 0.0f : sum / static_cast<float>(readings.size());
    // Feed avg into controller - details omitted
    (void)avg;
}
```

Notice that `process_cycle` iterates over at most 64 elements - that upper bound is baked into the type. A WCET tool analyzing this function sees a loop with a provable trip count of 64 and can compute a tight timing bound. With a heap-backed `std::vector` you'd have no such guarantee.

### Q2: How do you measure WCET at the hardware level using the ARM DWT cycle counter

The DWT (Data Watchpoint and Trace) unit is a free cycle counter built into Cortex-M processors. It counts processor cycles at zero runtime overhead. The pattern below uses RAII to bracket any section of code and track the maximum observed cycle count across all invocations - that running maximum is your measured WCET estimate.

```cpp
#include <cstdint>

// ARM Cortex-M DWT register definitions (memory-mapped)
namespace dwt {
    inline volatile uint32_t& DEMCR =
        *reinterpret_cast<volatile uint32_t*>(0xE000EDFC);
    inline volatile uint32_t& DWT_CTRL =
        *reinterpret_cast<volatile uint32_t*>(0xE0001000);
    inline volatile uint32_t& DWT_CYCCNT =
        *reinterpret_cast<volatile uint32_t*>(0xE0001004);

    constexpr uint32_t DEMCR_TRCENA  = 1u << 24;
    constexpr uint32_t DWT_CTRL_ENABLE = 1u << 0;

    inline void enable_cycle_counter() {
        DEMCR    |= DEMCR_TRCENA;       // Enable trace
        DWT_CYCCNT = 0;                  // Reset counter
        DWT_CTRL |= DWT_CTRL_ENABLE;    // Start counting
    }

    inline uint32_t read_cycles() {
        return DWT_CYCCNT;
    }
}

// RAII cycle measurement - records high-water mark for WCET estimation
class CycleMeasurement {
    uint32_t start_;
    uint32_t& max_observed_;

public:
    explicit CycleMeasurement(uint32_t& max_out) noexcept
        : start_(dwt::read_cycles()), max_observed_(max_out) {}

    ~CycleMeasurement() noexcept {
        uint32_t elapsed = dwt::read_cycles() - start_; // Handles wrap
        if (elapsed > max_observed_)
            max_observed_ = elapsed;
    }

    CycleMeasurement(const CycleMeasurement&) = delete;
    CycleMeasurement& operator=(const CycleMeasurement&) = delete;
};

// Usage: measure WCET of a control function across all invocations
static uint32_t pid_wcet_cycles = 0;

struct PIDState { float integral; float prev_error; };

float pid_update(PIDState& state, float setpoint, float measured,
                 float kp, float ki, float kd, float dt) {
    CycleMeasurement measure(pid_wcet_cycles);

    float error = setpoint - measured;
    state.integral += error * dt;
    // Clamp integral (bounded operation - deterministic)
    constexpr float integral_limit = 1000.0f;
    if (state.integral > integral_limit) state.integral = integral_limit;
    if (state.integral < -integral_limit) state.integral = -integral_limit;

    float derivative = (error - state.prev_error) / dt;
    state.prev_error = error;

    return kp * error + ki * state.integral + kd * derivative;
}

// After running N cycles, pid_wcet_cycles holds the observed WCET.
// Convert: time_us = pid_wcet_cycles / (SystemCoreClock / 1'000'000)
```

After running the function across all representative inputs and stress scenarios, `pid_wcet_cycles` holds the observed worst case. You then divide by the clock frequency to get microseconds. The key caution: DWT counts include time spent in interrupt handlers that preempted the function, so for a true function-level WCET you should either disable interrupts during the measurement or subtract observed preemption overhead.

### Q3: How do you eliminate virtual dispatch overhead in hot paths while retaining polymorphic design

Virtual dispatch is a problem for WCET tools because it's an indirect branch through a vtable pointer - the tool can't always resolve the target statically, and the indirect call can miss the instruction cache. The two clean alternatives are CRTP (resolved entirely at compile time) and `std::variant` dispatch (generates a bounded switch/if-else chain). Both give the compiler full visibility into every possible call target.

```cpp
#include <cstdint>
#include <variant>

// CRTP: static polymorphism - zero overhead, fully analyzable by WCET tools.
// The compiler sees the exact function at every call site.
template <typename Derived>
class ControllerBase {
public:
    // Deterministic: no vtable lookup, no indirect branch
    float compute(float error, float dt) noexcept {
        return static_cast<Derived*>(this)->compute_impl(error, dt);
    }
};

class PController : public ControllerBase<PController> {
    float kp_;
public:
    explicit PController(float kp) noexcept : kp_(kp) {}
    float compute_impl(float error, float /*dt*/) noexcept {
        return kp_ * error;
    }
};

class PIController : public ControllerBase<PIController> {
    float kp_, ki_, integral_ = 0.0f;
public:
    PIController(float kp, float ki) noexcept : kp_(kp), ki_(ki) {}
    float compute_impl(float error, float dt) noexcept {
        integral_ += error * dt;
        constexpr float clamp = 500.0f;
        if (integral_ > clamp) integral_ = clamp;
        if (integral_ < -clamp) integral_ = -clamp;
        return kp_ * error + ki_ * integral_;
    }
};

// std::variant dispatch - deterministic, bounded set of types,
// no heap allocation, compiler can inline all paths.
using AnyController = std::variant<PController, PIController>;

float dispatch_compute(AnyController& ctrl, float error, float dt) noexcept {
    return std::visit([&](auto& c) noexcept -> float {
        return c.compute(error, dt);
    }, ctrl);
}

// In the real-time loop - everything is stack/static, fully deterministic
void control_tick(AnyController& ctrl, float setpoint, float measured) {
    constexpr float dt = 0.001f; // 1 kHz loop
    float error = setpoint - measured;
    float output = dispatch_compute(ctrl, error, dt);

    // Write to DAC / PWM - hardware-specific, omitted
    (void)output;
}
```

`std::variant` visit generates a jump table or if-else chain over a bounded set of types. Both forms are fully analyzable by WCET tools, unlike a virtual dispatch through an arbitrary function pointer. The reason this matters so much is that WCET tools like aiT need to know every possible code path to compute a safe bound - an indirect call with unknown targets makes that impossible.

---

## Notes

- **Ban `new`/`delete` in real-time paths.** Use `StaticVector`, `std::array`, pool allocators, or placement new into pre-allocated buffers. Override global `operator new` to trap accidental allocations in debug builds.
- **Compile with `-fno-exceptions -fno-rtti`** for hard real-time targets. This eliminates hidden unwinding tables and RTTI overhead, and makes binaries more amenable to static WCET analysis.
- **Every loop must have a provable upper bound.** WCET tools like aiT require bounded loop annotations or automatically infer them. Unbounded iteration makes static analysis impossible.
- **Avoid `std::unordered_map` and other hash containers** - rehashing is non-deterministic. Prefer `std::array`-backed lookup tables or sorted `StaticVector` with binary search (O(log N), bounded).
- **DWT cycle counts include interrupt preemption time.** For true function-level WCET, disable interrupts around the measurement or subtract observed preemption. On Cortex-M, `__disable_irq()` / `__enable_irq()` bracket the critical section.
- **Static WCET analysis (aiT, OTAWA) operates on the compiled binary**, not source code. Always analyze the exact binary deployed to the target, with the same optimization level and linker configuration.
- **Cache effects dominate WCET on Cortex-A / Cortex-R.** On Cortex-M (typically no cache), cycle counts are more predictable. For cached cores, WCET tools model cache state per program point - expect 3-10x difference between BCET and WCET.
- **`std::variant` visit** generates a jump table or if-else chain - both are bounded and analyzable, unlike virtual dispatch through an arbitrary function pointer.
- **Mark hot-path functions `[[gnu::flatten]]` or `__attribute__((always_inline))`** to ensure inlining. Indirect calls defeat WCET flow analysis.
