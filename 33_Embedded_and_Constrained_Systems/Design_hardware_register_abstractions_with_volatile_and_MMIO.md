# Design Hardware Register Abstractions with volatile and MMIO

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++20  
**Reference:** https://en.cppreference.com/w/cpp/language/cv  

---

## Topic Overview

Memory-Mapped I/O (MMIO) is the universal mechanism for CPU-peripheral communication on embedded systems. Hardware registers appear at fixed physical addresses, and reads/writes to these addresses trigger side effects in the peripheral hardware. C++ must express this through `volatile` qualified accesses, but raw `volatile` pointers are error-prone - wrong offsets, incorrect bit widths, and missing read-modify-write sequences cause subtle, hard-to-debug hardware glitches.

A well-designed register abstraction layer (RAL) provides: (1) type-safe access to named registers at compile-time-verified offsets, (2) bit-field manipulation through named constants instead of magic numbers, (3) enforcement of access policies (read-only, write-only, read-write, write-1-to-clear), and (4) zero overhead - the abstraction must compile down to the same single `LDR`/`STR` instructions as hand-written C.

| Access Type     | Read Behavior            | Write Behavior             | Example                |
| --- | --- | --- | --- |
| Read-Write (RW) | Returns current value    | Sets value                 | GPIO ODR               |
| Read-Only (RO)  | Returns current value    | Compile error              | GPIO IDR               |
| Write-Only (WO) | Compile error / UB       | Sets value                 | USART TDR              |
| Write-1-Clear   | Returns set bits         | Writing 1 clears the bit   | Interrupt status flags |
| Read-Clear (RC) | Returns & clears value   | No effect or compile error | Some status registers  |

Here is a typical GPIO peripheral register block layout. Each register lives at a known byte offset from the peripheral base address, and each has a specific access policy - IDR (input data register) is read-only, BSRR (bit set/reset register) is write-only.

```text
  Peripheral Register Block (e.g., GPIO):
  Base: 0x4002_0000
  +--------------+--------+--------+
  | Register     | Offset | Access |
  +--------------+--------+--------+
  | MODER        | 0x00   | RW     |
  | OTYPER       | 0x04   | RW     |
  | OSPEEDR      | 0x08   | RW     |
  | PUPDR        | 0x0C   | RW     |
  | IDR          | 0x10   | RO     |
  | ODR          | 0x14   | RW     |
  | BSRR         | 0x18   | WO     |
  | LCKR         | 0x1C   | RW     |
  +--------------+--------+--------+
```

The key C++ insight: `volatile` only prevents the compiler from eliding or reordering accesses to that specific object - it does NOT provide atomicity, memory ordering between volatile and non-volatile accesses, or thread safety. For multi-core MMIO, explicit memory barriers (`__DMB()`, `std::atomic_signal_fence`) are also required.

---

## Self-Assessment

### Q1: How do you build a zero-cost, type-safe register access layer that enforces read/write policies at compile time

The policy tags below are just empty structs - they carry no data and cost nothing at runtime. Their entire job is to be different types so that `static_assert` and `if constexpr` can gate which operations are legal. If you call `IDR::write(0)`, you get a compile error. There's no runtime check, no overhead - the policy enforcement is entirely in the type system.

```cpp
#include <cstdint>
#include <type_traits>

// Access policy tags - used for SFINAE/static_assert gating
struct ReadWrite  {};
struct ReadOnly   {};
struct WriteOnly  {};
struct Write1Clear{};

// Core register accessor: compiles to a single LDR/STR
template <std::uintptr_t Address, typename Policy, typename T = std::uint32_t>
class Register {
    static_assert(std::is_unsigned_v<T>, "Register storage must be unsigned");

    static volatile T& ref() noexcept {
        return *reinterpret_cast<volatile T*>(Address);
    }

public:
    // Read: allowed for RW, RO, W1C; blocked for WO
    static T read() noexcept {
        static_assert(!std::is_same_v<Policy, WriteOnly>,
                      "Cannot read a write-only register");
        return ref();
    }

    // Write: allowed for RW, WO; blocked for RO
    static void write(T value) noexcept {
        static_assert(!std::is_same_v<Policy, ReadOnly>,
                      "Cannot write a read-only register");
        ref() = value;
    }

    // Read-modify-write: only for RW
    static void modify(T clear_mask, T set_mask) noexcept {
        static_assert(std::is_same_v<Policy, ReadWrite>,
                      "Read-modify-write only on RW registers");
        T val = ref();
        val &= ~clear_mask;
        val |= set_mask;
        ref() = val;
    }

    // Set bits: for RW registers (read-modify-write) or W1C (direct write)
    static void set_bits(T mask) noexcept {
        if constexpr (std::is_same_v<Policy, Write1Clear>) {
            ref() = mask;  // writing 1 clears the bit
        } else {
            static_assert(std::is_same_v<Policy, ReadWrite>,
                          "set_bits only valid for RW or W1C");
            ref() |= mask;
        }
    }

    // Clear bits: only for RW
    static void clear_bits(T mask) noexcept {
        static_assert(std::is_same_v<Policy, ReadWrite>,
                      "clear_bits only valid for RW registers");
        ref() &= ~mask;
    }
};

// GPIO Port A register definitions for STM32
namespace gpioa {
    constexpr std::uintptr_t BASE = 0x40020000;

    using MODER   = Register<BASE + 0x00, ReadWrite>;
    using OTYPER  = Register<BASE + 0x04, ReadWrite>;
    using OSPEEDR = Register<BASE + 0x08, ReadWrite>;
    using PUPDR   = Register<BASE + 0x0C, ReadWrite>;
    using IDR     = Register<BASE + 0x10, ReadOnly>;
    using ODR     = Register<BASE + 0x14, ReadWrite>;
    using BSRR    = Register<BASE + 0x18, WriteOnly>;
}

// Usage - compiler enforces access policies
void configure_pa5_output() noexcept {
    // Set PA5 to general-purpose output mode
    gpioa::MODER::modify(0x3u << 10, 0x1u << 10);  // clear 2 bits, set to 01

    // Toggle PA5 via BSRR (write-only: set bit 5)
    gpioa::BSRR::write(1u << 5);      // set PA5
    gpioa::BSRR::write(1u << 21);     // reset PA5 (upper 16 bits)

    // Read input - IDR is read-only
    std::uint32_t val = gpioa::IDR::read();
    (void)val;

    // gpioa::IDR::write(0);    // COMPILE ERROR: cannot write read-only register
    // gpioa::BSRR::read();     // COMPILE ERROR: cannot read write-only register
}
```

The `using` aliases in `namespace gpioa` are the real win here - they give human-readable names to what would otherwise be `*reinterpret_cast<volatile uint32_t*>(0x40020010)`. When you read `gpioa::IDR::read()`, it's immediately clear what's happening. And you can't accidentally call `gpioa::IDR::write()` without getting a compile error.

### Q2: How do you create type-safe bit-field abstractions that name individual fields and prevent cross-register mistakes

The `BitField` template below encodes a field's position and width entirely at compile time. `extract` pulls a field out of a raw register value, and `encode` shifts a value into position ready for a write. The `RegModifier` class then lets you chain multiple field changes and flush them all in a single read-modify-write cycle - important for peripherals where writing intermediate states can cause glitches.

```cpp
#include <cstdint>

// Bit-field descriptor: encodes position and width at compile time
template <unsigned Offset, unsigned Width>
struct BitField {
    static_assert(Offset + Width <= 32, "Field exceeds register width");
    static constexpr std::uint32_t mask = ((1u << Width) - 1u) << Offset;
    static constexpr unsigned offset = Offset;
    static constexpr unsigned width  = Width;

    // Extract field value from a raw register value
    static constexpr std::uint32_t extract(std::uint32_t reg) noexcept {
        return (reg >> Offset) & ((1u << Width) - 1u);
    }

    // Create field value shifted into position
    static constexpr std::uint32_t encode(std::uint32_t value) noexcept {
        return (value & ((1u << Width) - 1u)) << Offset;
    }
};

// Named fields for USART Control Register 1 (STM32)
namespace usart1_cr1 {
    constexpr std::uintptr_t ADDR = 0x40011000;  // USART1_CR1

    using UE    = BitField<0,  1>;   // USART enable
    using RE    = BitField<2,  1>;   // Receiver enable
    using TE    = BitField<3,  1>;   // Transmitter enable
    using IDLEIE= BitField<4,  1>;   // Idle interrupt enable
    using RXNEIE= BitField<5,  1>;   // RX not empty interrupt enable
    using TCIE  = BitField<6,  1>;   // Transmission complete IE
    using TXEIE = BitField<7,  1>;   // TX empty interrupt enable
    using M0    = BitField<12, 1>;   // Word length bit 0
    using OVER8 = BitField<15, 1>;   // Oversampling mode
    using BRR_FIELD = BitField<0, 16>;  // For BRR register
}

// Fluent register modifier - accumulates changes then writes once
template <std::uintptr_t Address>
class RegModifier {
    std::uint32_t clear_mask_ = 0;
    std::uint32_t set_mask_   = 0;

public:
    template <typename Field>
    RegModifier& set(std::uint32_t value = 1) noexcept {
        clear_mask_ |= Field::mask;
        set_mask_   |= Field::encode(value);
        return *this;
    }

    template <typename Field>
    RegModifier& clear() noexcept {
        clear_mask_ |= Field::mask;
        // set_mask_ bit stays 0
        return *this;
    }

    void apply() noexcept {
        auto& reg = *reinterpret_cast<volatile std::uint32_t*>(Address);
        std::uint32_t val = reg;
        val &= ~clear_mask_;
        val |= set_mask_;
        reg = val;
    }
};

// Usage: configure USART1 - single read-modify-write
void usart1_init() noexcept {
    RegModifier<usart1_cr1::ADDR>{}
        .set<usart1_cr1::UE>()          // enable USART
        .set<usart1_cr1::TE>()          // enable transmitter
        .set<usart1_cr1::RE>()          // enable receiver
        .set<usart1_cr1::OVER8>(0)      // 16x oversampling
        .clear<usart1_cr1::M0>()        // 8-bit word length
        .apply();                        // single RMW cycle
}
```

The fluent `.set().set().clear().apply()` chain reads almost like a hardware reference manual description, and it generates a single read-modify-write sequence at runtime. That single-write property matters for peripherals whose behavior is undefined if you write partial configuration states.

### Q3: How do you prevent reordering of MMIO accesses across memory barriers and between volatile accesses

This is where `volatile` falls short. It prevents the compiler from optimizing away a specific access, but the C++ standard says that the ordering of volatile accesses relative to each other is implementation-defined, and hardware on weakly-ordered architectures (ARM Cortex-A, Cortex-R) can reorder them further. The concrete danger: if you write the DMA source address, destination, and length, then enable the DMA channel, the hardware must see those writes in that order. Without barriers, you might enable DMA before the address registers have been written.

```cpp
#include <cstdint>
#include <atomic>

// The problem: volatile prevents compiler elision of *that specific access*,
// but the compiler may reorder volatile accesses relative to each other
// and relative to non-volatile accesses (implementation-defined in C++).
// Hardware may also reorder on weakly-ordered architectures.

namespace mmio_barriers {

    // ARM barrier intrinsics
    inline void dmb() noexcept { asm volatile("dmb sy" ::: "memory"); }
    inline void dsb() noexcept { asm volatile("dsb sy" ::: "memory"); }
    inline void isb() noexcept { asm volatile("isb"    ::: "memory"); }

    // Compiler-only fence (no hardware barrier)
    inline void compiler_fence() noexcept {
        asm volatile("" ::: "memory");
    }

    // Safe MMIO write with ordering guarantees
    inline void mmio_write32(std::uintptr_t addr, std::uint32_t value) noexcept {
        *reinterpret_cast<volatile std::uint32_t*>(addr) = value;
    }

    inline std::uint32_t mmio_read32(std::uintptr_t addr) noexcept {
        return *reinterpret_cast<volatile std::uint32_t*>(addr);
    }
}

// DMA transfer setup: ordering is CRITICAL
// The DMA controller reads configuration registers - if the enable bit
// is written before the address/length, DMA fires with stale config.
namespace dma_example {
    constexpr std::uintptr_t DMA_SRC   = 0x40026010;
    constexpr std::uintptr_t DMA_DST   = 0x40026014;
    constexpr std::uintptr_t DMA_LEN   = 0x40026018;
    constexpr std::uintptr_t DMA_CTRL  = 0x4002601C;

    void start_dma_transfer(std::uintptr_t src, std::uintptr_t dst,
                            std::uint32_t length) noexcept {
        using namespace mmio_barriers;

        // Step 1: Configure source, destination, length
        mmio_write32(DMA_SRC, static_cast<std::uint32_t>(src));
        mmio_write32(DMA_DST, static_cast<std::uint32_t>(dst));
        mmio_write32(DMA_LEN, length);

        // Step 2: Ensure all config writes reach the peripheral
        // before enabling the DMA channel
        dsb();  // Data Synchronization Barrier: completes all pending writes

        // Step 3: Enable DMA channel
        mmio_write32(DMA_CTRL, 0x1u);  // enable bit

        // Step 4: ISB if we need subsequent instructions to see DMA active
        dsb();
    }

    // After DMA completion (interrupt or polling):
    void process_dma_result(volatile std::uint8_t* dest_buffer,
                            std::uint32_t length) noexcept {
        using namespace mmio_barriers;

        // After DMA writes to memory, CPU cache may hold stale data
        // DSB ensures DMA writes are visible, then invalidate cache if needed
        dsb();

        // Now safe to read dest_buffer
        std::uint32_t checksum = 0;
        for (std::uint32_t i = 0; i < length; ++i) {
            checksum += dest_buffer[i];
        }
        (void)checksum;
    }
}
```

The `dsb()` between writing the DMA configuration and writing the enable bit is non-negotiable. Without it, the DMA engine might read its control registers before your writes have propagated through the bus fabric. The table below summarizes which barrier to use in each situation:

| Barrier | Effect                                              | Use Case              |
|---------|-----------------------------------------------------|-----------------------|
| DMB     | Orders memory accesses (but not instruction fetch)  | Between MMIO writes   |
| DSB     | Completes all pending memory accesses               | Before DMA enable     |
| ISB     | Flushes pipeline, refetches instructions            | After MPU/SCB change  |
| `"" ::: "memory"` | Compiler fence only, no hardware barrier | Prevent reordering    |

---

## Notes

- `volatile` in C++ guarantees the compiler emits the access; it does NOT guarantee ordering between different volatile objects on weakly-ordered hardware.
- Always use DSB between writing peripheral configuration and writing the enable/trigger bit.
- ISB is required after modifying the MPU, SCB, or any register that affects instruction fetch behavior.
- `std::atomic_signal_fence(std::memory_order_seq_cst)` is a portable compiler fence but does NOT emit hardware barriers.
- Bit-field structs (`struct { uint32_t x : 3; }`) for MMIO are non-portable - compilers disagree on bit ordering, padding, and whether the access is 8/16/32-bit.
- Prefer explicit mask-and-shift over C bit-fields for register access; the generated code is identical but portable.
- For multi-core systems (e.g., Cortex-A with MMIO), use DMB between core-to-peripheral and core-to-core shared memory accesses.
