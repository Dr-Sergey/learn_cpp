# Test embedded C++ code on host with hardware abstraction layers

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Hardware Abstraction Layers (HAL)** are the architectural pattern that makes embedded code testable. The idea is simple: instead of calling GPIO or UART hardware registers directly from your application logic, you call through a pure virtual interface. On the real device, that interface is backed by the actual hardware driver. On your development PC, it's backed by a mock or simulator class that you control.

This matters enormously for your development speed. Flashing firmware to a microcontroller, waiting for it to boot, and attaching a debugger takes at minimum tens of seconds per iteration. Running a host test suite takes milliseconds. If you can catch 80% of your bugs before you ever touch hardware, you've dramatically shortened the debug cycle for the harder 20% that genuinely requires the real device.

The reason this trips people up is the architecture has to be decided early. It's very hard to retrofit a HAL onto firmware that was written with direct hardware register access throughout. The time to set up the interfaces and the mock implementations is at the start of the project, not after the firmware is mostly written.

### Testing Strategy Layers

```cpp
  +-----------------------------+
  | Application Logic           |  <-- Tested on host (fast)
  +-----------------------------+
  | HAL Interface (abstract)    |  <-- Swap implementation
  +-----------------------------+
  | Host Mock  | Real Hardware  |  <-- Test vs Production
  +------------+----------------+
```

| Approach | Speed | Fidelity | Setup |
| --- | --- | --- | --- |
| Host + HAL mock | ms | Logic only | Easy |
| QEMU emulation | seconds | CPU accurate | Medium |
| Hardware-in-Loop | seconds | Full | Expensive |

---

## Self-Assessment

### Q1: Design a HAL interface and swap implementations for host testing

**Answer:**

Here are three minimal HAL interfaces covering the most common peripherals. Notice that each interface is deliberately narrow - it only exposes what application code actually needs, not every feature the hardware supports. A narrow interface is easier to mock, easier to keep stable, and less likely to leak hardware-specific details into your application.

```cpp
// === hal/gpio.h - Abstract interface ===
#pragma once
#include <cstdint>

class IGpio {
public:
    virtual ~IGpio() = default;

    enum class Direction { Input, Output };
    enum class Level { Low, High };

    virtual void configure(uint8_t pin, Direction dir) = 0;
    virtual void write(uint8_t pin, Level level) = 0;
    virtual Level read(uint8_t pin) = 0;
};

// === hal/uart.h - Abstract interface ===
#pragma once
#include <cstdint>
#include <span>

class IUart {
public:
    virtual ~IUart() = default;
    virtual void init(uint32_t baud_rate) = 0;
    virtual void send(std::span<const uint8_t> data) = 0;
    virtual size_t receive(std::span<uint8_t> buffer, uint32_t timeout_ms) = 0;
};

// === hal/timer.h - Abstract interface ===
#pragma once
#include <cstdint>
#include <functional>

class ITimer {
public:
    virtual ~ITimer() = default;
    virtual void start_periodic(uint32_t period_ms, std::function<void()> callback) = 0;
    virtual void stop() = 0;
    virtual uint32_t elapsed_ms() const = 0;
};
```

On the target hardware, the implementation wraps the vendor HAL (here STM32's `HAL_GPIO_*` API). This is the only place where vendor-specific code lives - everything above the interface boundary is portable:

```cpp
// === hal/stm32/gpio_stm32.h - Real hardware implementation ===
#pragma once
#include "hal/gpio.h"
#include "stm32f4xx_hal.h"  // Vendor HAL

class GpioStm32 : public IGpio {
public:
    void configure(uint8_t pin, Direction dir) override {
        GPIO_InitTypeDef init{};
        init.Pin = (1U << pin);
        init.Mode = (dir == Direction::Output) ? GPIO_MODE_OUTPUT_PP
                                                : GPIO_MODE_INPUT;
        HAL_GPIO_Init(GPIOA, &init);
    }

    void write(uint8_t pin, Level level) override {
        HAL_GPIO_WritePin(GPIOA, (1U << pin),
                          level == Level::High ? GPIO_PIN_SET : GPIO_PIN_RESET);
    }

    Level read(uint8_t pin) override {
        return HAL_GPIO_ReadPin(GPIOA, (1U << pin)) ? Level::High : Level::Low;
    }
};
```

For testing you have two choices: a `GpioMock` that uses gmock's `MOCK_METHOD` for precise call expectations, or a `GpioSimulator` that actually tracks pin states. The simulator is better when you're testing multi-step interactions where you need to read back what was written:

```cpp
// === hal/host/gpio_mock.h - Host test mock ===
#pragma once
#include "hal/gpio.h"
#include <gmock/gmock.h>
#include <map>

class GpioMock : public IGpio {
public:
    MOCK_METHOD(void, configure, (uint8_t, Direction), (override));
    MOCK_METHOD(void, write, (uint8_t, Level), (override));
    MOCK_METHOD(Level, read, (uint8_t), (override));
};

// Simulated GPIO with state tracking (for integration tests)
class GpioSimulator : public IGpio {
public:
    void configure(uint8_t pin, Direction dir) override {
        directions_[pin] = dir;
    }

    void write(uint8_t pin, Level level) override {
        pin_states_[pin] = level;
    }

    Level read(uint8_t pin) override {
        auto it = pin_states_.find(pin);
        return (it != pin_states_.end()) ? it->second : Level::Low;
    }

    // Test helpers
    Level get_pin_state(uint8_t pin) const {
        auto it = pin_states_.find(pin);
        return (it != pin_states_.end()) ? it->second : Level::Low;
    }

    void simulate_input(uint8_t pin, Level level) {
        pin_states_[pin] = level;
    }

private:
    std::map<uint8_t, Direction> directions_;
    std::map<uint8_t, Level> pin_states_;
};
```

### Q2: Test application logic using HAL mocks

**Answer:**

Here are two application classes that each take a HAL interface by reference. Neither class knows anything about how GPIO or UART is implemented - it only knows the interface. This is dependency injection, and it's what makes the mocking work.

```cpp
// === app/led_controller.h ===
#pragma once
#include "hal/gpio.h"
#include "hal/timer.h"
#include <memory>

class LedController {
public:
    LedController(IGpio& gpio, uint8_t led_pin)
        : gpio_(gpio), pin_(led_pin) {
        gpio_.configure(pin_, IGpio::Direction::Output);
    }

    void on()  { gpio_.write(pin_, IGpio::Level::High); state_ = true; }
    void off() { gpio_.write(pin_, IGpio::Level::Low);  state_ = false; }
    void toggle() { state_ ? off() : on(); }
    bool is_on() const { return state_; }

private:
    IGpio& gpio_;
    uint8_t pin_;
    bool state_ = false;
};

// === app/sensor_reader.h ===
#pragma once
#include "hal/uart.h"
#include <optional>
#include <array>

class SensorReader {
public:
    explicit SensorReader(IUart& uart) : uart_(uart) {}

    struct Reading {
        float temperature;
        float humidity;
    };

    std::optional<Reading> poll() {
        // Send poll command
        uint8_t cmd[] = {0x01, 0x03};
        uart_.send(cmd);

        // Read response
        std::array<uint8_t, 8> buf{};
        size_t n = uart_.receive(buf, 1000);
        if (n < 8) return std::nullopt;

        // Parse (simplified)
        float temp = static_cast<float>((buf[0] << 8) | buf[1]) / 100.0f;
        float hum  = static_cast<float>((buf[2] << 8) | buf[3]) / 100.0f;
        return Reading{temp, hum};
    }

private:
    IUart& uart_;
};
```

Now the tests. Notice the distinction between `GpioMock` (for the first two tests, where we care about exact call sequences) and `GpioSimulator` (for the toggle test, where we want to verify the actual pin state after two calls):

```cpp
// === tests/test_led_controller.cpp ===
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include "app/led_controller.h"
#include "hal/host/gpio_mock.h"

using namespace testing;

TEST(LedControllerTest, InitConfiguresPinAsOutput) {
    GpioMock gpio;
    EXPECT_CALL(gpio, configure(5, IGpio::Direction::Output));
    LedController led(gpio, 5);
}

TEST(LedControllerTest, OnWritesHigh) {
    NiceMock<GpioMock> gpio;
    LedController led(gpio, 5);

    EXPECT_CALL(gpio, write(5, IGpio::Level::High));
    led.on();
    EXPECT_TRUE(led.is_on());
}

TEST(LedControllerTest, ToggleAlternates) {
    GpioSimulator gpio;
    LedController led(gpio, 3);

    led.toggle();
    EXPECT_EQ(gpio.get_pin_state(3), IGpio::Level::High);

    led.toggle();
    EXPECT_EQ(gpio.get_pin_state(3), IGpio::Level::Low);
}
```

`NiceMock<GpioMock>` in the second test suppresses gmock warnings for calls that don't have matching `EXPECT_CALL` entries. You need this for the constructor's `configure()` call - you only care about the `write()` call in that particular test.

### Q3: CMake dual-target build for host and embedded

**Answer:**

The `CMAKE_CROSSCOMPILING` variable is the natural dividing line. When you configure with a cross-compilation toolchain file, CMake sets `CMAKE_CROSSCOMPILING` to `TRUE` and you build the embedded target. When you configure natively, you build the host tests. The same `CMakeLists.txt` handles both cases:

```cmake
# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.20)
project(embedded_app)

# Common application logic (platform-independent)
add_library(app_logic
    src/led_controller.cpp
    src/sensor_reader.cpp
)
target_include_directories(app_logic PUBLIC include/)

if(CMAKE_CROSSCOMPILING)
    # === Embedded target ===
    add_library(hal_stm32
        hal/stm32/gpio_stm32.cpp
        hal/stm32/uart_stm32.cpp
    )
    target_link_libraries(hal_stm32 PUBLIC app_logic stm32_hal_driver)

    add_executable(firmware main_embedded.cpp)
    target_link_libraries(firmware hal_stm32)
else()
    # === Host testing ===
    enable_testing()
    find_package(GTest REQUIRED)

    add_executable(host_tests
        tests/test_led_controller.cpp
        tests/test_sensor_reader.cpp
    )
    target_link_libraries(host_tests
        app_logic
        GTest::gtest_main
        GTest::gmock
    )
    add_test(NAME host_tests COMMAND host_tests)
endif()
```

The two build commands are cleanly separated:

```bash
# Build for host (testing)
cmake -B build-host
cmake --build build-host
ctest --test-dir build-host

# Build for target (flashing)
cmake -B build-arm --toolchain arm-toolchain.cmake
cmake --build build-arm
```

The separation of `app_logic` from any HAL implementation is the key architectural decision. `app_logic` has no platform-specific includes and no `#ifdef` soup - it's plain C++ that compiles and runs anywhere.

---

## Notes

- HAL interfaces must be stable - changing the interface requires updating all implementations and tests.
- Keep HAL interfaces minimal - expose only what the application needs, not every hardware register.
- Use `NiceMock<>` for interactions you don't care about; use strict mocks for critical sequences.
- `GpioSimulator` (with state) is better than `GpioMock` when testing multi-step interactions.
- Host tests catch roughly 80% of firmware bugs (logic, state machines, protocol parsing) in milliseconds.
- The remaining 20% (timing, interrupts, electrical) require hardware-in-the-loop testing.
- Separating `app_logic` from the HAL is the key architectural decision for testability.
