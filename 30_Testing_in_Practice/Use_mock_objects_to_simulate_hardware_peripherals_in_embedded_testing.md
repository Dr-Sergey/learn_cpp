# Use mock objects to simulate hardware peripherals in embedded testing

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Embedded firmware interacts with hardware peripherals (SPI flash, ADC, I2C sensors, DMA controllers). **Mock objects** simulate these peripherals in host-based tests, enabling fast verification of firmware logic without physical hardware. Google Mock provides the framework; the key challenge is designing mockable peripheral interfaces.

The reason this matters so much in embedded development is that you cannot practically run every logical test on real silicon. Flashing, booting, and communicating with a device is slow. If you put an abstract interface between your firmware logic and the hardware registers, you can run the firmware logic entirely on your development machine - catching most bugs before the hardware is even involved.

### Mock Strategy by Peripheral Type

Different peripherals call for different simulation strategies. A GPIO is simple state - just track high/low. An ADC benefits from realistic noise modeling because filtering bugs only appear when the data is not perfectly clean.

| Peripheral | Mock Approach | Key Behaviors to Simulate |
| --- | --- | --- |
| **GPIO** | State-tracking mock | Pin levels, direction changes |
| **SPI/I2C** | Request-response mock | Register reads/writes, NAK errors |
| **UART** | Buffer-based mock | TX/RX buffers, timeouts, framing errors |
| **ADC** | Value injection | Conversion results, overrun, noise |
| **Timer** | Time-advancing mock | Callbacks, overflow, capture events |
| **Flash/EEPROM** | Memory-backed mock | Read/write/erase, wear-out simulation |

---

## Self-Assessment

### Q1: Mock an SPI flash peripheral for firmware testing

**Answer:**

There are two distinct simulation tools shown here. `SpiFlashMock` is a Google Mock - use it when you want to verify exactly what function calls your code makes (protocol compliance, call ordering). `SpiFlashSimulator` actually stores data in a `std::vector` and models flash physics - use it for integration-style tests where you want real data to persist across reads and writes. Both implement the same `ISpiFlash` interface, so you can swap them out without changing the firmware code under test.

```cpp
// === Interface: SPI Flash ===
#pragma once
#include <cstdint>
#include <span>

class ISpiFlash {
public:
    virtual ~ISpiFlash() = default;

    virtual bool read(uint32_t address, std::span<uint8_t> buffer) = 0;
    virtual bool write(uint32_t address, std::span<const uint8_t> data) = 0;
    virtual bool erase_sector(uint32_t sector_address) = 0;
    virtual uint32_t capacity() const = 0;
    virtual bool is_busy() const = 0;
};

// === Google Mock for unit tests ===
#include <gmock/gmock.h>

class SpiFlashMock : public ISpiFlash {
public:
    MOCK_METHOD(bool, read, (uint32_t, std::span<uint8_t>), (override));
    MOCK_METHOD(bool, write, (uint32_t, std::span<const uint8_t>), (override));
    MOCK_METHOD(bool, erase_sector, (uint32_t), (override));
    MOCK_METHOD(uint32_t, capacity, (), (const, override));
    MOCK_METHOD(bool, is_busy, (), (const, override));
};

// === Simulated flash (memory-backed, for integration tests) ===
#include <vector>
#include <algorithm>
#include <cstring>

class SpiFlashSimulator : public ISpiFlash {
public:
    static constexpr uint32_t SECTOR_SIZE = 4096;
    static constexpr uint32_t TOTAL_SIZE = 1024 * 1024;  // 1MB

    SpiFlashSimulator() : memory_(TOTAL_SIZE, 0xFF) {}  // Erased = 0xFF

    bool read(uint32_t address, std::span<uint8_t> buffer) override {
        if (address + buffer.size() > TOTAL_SIZE) return false;
        std::memcpy(buffer.data(), &memory_[address], buffer.size());
        return true;
    }

    bool write(uint32_t address, std::span<const uint8_t> data) override {
        if (address + data.size() > TOTAL_SIZE) return false;
        // Flash can only clear bits (AND operation)
        for (size_t i = 0; i < data.size(); ++i)
            memory_[address + i] &= data[i];
        return true;
    }

    bool erase_sector(uint32_t sector_address) override {
        if (sector_address % SECTOR_SIZE != 0) return false;
        if (sector_address + SECTOR_SIZE > TOTAL_SIZE) return false;
        std::fill_n(memory_.begin() + sector_address, SECTOR_SIZE, 0xFF);
        erase_count_++;
        return true;
    }

    uint32_t capacity() const override { return TOTAL_SIZE; }
    bool is_busy() const override { return false; }

    // Test helpers
    uint32_t erase_count() const { return erase_count_; }
    const std::vector<uint8_t>& raw_memory() const { return memory_; }

private:
    std::vector<uint8_t> memory_;
    uint32_t erase_count_ = 0;
};
```

Notice the `write` implementation uses AND (`&=`) rather than direct assignment. Real NOR flash can only change bits from 1 to 0 - you need an erase cycle to turn bits back to 1. Modeling this correctly means your tests will catch bugs where the firmware writes to an already-written sector without erasing first.

### Q2: Test firmware file system logic using the flash mock

**Answer:**

The `FileStorage` class below is the firmware code under test - it knows nothing about how the flash is implemented. When tests use `SpiFlashSimulator`, they exercise the real read/write/erase logic. When the `StrictMock` test runs, it verifies the exact sequence of flash operations, catching bugs like writing without erasing first.

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include <cstring>
#include <string>
#include <optional>

using namespace testing;

// === Simple firmware file storage (under test) ===
class FileStorage {
public:
    explicit FileStorage(ISpiFlash& flash) : flash_(flash) {}

    bool save(const std::string& name, std::span<const uint8_t> data) {
        // Find free sector (simplified: each file = 1 sector)
        uint32_t addr = find_free_sector();
        if (addr == UINT32_MAX) return false;

        // Write header: [name_len(1)][name(N)][data_len(2)][data(...)]
        std::vector<uint8_t> buf;
        buf.push_back(static_cast<uint8_t>(name.size()));
        buf.insert(buf.end(), name.begin(), name.end());
        uint16_t len = static_cast<uint16_t>(data.size());
        buf.push_back(len & 0xFF);
        buf.push_back(len >> 8);
        buf.insert(buf.end(), data.begin(), data.end());

        flash_.erase_sector(addr);
        return flash_.write(addr, buf);
    }

    std::optional<std::vector<uint8_t>> load(const std::string& name) {
        for (uint32_t addr = 0; addr < flash_.capacity();
             addr += SpiFlashSimulator::SECTOR_SIZE) {
            std::array<uint8_t, 256> header{};
            flash_.read(addr, header);

            uint8_t name_len = header[0];
            if (name_len == 0xFF || name_len == 0) continue;

            std::string stored_name(header.begin() + 1,
                                     header.begin() + 1 + name_len);
            if (stored_name != name) continue;

            size_t offset = 1 + name_len;
            uint16_t data_len = header[offset] | (header[offset + 1] << 8);

            std::vector<uint8_t> data(data_len);
            flash_.read(addr + offset + 2, data);
            return data;
        }
        return std::nullopt;
    }

private:
    uint32_t find_free_sector() {
        for (uint32_t addr = 0; addr < flash_.capacity();
             addr += SpiFlashSimulator::SECTOR_SIZE) {
            uint8_t first_byte;
            flash_.read(addr, std::span(&first_byte, 1));
            if (first_byte == 0xFF) return addr;  // Erased sector
        }
        return UINT32_MAX;
    }

    ISpiFlash& flash_;
};

// === Tests with simulated flash ===
class FileStorageTest : public ::testing::Test {
protected:
    SpiFlashSimulator flash_;
    FileStorage storage_{flash_};
};

TEST_F(FileStorageTest, SaveAndLoadRoundtrip) {
    std::vector<uint8_t> data = {1, 2, 3, 4, 5};
    ASSERT_TRUE(storage_.save("config", data));

    auto loaded = storage_.load("config");
    ASSERT_TRUE(loaded.has_value());
    EXPECT_EQ(*loaded, data);
}

TEST_F(FileStorageTest, LoadMissingFileReturnsNullopt) {
    EXPECT_FALSE(storage_.load("nonexistent").has_value());
}

TEST_F(FileStorageTest, MultipleFiles) {
    std::vector<uint8_t> d1 = {10, 20};
    std::vector<uint8_t> d2 = {30, 40, 50};

    ASSERT_TRUE(storage_.save("file1", d1));
    ASSERT_TRUE(storage_.save("file2", d2));

    EXPECT_EQ(storage_.load("file1").value(), d1);
    EXPECT_EQ(storage_.load("file2").value(), d2);
}

// === Unit test with strict mock (verify exact SPI transactions) ===
TEST(FileStorageMockTest, SaveErasesBeforeWriting) {
    StrictMock<SpiFlashMock> flash;
    FileStorage storage(flash);

    // Expect read to find_free_sector
    EXPECT_CALL(flash, read(0, _)).WillOnce(DoAll(
        SetArgReferee<1>(std::array<uint8_t, 1>{0xFF}),
        Return(true)));
    EXPECT_CALL(flash, capacity()).WillRepeatedly(Return(1024 * 1024));

    // Expect erase before write
    {
        InSequence seq;
        EXPECT_CALL(flash, erase_sector(0)).WillOnce(Return(true));
        EXPECT_CALL(flash, write(0, _)).WillOnce(Return(true));
    }

    std::vector<uint8_t> data = {1, 2, 3};
    storage.save("test", data);
}
```

The `StrictMock` test verifies the erase-before-write ordering using `InSequence`. If the firmware were ever refactored to call `write` before `erase_sector`, the `InSequence` expectation would fail and catch that regression immediately.

### Q3: Mock an ADC with noise injection and error simulation

**Answer:**

A completely clean ADC that always returns the exact configured value is unrealistic and will hide filtering bugs. The `AdcSimulator` below adds configurable Gaussian noise so you can test that your noise-filtering code actually works before running on hardware.

```cpp
// === ADC interface ===
class IAdc {
public:
    virtual ~IAdc() = default;

    enum class Error { None, Timeout, Overrun, CalibrationFailed };

    struct Result {
        uint16_t raw_value;  // 12-bit ADC: 0-4095
        Error error;
    };

    virtual void init(uint8_t channel, uint32_t sample_rate) = 0;
    virtual Result read_single(uint8_t channel) = 0;
    virtual void start_continuous(uint8_t channel) = 0;
    virtual void stop_continuous() = 0;
};

// === ADC simulator with realistic behavior ===
#include <random>

class AdcSimulator : public IAdc {
public:
    void init(uint8_t, uint32_t) override {}

    Result read_single(uint8_t channel) override {
        if (force_error_ != Error::None)
            return {0, force_error_};

        auto it = base_values_.find(channel);
        uint16_t base = (it != base_values_.end()) ? it->second : 2048;

        // Add Gaussian noise
        std::normal_distribution<double> noise(0.0, noise_stddev_);
        int noisy = static_cast<int>(base + noise(rng_));
        noisy = std::clamp(noisy, 0, 4095);

        return {static_cast<uint16_t>(noisy), Error::None};
    }

    void start_continuous(uint8_t) override {}
    void stop_continuous() override {}

    // Test helpers
    void set_base_value(uint8_t channel, uint16_t value) {
        base_values_[channel] = value;
    }
    void set_noise(double stddev) { noise_stddev_ = stddev; }
    void force_error(Error err) { force_error_ = err; }

private:
    std::map<uint8_t, uint16_t> base_values_;
    double noise_stddev_ = 2.0;  // ~0.05% noise on 12-bit
    Error force_error_ = Error::None;
    std::mt19937 rng_{42};
};

// === Test: temperature reading with noise filtering ===
class TemperatureSensor {
public:
    explicit TemperatureSensor(IAdc& adc, uint8_t channel)
        : adc_(adc), channel_(channel) {}

    std::optional<float> read_celsius() {
        // Average 10 readings for noise reduction
        float sum = 0;
        for (int i = 0; i < 10; ++i) {
            auto result = adc_.read_single(channel_);
            if (result.error != IAdc::Error::None) return std::nullopt;
            sum += result.raw_value;
        }
        float avg = sum / 10.0f;
        return (avg / 4095.0f) * 150.0f - 50.0f;  // -50 to 100 degrees C range
    }

private:
    IAdc& adc_;
    uint8_t channel_;
};

TEST(TemperatureSensorTest, ReadsWithNoise) {
    AdcSimulator adc;
    adc.set_base_value(0, 2730);  // ~50 degrees C
    adc.set_noise(5.0);           // Moderate noise

    TemperatureSensor sensor(adc, 0);
    auto temp = sensor.read_celsius();
    ASSERT_TRUE(temp.has_value());
    EXPECT_NEAR(*temp, 50.0f, 2.0f);  // Within 2 degrees C due to noise
}

TEST(TemperatureSensorTest, HandlesAdcError) {
    AdcSimulator adc;
    adc.force_error(IAdc::Error::Timeout);

    TemperatureSensor sensor(adc, 0);
    EXPECT_FALSE(sensor.read_celsius().has_value());
}
```

The `force_error` mechanism lets you verify that your firmware's error-handling path is correct. Without it you would need to physically break the ADC hardware to test that path. With it, one line of test setup is all it takes.

---

## Notes

- **Simulators with state** (flash memory contents, pin levels) are better than mock expectations for integration tests.
- **Strict mocks** verify exact call sequences - use for protocol compliance testing (SPI transaction order).
- **Noise injection** in ADC simulators catches filtering bugs that clean test data misses.
- **Error simulation** (`force_error()`) verifies firmware gracefully handles hardware failures.
- Keep mock/simulator complexity proportional to the test value - don't over-simulate.
- The flash simulator correctly models flash physics: erase sets to 0xFF, write can only clear bits.
- Test both the "happy path" (simulator) and exact call sequences (mock) for complete coverage.
