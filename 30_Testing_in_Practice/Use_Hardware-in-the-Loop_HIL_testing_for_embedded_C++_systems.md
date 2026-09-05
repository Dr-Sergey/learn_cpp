# Use Hardware-in-the-Loop (HIL) testing for embedded C++ systems

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Hardware-in-the-Loop (HIL)** testing connects real embedded hardware (the device under test) to a test system that simulates the environment. The test host drives inputs, monitors outputs, and verifies timing and behavior against requirements. HIL bridges the gap between host simulation and field testing - it gives you real silicon running your real firmware, but in a controlled, repeatable environment driven by scripts.

The reason HIL exists as its own testing level is that host unit tests and emulators, however good, cannot catch everything. Real hardware has timing tolerances, ADC noise floors, GPIO glitch behavior, and watchdog interactions that only show up when you are running on the actual chip. HIL lets you stress-test that behavior automatically.

### Testing Pyramid for Embedded

The diagram below shows where HIL sits. Each layer builds on the one below it - you should have many more host unit tests than HIL tests, because HIL is slower and requires physical hardware.

```cpp
         /  Field Test  \         <- Real environment, expensive
        /     HIL Test    \       <- Real HW, simulated environment
       /   QEMU / Emulator  \     <- Simulated HW, real firmware
      /    Host Unit Tests    \   <- No HW, logic only
     /_________________________\
```

| Level | Hardware | Environment | Speed | Catches |
| --- | --- | --- | --- | --- |
| Host test | None | None | ms | Logic bugs |
| Emulator | Simulated | Simulated | seconds | CPU-level bugs |
| **HIL** | **Real** | **Simulated** | **seconds-min** | **Timing, HW interaction** |
| Field test | Real | Real | hours | Everything |

---

## Self-Assessment

### Q1: Design a HIL test framework for embedded systems

**Answer:**

The pattern here is a Python class that speaks a simple binary protocol over UART to a test adapter board. The adapter board (an Arduino, Raspberry Pi, or a dedicated HIL board) is wired to the device under test and can drive its GPIO pins, inject ADC values, and measure timing. The CI runner talks to the adapter; the adapter talks to the DUT.

```python
# === hil_framework.py: Python-based HIL test runner ===
import serial
import time
import struct
from dataclasses import dataclass
from typing import Optional

@dataclass
class PinState:
    pin: int
    level: bool  # True = HIGH, False = LOW
    timestamp_ms: float

class HilTestBench:
    """Communicates with DUT (Device Under Test) via UART/SWD."""

    def __init__(self, port: str, baud: int = 115200):
        self.serial = serial.Serial(port, baud, timeout=2)
        self.gpio_states: dict[int, bool] = {}

    # === GPIO stimulus ===
    def set_input(self, pin: int, level: bool):
        """Drive an input pin on the DUT."""
        cmd = struct.pack('<BBB', 0x01, pin, int(level))
        self.serial.write(cmd)
        self._wait_ack()

    def read_output(self, pin: int) -> bool:
        """Read an output pin from the DUT."""
        cmd = struct.pack('<BB', 0x02, pin)
        self.serial.write(cmd)
        resp = self.serial.read(3)
        return bool(resp[2])

    # === ADC stimulus ===
    def inject_adc(self, channel: int, value: int):
        """Set a simulated ADC value on the test bench."""
        cmd = struct.pack('<BBH', 0x03, channel, value)
        self.serial.write(cmd)
        self._wait_ack()

    # === Timing measurement ===
    def measure_response_time(self, stimulus_pin: int,
                               response_pin: int,
                               timeout_ms: float = 100) -> Optional[float]:
        """Measure time from stimulus to response."""
        cmd = struct.pack('<BBBf', 0x10, stimulus_pin, response_pin, timeout_ms)
        self.serial.write(cmd)
        resp = self.serial.read(5)
        if resp[0] == 0x00:  # Timeout
            return None
        return struct.unpack('<f', resp[1:5])[0]

    # === UART monitoring ===
    def capture_uart_output(self, duration_s: float) -> bytes:
        """Capture what the DUT sends on its UART TX."""
        time.sleep(duration_s)
        return self.serial.read_all()

    def send_uart_to_dut(self, data: bytes):
        """Send data to DUT's UART RX."""
        self.serial.write(data)

    # === Power control ===
    def power_cycle_dut(self, off_time_s: float = 0.5):
        """Simulate power cycle."""
        cmd = struct.pack('<Bf', 0x20, off_time_s)
        self.serial.write(cmd)
        time.sleep(off_time_s + 0.5)  # Wait for reboot

    def _wait_ack(self, timeout: float = 1.0):
        resp = self.serial.read(1)
        if not resp or resp[0] != 0xAA:
            raise RuntimeError("DUT did not acknowledge")
```

The `_wait_ack` method is the small but important detail - every command that modifies hardware state waits for a confirmation byte so the test never races ahead before the adapter has had time to act.

### Q2: Write HIL tests with timing constraints

**Answer:**

These tests use pytest as the runner. The `scope="session"` fixture opens the serial port once for the whole test session, while the `autouse` fixture power-cycles the DUT before every individual test so each test starts from a known clean boot state. Notice how the timing test has an explicit requirement baked into the assertion message.

```python
# === test_hil.py: pytest-based HIL tests ===
import pytest
from hil_framework import HilTestBench

LED_PIN = 13
BUTTON_PIN = 2
ADC_CHANNEL = 0

@pytest.fixture(scope="session")
def bench():
    # Connect to test bench hardware
    b = HilTestBench("/dev/ttyUSB0", 115200)
    yield b
    b.serial.close()

@pytest.fixture(autouse=True)
def reset_dut(bench):
    """Reset DUT before each test."""
    bench.power_cycle_dut(off_time_s=0.2)
    yield

# === Test: Button press toggles LED ===
def test_button_toggles_led(bench):
    # Initial state: LED off
    assert bench.read_output(LED_PIN) == False

    # Press button
    bench.set_input(BUTTON_PIN, True)
    time.sleep(0.05)  # Debounce time
    bench.set_input(BUTTON_PIN, False)

    # LED should be on
    assert bench.read_output(LED_PIN) == True

    # Press again
    bench.set_input(BUTTON_PIN, True)
    time.sleep(0.05)
    bench.set_input(BUTTON_PIN, False)

    # LED should be off
    assert bench.read_output(LED_PIN) == False

# === Test: Response time requirement ===
def test_button_response_time(bench):
    """Requirement: LED must respond within 10ms of button press."""
    response_ms = bench.measure_response_time(
        stimulus_pin=BUTTON_PIN,
        response_pin=LED_PIN,
        timeout_ms=50
    )
    assert response_ms is not None, "No response detected"
    assert response_ms < 10.0, f"Response too slow: {response_ms}ms"

# === Test: ADC threshold triggers alarm ===
def test_adc_temperature_alarm(bench):
    ALARstd::numbers::piN = 7

    # Below threshold: no alarm
    bench.inject_adc(ADC_CHANNEL, 2000)  # ~49C
    time.sleep(0.1)
    assert bench.read_output(ALARstd::numbers::piN) == False

    # Above threshold: alarm active
    bench.inject_adc(ADC_CHANNEL, 3000)  # ~73C
    time.sleep(0.1)
    assert bench.read_output(ALARstd::numbers::piN) == True

    # Back below: alarm clears
    bench.inject_adc(ADC_CHANNEL, 2000)
    time.sleep(0.5)  # Hysteresis delay
    assert bench.read_output(ALARstd::numbers::piN) == False

# === Test: Watchdog recovery ===
def test_watchdog_recovery(bench):
    """DUT should recover from watchdog reset within 500ms."""
    # Send command that triggers intentional hang
    bench.send_uart_to_dut(b'\x99')  # Trigger infinite loop in test mode

    # Watchdog should fire and reset
    time.sleep(1.0)  # Wait for WDT timeout + boot

    # DUT should be responsive again
    assert bench.read_output(LED_PIN) == False  # Default state after boot

# === Test: Power-loss resilience ===
def test_settings_survive_power_cycle(bench):
    # Write a setting
    bench.send_uart_to_dut(b'SET:threshold=75\n')
    time.sleep(0.1)

    # Power cycle
    bench.power_cycle_dut(off_time_s=0.5)

    # Read back setting
    bench.send_uart_to_dut(b'GET:threshold\n')
    response = bench.capture_uart_output(0.5)
    assert b'75' in response
```

The `test_settings_survive_power_cycle` test is a good example of what makes HIL uniquely valuable - you literally cannot write this test as a host unit test. It verifies that data written to flash actually persists across a real power loss, which involves the flash driver, the wear-leveling code, and the boot-time initialization code all working together correctly on real hardware.

### Q3: CI pipeline with HIL test execution

**Answer:**

HIL tests need a self-hosted GitHub Actions runner - a machine physically connected to the test bench. The workflow below flashes the freshly-built firmware onto the DUT, then runs the test suite. The `runs-on: [self-hosted, hil-bench-1]` label routes the job to exactly that machine.

```yaml
# === .github/workflows/hil.yml ===
# Requires a self-hosted runner connected to HIL test bench
name: HIL Tests

on:
  push:
    branches: [main]
  workflow_dispatch:  # Manual trigger

jobs:
  hil-test:
    runs-on: [self-hosted, hil-bench-1]  # Specific runner with HW
    timeout-minutes: 30

    steps:

      - uses: actions/checkout@v4

      - name: Cross-compile firmware

        run: |
          cmake -B build --toolchain cmake/arm-toolchain.cmake
          cmake --build build --target firmware

      - name: Flash DUT

        run: |
          # Use OpenOCD, pyOCD, or vendor tool
          pyocd flash build/firmware.bin --target stm32f407

      - name: Run HIL tests

        run: |
          pip install pytest pyserial
          pytest tests/hil/ -v --tb=short \
              --junitxml=hil_results.xml \
              --timeout=60

      - name: Upload results

        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: hil-results
          path: hil_results.xml

      - name: Publish test report

        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: HIL Test Results
          path: hil_results.xml
          reporter: java-junit
```

The architecture of the whole system looks like this. The CI runner flashes the DUT over SWD/JTAG, then talks to it through a test adapter over GPIO and UART:

```cpp
HIL Test Bench Architecture:

  +-----------+      USB/UART      +-----------+
  | CI Runner | <=================> | Test      |
  | (PC)      |                     | Adapter   |
  +-----------+      SWD/JTAG      | (Arduino/ |
       |         +=================> | RPi)      |
       |         |                  +-----------+
       |         |                       |
       |         |    GPIO/ADC/SPI       |
       |         |  +-----------+        |
       |         +->|    DUT    |<-------+
       |            | (Target)  |
       |            +-----------+
       |
  Flash via
  pyOCD/OpenOCD
```

---

## Notes

- **HIL tests are slow** (seconds per test) - keep the count small and focused on HW-dependent behavior.
- Use self-hosted GitHub Actions runners connected to physical test benches.
- Run HIL tests on main branch only (not every PR) to avoid hardware contention.
- Test timing requirements (response latency, watchdog timeout) that host tests can't verify.
- Power-cycle testing catches flash corruption, boot failures, and state persistence bugs.
- The test adapter (Arduino, RPi, dedicated HIL board) drives DUT inputs and reads outputs.
- Python + pytest is the standard HIL test framework - serial communication is simple.
- Label HIL tests separately; developers run host tests locally, HIL runs in CI only.
