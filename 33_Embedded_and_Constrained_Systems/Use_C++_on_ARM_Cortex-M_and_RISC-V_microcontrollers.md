# Use C++ on ARM Cortex-M and RISC-V Microcontrollers

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** <https://developer.arm.com/documentation/dui0553/latest/>  

---

## Topic Overview

ARM Cortex-M (M0/M0+/M3/M4/M7/M33) and RISC-V (RV32I/RV32IMAC) are the dominant architectures for microcontroller-class embedded systems. Using C++ effectively on these targets requires understanding the ROM/RAM constraints, the toolchain flags that control code generation, and which C++ features have hidden costs (RTTI, exceptions, `atexit` destructors).

A typical Cortex-M0 target has 16-64 KB flash and 4-8 KB RAM. A Cortex-M7 may have 2 MB flash and 512 KB RAM. The C++ runtime footprint with `-fno-exceptions -fno-rtti` and a minimal `libc` (newlib-nano, picolibc) can be as small as 1-2 KB. The critical optimization is choosing the right ABI, instruction set, and library variant. Getting these wrong is the most common reason a C++ firmware ends up much larger than expected.

| Feature | Cortex-M0/M0+ | Cortex-M4/M7 | RV32IMAC |
| --- | --- | --- | --- |
| ISA | Thumb (16-bit) | Thumb-2 (16/32-bit) | RV32I + extensions |
| FPU | None | Optional (SP/DP) | F extension optional |
| Multiply | 1 or 32 cycles | 1 cycle (DSP) | M extension: 1 cycle |
| Divide | Software | 2-12 cycles (SDIV) | M extension: ~34 cyc |
| Min flash for C++ | ~4 KB | ~4 KB | ~6 KB |
| Interrupt latency | 16 cycles | 12 cycles | Implementation-defined |
| Vector table | Fixed at 0x0 | Relocatable (VTOR) | mtvec CSR |

Here is how the Cortex-M memory map is laid out - understanding this is essential before writing your first linker script:

```cpp
  Cortex-M Memory Map:
  ┌────────────────────┐ 0xFFFF_FFFF
  │ System / Debug     │
  ├────────────────────┤ 0xE000_0000
  │ Private Peripherals│ (NVIC, SysTick, SCB)
  ├────────────────────┤ 0xA000_0000
  │ External Device    │
  ├────────────────────┤ 0x6000_0000
  │ External RAM       │
  ├────────────────────┤ 0x4000_0000
  │ Peripherals        │ (GPIO, UART, SPI, DMA)
  ├────────────────────┤ 0x2000_0000
  │ SRAM               │ (.data, .bss, stack, heap)
  ├────────────────────┤ 0x0000_0000
  │ Flash / Code       │ (.text, .rodata, vector table)
  └────────────────────┘

  RISC-V RV32 typical (vendor-specific):
  Flash: 0x0800_0000   RAM: 0x2000_0000
```

Toolchain selection matters enormously. `arm-none-eabi-g++` with newlib-nano, or `clang --target=thumbv7em-none-eabi` with picolibc, yield the smallest binaries. For RISC-V: `riscv32-unknown-elf-g++` or `clang --target=riscv32`.

---

## Self-Assessment

### Q1: What are the essential compiler flags for minimal C++ on Cortex-M4F and RV32IMAC, and what does each flag eliminate

Every flag listed here removes a specific runtime feature. The comments explain what you are trading away and why the trade is usually worth it on a constrained target.

```cpp
// === ARM Cortex-M4F (STM32F4) toolchain flags ===
//
// arm-none-eabi-g++ \
//   -mcpu=cortex-m4           # Target CPU: enables Thumb-2, DSP
//   -mthumb                   # Use Thumb instruction set
//   -mfloat-abi=hard          # Hardware FPU calling convention
//   -mfpu=fpv4-sp-d16         # Single-precision FPU, 16 D-regs
//   -fno-exceptions           # Removes ~10-20 KB unwind tables
//   -fno-rtti                 # Removes ~2-5 KB typeinfo tables
//   -fno-threadsafe-statics   # Removes __cxa_guard (no OS futex)
//   -fno-use-cxa-atexit       # Removes global destructor registry
//   -ffunction-sections       # Each function in own ELF section
//   -fdata-sections           # Each global in own ELF section
//   -Os                       # Optimize for size
//   -specs=nano.specs         # Use newlib-nano (printf ~1 KB vs ~50 KB)
//   -specs=nosys.specs        # Stub syscalls (no semihosting)
//   -Wl,--gc-sections         # Linker: discard unused sections
//   -T stm32f407.ld           # Linker script with memory regions

// === RISC-V RV32IMAC toolchain flags ===
//
// riscv32-unknown-elf-g++ \
//   -march=rv32imac           # Integer + Multiply + Atomic + Compressed
//   -mabi=ilp32               # 32-bit int/long/pointer, soft float
//   -fno-exceptions           # Same rationale as ARM
//   -fno-rtti                 #
//   -fno-threadsafe-statics   #
//   -ffunction-sections       #
//   -fdata-sections           #
//   -Os                       #
//   -Wl,--gc-sections         #
//   -T link.ld                #

// Code size impact measurement:
// | Feature              | ARM M4 (bytes) | RV32IMAC (bytes) |
// |----------------------|----------------|------------------|
// | Bare minimum (main)  |            348 |              412 |
// | + exceptions         |        +15,280 |          +18,640 |
// | + RTTI               |         +3,720 |           +4,100 |
// | + full printf        |        +48,000 |          +52,000 |
// | + nano printf        |         +1,200 |           +1,400 |
// | + iostream           |       +120,000 |         +140,000 |

// NEVER use iostream on constrained targets. Use custom formatters:
#include <cstdint>

class UartWriter {
    volatile std::uint32_t* const tdr_;  // Transmit data register
    volatile std::uint32_t* const sr_;   // Status register

public:
    explicit UartWriter(std::uintptr_t base) noexcept
        : tdr_(reinterpret_cast<volatile std::uint32_t*>(base + 0x04))
        , sr_(reinterpret_cast<volatile std::uint32_t*>(base + 0x00)) {}

    void put_char(char c) noexcept {
        while (!(*sr_ & (1u << 7))) {}  // wait for TXE
        *tdr_ = static_cast<std::uint32_t>(c);
    }

    void put_string(const char* s) noexcept {
        while (*s) put_char(*s++);
    }

    // Integer-to-string without printf: ~80 bytes flash
    void put_int(std::int32_t value) noexcept {
        if (value < 0) { put_char('-'); value = -value; }
        char buf[11];
        int i = 0;
        do {
            buf[i++] = '0' + static_cast<char>(value % 10);
            value /= 10;
        } while (value > 0);
        while (i > 0) put_char(buf[--i]);
    }
};
```

The size table makes the case vividly: `iostream` alone costs 120 KB on ARM M4. That is more than the entire flash of most Cortex-M0 devices. A custom `UartWriter` like the one above costs about 80 bytes.

### Q2: How do you write a portable C++ startup sequence (reset handler, .data/.bss init) for both Cortex-M and RISC-V

Both architectures need the same sequence: copy `.data`, zero `.bss`, call static constructors, then enter main. The architecture-specific part is just the halt instruction at the end.

```cpp
#include <cstdint>
#include <cstddef>

// Linker script exports these symbols
extern "C" {
    extern std::uint32_t _sidata;  // .data initializers in flash (LMA)
    extern std::uint32_t _sdata;   // .data start in RAM (VMA)
    extern std::uint32_t _edata;   // .data end in RAM
    extern std::uint32_t _sbss;    // .bss start
    extern std::uint32_t _ebss;    // .bss end
    extern std::uint32_t _estack;  // initial stack pointer

    // C++ static constructors
    using InitFunc = void(*)();
    extern InitFunc __init_array_start[];
    extern InitFunc __init_array_end[];
}

// Forward declarations
extern "C" int main();

// Reset handler: runs before main, initializes C++ runtime
extern "C" [[noreturn]] void Reset_Handler() {
    // 1. Copy .data from flash to RAM
    {
        std::uint32_t* src = &_sidata;
        std::uint32_t* dst = &_sdata;
        while (dst < &_edata) {
            *dst++ = *src++;
        }
    }

    // 2. Zero .bss
    {
        std::uint32_t* dst = &_sbss;
        while (dst < &_ebss) {
            *dst++ = 0;
        }
    }

    // 3. Call C++ static constructors (__init_array)
    {
        for (InitFunc* fn = __init_array_start; fn < __init_array_end; ++fn) {
            (*fn)();
        }
    }

    // 4. Call main
    main();

    // 5. If main returns, halt
    while (true) {
        #if defined(__ARM_ARCH)
        asm volatile("wfi");  // Wait For Interrupt - low power halt
        #elif defined(__riscv)
        asm volatile("wfi");  // RISC-V also supports WFI
        #endif
    }
}

// Cortex-M vector table (first 16 entries are core exceptions)
#if defined(__ARM_ARCH)
extern "C" {
    void NMI_Handler()       __attribute__((weak, alias("Default_Handler")));
    void HardFault_Handler() __attribute__((weak, alias("Default_Handler")));
    void SysTick_Handler()   __attribute__((weak, alias("Default_Handler")));

    [[noreturn]] void Default_Handler() { while (true) { asm volatile("bkpt #0"); } }
}

__attribute__((section(".isr_vector"), used))
const std::uintptr_t vector_table[] = {
    reinterpret_cast<std::uintptr_t>(&_estack),       // Initial SP
    reinterpret_cast<std::uintptr_t>(&Reset_Handler),  // Reset
    reinterpret_cast<std::uintptr_t>(&NMI_Handler),    // NMI
    reinterpret_cast<std::uintptr_t>(&HardFault_Handler),
    // ... remaining vectors per device datasheet
};
#endif
```

### Q3: How do you handle the "static initialization order fiasco" safely on bare-metal targets with no heap

The static initialization order fiasco is when one global object's constructor depends on another global object that has not been constructed yet. On hosted systems this is an occasional headache; on bare-metal it can cause crashes that are nearly impossible to reproduce. Here are three strategies, each with different tradeoffs:

```cpp
#include <cstdint>
#include <new>

// Problem: C++ globals with constructors initialize in translation-unit order,
// which is unspecified across TUs. On bare metal, this can mean a peripheral
// driver initializes before the clock system it depends on.

// Solution 1: Construct-on-first-use with placement new in static storage
// Zero overhead after first call; no heap; deterministic order.

class ClockSystem {
    std::uint32_t core_freq_hz_;
    bool initialized_ = false;
public:
    void init(std::uint32_t freq) noexcept {
        core_freq_hz_ = freq;
        initialized_ = true;
    }
    std::uint32_t freq() const noexcept { return core_freq_hz_; }
};

class UartDriver {
    std::uint32_t baud_;
public:
    UartDriver(ClockSystem& clk, std::uint32_t baud) noexcept
        : baud_(baud) {
        // Compute divider from clock - clock MUST be initialized first
        std::uint32_t divider = clk.freq() / baud;
        (void)divider; // write to BRR register
    }
    std::uint32_t baud() const noexcept { return baud_; }
};

// Construct-on-first-use: aligned storage, no heap, explicit init order
template <typename T>
class LazyStatic {
    alignas(T) std::byte storage_[sizeof(T)];
    bool constructed_ = false;

public:
    template <typename... Args>
    T& construct(Args&&... args) noexcept {
        if (!constructed_) {
            ::new (storage_) T(static_cast<Args&&>(args)...);
            constructed_ = true;
        }
        return *reinterpret_cast<T*>(storage_);
    }

    T& get() noexcept { return *reinterpret_cast<T*>(storage_); }
    bool is_constructed() const noexcept { return constructed_; }
};

// Global instances - construction deferred, order controlled in init()
LazyStatic<ClockSystem> g_clock;
LazyStatic<UartDriver>  g_uart;

// Explicit initialization sequence - called from Reset_Handler or main
void system_init() noexcept {
    // 1. Clock first - no dependencies
    auto& clk = g_clock.construct();
    clk.init(168'000'000);  // 168 MHz

    // 2. UART depends on clock - safe because clock is initialized
    g_uart.construct(g_clock.get(), 115200);
}

// Solution 2: constinit (C++20) for trivially constructible globals
// Guarantees constant initialization - no runtime constructor needed
struct GpioPin {
    std::uintptr_t port_base;
    std::uint8_t   pin;
    constexpr GpioPin(std::uintptr_t base, std::uint8_t p) noexcept
        : port_base(base), pin(p) {}
};

// constinit: resolved at compile time, in .data not __init_array
constinit GpioPin led_pin{0x40020000, 5};
constinit GpioPin button_pin{0x40020400, 13};
```

The `constinit` solution is the cleanest when your type supports it - the object is placed in `.data` with its constant values already there, so no constructor runs at startup at all and the ordering problem simply does not exist.

---

## Notes

- `-fno-exceptions` alone saves 10-20 KB on Cortex-M; `-fno-rtti` saves another 2-5 KB - always use both on constrained targets.
- `-fno-threadsafe-statics` removes hidden mutex calls (`__cxa_guard_acquire`) that require an OS; it is safe on single-threaded bare metal.
- On Cortex-M0, avoid 64-bit divides - they expand to a ~500-byte software routine; use shifts and reciprocal multiplication instead.
- RISC-V compressed instructions (`C` extension) reduce code size by ~25% - always include `c` in `-march` if the core supports it.
- newlib-nano's `printf` supports `%d`, `%s`, `%x` but not `%f` by default; add `-u _printf_float` only if you need it (+8 KB).
- `constinit` (C++20) eliminates the static initialization order problem for constexpr-constructible types with zero overhead.
- Flash wait states increase with clock frequency; on STM32F4 at 168 MHz, 5 wait states are needed - enable prefetch and ART accelerator.
