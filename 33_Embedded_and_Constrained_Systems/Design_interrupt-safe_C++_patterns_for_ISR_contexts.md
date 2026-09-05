# Design Interrupt-Safe C++ Patterns for ISR Contexts

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20 (`<atomic>`)  
**Reference:** https://en.cppreference.com/w/cpp/atomic  

---

## Topic Overview

Interrupt Service Routines (ISRs) run in a radically constrained context. On Cortex-M, an ISR preempts the main code (or a lower-priority ISR) at any instruction boundary. Inside an ISR you **cannot** use heap allocation, mutexes, blocking calls, or any function with unbounded execution time. You have microseconds - sometimes nanoseconds - before you must return. Violating these constraints causes priority inversion, stack overflow, missed deadlines, or outright hardware faults.

C++ adds specific dangers: virtual function calls through vtables add indirection overhead; `std::function` may allocate; template instantiations can silently pull in large code; and `std::mutex` / `std::lock_guard` will deadlock if called from an ISR that interrupted the lock holder.

| Constraint | Allowed in ISR | Forbidden in ISR |
| --- | --- | --- |
| Execution time | Bounded, minimal | Unbounded loops, complex algorithms |
| Memory allocation | Static/stack only | `new`, `malloc`, `std::vector::push_back` |
| Synchronization | `std::atomic`, LDREX/STREX | `std::mutex`, `std::condition_variable` |
| Data access | `volatile` for HW regs, `std::atomic` for shared data | Non-atomic shared state |
| Function calls | `constexpr`, trivial, `__attribute__((always_inline))` | Virtual calls (cache miss risk), `std::function` |
| C++ features | Enum class, `static_assert`, `constexpr`, templates | Exceptions, RTTI, iostream |
| FPU | Depends on lazy stacking config | Unguarded FPU use without stacking |

The diagram below shows what actually happens at the hardware level when an interrupt fires. The processor automatically saves eight registers onto the stack, runs your ISR, then restores them and resumes exactly where it left off. Your ISR code runs between those two hardware-managed stack frames.

```text
ISR Execution Model (Cortex-M):

Main Thread              ISR (higher priority)
    |                         |
    +- instruction N          |
    +- instruction N+1        |
    |   <- HW interrupt ------+
    |   (context saved        +- ISR prologue
    |    to stack:            +- read HW register
    |    R0-R3, R12, LR,     +- set flag / enqueue
    |    PC, xPSR)            +- clear interrupt
    |                         +- ISR epilogue
    |   <- context restored ---+
    +- instruction N+2        |
    |   (resumes exactly      |
    |    where preempted)     |
    +-------------------------+
```

The fundamental ISR pattern is: **acknowledge the interrupt, capture minimal data, signal the main loop, and return**. This is the "deferred processing" or "bottom-half" pattern. The ISR is the "top half" - it sets a flag or enqueues data. The main loop (or an RTOS task) is the "bottom half" - it processes the data with full C++ capabilities.

---

## Self-Assessment

### Q1: Implement a lock-free, ISR-safe SPSC (Single-Producer Single-Consumer) ring buffer using `std::atomic` with correct memory ordering for Cortex-M

The reason this trips people up is the memory ordering. On a single-core Cortex-M, interrupt entry and exit act as full memory barriers - but if you ever port to a dual-core device or want portable code, `acquire`/`release` ordering is the right choice. The producer (ISR) owns the write index, the consumer (main loop) owns the read index, and neither side needs to do a heavyweight CAS because only one producer and one consumer ever exist.

```cpp
// isr_ringbuf.h - Lock-free SPSC ring buffer for ISR -> main communication
#pragma once

#include <cstdint>
#include <cstddef>
#include <atomic>
#include <type_traits>
#include <optional>
#include <array>

namespace isr {

// T must be trivially copyable - no constructors/destructors to worry about
// N must be power of 2 for efficient modulo via bitmask
template <typename T, std::size_t N>
class SpscRingBuffer {
    static_assert(std::is_trivially_copyable_v<T>,
                  "ISR ring buffer elements must be trivially copyable");
    static_assert(N > 0 && (N & (N - 1)) == 0,
                  "Buffer size must be a power of 2");

    std::array<T, N> buffer_{};

    // Producer (ISR) owns write_idx_; consumer (main) owns read_idx_
    // Both are atomic to allow safe cross-context visibility
    //
    // Memory ordering rationale:
    //   - On Cortex-M (single-core), relaxed is sufficient because
    //     interrupt entry/exit implies a full memory barrier.
    //   - We use acquire/release for portability to multi-core / DMA scenarios.
    alignas(64) std::atomic<std::size_t> write_idx_{0};
    alignas(64) std::atomic<std::size_t> read_idx_{0};

    static constexpr std::size_t MASK = N - 1;

public:
    // Called from ISR context (producer)
    [[nodiscard]]
    bool try_push(const T& item) {
        const auto w = write_idx_.load(std::memory_order_relaxed);
        const auto r = read_idx_.load(std::memory_order_acquire);

        // Full when write is one slot behind read
        if (((w + 1) & MASK) == r) {
            return false;  // Buffer full - drop or handle overflow
        }

        buffer_[w] = item;  // Trivial copy - safe in ISR

        write_idx_.store((w + 1) & MASK, std::memory_order_release);
        return true;
    }

    // Called from main/thread context (consumer)
    [[nodiscard]]
    std::optional<T> try_pop() {
        const auto r = read_idx_.load(std::memory_order_relaxed);
        const auto w = write_idx_.load(std::memory_order_acquire);

        if (r == w) {
            return std::nullopt;  // Buffer empty
        }

        T item = buffer_[r];  // Trivial copy

        read_idx_.store((r + 1) & MASK, std::memory_order_release);
        return item;
    }

    [[nodiscard]]
    std::size_t size() const {
        const auto w = write_idx_.load(std::memory_order_acquire);
        const auto r = read_idx_.load(std::memory_order_acquire);
        return (w - r) & MASK;
    }

    [[nodiscard]] bool empty() const { return size() == 0; }
    [[nodiscard]] bool full()  const { return size() == N - 1; }
    [[nodiscard]] static constexpr std::size_t capacity() { return N - 1; }
};

// ---- Usage example ----

struct AdcSample {
    std::uint16_t channel;
    std::uint16_t value;
    std::uint32_t timestamp;
};
static_assert(std::is_trivially_copyable_v<AdcSample>);

// Global ring buffer - ISR writes, main loop reads
inline SpscRingBuffer<AdcSample, 64> g_adc_fifo;

// ISR handler - kept minimal
extern "C" void ADC_IRQHandler() {
    // Read HW registers
    volatile auto* adc_dr   = reinterpret_cast<volatile std::uint32_t*>(0x4001'204C);
    volatile auto* systick   = reinterpret_cast<volatile std::uint32_t*>(0xE000'E018);

    AdcSample s{
        .channel   = static_cast<std::uint16_t>((*adc_dr >> 16) & 0x1F),
        .value     = static_cast<std::uint16_t>(*adc_dr & 0xFFFF),
        .timestamp = *systick
    };

    g_adc_fifo.try_push(s);  // Lock-free, bounded time, no allocation

    // Clear interrupt flag
    volatile auto* adc_sr = reinterpret_cast<volatile std::uint32_t*>(0x4001'2000);
    *adc_sr &= ~(1u << 1);  // Clear EOC flag
}

} // namespace isr
```

Notice that `try_push` uses `release` when storing the updated write index. That store is the "handshake" that makes the written data visible to the consumer. The consumer does an `acquire` load on the write index, which synchronizes with that release store - ensuring the buffer contents are visible before the index change is seen. Without that pairing, the consumer could read a stale buffer slot.

### Q2: Design a type-safe hardware register abstraction using `volatile` correctly, ensuring the compiler does not optimize away reads/writes to memory-mapped I/O

The C++20 version below uses concepts (`requires`) to gate access policies, which gives cleaner error messages than `static_assert`. The pattern is the same as the template approach: wrap a raw `volatile` pointer in a class that only exposes operations consistent with the register's access policy. Every method compiles down to a single load or store instruction.

```cpp
// hw_register.h - Volatile-correct register access patterns
#pragma once

#include <cstdint>
#include <type_traits>

namespace hw {

// Access policy tags
struct ReadOnly  {};
struct WriteOnly {};
struct ReadWrite {};

// Memory-mapped register with access policy
template <std::uintptr_t Addr, typename Policy = ReadWrite,
          typename RegType = std::uint32_t>
class Register {
    static_assert(std::is_unsigned_v<RegType>, "Register type must be unsigned");

    // The volatile pointer - critical for MMIO correctness
    static auto ptr() {
        return reinterpret_cast<volatile RegType*>(Addr);
    }

public:
    // Read - available for ReadOnly and ReadWrite
    [[nodiscard]]
    static RegType read()
        requires (std::is_same_v<Policy, ReadOnly> ||
                  std::is_same_v<Policy, ReadWrite>)
    {
        return *ptr();  // volatile read - never optimized away
    }

    // Write - available for WriteOnly and ReadWrite
    static void write(RegType val)
        requires (std::is_same_v<Policy, WriteOnly> ||
                  std::is_same_v<Policy, ReadWrite>)
    {
        *ptr() = val;   // volatile write - never optimized away
    }

    // Set bits (read-modify-write) - ReadWrite only
    static void set_bits(RegType mask)
        requires std::is_same_v<Policy, ReadWrite>
    {
        *ptr() |= mask;
    }

    // Clear bits - ReadWrite only
    static void clear_bits(RegType mask)
        requires std::is_same_v<Policy, ReadWrite>
    {
        *ptr() &= ~mask;
    }

    // Modify a bitfield: clear field, then set new value
    template <unsigned Pos, unsigned Width>
    static void write_field(RegType value)
        requires std::is_same_v<Policy, ReadWrite>
    {
        static_assert(Pos + Width <= sizeof(RegType) * 8);
        constexpr RegType mask = ((RegType{1} << Width) - 1) << Pos;
        RegType reg = *ptr();
        reg &= ~mask;
        reg |= (value << Pos) & mask;
        *ptr() = reg;
    }

    // Read a bitfield
    template <unsigned Pos, unsigned Width>
    [[nodiscard]]
    static RegType read_field()
        requires (std::is_same_v<Policy, ReadOnly> ||
                  std::is_same_v<Policy, ReadWrite>)
    {
        static_assert(Pos + Width <= sizeof(RegType) * 8);
        constexpr RegType mask = ((RegType{1} << Width) - 1) << Pos;
        return (*ptr() & mask) >> Pos;
    }
};

// ---------- Example: UART peripheral registers ----------

// STM32 USART1 base: 0x4001'1000
namespace uart1 {
    using SR   = Register<0x4001'1000, ReadWrite>;  // Status register
    using DR   = Register<0x4001'1004, ReadWrite>;  // Data register
    using BRR  = Register<0x4001'1008, ReadWrite>;  // Baud rate
    using CR1  = Register<0x4001'100C, ReadWrite>;  // Control reg 1
    using CR2  = Register<0x4001'1010, ReadWrite>;  // Control reg 2

    // Status register bits
    constexpr std::uint32_t SR_TXE  = 1u << 7;  // TX empty
    constexpr std::uint32_t SR_RXNE = 1u << 5;  // RX not empty
    constexpr std::uint32_t SR_TC   = 1u << 6;  // TX complete

    // ISR-safe transmit - bounded, no allocation
    inline bool try_send_byte(std::uint8_t byte) {
        if (SR::read() & SR_TXE) {
            DR::write(byte);
            return true;
        }
        return false;
    }

    // ISR-safe receive
    inline std::optional<std::uint8_t> try_recv_byte() {
        if (SR::read() & SR_RXNE) {
            return static_cast<std::uint8_t>(DR::read() & 0xFF);
        }
        return std::nullopt;
    }
}

} // namespace hw
```

`try_send_byte` and `try_recv_byte` are the kind of functions you'd call directly from an ISR. They're bounded (a single register read, a conditional, an optional write), they don't allocate, and they can't throw. That's the ISR contract: do the minimum work, return promptly.

### Q3: Implement the deferred processing pattern (top-half/bottom-half) using atomic flags and a priority-aware event queue that is safe to call from nested ISRs

The top-half/bottom-half split is the most important architectural pattern for ISR-to-task communication. The ISR (top half) does the absolute minimum - acknowledge the hardware and set a flag. The main loop (bottom half) processes everything. The `EventFlags` class uses `fetch_or` for setting flags, which is lock-free on Cortex-M for 32-bit types and safe even from nested ISRs. The `EventDispatcher` then processes pending events in priority order from the main loop.

```cpp
// deferred_processing.h - ISR-safe event dispatch
#pragma once

#include <cstdint>
#include <atomic>
#include <array>
#include <type_traits>

namespace isr {

// Event types - kept as a bitfield for O(1) pending check
enum class Event : std::uint32_t {
    None        = 0,
    AdcComplete = 1u << 0,
    UartRx      = 1u << 1,
    TimerTick   = 1u << 2,
    SpiDone     = 1u << 3,
    GpioEdge    = 1u << 4,
    DmaComplete = 1u << 5,
    Watchdog    = 1u << 6,
    ErrorFlag   = 1u << 7,
};

// Bitwise operators for Event
constexpr Event operator|(Event a, Event b) {
    return static_cast<Event>(
        static_cast<std::uint32_t>(a) | static_cast<std::uint32_t>(b));
}
constexpr Event operator&(Event a, Event b) {
    return static_cast<Event>(
        static_cast<std::uint32_t>(a) & static_cast<std::uint32_t>(b));
}
constexpr Event operator~(Event a) {
    return static_cast<Event>(~static_cast<std::uint32_t>(a));
}

// ---------- Atomic Event Flags ----------
// ISRs set flags atomically; main loop processes and clears them.
// Works correctly even with nested interrupts.

class EventFlags {
    std::atomic<std::uint32_t> flags_{0};

public:
    // Called from ISR - O(1), lock-free, safe in nested ISR
    void set(Event e) {
        flags_.fetch_or(static_cast<std::uint32_t>(e),
                        std::memory_order_release);
    }

    // Called from main loop - atomically read and clear all flags
    [[nodiscard]]
    Event consume_all() {
        return static_cast<Event>(
            flags_.exchange(0, std::memory_order_acq_rel));
    }

    // Called from main loop - atomically consume specific flags
    [[nodiscard]]
    bool consume(Event e) {
        auto expected = flags_.load(std::memory_order_relaxed);
        auto mask = static_cast<std::uint32_t>(e);
        while (expected & mask) {
            if (flags_.compare_exchange_weak(
                    expected, expected & ~mask,
                    std::memory_order_acq_rel,
                    std::memory_order_relaxed)) {
                return true;
            }
        }
        return false;
    }

    [[nodiscard]]
    bool any_pending() const {
        return flags_.load(std::memory_order_acquire) != 0;
    }
};

// ---------- Handler Table ----------
// Static dispatch - no virtual calls, no std::function, no allocation

using EventHandler = void(*)(void* context);

struct HandlerEntry {
    EventHandler handler = nullptr;
    void*        context = nullptr;
    std::uint8_t priority = 255;  // Lower = higher priority
};

class EventDispatcher {
    static constexpr std::size_t MAX_EVENTS = 8;

    EventFlags flags_;
    std::array<HandlerEntry, MAX_EVENTS> handlers_{};

    static constexpr unsigned event_index(Event e) {
        // Find bit position - assumes single-bit Event values
        auto val = static_cast<std::uint32_t>(e);
        unsigned idx = 0;
        while (val >>= 1) ++idx;
        return idx;
    }

public:
    // Register a handler (called during init, not from ISR)
    void register_handler(Event e, EventHandler h, void* ctx,
                          std::uint8_t prio = 128) {
        auto idx = event_index(e);
        if (idx < MAX_EVENTS) {
            handlers_[idx] = {h, ctx, prio};
        }
    }

    // Called from ISR - just set the flag
    void signal(Event e) {
        flags_.set(e);
    }

    // Called from main loop - dispatch pending events by priority
    void process() {
        auto pending = flags_.consume_all();
        if (pending == Event::None) return;

        // Sort by priority: collect pending handlers
        struct PendingHandler {
            EventHandler handler;
            void* context;
            std::uint8_t priority;
        };
        std::array<PendingHandler, MAX_EVENTS> to_run{};
        std::size_t count = 0;

        for (std::size_t i = 0; i < MAX_EVENTS; ++i) {
            auto bit = static_cast<Event>(1u << i);
            if ((pending & bit) != Event::None && handlers_[i].handler) {
                to_run[count++] = {
                    handlers_[i].handler,
                    handlers_[i].context,
                    handlers_[i].priority
                };
            }
        }

        // Simple insertion sort (bounded N=8, faster than qsort overhead)
        for (std::size_t i = 1; i < count; ++i) {
            auto key = to_run[i];
            std::size_t j = i;
            while (j > 0 && to_run[j-1].priority > key.priority) {
                to_run[j] = to_run[j-1];
                --j;
            }
            to_run[j] = key;
        }

        // Dispatch in priority order
        for (std::size_t i = 0; i < count; ++i) {
            to_run[i].handler(to_run[i].context);
        }
    }
};

// ---------- Global instance ----------
inline EventDispatcher g_dispatcher;

// ---------- ISR implementations (top-half) ----------
extern "C" void ADC_IRQHandler() {
    // Minimal: acknowledge + signal
    volatile auto* adc_sr = reinterpret_cast<volatile std::uint32_t*>(0x4001'2000);
    *adc_sr &= ~(1u << 1);  // Clear EOC
    g_dispatcher.signal(Event::AdcComplete);
}

extern "C" void USART1_IRQHandler() {
    volatile auto* sr = reinterpret_cast<volatile std::uint32_t*>(0x4001'1000);
    if (*sr & (1u << 5)) {  // RXNE
        g_dispatcher.signal(Event::UartRx);
    }
}

} // namespace isr
```

The handler table uses raw function pointers and a `void*` context rather than `std::function`. The reason: `std::function` may allocate on the heap internally (for large callables), which is forbidden in ISR-reachable code. Raw function pointers are always one word, always lock-free, and the `void*` context covers the same use cases with no overhead.

---

## Notes

- `std::atomic::fetch_or` (C++11) is **lock-free on Cortex-M** for 32-bit types - it compiles to `LDREX`/`STREX` which are safe even in nested ISRs.
- `volatile` is for **hardware registers** - it guarantees the compiler performs the read/write. It does **not** guarantee atomicity or ordering between cores. Use `std::atomic` for shared data between ISR and main.
- On Cortex-M, interrupt entry/exit acts as a **compiler barrier** (but not a hardware memory barrier on multi-core). Single-core Cortex-M0/M3/M4 can use `memory_order_relaxed` safely for ISR communication, but `acquire`/`release` is preferred for portability.
- Never use `std::mutex` or `std::lock_guard` in any code path reachable from an ISR - these are blocking primitives that will deadlock.
- The "critical section" pattern on Cortex-M is `__disable_irq()` / `__enable_irq()` (CPSID/CPSIE instructions). Use sparingly - it increases interrupt latency for **all** peripherals.
- MISRA C++ 2023 Rule 6.8.1 requires that ISR functions have `extern "C"` linkage - the vector table expects C calling convention.
- FPU registers (S0-S15) are lazily stacked on Cortex-M4F by default. If your ISR uses `float`, ensure lazy stacking is enabled in the FPCCR register, or manually save/restore FPU context.
