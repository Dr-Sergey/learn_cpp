# Write Bare-Metal Startup Code and Linker Scripts for C++

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++23  
**Reference:** <https://developer.arm.com/documentation/dui0474/m/>  

---

## Topic Overview

On a hosted system, the C/C++ runtime (crt0, crti, crtn) handles everything before `main()`: zeroing `.bss`, copying `.data` from flash to SRAM, calling global constructors, and setting up the stack. On bare-metal, **you** are responsible for all of this. A single mistake - forgetting to call static constructors, placing the stack at the wrong address, or misconfiguring the linker script sections - causes silent corruption that is extraordinarily difficult to debug.

For C++ specifically, the startup sequence must include the **static initialization** phase: calling all functions registered in the `.init_array` section. These are the constructors of global/namespace-scope objects and `__attribute__((constructor))` functions. Missing this step means every global object with a non-trivial constructor is in an undefined state - and the resulting bugs are notoriously hard to reproduce because the symptom often appears far from the root cause.

| Startup Phase | Responsibility | What Happens |
| --- | --- | --- |
| Reset vector | Hardware | PC loaded from vector table offset 0x04 |
| Stack pointer init | Vector table / startup | MSP loaded from vector table offset 0x00 |
| `.data` copy | Startup code | Copy initialized globals from flash (LMA) to SRAM (VMA) |
| `.bss` zero | Startup code | Zero-fill uninitialized globals in SRAM |
| FPU enable | Startup code | Set CPACR bits for Cortex-M4F/M7 |
| `.init_array` calls | Startup code | Call all C++ global constructors |
| `main()` / `app_main()` | Application | User code begins |

Here is how the flash and SRAM are laid out after linking:

```cpp
Flash Memory Layout (Cortex-M):

Address     Section         Content
0x0800_0000 .isr_vector     Vector table (stack ptr + handlers)
0x0800_01C0 .text           Code (.text)
0x080X_XXXX .rodata         Read-only data, const strings
0x080Y_YYYY .data (LMA)     Initial values for .data (source)
0x080Z_ZZZZ .init_array     C++ constructor function pointers

SRAM Layout:
0x2000_0000 .data (VMA)     Initialized globals (destination)
0x200A_AAAA .bss            Zero-initialized globals
0x200B_BBBB                 Heap (grows up)
    ...                     <- gap ->
0x2001_FFF0                 Stack (grows down)
0x2002_0000                 End of SRAM (128 KB)
```

The linker script is the blueprint that assigns these sections to physical addresses. It must define symbols that the startup code references (`_sidata`, `_sdata`, `_edata`, `_sbss`, `_ebss`, `__init_array_start`, `__init_array_end`) to perform the copy and zero operations. If those symbols are missing or wrong, the startup code writes to the wrong addresses and your firmware corrupts memory silently.

---

## Self-Assessment

### Q1: Write a complete linker script for an STM32F4 (1 MB flash, 128 KB SRAM) that properly handles C++ static initialization, stack, and heap

Walk through this script section by section. Notice how each region serves a specific purpose, and pay special attention to the `.init_array` section - that is where the C++ constructor pointers live, and `KEEP()` is essential there because the linker's dead-code elimination would otherwise discard those pointers as "unreferenced."

```ld
/* stm32f4.ld - Linker script for STM32F407 with C++ support */

/* Memory regions */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
    SRAM  (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
    CCM   (rw)  : ORIGIN = 0x10000000, LENGTH = 64K    /* Core-Coupled Memory */
}

/* Entry point - Reset_Handler in startup code */
ENTRY(Reset_Handler)

/* Stack and heap sizes */
_stack_size = 8K;
_heap_size  = 16K;

SECTIONS
{
    /* --- Interrupt Vector Table (must be at flash start) --- */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } > FLASH

    /* --- Code --- */
    .text :
    {
        . = ALIGN(4);
        *(.text)
        *(.text*)
        *(.glue_7)         /* ARM-Thumb interworking */
        *(.glue_7t)

        /* C++ virtual function tables */
        *(.rodata .rodata*)

        . = ALIGN(4);
        _etext = .;
    } > FLASH

    /* --- C++ static constructors (CRITICAL for C++) --- */
    .preinit_array :
    {
        . = ALIGN(4);
        PROVIDE_HIDDEN(__preinit_array_start = .);
        KEEP(*(.preinit_array*))
        PROVIDE_HIDDEN(__preinit_array_end = .);
    } > FLASH

    .init_array :
    {
        . = ALIGN(4);
        PROVIDE_HIDDEN(__init_array_start = .);
        KEEP(*(SORT(.init_array.*)))
        KEEP(*(.init_array*))
        PROVIDE_HIDDEN(__init_array_end = .);
    } > FLASH

    /* --- C++ static destructors (usually unused on embedded) --- */
    .fini_array :
    {
        . = ALIGN(4);
        PROVIDE_HIDDEN(__fini_array_start = .);
        KEEP(*(SORT(.fini_array.*)))
        KEEP(*(.fini_array*))
        PROVIDE_HIDDEN(__fini_array_end = .);
    } > FLASH

    /* --- ARM exception unwind tables (omit with -fno-exceptions) --- */
    .ARM.extab :
    {
        *(.ARM.extab* .gnu.linkonce.armextab.*)
    } > FLASH

    .ARM.exidx :
    {
        PROVIDE_HIDDEN(__exidx_start = .);
        *(.ARM.exidx* .gnu.linkonce.armexidx.*)
        PROVIDE_HIDDEN(__exidx_end = .);
    } > FLASH

    /* --- Initialized data (flash -> SRAM copy) --- */
    _sidata = LOADADDR(.data);   /* Source address in flash */

    .data :
    {
        . = ALIGN(4);
        _sdata = .;              /* Destination start in SRAM */
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;              /* Destination end in SRAM */
    } > SRAM AT> FLASH           /* VMA in SRAM, LMA in FLASH */

    /* --- Zero-initialized data --- */
    .bss :
    {
        . = ALIGN(4);
        _sbss = .;
        __bss_start__ = _sbss;
        *(.bss)
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
        __bss_end__ = _ebss;
    } > SRAM

    /* --- Heap (grows up from end of BSS) --- */
    .heap :
    {
        . = ALIGN(8);
        _heap_start = .;
        . = . + _heap_size;
        . = ALIGN(8);
        _heap_end = .;
    } > SRAM

    /* --- Stack (grows down from end of SRAM) --- */
    .stack :
    {
        . = ALIGN(8);
        . = . + _stack_size;
        . = ALIGN(8);
        _estack = .;             /* Initial stack pointer value */
    } > SRAM

    /* Verify RAM fits */
    ASSERT((_heap_end < _estack - _stack_size),
           "ERROR: Heap and stack overlap! Reduce heap or stack size.")

    /* Discard unneeded sections */
    /DISCARD/ :
    {
        libc.a(*)
        libm.a(*)
        libgcc.a(*)
    }
}
```

### Q2: Write the C++ startup code (Reset_Handler) that correctly initializes .data, .bss, the FPU, and calls all C++ global constructors before entering main

The startup code references those linker-script symbols by declaring them as `extern` variables. The `Reset_Handler` must be in `extern "C"` because the vector table stores its address as a raw function pointer - no C++ name mangling allowed.

```cpp
// startup.cpp - Bare-metal C++ startup for Cortex-M4F
#include <cstdint>
#include <cstddef>

// ---------- Linker-defined symbols ----------
extern "C" {
    extern std::uint32_t _sidata;       // .data LMA (flash source)
    extern std::uint32_t _sdata;        // .data VMA start (SRAM)
    extern std::uint32_t _edata;        // .data VMA end
    extern std::uint32_t _sbss;         // .bss start
    extern std::uint32_t _ebss;         // .bss end
    extern std::uint32_t _estack;       // Initial stack pointer

    // C++ constructor tables
    using InitFunc = void(*)();
    extern InitFunc __preinit_array_start[];
    extern InitFunc __preinit_array_end[];
    extern InitFunc __init_array_start[];
    extern InitFunc __init_array_end[];
}

// Forward declarations
extern "C" void Reset_Handler();
extern "C" void Default_Handler();
extern "C" [[noreturn]] void app_main();

// ---------- FPU Enable (Cortex-M4F) ----------
static void enable_fpu() {
    // CPACR at 0xE000'ED88 - set CP10 and CP11 to full access
    volatile auto* cpacr =
        reinterpret_cast<volatile std::uint32_t*>(0xE000'ED88);
    *cpacr |= (0xFu << 20);  // CP10 + CP11 = full access
    // Data Synchronization Barrier + Instruction Sync Barrier
    asm volatile("dsb");
    asm volatile("isb");
}

// ---------- Data/BSS Initialization ----------
static void init_data() {
    // Copy .data from flash (LMA) to SRAM (VMA)
    auto* src = &_sidata;
    for (auto* dst = &_sdata; dst < &_edata; ) {
        *dst++ = *src++;
    }
}

static void init_bss() {
    // Zero .bss
    for (auto* dst = &_sbss; dst < &_ebss; ) {
        *dst++ = 0;
    }
}

// ---------- C++ Static Constructors ----------
static void call_static_constructors() {
    // Pre-init array (compiler-internal, e.g. sanitizers)
    for (auto* fn = __preinit_array_start; fn < __preinit_array_end; ++fn) {
        (*fn)();
    }
    // Init array (C++ global object constructors)
    for (auto* fn = __init_array_start; fn < __init_array_end; ++fn) {
        (*fn)();
    }
}

// ---------- Reset Handler (entry point) ----------
extern "C" [[noreturn]]
void Reset_Handler() {
    // 1. Stack pointer is already set from vector table[0]
    // 2. Enable FPU before any float usage
    enable_fpu();
    // 3. Copy .data section
    init_data();
    // 4. Zero .bss section
    init_bss();
    // 5. Call C++ global constructors
    call_static_constructors();
    // 6. Enter application
    app_main();
    // Should never return
    __builtin_unreachable();
}

// ---------- Vector Table ----------
extern "C" void Default_Handler() {
    while (true) { asm volatile("bkpt #0"); }
}

// Weak aliases - user overrides specific handlers
#define WEAK_ALIAS __attribute__((weak, alias("Default_Handler")))

extern "C" {
    void NMI_Handler()        WEAK_ALIAS;
    void HardFault_Handler()  WEAK_ALIAS;
    void MemManage_Handler()  WEAK_ALIAS;
    void BusFault_Handler()   WEAK_ALIAS;
    void UsageFault_Handler() WEAK_ALIAS;
    void SVC_Handler()        WEAK_ALIAS;
    void PendSV_Handler()     WEAK_ALIAS;
    void SysTick_Handler()    WEAK_ALIAS;
}

// Vector table placed in .isr_vector by linker script
__attribute__((section(".isr_vector"), used))
const std::uintptr_t vector_table[] = {
    reinterpret_cast<std::uintptr_t>(&_estack),     // 0: Initial SP
    reinterpret_cast<std::uintptr_t>(Reset_Handler), // 1: Reset
    reinterpret_cast<std::uintptr_t>(NMI_Handler),   // 2: NMI
    reinterpret_cast<std::uintptr_t>(HardFault_Handler),  // 3
    reinterpret_cast<std::uintptr_t>(MemManage_Handler),  // 4
    reinterpret_cast<std::uintptr_t>(BusFault_Handler),   // 5
    reinterpret_cast<std::uintptr_t>(UsageFault_Handler), // 6
    0, 0, 0, 0,                                           // 7-10: Reserved
    reinterpret_cast<std::uintptr_t>(SVC_Handler),        // 11
    0, 0,                                                  // 12-13: Reserved
    reinterpret_cast<std::uintptr_t>(PendSV_Handler),     // 14
    reinterpret_cast<std::uintptr_t>(SysTick_Handler),    // 15
    // IRQ 0-239 would follow for specific STM32 peripherals...
};
```

The `enable_fpu()` call must come before `init_data()` if any global constructor uses floating point. The `dsb`/`isb` barrier instructions ensure the CPACR write has propagated before the pipeline executes any FP instruction.

### Q3: Demonstrate the static initialization order fiasco and how to prevent it in bare-metal C++ using constinit, Nifty Counter, and placement new

The static initialization order fiasco is when object A's constructor runs before object B's, but A's constructor calls B - and B is not initialized yet. On hosted systems this is a known pitfall; on bare-metal it is especially insidious because the crash happens before `main()` starts, often with no stack trace.

```cpp
// init_order.cpp - Static initialization pitfalls and solutions
#include <cstdint>
#include <new>          // placement new (freestanding)
#include <type_traits>
#include <atomic>

// ======================================================
// PROBLEM: Static Initialization Order Fiasco
// ======================================================
// File A:
struct Logger {
    std::uint32_t log_count = 0;
    bool initialized = false;
    void init() { initialized = true; }
    void log(const char*) { ++log_count; }
};

// File B:
struct Sensor {
    Logger& logger;
    Sensor(Logger& l) : logger(l) {
        logger.log("Sensor init");  // BUG if Logger not yet constructed!
    }
};

// If Sensor's TU is initialized before Logger's TU -> undefined behavior

// ======================================================
// SOLUTION 1: constinit (C++20) - guarantee constant initialization
// ======================================================
struct SafeCounter {
    std::uint32_t value;
    constexpr SafeCounter() : value(0) {}
    constexpr void increment() { ++value; }
};

// constinit guarantees this is initialized at compile time,
// before any dynamic initialization runs.
// No ordering issues possible.
constinit SafeCounter global_counter{};

// ======================================================
// SOLUTION 2: Nifty Counter (Schwarz Counter) idiom
// ======================================================
// This is how <iostream> solves the problem for std::cout

// logger_nifty.h
struct NiftyLogger {
    char buffer[256];
    std::uint32_t write_pos;
    void init();
    void log(const char* msg);
};

// Storage declared in header - all TUs share one definition
extern NiftyLogger& g_logger;

struct NiftyLoggerInit {
    NiftyLoggerInit();
    ~NiftyLoggerInit();
};

// One instance per TU that includes this header.
// First constructor initializes; last destructor destroys.
static NiftyLoggerInit nifty_logger_init_obj;

// logger_nifty.cpp (implementation)
namespace {
    // Raw storage - no constructor called at file scope
    alignas(NiftyLogger) std::uint8_t logger_storage[sizeof(NiftyLogger)];
    std::uint32_t nifty_counter = 0;
}

NiftyLogger& g_logger = *reinterpret_cast<NiftyLogger*>(logger_storage);

NiftyLoggerInit::NiftyLoggerInit() {
    if (nifty_counter++ == 0) {
        // First TU to initialize - construct the logger
        ::new (logger_storage) NiftyLogger{};
        g_logger.init();
    }
}

NiftyLoggerInit::~NiftyLoggerInit() {
    if (--nifty_counter == 0) {
        g_logger.~NiftyLogger();
    }
}

// ======================================================
// SOLUTION 3: Placement new with explicit init order
// ======================================================
// Best for embedded: full control, no hidden constructors

template <typename T>
class ManualInit {
    alignas(T) std::uint8_t storage_[sizeof(T)];
    bool initialized_ = false;

public:
    // No constructor, no destructor - trivial lifetime
    template <typename... Args>
    void construct(Args&&... args) {
        ::new (storage_) T(static_cast<Args&&>(args)...);
        initialized_ = true;
    }

    void destroy() {
        if (initialized_) {
            get().~T();
            initialized_ = false;
        }
    }

    [[nodiscard]] T&       get()       { return *reinterpret_cast<T*>(storage_); }
    [[nodiscard]] const T& get() const { return *reinterpret_cast<const T*>(storage_); }
    [[nodiscard]] bool is_initialized() const { return initialized_; }
};

// Global objects - no constructors called at static init time
ManualInit<Logger> g_lazy_logger;
ManualInit<Sensor> g_lazy_sensor;

// Explicit initialization in guaranteed order
void system_init() {
    // 1. Logger first
    g_lazy_logger.construct();
    g_lazy_logger.get().init();

    // 2. Sensor second - logger is guaranteed ready
    g_lazy_sensor.construct(g_lazy_logger.get());
}
```

Solution 3 is the recommended approach for embedded: call `system_init()` explicitly from your `Reset_Handler` or early in `main()`, in whatever order the dependencies require. There is no hidden ordering, no surprise, and no risk of the wrong thing running first.

---

## Notes

- The `KEEP()` directive in the linker script is **essential** for `.init_array` - without it, `--gc-sections` will discard constructor pointers as "unreferenced" and global objects will never be constructed.
- `SORT(.init_array.*)` ensures constructors with explicit `__attribute__((init_priority(N)))` are called in the correct order.
- On Cortex-M, the initial stack pointer is loaded from vector table offset 0 by hardware. The `Reset_Handler` does **not** need to set SP - but it must be correct in the vector table.
- Never use `__libc_init_array()` from newlib in production - it pulls in `malloc` and other dependencies. Write your own `.init_array` loop instead.
- `constinit` (C++20) **guarantees** constant initialization. If the initializer is not a constant expression, the program is ill-formed - this is a compile-time safety net.
- On Cortex-M4F, the FPU **must** be enabled before any function that uses `float`/`double`. If the startup code itself uses floating-point (even implicitly via a global constructor), enable FPU first.
- The `.data` LMA/VMA split (`> SRAM AT> FLASH`) is the most common linker script mistake: omit `AT> FLASH` and your initialized globals will be zero (or random) after reset.
