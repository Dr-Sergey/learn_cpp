# Analyze Stack Usage and Enforce Static Memory Budgets

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** https://gcc.gnu.org/onlinedocs/gccint/Stack-Checking.html  

---

## Topic Overview

Embedded systems operate with fixed, often tiny RAM budgets - a Cortex-M0 may have 8 KB total SRAM shared between stack, heap, and static data. Stack overflow is the number one cause of mysterious embedded crashes, and it manifests as corrupted variables, HardFaults, or silent data corruption. Unlike desktop systems, there is no virtual memory to catch the fault gracefully.

Static stack analysis determines the maximum stack depth at compile/link time by building a call graph and summing frame sizes. GCC's `-fstack-usage` emits per-function `.su` files; tools like `avstack.py`, `Cppcheck`, or Arm Compiler's `--callgraph` produce whole-program worst-case stack depth. Dynamic detection uses canary values or MPU-based guard regions to catch overflow at runtime.

| Technique                     | When                | Precision        | Overhead         |
| --- | --- | --- | --- |
| `-fstack-usage` (.su files)   | Compile time        | Per-function     | Zero runtime     |
| Call graph analysis            | Link time           | Whole-program    | Zero runtime     |
| Linker map file               | Link time           | Section sizes    | Zero runtime     |
| Stack canary (painting)       | Runtime             | Detects overflow | ~0.1% CPU        |
| MPU guard region              | Runtime             | Hardware trap    | Zero CPU cost    |
| `-fstack-protector`           | Runtime             | Per-function     | ~1-2% overhead   |

Here is a typical Cortex-M RAM layout to keep in mind. The stack grows downward from the top of SRAM; if it ever collides with the heap or static data growing upward, you have overflow and probably a crash.

```text
  RAM Layout (typical Cortex-M):
  +------------------+  0x2000_0000
  |  .data (init'd)  |
  +------------------+
  |  .bss  (zeroed)  |
  +------------------+
  |  Heap -------->  |  (grows up, if used)
  |                  |
  |    <-------- Stack |  (grows down)
  +------------------+
  |  Stack guard     |  <- canary / MPU region
  +------------------+  0x2000_2000 (8KB)
```

The memory budget must account for: interrupt stacking (Cortex-M pushes 8 registers = 32 bytes per exception), nested interrupts (each level adds a frame), and RTOS task stacks (each task has its own). A common rule: analyze maximum call depth, add interrupt overhead, then add a 20% safety margin.

---

## Self-Assessment

### Q1: How do you parse `-fstack-usage` output and build a worst-case call graph analysis tool

When you compile with `-fstack-usage`, GCC writes a `.su` file next to each object file. Each line tells you a function's name and its stack frame size. The trick is to combine those per-function numbers with your call graph to get a whole-program worst-case stack depth - you walk the deepest call chain and sum the frame sizes. The example below shows how to encode that analysis in `constexpr` so the compiler enforces your budget at build time.

```cpp
// Compile with: g++ -fstack-usage -c myfile.cpp
// Produces myfile.su with lines like:
//   myfile.cpp:12:6:foo  64  static
//   myfile.cpp:25:6:bar  128 dynamic,bounded

#include <cstdint>
#include <cstddef>

// Simulating what the .su file tells us - encode as constexpr for static checks
struct StackEntry {
    const char* function_name;
    std::size_t frame_size;
    bool        is_dynamic;  // true if VLA or alloca detected
};

// Extracted from .su files (in practice, scripted from build output)
constexpr StackEntry stack_table[] = {
    {"main",                    96,  false},
    {"init_peripherals",        48,  false},
    {"run_control_loop",        64,  false},
    {"pid_update",             128,  false},
    {"read_sensor_spi",         32,  false},
    {"apply_actuator_pwm",      24,  false},
    {"uart_log_fixed",          56,  false},
    {"error_handler",           80,  false},
    {"ISR_SysTick",             40,  false},
    {"ISR_DMA_Complete",        36,  false},
};

// Call graph: worst-case path analysis
// main -> run_control_loop -> pid_update -> read_sensor_spi
// Plus SysTick interrupt nesting

constexpr std::size_t worst_case_call_chain() {
    // Deepest call path: main + run_control_loop + pid_update + read_sensor_spi
    std::size_t call_path = 96 + 64 + 128 + 32;  // = 320 bytes

    // Cortex-M exception frame: 8 registers x 4 bytes = 32 bytes
    constexpr std::size_t exception_frame = 32;

    // FPU context (if Cortex-M4F): additional 68 bytes (lazy stacking)
    constexpr std::size_t fpu_frame = 68;

    // Worst case: deepest call path + 1 ISR nesting
    std::size_t isr_overhead = exception_frame + fpu_frame + 40; // SysTick frame
    std::size_t total = call_path + isr_overhead;

    // 20% safety margin
    return total + (total / 5);
}

constexpr std::size_t STACK_BUDGET = 1024;
static_assert(worst_case_call_chain() <= STACK_BUDGET,
              "WCSS exceeds allocated stack budget!");

// Linker script enforcement:
// STACK_SIZE = 1024;
// ._stack_guard (NOLOAD) : { . += 32; } > RAM  /* MPU guard */
// ._stack (NOLOAD) : { . += STACK_SIZE; } > RAM
```

The `static_assert` at the bottom is the payoff. If you add a new function to a deep call chain and the budget is exceeded, the build breaks immediately - you catch the problem at compile time, not during a field deployment. In practice, the numbers in the analysis would be extracted from `.su` files by a build script rather than hand-typed.

### Q2: How do you implement runtime stack painting to detect high-water-mark usage post-mortem

Stack painting fills the unused portion of the stack with a known pattern at startup. Later, you scan from the bottom upward and see how far the paint got overwritten - that "high-water mark" tells you the maximum stack depth ever reached. It's a great complement to static analysis: statics tell you the theoretical bound, painting tells you what actually happened during a test run.

```cpp
#include <cstdint>
#include <cstring>

// Stack painting: fill unused stack with a known pattern at startup,
// then scan to find how deep the stack actually grew.

namespace stack_monitor {

    // Called in early startup before main(), after .bss init
    // The linker script must export these symbols
    extern "C" {
        extern std::uint32_t _stack_start;  // bottom of stack (low address)
        extern std::uint32_t _stack_end;    // top of stack (initial SP)
    }

    inline constexpr std::uint32_t PAINT_PATTERN = 0xDEAD'C0DE;

    // Paint the stack region below current SP with known pattern
    // MUST be called very early, with minimal stack usage
    void paint_stack() noexcept {
        volatile std::uint32_t* sp;
        // Read current stack pointer
        #if defined(__ARM_ARCH)
        asm volatile("mov %0, sp" : "=r"(sp));
        #else
        // Fallback for testing on host
        std::uint32_t local;
        sp = &local;
        #endif

        // Paint from stack_start up to (current_sp - safety_margin)
        volatile std::uint32_t* bottom = &_stack_start;
        volatile std::uint32_t* top = sp - 16;  // 64-byte safety margin

        for (volatile std::uint32_t* p = bottom; p < top; ++p) {
            *p = PAINT_PATTERN;
        }
    }

    // Measure how much stack was actually used (high-water mark)
    struct StackUsageReport {
        std::size_t total_bytes;
        std::size_t used_bytes;
        std::size_t free_bytes;
        std::uint8_t percent_used;
    };

    StackUsageReport measure_usage() noexcept {
        const std::uint32_t* bottom = &_stack_start;
        const std::uint32_t* top    = &_stack_end;

        std::size_t total = reinterpret_cast<std::uintptr_t>(top) -
                            reinterpret_cast<std::uintptr_t>(bottom);

        // Scan from bottom up: first non-painted word = deepest stack reach
        const std::uint32_t* p = bottom;
        while (p < top && *p == PAINT_PATTERN) {
            ++p;
        }

        std::size_t free_words = static_cast<std::size_t>(p - bottom);
        std::size_t free_bytes = free_words * sizeof(std::uint32_t);
        std::size_t used = total - free_bytes;

        return StackUsageReport{
            .total_bytes = total,
            .used_bytes  = used,
            .free_bytes  = free_bytes,
            .percent_used = static_cast<std::uint8_t>((used * 100) / total),
        };
    }

    // Periodic check - call from idle task or watchdog
    bool is_stack_critical(std::uint8_t threshold_percent = 80) noexcept {
        return measure_usage().percent_used >= threshold_percent;
    }
}
```

The reason this trips people up is timing: `paint_stack()` must be called as early as possible - ideally from the startup code before any C++ constructors run - otherwise you'll miss the stack depth used during those constructors. Also note that painting only tells you about the deepest point reached in test scenarios. If your tests don't exercise the worst-case call path, the high-water mark will be optimistic.

### Q3: How do you use the Cortex-M MPU to create a hardware stack guard that triggers a HardFault on overflow

The MPU (Memory Protection Unit) lets you mark a small region at the bottom of the stack as "no access". The moment the stack pointer steps into that region, the hardware fires a MemManage fault (or falls back to a HardFault) immediately. This is the gold standard for overflow detection because it catches the problem at the exact instruction that caused it - no polling, no scan, no false negatives.

```cpp
#include <cstdint>

// ARM Cortex-M MPU registers
namespace mpu {
    struct Regs {
        volatile std::uint32_t TYPE;    // 0xE000ED90
        volatile std::uint32_t CTRL;    // 0xE000ED94
        volatile std::uint32_t RNR;     // 0xE000ED98
        volatile std::uint32_t RBAR;    // 0xE000ED9C
        volatile std::uint32_t RASR;    // 0xE000EDA0
    };

    inline Regs& regs() noexcept {
        return *reinterpret_cast<Regs*>(0xE000ED90);
    }

    // RASR field encodings
    constexpr std::uint32_t RASR_ENABLE     = 1u << 0;
    constexpr std::uint32_t RASR_SIZE_32B   = (4u << 1);   // 2^(4+1) = 32
    constexpr std::uint32_t RASR_SIZE_256B  = (7u << 1);   // 2^(7+1) = 256
    constexpr std::uint32_t RASR_AP_NONE    = (0u << 24);  // no access
    constexpr std::uint32_t RASR_XN         = (1u << 28);  // execute never

    constexpr std::uint32_t CTRL_ENABLE     = 1u << 0;
    constexpr std::uint32_t CTRL_HFNMIENA   = 1u << 1;
    constexpr std::uint32_t CTRL_PRIVDEFENA = 1u << 2;
}

// Stack guard configuration
// Place a 256-byte no-access MPU region at the bottom of the stack
// Any access triggers MemManage or HardFault -> immediate detection
void configure_stack_guard(std::uintptr_t stack_bottom) noexcept {
    auto& m = mpu::regs();

    // Verify MPU exists
    if ((m.TYPE & 0xFF00) == 0) return;  // no regions available

    // Region 0: stack guard - 256 bytes, no access, execute never
    m.RNR  = 0;  // select region 0
    m.RBAR = stack_bottom & ~0xFFu;  // aligned to region size
    m.RASR = mpu::RASR_ENABLE
           | mpu::RASR_SIZE_256B
           | mpu::RASR_AP_NONE
           | mpu::RASR_XN;

    // Enable MPU with default memory map for privileged access
    m.CTRL = mpu::CTRL_ENABLE | mpu::CTRL_PRIVDEFENA;

    // Memory barrier to ensure MPU is active before proceeding
    asm volatile("dsb" ::: "memory");
    asm volatile("isb" ::: "memory");
}

// Fault handler: log stack overflow location
extern "C" void HardFault_Handler() {
    // Read stacked PC to identify faulting instruction
    volatile std::uint32_t* sp;
    asm volatile("mrs %0, msp" : "=r"(sp));

    struct FaultFrame {
        std::uint32_t r0, r1, r2, r3, r12, lr, pc, xpsr;
    };
    volatile auto* frame = reinterpret_cast<volatile FaultFrame*>(sp);

    // In production: log frame->pc and frame->lr to persistent storage
    // then reset or enter safe state
    (void)frame->pc;
    (void)frame->lr;

    // Halt in debug, reset in production
    #ifdef DEBUG
    asm volatile("bkpt #0");
    #else
    NVIC_SystemReset();
    #endif
    while (true) {}
}
```

The fault handler captures the stacked PC register, which points directly at the instruction that caused the overflow. In a debug build you breakpoint there; in production you log it to flash and reset. The table below summarizes how the three main protection layers compare:

| Protection Layer   | Catches Overflow? | Latency   | False Negatives |
|--------------------|-------------------|-----------|-----------------|
| Stack painting     | Post-mortem only  | Zero      | Possible        |
| -fstack-protector  | At function return| ~5 cycles | If no return    |
| MPU guard region   | Immediately       | Zero      | None            |
| Linker script gaps | Link-time only    | Zero      | At runtime      |

---

## Notes

- `-fstack-usage` outputs per-function frame sizes, but does not account for register spills caused by optimization level changes - always analyze at the final `-O` level.
- Dynamic stack usage (VLA, `alloca`) is flagged as "dynamic" in `.su` files - ban these in safety-critical code.
- Stack painting uses zero cycles at runtime after initialization; the scan is only done during diagnostics or idle time.
- MPU guard regions must be size-aligned (32B minimum on Cortex-M); place the guard between heap and stack.
- For RTOS systems, each task stack needs its own painting and guard - FreeRTOS provides `uxTaskGetStackHighWaterMark()`.
- Linker map files (`.map`) show exact section sizes; script a CI check: if `.bss + .data + stack + heap > RAM_SIZE` then fail.
- Recursive functions are the enemy of static analysis - replace with iterative equivalents or prove bounded recursion depth.
