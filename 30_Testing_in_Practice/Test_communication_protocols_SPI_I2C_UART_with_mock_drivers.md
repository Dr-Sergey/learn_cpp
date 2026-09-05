# Test communication protocols (SPI, I2C, UART) with mock drivers

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Embedded systems spend a lot of their time talking to sensors, actuators, and other microcontrollers over serial buses. The challenge is that the protocol handling logic - framing, addressing, checksums, register maps, error recovery - is exactly the kind of code that's rich with bugs, but it's also tightly coupled to hardware that you can't easily access in a unit test or CI pipeline.

The solution is **mock drivers**: software implementations of the bus interface that capture transmitted data and inject controlled responses. Your protocol code calls the same interface it would use with real hardware, but the underlying implementation is a C++ class under your control. This lets you test that your driver sends the right bytes, handles NAKs and timeouts correctly, and interprets sensor responses accurately - all without a single piece of physical hardware.

The reason to write a simulator mock (one that actually models the device register state) rather than just using `EXPECT_CALL` everywhere is depth. A simple mock verifies the sequence of calls. A simulator verifies that the right data flows through those calls in response to different device states, which catches a much wider class of bugs.

### Protocol Quick Reference

| Protocol | Wires | Speed | Addressing | Use Case |
| --- | --- | --- | --- | --- |
| **SPI** | 4 (MOSI/MISO/CLK/CS) | 10-100 MHz | Chip select | Flash, ADC, displays |
| **I2C** | 2 (SDA/SCL) | 100-400 kHz | 7/10-bit address | Sensors, EEPROM |
| **UART** | 2 (TX/RX) | 9600-3M baud | None (point-to-point) | Debug, GPS, modems |

---

## Self-Assessment

### Q1: Mock an I2C bus for testing sensor drivers

**Answer:**

The design starts with a pure abstract interface for the I2C bus. Your application code depends only on this interface, which is what makes the mock substitution possible. The interface covers the three fundamental I2C operations: write, read, and the combined write-then-read (used to set a register address before reading its value).

```cpp
#include <cstdint>
#include <vector>
#include <span>
#include <optional>
#include <map>
#include <gmock/gmock.h>

// === I2C bus interface ===
class II2cBus {
public:
    virtual ~II2cBus() = default;

    enum class Error { Ok, Nak, Timeout, BusError };

    virtual Error write(uint8_t address, std::span<const uint8_t> data) = 0;
    virtual Error read(uint8_t address, std::span<uint8_t> buffer) = 0;
    virtual Error write_read(uint8_t address,
                              std::span<const uint8_t> write_data,
                              std::span<uint8_t> read_buffer) = 0;
};

// === I2C device simulator ===
class I2cDeviceSimulator : public II2cBus {
public:
    // Register a simulated device with register map
    void add_device(uint8_t address,
                    const std::map<uint8_t, uint8_t>& registers) {
        devices_[address] = registers;
    }

    Error write(uint8_t address, std::span<const uint8_t> data) override {
        auto it = devices_.find(address);
        if (it == devices_.end()) return Error::Nak;

        // First byte = register address, rest = data
        if (data.size() >= 2) {
            uint8_t reg = data[0];
            for (size_t i = 1; i < data.size(); ++i)
                it->second[reg + i - 1] = data[i];
        }
        tx_log_.push_back({address, {data.begin(), data.end()}});
        return Error::Ok;
    }

    Error read(uint8_t address, std::span<uint8_t> buffer) override {
        auto it = devices_.find(address);
        if (it == devices_.end()) return Error::Nak;

        // Read from last written register address
        uint8_t reg = last_reg_[address];
        for (size_t i = 0; i < buffer.size(); ++i)
            buffer[i] = it->second[reg + i];
        return Error::Ok;
    }

    Error write_read(uint8_t address,
                      std::span<const uint8_t> write_data,
                      std::span<uint8_t> read_buffer) override {
        auto it = devices_.find(address);
        if (it == devices_.end()) return Error::Nak;

        if (!write_data.empty()) {
            last_reg_[address] = write_data[0];
        }
        return read(address, read_buffer);
    }

    // Inject error for specific device
    void force_nak(uint8_t address) { devices_.erase(address); }

    // Inspect transmitted data
    struct Transaction {
        uint8_t address;
        std::vector<uint8_t> data;
    };
    const std::vector<Transaction>& tx_log() const { return tx_log_; }

private:
    std::map<uint8_t, std::map<uint8_t, uint8_t>> devices_;
    std::map<uint8_t, uint8_t> last_reg_;
    std::vector<Transaction> tx_log_;
};

// === BME280 sensor driver (under test) ===
class Bme280Driver {
public:
    static constexpr uint8_t I2C_ADDR = 0x76;
    static constexpr uint8_t REG_CHIP_ID = 0xD0;
    static constexpr uint8_t REG_TEMP_MSB = 0xFA;
    static constexpr uint8_t CHIP_ID_VALUE = 0x60;

    explicit Bme280Driver(II2cBus& bus) : bus_(bus) {}

    bool init() {
        uint8_t reg = REG_CHIP_ID;
        uint8_t chip_id = 0;
        auto err = bus_.write_read(I2C_ADDR, {&reg, 1}, {&chip_id, 1});
        return err == II2cBus::Error::Ok && chip_id == CHIP_ID_VALUE;
    }

    std::optional<float> read_temperature() {
        uint8_t reg = REG_TEMP_MSB;
        uint8_t data[3]{};
        auto err = bus_.write_read(I2C_ADDR, {&reg, 1}, data);
        if (err != II2cBus::Error::Ok) return std::nullopt;

        int32_t raw = (data[0] << 12) | (data[1] << 4) | (data[2] >> 4);
        return static_cast<float>(raw) / 100.0f;  // Simplified conversion
    }

private:
    II2cBus& bus_;
};
```

Now the tests. Notice that `SetUp` populates the simulator with the exact register values a real BME280 would present, including the chip ID that `init()` verifies. The `force_nak` test checks that the driver handles a missing device correctly - this is the kind of error path that's tedious to trigger with real hardware but trivial with a mock:

```cpp
// === Tests ===
#include <gtest/gtest.h>

class Bme280Test : public ::testing::Test {
protected:
    void SetUp() override {
        bus_.add_device(0x76, {
            {0xD0, 0x60},   // Chip ID register
            {0xFA, 0x50},   // Temp MSB
            {0xFB, 0x00},   // Temp LSB
            {0xFC, 0x00},   // Temp XLSB
        });
    }

    I2cDeviceSimulator bus_;
    Bme280Driver sensor_{bus_};
};

TEST_F(Bme280Test, InitSucceedsWithCorrectChipId) {
    EXPECT_TRUE(sensor_.init());
}

TEST_F(Bme280Test, InitFailsIfDeviceNotPresent) {
    bus_.force_nak(0x76);
    EXPECT_FALSE(sensor_.init());
}

TEST_F(Bme280Test, ReadsTemperature) {
    auto temp = sensor_.read_temperature();
    ASSERT_TRUE(temp.has_value());
    EXPECT_GT(*temp, 0.0f);
}
```

### Q2: Mock a UART protocol with framing and checksums

**Answer:**

UART mocks work on a different model from I2C. Instead of simulating device registers, you capture a transmit buffer (what the code under test sends) and maintain an inject-able receive buffer (what it will read back). This pattern directly tests that protocol framing is correct.

```cpp
// === UART mock with TX logging and RX injection ===
class UartMock : public IUart {
public:
    void init(uint32_t baud_rate) override { baud_ = baud_rate; }

    void send(std::span<const uint8_t> data) override {
        tx_buffer_.insert(tx_buffer_.end(), data.begin(), data.end());
    }

    size_t receive(std::span<uint8_t> buffer, uint32_t) override {
        size_t n = std::min(buffer.size(), rx_buffer_.size());
        std::copy_n(rx_buffer_.begin(), n, buffer.begin());
        rx_buffer_.erase(rx_buffer_.begin(), rx_buffer_.begin() + n);
        return n;
    }

    // Test helpers
    void inject_rx(std::span<const uint8_t> data) {
        rx_buffer_.insert(rx_buffer_.end(), data.begin(), data.end());
    }
    const std::vector<uint8_t>& tx_data() const { return tx_buffer_; }
    void clear() { tx_buffer_.clear(); rx_buffer_.clear(); }

private:
    uint32_t baud_ = 0;
    std::vector<uint8_t> tx_buffer_;
    std::vector<uint8_t> rx_buffer_;
};

// === Modbus RTU protocol handler (simplified) ===
class ModbusRtu {
public:
    explicit ModbusRtu(IUart& uart) : uart_(uart) {}

    struct Response {
        uint8_t slave_id;
        uint8_t function;
        std::vector<uint8_t> data;
        bool valid;
    };

    Response read_holding_registers(uint8_t slave, uint16_t start, uint16_t count) {
        // Build request frame
        std::vector<uint8_t> frame = {
            slave, 0x03,
            static_cast<uint8_t>(start >> 8),
            static_cast<uint8_t>(start & 0xFF),
            static_cast<uint8_t>(count >> 8),
            static_cast<uint8_t>(count & 0xFF)
        };
        auto crc = calculate_crc(frame);
        frame.push_back(crc & 0xFF);
        frame.push_back(crc >> 8);
        uart_.send(frame);

        // Read response
        std::array<uint8_t, 256> buf{};
        size_t n = uart_.receive(buf, 1000);
        if (n < 5) return {0, 0, {}, false};

        return {buf[0], buf[1], {buf.begin() + 3, buf.begin() + 3 + buf[2]}, true};
    }

private:
    uint16_t calculate_crc(std::span<const uint8_t> data) {
        uint16_t crc = 0xFFFF;
        for (uint8_t b : data) {
            crc ^= b;
            for (int i = 0; i < 8; ++i)
                crc = (crc & 1) ? (crc >> 1) ^ 0xA001 : crc >> 1;
        }
        return crc;
    }

    IUart& uart_;
};

TEST(ModbusTest, SendsCorrectFrame) {
    UartMock uart;
    uart.init(9600);
    ModbusRtu modbus(uart);

    // Inject a valid response
    uart.inject_rx({0x01, 0x03, 0x04, 0x00, 0x0A, 0x00, 0x0B, 0xAA, 0xBB});

    auto resp = modbus.read_holding_registers(0x01, 0x0000, 0x0002);

    // Verify request was sent correctly
    auto& tx = uart.tx_data();
    EXPECT_EQ(tx[0], 0x01);  // Slave ID
    EXPECT_EQ(tx[1], 0x03);  // Function: read holding registers
    EXPECT_EQ(tx.size(), 8);  // 6 bytes + 2 CRC
}
```

The test pre-loads the mock's receive buffer with a well-formed response, then asserts on the bytes the driver actually transmitted. This is a clean way to test both sides of a transaction in a single test case.

### Q3: Mock SPI with full-duplex transaction recording

**Answer:**

SPI is full-duplex - every byte you send, you also receive a byte back simultaneously. The `SpiRecorder` below captures complete transactions: for each chip-select cycle it records both the TX bytes sent and the RX bytes returned. This makes it easy to write tests that verify not just what was sent but also that the driver reads the right positions in the response buffer.

```cpp
// === SPI interface ===
class ISpi {
public:
    virtual ~ISpi() = default;
    virtual void select(uint8_t device) = 0;
    virtual void deselect(uint8_t device) = 0;
    virtual uint8_t transfer(uint8_t tx_byte) = 0;  // Full duplex
    virtual void transfer_bulk(std::span<const uint8_t> tx,
                                std::span<uint8_t> rx) = 0;
};

// === SPI recorder mock ===
class SpiRecorder : public ISpi {
public:
    void select(uint8_t device) override {
        selected_ = device;
        transactions_.push_back({device, {}, {}});
    }

    void deselect(uint8_t) override { selected_ = -1; }

    uint8_t transfer(uint8_t tx_byte) override {
        transactions_.back().tx_bytes.push_back(tx_byte);
        uint8_t rx = rx_queue_.empty() ? 0xFF : rx_queue_.front();
        if (!rx_queue_.empty()) rx_queue_.erase(rx_queue_.begin());
        transactions_.back().rx_bytes.push_back(rx);
        return rx;
    }

    void transfer_bulk(std::span<const uint8_t> tx,
                        std::span<uint8_t> rx) override {
        for (size_t i = 0; i < tx.size(); ++i)
            rx[i] = transfer(tx[i]);
    }

    // Test helpers
    void queue_rx(std::initializer_list<uint8_t> bytes) {
        rx_queue_.insert(rx_queue_.end(), bytes);
    }

    struct Transaction {
        uint8_t device;
        std::vector<uint8_t> tx_bytes;
        std::vector<uint8_t> rx_bytes;
    };
    const std::vector<Transaction>& transactions() const { return transactions_; }

private:
    int selected_ = -1;
    std::vector<uint8_t> rx_queue_;
    std::vector<Transaction> transactions_;
};
```

With the recorder you can assert on `transactions()[0].tx_bytes` to verify the exact command bytes sent during a chip-select window, and check `transactions()[0].rx_bytes` to verify the driver correctly consumed the response. This is especially useful for SPI flash and ADC drivers where the command-response structure is precise.

---

## Notes

- Simulator mocks (with state and register maps) test protocol logic more thoroughly than simple `EXPECT_CALL` mocks.
- Always test both success paths and error conditions (NAK, timeout, CRC mismatch).
- SPI full-duplex recording catches bugs where TX/RX timing matters.
- For I2C, simulate multi-byte register reads with auto-incrementing addresses.
- Protocol CRC/checksum validation should be tested with known-good vectors.
- Keep mock drivers simple - they should model the bus, not reimplement the real hardware.
- Transaction logging (TX/RX buffers) helps debug protocol compliance issues in test output.
