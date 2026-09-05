# Design embedded software architecture with layered HAL and BSP

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

**Layered embedded architecture** separates hardware-specific code from application logic through two layers: a **BSP** (Board Support Package) and a **HAL** (Hardware Abstraction Layer). The BSP contains board-specific initialization - clocks, pin assignments, peripheral setup. The HAL provides hardware-independent interfaces that the application code calls. Application code never touches vendor headers or hardware registers directly; it only talks to the HAL.

The payoff is portability and testability. If you swap from an STM32F4 to an nRF52840, you write a new BSP and BSP implementations, but the application logic does not change at all. And because application code depends only on interfaces, you can implement mock HAL objects that run on your host PC - no hardware required to run unit tests.

### Layer Architecture

| Layer | Responsibility | Changes When | Example |
| --- | --- | --- | --- |
| **Application** | Business logic, state machines | Requirements change | Motor controller logic |
| **Service/Middleware** | RTOS tasks, protocols, drivers | Feature addition | CAN bus protocol stack |
| **HAL** | Hardware-independent interfaces | Never (stable API) | `IGpio`, `ISpi`, `ITimer` |
| **BSP** | Board-specific initialization + config | Board revision changes | Pin mappings, clock setup |
| **Hardware** | Physical MCU + peripherals | PCB redesign | STM32F4, nRF52840 |

---

## Self-Assessment

### Q1: Implement HAL interfaces and BSP implementation

**Answer:**

The HAL interfaces below define what the application can ask the hardware to do - set a pin, perform a SPI transfer, send bytes over UART, start a timer. Every method is pure virtual. The application only ever sees these interfaces. Below them is the BSP implementation for a specific MCU, which knows about `GPIO_TypeDef` and `HAL_GPIO_Init` - that knowledge is entirely contained in the BSP files and never leaks upward.

```cpp
// === HAL interfaces (platform-independent) ===
// hal/gpio.h
#pragma once
#include <cstdint>

namespace hal {

enum class PinMode { Input, Output, AlternateFunction, Analog };
enum class PullMode { None, PullUp, PullDown };
enum class PinState { Low, High };

class IGpio {
public:
    virtual ~IGpio() = default;
    virtual void configure(PinMode mode, PullMode pull = PullMode::None) = 0;
    virtual void set(PinState state) = 0;
    virtual PinState read() = 0;
    virtual void toggle() = 0;
};

// hal/spi.h
class ISpi {
public:
    virtual ~ISpi() = default;
    virtual void configure(uint32_t clock_hz, uint8_t mode) = 0;
    virtual void transfer(const uint8_t* tx, uint8_t* rx, size_t len) = 0;
    virtual void write(const uint8_t* data, size_t len) = 0;
    virtual uint8_t read_byte() = 0;
};

// hal/uart.h
class IUart {
public:
    virtual ~IUart() = default;
    virtual void configure(uint32_t baud, uint8_t data_bits = 8) = 0;
    virtual void send(const uint8_t* data, size_t len) = 0;
    virtual size_t receive(uint8_t* buffer, size_t max_len,
                           uint32_t timeout_ms) = 0;
};

// hal/timer.h
class ITimer {
public:
    virtual ~ITimer() = default;
    virtual void start(uint32_t period_us) = 0;
    virtual void stop() = 0;
    virtual uint32_t elapsed_us() = 0;
    virtual void set_callback(void(*cb)(void*), void* ctx) = 0;
};

} // namespace hal


// === BSP implementation for STM32F4 ===
// bsp/stm32f4/gpio_stm32f4.h
#include "hal/gpio.h"
#include "stm32f4xx.h"

namespace bsp {

class GpioStm32F4 : public hal::IGpio {
public:
    GpioStm32F4(GPIO_TypeDef* port, uint16_t pin)
        : port_(port), pin_(pin) {}

    void configure(hal::PinMode mode, hal::PullMode pull) override {
        GPIO_InitTypeDef init{};
        init.Pin = pin_;
        init.Mode = to_hal_mode(mode);
        init.Pull = to_hal_pull(pull);
        init.Speed = GPIO_SPEED_FREQ_HIGH;
        HAL_GPIO_Init(port_, &init);
    }

    void set(hal::PinState state) override {
        HAL_GPIO_WritePin(port_, pin_,
            state == hal::PinState::High ? GPIO_PIN_SET : GPIO_PIN_RESET);
    }

    hal::PinState read() override {
        return HAL_GPIO_ReadPin(port_, pin_) == GPIO_PIN_SET
            ? hal::PinState::High : hal::PinState::Low;
    }

    void toggle() override {
        HAL_GPIO_TogglePin(port_, pin_);
    }

private:
    GPIO_TypeDef* port_;
    uint16_t pin_;

    static uint32_t to_hal_mode(hal::PinMode m) {
        switch (m) {
            case hal::PinMode::Input: return GPIO_MODE_INPUT;
            case hal::PinMode::Output: return GPIO_MODE_OUTPUT_PP;
            case hal::PinMode::AlternateFunction: return GPIO_MODE_AF_PP;
            case hal::PinMode::Analog: return GPIO_MODE_ANALOG;
        }
        return GPIO_MODE_INPUT;
    }

    static uint32_t to_hal_pull(hal::PullMode p) {
        switch (p) {
            case hal::PullMode::None: return GPIO_NOPULL;
            case hal::PullMode::PullUp: return GPIO_PULLUP;
            case hal::PullMode::PullDown: return GPIO_PULLDOWN;
        }
        return GPIO_NOPULL;
    }
};

} // namespace bsp
```

The `to_hal_mode` and `to_hal_pull` helpers are private and static because they are pure translation functions. They map from the HAL's vocabulary to the vendor HAL's vocabulary. That translation is entirely a BSP concern and should never be visible to application code.

### Q2: Board-level initialization and dependency wiring

**Answer:**

The `Board` class is the top-level wiring object for a specific PCB revision. It owns all the BSP implementation objects and initializes the clocks and peripherals they depend on. Application code gets handles to the HAL interfaces through accessor methods - it receives an `IGpio&`, not a `GpioStm32F4&`. The concrete type stays hidden inside the BSP.

<!-- compile: fragment: depends on a header defined elsewhere in the topic -->
```cpp
// === BSP board initialization ===
// bsp/stm32f4_discovery/board.h
#pragma once
#include "bsp/stm32f4/gpio_stm32f4.h"
#include "bsp/stm32f4/spi_stm32f4.h"
#include "bsp/stm32f4/uart_stm32f4.h"

namespace bsp {

// Board-specific pin assignments
struct BoardPins {
    static constexpr auto LED_PORT = GPIOD;
    static constexpr uint16_t LED_GREEN = GPIO_PIN_12;
    static constexpr uint16_t LED_RED = GPIO_PIN_14;

    static constexpr auto SPI_PORT = SPI1;
    static constexpr auto DEBUG_UART = USART2;
    static constexpr uint32_t DEBUG_BAUD = 115200;
};

class Board {
public:
    void init() {
        // Clock initialization
        __HAL_RCC_GPIOD_CLK_ENABLE();
        __HAL_RCC_SPI1_CLK_ENABLE();
        __HAL_RCC_USART2_CLK_ENABLE();

        // Configure peripherals
        led_green_.configure(hal::PinMode::Output);
        led_red_.configure(hal::PinMode::Output);
        debug_uart_.configure(BoardPins::DEBUG_BAUD);
        sensor_spi_.configure(1000000, 0);  // 1MHz, mode 0
    }

    // HAL interface accessors
    hal::IGpio& led_green() { return led_green_; }
    hal::IGpio& led_red() { return led_red_; }
    hal::IUart& debug_uart() { return debug_uart_; }
    hal::ISpi& sensor_spi() { return sensor_spi_; }

private:
    GpioStm32F4 led_green_{BoardPins::LED_PORT, BoardPins::LED_GREEN};
    GpioStm32F4 led_red_{BoardPins::LED_PORT, BoardPins::LED_RED};
    UartStm32F4 debug_uart_{BoardPins::DEBUG_UART};
    SpiStm32F4 sensor_spi_{BoardPins::SPI_PORT};
};

} // namespace bsp


// === Application code: depends ONLY on HAL interfaces ===
// app/sensor_reader.h
#include "hal/spi.h"
#include "hal/gpio.h"

class SensorReader {
public:
    SensorReader(hal::ISpi& spi, hal::IGpio& cs_pin)
        : spi_(spi), cs_(cs_pin) {}

    float read_temperature() {
        cs_.set(hal::PinState::Low);
        uint8_t cmd = 0x01;  // Read temp register
        uint8_t response[2]{};
        spi_.transfer(&cmd, response, 2);
        cs_.set(hal::PinState::High);

        int16_t raw = (response[0] << 8) | response[1];
        return raw * 0.0625f;
    }

private:
    hal::ISpi& spi_;
    hal::IGpio& cs_;
};

// === main.cpp: wiring ===
int main() {
    bsp::Board board;
    board.init();

    SensorReader sensor(board.sensor_spi(), board.led_green());
    float temp = sensor.read_temperature();
}
```

The `main.cpp` shows the key property of this architecture: `SensorReader` is constructed with `hal::ISpi&` and `hal::IGpio&`. It does not know or care which MCU it is running on. That same class compiles and runs in unit tests on your development machine using mock objects.

### Q3: Host-side test doubles for HAL

**Answer:**

The mock HAL classes below implement the same interfaces as the real BSP, but instead of touching hardware registers they record what was called. Tests can then inspect `cs.state_history` to verify the CS pin was toggled in the right order, or check `spi.tx_log` to verify the correct command byte was sent. You are testing protocol logic, not hardware behavior.

```cpp
// === Mock HAL for host-PC testing ===
// test/mock_gpio.h
#include "hal/gpio.h"
#include <vector>

class MockGpio : public hal::IGpio {
public:
    void configure(hal::PinMode mode, hal::PullMode pull) override {
        configured_mode = mode;
        configured_pull = pull;
    }

    void set(hal::PinState state) override {
        current_state = state;
        state_history.push_back(state);
    }

    hal::PinState read() override {
        return next_read_value;
    }

    void toggle() override {
        current_state = (current_state == hal::PinState::High)
            ? hal::PinState::Low : hal::PinState::High;
        state_history.push_back(current_state);
    }

    // Test inspection
    hal::PinMode configured_mode{};
    hal::PullMode configured_pull{};
    hal::PinState current_state = hal::PinState::Low;
    hal::PinState next_read_value = hal::PinState::Low;
    std::vector<hal::PinState> state_history;
};

class MockSpi : public hal::ISpi {
public:
    void configure(uint32_t clock_hz, uint8_t mode) override {
        configured_clock = clock_hz;
    }

    void transfer(const uint8_t* tx, uint8_t* rx, size_t len) override {
        tx_log.insert(tx_log.end(), tx, tx + len);
        for (size_t i = 0; i < len && i < rx_data.size(); ++i)
            rx[i] = rx_data[i];
    }

    void write(const uint8_t* data, size_t len) override {
        tx_log.insert(tx_log.end(), data, data + len);
    }

    uint8_t read_byte() override { return 0; }

    uint32_t configured_clock = 0;
    std::vector<uint8_t> tx_log;
    std::vector<uint8_t> rx_data;  // Pre-loaded responses
};

// === Unit test (runs on host PC, no hardware needed) ===
void test_sensor_reader() {
    MockSpi spi;
    MockGpio cs;

    // Pre-load SPI response: 25.5 degrees C = 0x01 0x98
    spi.rx_data = {0x01, 0x98};

    SensorReader reader(spi, cs);
    float temp = reader.read_temperature();

    // Verify CS pin toggled correctly
    assert(cs.state_history.size() == 2);
    assert(cs.state_history[0] == hal::PinState::Low);   // CS asserted
    assert(cs.state_history[1] == hal::PinState::High);  // CS deasserted

    // Verify SPI command sent
    assert(spi.tx_log[0] == 0x01);  // Read temp command

    // Verify temperature conversion
    assert(std::abs(temp - 25.5f) < 0.1f);
}
```

The `rx_data` field on `MockSpi` lets you pre-load the bytes you want the mock to return from a `transfer()` call. That is how you simulate a sensor responding with a specific temperature reading. You control both sides of the conversation, so the test is fully deterministic and hardware-independent.

---

## Notes

- Application code should never include vendor headers like `stm32f4xx.h` - only HAL interface headers.
- The BSP changes when you switch boards; HAL interfaces stay stable; application code is unchanged throughout.
- Mock HAL objects enable unit testing on the host PC without any hardware - you can realistically cover 90% of application logic this way.
- Use a `Board` class to wire BSP implementations to HAL interfaces - it acts as a simple, manual dependency injection root.
- Each supported board gets its own BSP directory: `bsp/stm32f4/`, `bsp/nrf52/`, `bsp/host/`.
- CMake selects the correct BSP at build time using a define like `-DBSP=stm32f4_discovery`.
- Keep HAL interfaces minimal and stable - they are the contract between portable application code and platform-specific code, and breaking them is expensive.
