# Implement power management and low-power modes from C++

**Category:** Embedded and Constrained Systems  
**Standard:** C++17  
**Reference:** ARM Cortex-M documentation · <https://developer.arm.com/documentation>  

---

## Topic Overview

### Why Power Management Matters

Battery-powered and energy-harvesting devices must minimize power consumption. Modern MCUs provide multiple low-power modes that trade off wake-up latency for power savings. The further down the table you go, the less the MCU draws - but the longer it takes to wake up and the less state it retains.

| Mode | Cortex-M | Power | Wake-up | What's preserved |
| --- | --- | --- | --- | --- |
| **Run** | Normal | ~10-50 mA | - | Everything |
| **Sleep** | WFI/WFE | ~1-10 mA | µs | CPU state, RAM, peripherals |
| **Deep Sleep** | STOP | ~10-100 µA | ms | RAM, RTC, wake-up pins |
| **Standby** | STANDBY | ~1-5 µA | ms-s | RTC, backup registers only |
| **Shutdown** | SHUTDOWN | ~100 nA | s | Nothing (cold boot) |

### Register-Level Power Control in C++

The `PowerManager` class maps each logical power mode to the correct register writes and ARM intrinsics. The key register is the System Control Register (SCR) in the ARM System Control Block - specifically the `SLEEPDEEP` bit, which tells the processor whether `WFI` should enter normal sleep or deep sleep. Notice that `Standby` and `Shutdown` are marked `[[noreturn]]` in practice because the device resets on wake - execution never returns to the call site.

```cpp
#include <cstdint>

// ARM Cortex-M System Control Block
namespace scb {
    inline constexpr uintptr_t SCR_ADDR = 0xE000'ED10;

    enum class SCR : uint32_t {
        SLEEPDEEP  = (1U << 2),  // Deep sleep vs normal sleep
        SLEEPONEXIT = (1U << 1), // Sleep on ISR return
        SEVONPEND  = (1U << 4),  // Wake on pending interrupt
    };

    inline volatile uint32_t& SCR_REG() {
        return *reinterpret_cast<volatile uint32_t*>(SCR_ADDR);
    }
}

// Type-safe power mode abstraction
enum class PowerMode { Run, Sleep, DeepSleep, Standby, Shutdown };

class PowerManager {
public:
    static void enter(PowerMode mode) {
        switch (mode) {
        case PowerMode::Sleep:
            scb::SCR_REG() &= ~static_cast<uint32_t>(scb::SCR::SLEEPDEEP);
            __DSB();
            __WFI();  // Wait For Interrupt - CPU sleeps, wakes on any interrupt
            break;

        case PowerMode::DeepSleep:
            scb::SCR_REG() |= static_cast<uint32_t>(scb::SCR::SLEEPDEEP);
            configure_stop_mode();
            __DSB();
            __WFI();
            // After wake: restore clocks (PLL, etc.)
            restore_clocks();
            break;

        case PowerMode::Standby:
            scb::SCR_REG() |= static_cast<uint32_t>(scb::SCR::SLEEPDEEP);
            configure_standby_mode();
            __DSB();
            __WFI();
            // Never returns - device resets on wake
            __builtin_unreachable();

        case PowerMode::Shutdown:
            configure_shutdown_mode();
            __DSB();
            __WFI();
            __builtin_unreachable();

        case PowerMode::Run:
            break;  // Already running
        }
    }

    // Disable unused peripherals to save power
    static void disable_peripheral_clocks() {
        // STM32 example: disable GPIO port clocks not in use
        RCC->AHB1ENR &= ~(RCC_AHB1ENR_GPIOCEN | RCC_AHB1ENR_GPIODEN);
        // Disable unused bus clocks
        RCC->APB1ENR &= ~(RCC_APB1ENR_TIM3EN | RCC_APB1ENR_SPI2EN);
    }

    // Enable sleep-on-exit: CPU sleeps after every ISR
    static void enable_sleep_on_exit() {
        scb::SCR_REG() |= static_cast<uint32_t>(scb::SCR::SLEEPONEXIT);
    }

private:
    static void configure_stop_mode();
    static void configure_standby_mode();
    static void configure_shutdown_mode();
    static void restore_clocks();
};
```

The `__DSB()` before `__WFI()` is important - it's a Data Synchronization Barrier that ensures all pending memory writes complete before the CPU enters sleep. Without it, a peripheral write that hasn't propagated yet could behave unpredictably when the CPU wakes up.

### RAII Wake Lock

The wake-lock pattern solves a real problem: how do you prevent the idle task from entering deep sleep while a UART transmission is still in progress? The answer is a reference-counted lock. Any code that needs peripherals to stay active acquires a `WakeLock` on entry. The idle task checks the count before deciding how deep to sleep. RAII ensures the lock is always released when the scope exits, even on early returns.

```cpp
#include <atomic>
#include <cstdint>

class WakeLockManager {
    static inline std::atomic<uint32_t> lock_count_{0};

public:
    static void acquire() { lock_count_.fetch_add(1, std::memory_order_relaxed); }
    static void release() { lock_count_.fetch_sub(1, std::memory_order_relaxed); }

    [[nodiscard]] static bool can_deep_sleep() {
        return lock_count_.load(std::memory_order_relaxed) == 0;
    }
};

class WakeLock {
public:
    WakeLock()  { WakeLockManager::acquire(); }
    ~WakeLock() { WakeLockManager::release(); }
    WakeLock(const WakeLock&) = delete;
    WakeLock& operator=(const WakeLock&) = delete;
};

// Usage
void transmit_data(const uint8_t* buf, size_t len) {
    WakeLock hold;  // Prevent deep sleep during transmission
    uart_send(buf, len);
    // ~WakeLock releases automatically
}

// Idle task checks before sleeping
void idle_task() {
    while (true) {
        if (WakeLockManager::can_deep_sleep()) {
            PowerManager::enter(PowerMode::DeepSleep);
        } else {
            PowerManager::enter(PowerMode::Sleep);
        }
    }
}
```

If `transmit_data` is called from multiple concurrent tasks, the count correctly reflects that multiple operations are in flight - deep sleep only becomes legal when all of them have released their locks.

### Peripheral Power Gating with RAII

Every peripheral has a clock gate in the RCC (Reset and Clock Control) registers. Leaving a peripheral's clock enabled when you're not using it wastes power continuously. The `PeripheralPowerGuard` template turns this into a scoped operation: the constructor enables the clock and initializes the peripheral, the destructor deinitializes and disables the clock. You can't forget the shutdown step because the compiler enforces it.

```cpp
template<typename Peripheral>
class PeripheralPowerGuard {
public:
    PeripheralPowerGuard() {
        Peripheral::enable_clock();
        Peripheral::init();
    }

    ~PeripheralPowerGuard() {
        Peripheral::deinit();
        Peripheral::disable_clock();
    }

    PeripheralPowerGuard(const PeripheralPowerGuard&) = delete;
    PeripheralPowerGuard& operator=(const PeripheralPowerGuard&) = delete;
};

// Define peripheral traits
struct ADCPeripheral {
    static void enable_clock()  { RCC->APB2ENR |= RCC_APB2ENR_ADC1EN; }
    static void disable_clock() { RCC->APB2ENR &= ~RCC_APB2ENR_ADC1EN; }
    static void init()          { /* ADC calibration, channel setup */ }
    static void deinit()        { /* Power down ADC */ }
};

uint16_t read_battery_voltage() {
    PeripheralPowerGuard<ADCPeripheral> adc;  // Powers on ADC
    uint16_t value = adc_read_channel(BATTERY_CHANNEL);
    return value;
    // ~PeripheralPowerGuard powers off ADC automatically
}
```

### RTC Wake-Up Timer

For periodic low-power sensing, you don't want the CPU to stay awake between measurements - you want it to enter deep sleep and have the RTC (Real-Time Clock) wake it up on a schedule. The RTC keeps running in all low-power modes except Shutdown, drawing only a few microamps. The pattern is: do work, schedule the next wake-up, enter deep sleep, repeat.

```cpp
#include <chrono>

class RTCWakeup {
public:
    static void schedule(std::chrono::seconds interval) {
        uint32_t ticks = static_cast<uint32_t>(interval.count());

        // Configure RTC wake-up timer (STM32 example)
        RTC->WPR = 0xCA;  // Unlock write protection
        RTC->WPR = 0x53;
        RTC->CR &= ~RTC_CR_WUTE;             // Disable timer
        while (!(RTC->ISR & RTC_ISR_WUTWF));  // Wait for write flag
        RTC->WUTR = ticks - 1;                // Set period
        RTC->CR |= RTC_CR_WUTE | RTC_CR_WUTIE; // Enable timer + interrupt

        // Enable RTC interrupt in NVIC
        NVIC_EnableIRQ(RTC_WKUP_IRQn);
    }

    static void cancel() {
        RTC->CR &= ~(RTC_CR_WUTE | RTC_CR_WUTIE);
        NVIC_DisableIRQ(RTC_WKUP_IRQn);
    }
};

// Periodic low-power sensing pattern
void low_power_sensor_loop() {
    while (true) {
        // Wake: read sensor, transmit if threshold exceeded
        float temp = read_temperature();
        if (temp > THRESHOLD) {
            WakeLock hold;
            transmit_alert(temp);
        }

        // Schedule next wake-up and enter deep sleep
        RTCWakeup::schedule(std::chrono::seconds{60});
        PowerManager::enter(PowerMode::DeepSleep);
        // Execution resumes here after RTC wake-up interrupt
    }
}
```

The comment "Execution resumes here" is the key thing to understand. After a Sleep or DeepSleep entry via `WFI`, the CPU resumes execution at the instruction immediately following `WFI` once an interrupt fires. For Standby and Shutdown the device resets instead, so those modes are only used when you don't need to preserve any runtime state.

---

## Self-Assessment

### Q1: What is the difference between WFI and WFE on ARM Cortex-M

- **WFI (Wait For Interrupt)**: CPU sleeps until an interrupt fires. The interrupt must be enabled in the NVIC.
- **WFE (Wait For Event)**: CPU sleeps until an **event** occurs. Events can come from interrupts, `SEV` instruction, or external event signals. WFE is useful for spinlock-like patterns where you want to sleep between polls without requiring a full interrupt handler.

In practice, `WFI` is used for idle-loop sleeping, while `WFE` is used in multi-core synchronization or when `SEVONPEND` is set to wake on pending (but disabled) interrupts.

### Q2: Why use the wake-lock pattern

Without wake locks, entering deep sleep during an active UART transmission would corrupt the output because the peripheral clock gets gated off mid-transfer. The wake-lock pattern ensures:

- Any code that needs peripherals active holds a `WakeLock`.
- The idle task only enters deep sleep when the lock count is zero.
- RAII ensures the lock is always released, even if an operation fails or returns early.

### Q3: Why use RAII for peripheral power gating

Without RAII, you might forget to disable the ADC clock after reading, wasting ~1 mA continuously. With `PeripheralPowerGuard<ADCPeripheral>`, the destructor guarantees the peripheral is powered off when the scope exits - whether by normal return, early return, or (if exceptions are enabled) exception. You get the correct behavior automatically, with no discipline required at the call site.

---

## Notes

- Measure actual current draw with an oscilloscope/current probe - datasheet values are typical, not guaranteed.
- Always disable debug interfaces (SWD/JTAG) in production builds - they prevent deep sleep on most MCUs.
- Test wake-up paths thoroughly - bugs in low-power code are hard to debug because the system is asleep when the problem occurs.
- Use `__DSB()` (Data Synchronization Barrier) before `__WFI()` to ensure all memory writes complete before the CPU sleeps.
- Consider brown-out detection and voltage supervision for ultra-low-power designs.
