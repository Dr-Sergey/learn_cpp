# Test interrupt service routines (ISR) and real-time constraints

**Category:** Testing in Practice  
**Standard:** C++20  

---

## Topic Overview

**Interrupt Service Routines (ISRs)** are among the hardest embedded code to test, and the reason is fundamental: they execute asynchronously, they have strict timing requirements, they can't use blocking operations, and they interact with hardware registers that don't exist on your development machine. If you try to unit-test an ISR as-is, you'll find it's essentially impossible on the host.

The solution is a structural shift in how you write ISRs. Instead of putting logic inside the interrupt handler, you extract the logic into a plain C++ class and keep the ISR wrapper as thin as possible - ideally just a few lines. The ISR wrapper itself becomes too trivial to be worth testing. The extracted class becomes a regular C++ object that you can test freely on the host with no hardware involved.

### ISR Testing Challenges

| Challenge | Problem | Testing Strategy |
| --- | --- | --- |
| Asynchronous execution | Non-deterministic timing | Simulate with direct function calls |
| No blocking allowed | Can't use mutex, malloc, iostream | Static analysis + review |
| Shared state with main | Data races | Test with TSan, atomic verification |
| Hardware registers | Memory-mapped I/O | HAL abstraction + mocks |
| Timing constraints | Must complete in µs | Benchmark critical sections |
| Reentrance | Nested interrupts | Priority-based test scenarios |

---

## Self-Assessment

### Q1: Extract testable logic from ISRs

The core strategy is: everything in the ISR that has any logic in it belongs in a separate class. The actual interrupt handler should be trivially obvious at a glance. Here's what that looks like before and after the refactoring.

```cpp
// === BAD: Untestable ISR with embedded logic ===
// Can't test this on host - depends on hardware registers
// void TIM2_IRQHandler() {
//     if (TIM2->SR & TIM_SR_UIF) {
//         TIM2->SR &= ~TIM_SR_UIF;
//         static int count = 0;
//         count++;
//         if (count >= 1000) {
//             GPIOA->ODR ^= (1 << 5);  // Toggle LED
//             count = 0;
//         }
//     }
// }

// === GOOD: Separate logic from hardware ===

// Pure logic, fully testable on host
class BlinkController {
public:
    explicit BlinkController(uint32_t period_ticks)
        : period_(period_ticks) {}

    // Returns true when LED should toggle
    bool tick() {
        if (++count_ >= period_) {
            count_ = 0;
            return true;
        }
        return false;
    }

    void reset() { count_ = 0; }
    uint32_t count() const { return count_; }

private:
    uint32_t period_;
    uint32_t count_ = 0;
};

// === Thin ISR wrapper (not unit-tested, but trivially correct) ===
// In production:
// BlinkController blink_ctrl(1000);
// void TIM2_IRQHandler() {
//     timer_hal.clear_interrupt();
//     if (blink_ctrl.tick()) {
//         gpio_hal.toggle(LED_PIN);
//     }
// }
```

The production ISR becomes three lines of code, each of which is either a HAL call (testable via mocking) or a method call on the extracted class. There's nothing to go wrong. Now the interesting logic lives in `BlinkController` where you can drive it directly:

```cpp
// === Tests for extracted logic ===
#include <gtest/gtest.h>

TEST(BlinkControllerTest, TogglesAtPeriod) {
    BlinkController ctrl(100);

    // First 99 ticks: no toggle
    for (int i = 0; i < 99; ++i)
        EXPECT_FALSE(ctrl.tick()) << "Tick " << i;

    // 100th tick: toggle!
    EXPECT_TRUE(ctrl.tick());

    // Counter resets
    EXPECT_EQ(ctrl.count(), 0);
}

TEST(BlinkControllerTest, Period1TogglesEveryTick) {
    BlinkController ctrl(1);
    EXPECT_TRUE(ctrl.tick());
    EXPECT_TRUE(ctrl.tick());
    EXPECT_TRUE(ctrl.tick());
}

TEST(BlinkControllerTest, ResetClearsCounter) {
    BlinkController ctrl(100);
    for (int i = 0; i < 50; ++i) ctrl.tick();
    ctrl.reset();
    EXPECT_EQ(ctrl.count(), 0);
}
```

Notice how the `TogglesAtPeriod` test loops 99 times to verify nothing fires early, then checks that tick 100 fires. That's 100 function calls in a test, but it's the only rigorous way to confirm the timing is right. No shortcuts here.

### Q2: Test ISR-to-main communication (ring buffers, flags)

The standard way for an ISR to send data to the main loop is a lock-free ring buffer. The ISR is the producer, the main loop is the consumer, and neither can block waiting for the other. This is a genuinely tricky concurrent data structure to get right. The memory ordering in `push` and `pop` matters: the `release` on the write end and the `acquire` on the read end establish the happens-before relationship that prevents the consumer from reading data before the producer has finished writing it.

```cpp
#include <gtest/gtest.h>
#include <atomic>
#include <thread>
#include <array>
#include <cstdint>

// === Lock-free ring buffer for ISR-to-main communication ===
template<typename T, size_t N>
class RingBuffer {
    static_assert((N & (N - 1)) == 0, "N must be power of 2");
public:
    bool push(const T& item) {
        auto head = head_.load(std::memory_order_relaxed);
        auto next = (head + 1) & (N - 1);
        if (next == tail_.load(std::memory_order_acquire))
            return false;  // Full
        buffer_[head] = item;
        head_.store(next, std::memory_order_release);
        return true;
    }

    bool pop(T& item) {
        auto tail = tail_.load(std::memory_order_relaxed);
        if (tail == head_.load(std::memory_order_acquire))
            return false;  // Empty
        item = buffer_[tail];
        tail_.store((tail + 1) & (N - 1), std::memory_order_release);
        return true;
    }

    bool empty() const {
        return head_.load(std::memory_order_acquire)
            == tail_.load(std::memory_order_acquire);
    }

    size_t size() const {
        auto h = head_.load(std::memory_order_acquire);
        auto t = tail_.load(std::memory_order_acquire);
        return (h - t) & (N - 1);
    }

private:
    std::array<T, N> buffer_{};
    std::atomic<size_t> head_{0};
    std::atomic<size_t> tail_{0};
};

// === Unit tests ===
TEST(RingBufferTest, EmptyOnCreation) {
    RingBuffer<int, 8> rb;
    EXPECT_TRUE(rb.empty());
    EXPECT_EQ(rb.size(), 0);
}

TEST(RingBufferTest, PushPopRoundtrip) {
    RingBuffer<int, 8> rb;
    EXPECT_TRUE(rb.push(42));
    EXPECT_EQ(rb.size(), 1);

    int val;
    EXPECT_TRUE(rb.pop(val));
    EXPECT_EQ(val, 42);
    EXPECT_TRUE(rb.empty());
}

TEST(RingBufferTest, FullReturnsFalse) {
    RingBuffer<int, 4> rb;  // Capacity = 3 (one slot reserved)
    EXPECT_TRUE(rb.push(1));
    EXPECT_TRUE(rb.push(2));
    EXPECT_TRUE(rb.push(3));
    EXPECT_FALSE(rb.push(4));  // Full!
}

TEST(RingBufferTest, PopEmptyReturnsFalse) {
    RingBuffer<int, 4> rb;
    int val;
    EXPECT_FALSE(rb.pop(val));
}

TEST(RingBufferTest, FIFOOrder) {
    RingBuffer<int, 8> rb;
    rb.push(1); rb.push(2); rb.push(3);
    int val;
    rb.pop(val); EXPECT_EQ(val, 1);
    rb.pop(val); EXPECT_EQ(val, 2);
    rb.pop(val); EXPECT_EQ(val, 3);
}

// === Stress test simulating ISR producer + main consumer ===
TEST(RingBufferTest, ConcurrentProducerConsumer) {
    RingBuffer<uint32_t, 256> rb;
    constexpr uint32_t COUNT = 100000;
    std::atomic<uint32_t> consumed{0};
    std::atomic<bool> done{false};

    // Simulated ISR (producer thread)
    auto producer = std::thread([&] {
        for (uint32_t i = 0; i < COUNT; ++i) {
            while (!rb.push(i))
                std::this_thread::yield();  // Simulate ISR retry
        }
        done.store(true);
    });

    // Main loop (consumer)
    uint32_t expected = 0;
    while (expected < COUNT) {
        uint32_t val;
        if (rb.pop(val)) {
            ASSERT_EQ(val, expected) << "Out-of-order at " << expected;
            expected++;
        }
    }

    producer.join();
    EXPECT_TRUE(rb.empty());
}
```

The `ConcurrentProducerConsumer` test is the important one. It sends 100,000 integers through the buffer across two threads and verifies they arrive in order without any data being lost or reordered. Run this test with ThreadSanitizer enabled to catch memory ordering bugs that might pass on your machine but fail on different hardware.

### Q3: Verify ISR timing constraints on host

You can't measure absolute microsecond timings on the host - that requires a real target and an oscilloscope or cycle counter. But you can measure relative performance and spot algorithmic regressions before they ever reach hardware. The approach here is to extract the ISR hot path into a class, write functional tests to verify correctness, and then add a Google Benchmark microbenchmark to guard against performance regressions.

```cpp
#include <gtest/gtest.h>
#include <benchmark/benchmark.h>
#include <chrono>

// === Measure worst-case execution time of ISR logic ===
// Note: Host timing != target timing, but relative comparisons are valid

class UartRxHandler {
public:
    // This is the hot path called from ISR
    void on_byte_received(uint8_t byte) {
        if (byte == FRAME_START) {
            state_ = State::Header;
            frame_pos_ = 0;
            return;
        }

        switch (state_) {
            case State::Idle:
                break;
            case State::Header:
                if (frame_pos_ < 2) {
                    header_[frame_pos_++] = byte;
                    if (frame_pos_ == 2) {
                        payload_len_ = header_[1];
                        state_ = State::Payload;
                        frame_pos_ = 0;
                    }
                }
                break;
            case State::Payload:
                if (frame_pos_ < payload_len_ && frame_pos_ < MAX_PAYLOAD) {
                    payload_[frame_pos_++] = byte;
                    if (frame_pos_ == payload_len_) {
                        state_ = State::Checksum;
                    }
                }
                break;
            case State::Checksum:
                frame_complete_ = verify_checksum(byte);
                state_ = State::Idle;
                break;
        }
    }

    bool is_frame_complete() const { return frame_complete_; }
    void clear_frame() { frame_complete_ = false; }

private:
    static constexpr uint8_t FRAME_START = 0x7E;
    static constexpr size_t MAX_PAYLOAD = 64;

    enum class State { Idle, Header, Payload, Checksum };
    State state_ = State::Idle;
    uint8_t header_[2]{};
    uint8_t payload_[MAX_PAYLOAD]{};
    uint8_t payload_len_ = 0;
    size_t frame_pos_ = 0;
    bool frame_complete_ = false;

    bool verify_checksum(uint8_t checksum) {
        uint8_t calc = header_[0] ^ header_[1];
        for (size_t i = 0; i < payload_len_; ++i)
            calc ^= payload_[i];
        return calc == checksum;
    }
};

// === Functional test ===
TEST(UartRxHandlerTest, ParsesCompleteFrame) {
    UartRxHandler handler;

    // Frame: START, type=0x01, len=3, payload={0x10, 0x20, 0x30}, checksum
    handler.on_byte_received(0x7E);  // Start
    handler.on_byte_received(0x01);  // Type
    handler.on_byte_received(0x03);  // Length
    handler.on_byte_received(0x10);  // Payload
    handler.on_byte_received(0x20);
    handler.on_byte_received(0x30);
    handler.on_byte_received(0x01 ^ 0x03 ^ 0x10 ^ 0x20 ^ 0x30);  // Checksum

    EXPECT_TRUE(handler.is_frame_complete());
}

TEST(UartRxHandlerTest, RejectsBadChecksum) {
    UartRxHandler handler;
    handler.on_byte_received(0x7E);
    handler.on_byte_received(0x01);
    handler.on_byte_received(0x01);  // 1 byte payload
    handler.on_byte_received(0xAA);
    handler.on_byte_received(0xFF);  // Wrong checksum

    EXPECT_FALSE(handler.is_frame_complete());
}

// === Timing benchmark (use Google Benchmark) ===
static void BM_UartRxPerByte(benchmark::State& state) {
    UartRxHandler handler;
    uint8_t byte = 0x42;

    for (auto _ : state) {
        handler.on_byte_received(byte);
        benchmark::DoNotOptimize(handler);
    }
}
BENCHMARK(BM_UartRxPerByte);
```

The `BM_UartRxPerByte` benchmark measures how long a single call to `on_byte_received` takes in the steady state. If someone adds a loop, a dynamic allocation, or an expensive string operation to this function in the future, the benchmark will catch the regression on the next CI run - long before the code reaches hardware.

---

## Notes

- Extract, don't mock ISRs - move logic into plain C++ classes and call them from thin ISR wrappers. The wrappers become too trivial to break.
- ISR logic must be O(1) or bounded - no dynamic allocation, no unbounded loops, no blocking primitives of any kind.
- Ring buffers are the standard ISR-to-main communication primitive - test them thoroughly, including the full-buffer and empty-buffer edge cases.
- Use ThreadSanitizer on producer-consumer tests to verify lock-free correctness - data races in ring buffer code can be invisible without it.
- Host benchmarks show relative timing; always use real hardware profiling for absolute microsecond measurements before signing off on real-time budgets.
- Verify statically that ISR handlers never call blocking functions - static analysis tools can check this.
- Power-of-2 buffer sizes enable fast modulo via bitwise AND, which matters when `pop` and `push` are called from interrupt context.
