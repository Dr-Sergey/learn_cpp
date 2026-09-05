# Develop with -fno-exceptions and -fno-rtti Constraints

**Category:** Embedded and Constrained Systems  
**Standard:** C++17, C++23  
**Reference:** https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html  

---

## Topic Overview

In embedded and safety-critical systems, compiling with `-fno-exceptions` and `-fno-rtti` is standard practice. Exceptions introduce hidden control flow, non-deterministic unwinding costs, and significant binary size overhead from unwind tables (`.eh_frame`, `.gcc_except_table`). RTTI adds `type_info` objects, vtable pointers to `type_info`, and the `dynamic_cast` / `typeid` machinery - all of which bloat ROM and RAM.

Disabling these features is not merely a build flag; it fundamentally changes how you write C++. Any code that uses `throw`, `try`/`catch`, `dynamic_cast`, or `typeid` will fail to compile. Standard library facilities that rely on exceptions (`std::vector::at()`, `std::stoi()`, `std::any`) become unavailable or must be replaced. This forces a discipline of explicit error handling that, when done well, produces more predictable and auditable code.

| Feature Disabled | Compiler Flag | Binary Size Impact | What Breaks |
| --- | --- | --- | --- |
| Exceptions | `-fno-exceptions` | 10-30% ROM reduction | `throw`, `try`/`catch`, `.at()`, `std::stoi`, `std::any` |
| RTTI | `-fno-rtti` | 5-15% ROM reduction | `dynamic_cast`, `typeid`, `std::any`, `std::function` (some impl) |
| Both combined | Both flags | 15-40% ROM reduction | All of the above; must audit all dependencies |

Here is what the binary size difference looks like in practice on a typical Cortex-M4 project. The unwind tables and exception tables disappear entirely, and the code section shrinks because the compiler no longer needs to generate cleanup paths.

```text
Binary Layout Comparison (typical Cortex-M4 firmware):

With exceptions + RTTI:          Without:
+--------------------+           +--------------------+
| .text (code)  120K |           | .text (code)   95K |
| .rodata        25K |           | .rodata        18K |
| .eh_frame      18K |           | (removed)       0K |
| .gcc_except     8K |           | (removed)       0K |
| .data           4K |           | .data           4K |
| .bss           12K |           | .bss           12K |
+--------------------+           +--------------------+
| TOTAL:        187K |           | TOTAL:        129K |
+--------------------+           +--------------------+
                                   ~31% smaller
```

The C++23 `<expected>` header provides the standard replacement for exception-based error propagation. For C++17 targets, libraries like `tl::expected` or Boost.Outcome fill the gap. The key insight is that `-fno-exceptions` does **not** prevent you from using most of modern C++ - it requires you to be deliberate about error paths.

---

## Self-Assessment

### Q1: Implement a complete error-handling framework suitable for `-fno-exceptions` firmware, using `std::expected` (C++23) with monadic chaining

The biggest mental shift when moving to `-fno-exceptions` is that every function that can fail must say so in its return type. `std::expected<T, E>` is the standard tool for this: it either holds a `T` (success) or an `E` (error), and callers are forced to inspect which they got. The `and_then` method lets you chain operations together - if any step returns an error, the chain short-circuits and propagates it, giving you something close to Rust's `?` operator.

```cpp
// error_framework.h - No-exceptions error handling
#pragma once

#include <cstdint>
#include <expected>    // C++23 freestanding
#include <type_traits>
#include <optional>

namespace fw {

// Unified error code for the entire firmware
enum class Error : std::uint8_t {
    Ok              = 0,
    Timeout         = 1,
    InvalidParam    = 2,
    HardwareFault   = 3,
    BufferFull      = 4,
    NotInitialized  = 5,
    ChecksumFail    = 6,
    OutOfMemory     = 7,
};

// Result alias - the standard pattern
template <typename T>
using Result = std::expected<T, Error>;

using VoidResult = std::expected<void, Error>;

// Monadic error propagation macro (like Rust's `?` operator)
// Use when std::expected::and_then() is too verbose
#define TRY(expr)                                      \
    ({                                                 \
        auto _result = (expr);                         \
        if (!_result) return std::unexpected(_result.error()); \
        std::move(*_result);                           \
    })

// For void results
#define TRY_VOID(expr)                                 \
    do {                                               \
        auto _result = (expr);                         \
        if (!_result) return std::unexpected(_result.error()); \
    } while(0)


// ---------- Example sensor driver ----------

struct SensorConfig {
    std::uint8_t  i2c_addr;
    std::uint16_t sample_rate_hz;
    std::uint8_t  oversample;
};

struct RawSample {
    std::int32_t  adc_counts;
    std::uint32_t timestamp_us;
};

struct CalibratedSample {
    float    value;
    float    uncertainty;
    uint32_t timestamp_us;
};

// Each function returns Result<T> instead of throwing
inline Result<SensorConfig> validate_config(SensorConfig cfg) {
    if (cfg.i2c_addr == 0 || cfg.i2c_addr > 0x77)
        return std::unexpected(Error::InvalidParam);
    if (cfg.sample_rate_hz == 0 || cfg.sample_rate_hz > 10000)
        return std::unexpected(Error::InvalidParam);
    return cfg;
}

inline VoidResult init_sensor(const SensorConfig& cfg) {
    // Hypothetical HW write
    volatile auto* ctrl = reinterpret_cast<volatile std::uint32_t*>(0x4002'0000);
    *ctrl = (static_cast<std::uint32_t>(cfg.i2c_addr) << 16) |
             cfg.sample_rate_hz;
    // Check HW acknowledged
    if ((*ctrl & 0x8000'0000u) == 0)
        return std::unexpected(Error::HardwareFault);
    return {};
}

inline Result<RawSample> read_raw() {
    volatile auto* data = reinterpret_cast<volatile std::uint32_t*>(0x4002'0004);
    volatile auto* status = reinterpret_cast<volatile std::uint32_t*>(0x4002'0008);

    for (int i = 0; i < 1000; ++i) {
        if (*status & 1u) {
            return RawSample{
                .adc_counts   = static_cast<std::int32_t>(*data),
                .timestamp_us = *reinterpret_cast<volatile std::uint32_t*>(0xE000'E018)
            };
        }
    }
    return std::unexpected(Error::Timeout);
}

inline Result<CalibratedSample> calibrate(RawSample raw) {
    constexpr float SCALE  = 0.00125f;
    constexpr float OFFSET = -40.0f;
    return CalibratedSample{
        .value        = static_cast<float>(raw.adc_counts) * SCALE + OFFSET,
        .uncertainty  = 0.5f,
        .timestamp_us = raw.timestamp_us
    };
}

// Monadic chaining - the whole pipeline with no exceptions
inline Result<CalibratedSample> get_calibrated_reading(SensorConfig cfg) {
    return validate_config(cfg)
        .and_then([](SensorConfig c) -> Result<SensorConfig> {
            auto r = init_sensor(c);
            if (!r) return std::unexpected(r.error());
            return c;
        })
        .and_then([](SensorConfig) { return read_raw(); })
        .and_then([](RawSample raw) { return calibrate(raw); });
}

} // namespace fw
```

The `get_calibrated_reading` function at the bottom shows the payoff. If `validate_config` fails, the whole chain returns immediately with that error - `init_sensor`, `read_raw`, and `calibrate` never run. Each step is type-checked, each error is explicit in the return type, and none of this requires exceptions.

### Q2: Show how to replace `dynamic_cast` with a compile-time safe alternative under `-fno-rtti`, using CRTP-based type tagging

`dynamic_cast` requires RTTI to work - it queries the `type_info` object attached to each polymorphic class at runtime. Under `-fno-rtti` it simply won't compile. The replacement is to embed a type tag manually: each concrete class gets a unique `TypeId` value stored in the base class, and the `as<Derived>()` template checks that tag before doing a `static_cast`. The check is a single integer comparison - cheaper than `dynamic_cast` in almost every case.

```cpp
// type_tag.h - RTTI replacement for embedded polymorphism
#pragma once

#include <cstdint>
#include <type_traits>

namespace fw {

// Type ID registry - each concrete type gets a unique constexpr ID
enum class TypeId : std::uint16_t {
    Unknown     = 0,
    UartDriver  = 1,
    SpiDriver   = 2,
    I2cDriver   = 3,
    AdcDriver   = 4,
};

// Base interface with embedded type tag
class DriverBase {
public:
    constexpr explicit DriverBase(TypeId id) : type_id_(id) {}
    virtual ~DriverBase() = default;

    // Replacement for dynamic_cast - no RTTI needed
    [[nodiscard]] constexpr TypeId type_id() const { return type_id_; }

    template <typename Derived>
    [[nodiscard]] Derived* as() {
        static_assert(std::is_base_of_v<DriverBase, Derived>,
                      "Derived must inherit from DriverBase");
        if (type_id_ == Derived::static_type_id) {
            return static_cast<Derived*>(this);
        }
        return nullptr;
    }

    template <typename Derived>
    [[nodiscard]] const Derived* as() const {
        static_assert(std::is_base_of_v<DriverBase, Derived>,
                      "Derived must inherit from DriverBase");
        if (type_id_ == Derived::static_type_id) {
            return static_cast<const Derived*>(this);
        }
        return nullptr;
    }

    virtual void init()  = 0;
    virtual void reset() = 0;

private:
    TypeId type_id_;
};

// CRTP mixin that injects the type tag automatically
template <typename Derived, TypeId Id>
class DriverCRTP : public DriverBase {
public:
    static constexpr TypeId static_type_id = Id;
    constexpr DriverCRTP() : DriverBase(Id) {}
};

// ---------- Concrete drivers ----------

class UartDriver : public DriverCRTP<UartDriver, TypeId::UartDriver> {
    std::uint32_t baud_;
public:
    explicit UartDriver(std::uint32_t baud) : baud_(baud) {}
    void init()  override { /* configure UART registers */ }
    void reset() override { /* reset peripheral */ }
    [[nodiscard]] std::uint32_t baud() const { return baud_; }
};

class SpiDriver : public DriverCRTP<SpiDriver, TypeId::SpiDriver> {
    std::uint32_t clock_hz_;
public:
    explicit SpiDriver(std::uint32_t clk) : clock_hz_(clk) {}
    void init()  override { /* configure SPI */ }
    void reset() override { /* reset SPI */ }
    [[nodiscard]] std::uint32_t clock() const { return clock_hz_; }
};

// ---------- Usage without dynamic_cast or typeid ----------

inline void configure_driver(DriverBase* drv) {
    // Safe downcast - equivalent to dynamic_cast but no RTTI
    if (auto* uart = drv->as<UartDriver>()) {
        // UART-specific configuration
        if (uart->baud() > 115200) {
            // Enable DMA for high baud rates
        }
    } else if (auto* spi = drv->as<SpiDriver>()) {
        // SPI-specific configuration
        if (spi->clock() > 10'000'000) {
            // Enable high-speed mode
        }
    }
}

} // namespace fw
```

The CRTP mixin `DriverCRTP` is the piece that makes this ergonomic. Every new driver type inherits from it with its own `TypeId`, and the `static_type_id` constant is automatically available for the `as<>()` check. You don't have to remember to call a registration function or initialize anything - the type tag is set in the constructor and it's always correct.

### Q3: Measure and compare binary sizes with and without exceptions/RTTI using a build script, and show how to audit third-party libraries for hidden exception usage

Even after you compile your own code with `-fno-exceptions`, a third-party library you link against might quietly pull exception machinery back in. The audit approach below works at two levels: compile-time poisoning (replace the ABI symbols with traps so any TU that references them causes a link error) and post-build inspection (scan object files with `nm` to find the offending symbols).

```cpp
// audit_exceptions.cpp - Compile-time and link-time detection
// of hidden exception/RTTI usage in dependencies

// 1. Compile-time: poison the exception ABI symbols
// Place this in a "poison" header included early via -include flag

#ifdef NO_EXCEPTIONS_AUDIT

// If any TU pulls in exception machinery, these will cause
// multiply-defined symbol errors at link time
extern "C" {
    // GCC/Clang personality routine for C++ exceptions
    [[gnu::used]] void __gxx_personality_v0() {
        __builtin_trap();  // Should never be called
    }
    // Unwinder
    [[gnu::used]] void _Unwind_Resume() {
        __builtin_trap();
    }
    // operator new throwing version - replace with trap
    // (keeps non-throwing placement new available)
}

namespace __cxxabiv1 {
    // RTTI support functions - poison them
    [[gnu::used]] void __dynamic_cast() { __builtin_trap(); }
}

#endif // NO_EXCEPTIONS_AUDIT

// 2. Runtime: assert that exception tables are absent
#include <cstdint>
#include <cstddef>

namespace audit {

// Linker symbols - defined in linker script
extern "C" {
    extern std::uint8_t __eh_frame_start[];
    extern std::uint8_t __eh_frame_end[];
    extern std::uint8_t __gcc_except_table_start[];
    extern std::uint8_t __gcc_except_table_end[];
}

struct BinaryAudit {
    std::size_t eh_frame_size;
    std::size_t except_table_size;
    std::size_t text_size;
    std::size_t total_flash;
    float       overhead_pct;
};

inline BinaryAudit check_exception_overhead() {
    auto eh_sz  = static_cast<std::size_t>(
        __eh_frame_end - __eh_frame_start);
    auto exc_sz = static_cast<std::size_t>(
        __gcc_except_table_end - __gcc_except_table_start);

    extern std::uint8_t _stext[], _etext[];
    auto text_sz = static_cast<std::size_t>(_etext - _stext);
    auto total   = text_sz + eh_sz + exc_sz;

    return BinaryAudit{
        .eh_frame_size      = eh_sz,
        .except_table_size  = exc_sz,
        .text_size          = text_sz,
        .total_flash        = total,
        .overhead_pct       = static_cast<float>(eh_sz + exc_sz)
                              / static_cast<float>(total) * 100.0f
    };
}

} // namespace audit
```

The shell script below ties it together - it builds four variants and reports section sizes side-by-side, then scans all object files for the telltale ABI symbols that indicate hidden exception use.

```bash
#!/bin/bash
# measure_binary.sh - Compare binary sizes with/without exceptions and RTTI

PROJECT_DIR="$(dirname "$0")"
SRC="$PROJECT_DIR/main.cpp"
ARM_CXX="arm-none-eabi-g++"
COMMON="-std=c++23 -mcpu=cortex-m4 -mthumb -Os -ffunction-sections \
        -fdata-sections -Wl,--gc-sections -ffreestanding -nostdlib"

echo "=== Binary Size Comparison ==="
echo ""

# Build 4 variants
$ARM_CXX $COMMON                                    -o /tmp/fw_full.elf  "$SRC" 2>/dev/null
$ARM_CXX $COMMON -fno-exceptions                    -o /tmp/fw_noex.elf  "$SRC" 2>/dev/null
$ARM_CXX $COMMON                 -fno-rtti          -o /tmp/fw_nortti.elf "$SRC" 2>/dev/null
$ARM_CXX $COMMON -fno-exceptions -fno-rtti          -o /tmp/fw_bare.elf  "$SRC" 2>/dev/null

echo "Variant               .text   .rodata  .eh_frame  .data  .bss   TOTAL"
echo "----------------------------------------------------------------------"

for elf in /tmp/fw_full.elf /tmp/fw_noex.elf /tmp/fw_nortti.elf /tmp/fw_bare.elf; do
    name=$(basename "$elf" .elf)
    arm-none-eabi-size -A "$elf" | awk -v n="$name" '
    BEGIN { text=0; ro=0; eh=0; data=0; bss=0 }
    /\.text/       { text = $2 }
    /\.rodata/     { ro   = $2 }
    /\.eh_frame/   { eh   = $2 }
    /\.data/       { data = $2 }
    /\.bss/        { bss  = $2 }
    END {
        total = text + ro + eh + data + bss
        printf "%-22s %6d  %7d  %9d  %5d  %4d  %6d\n",
               n, text, ro, eh, data, bss, total
    }'
done

# Check for hidden exception usage in object files
echo ""
echo "=== Scanning for hidden exception symbols ==="
for obj in "$PROJECT_DIR"/build/*.o; do
    syms=$(arm-none-eabi-nm "$obj" 2>/dev/null | grep -E '__gxx_personality|_Unwind_Resume|__cxa_throw|__cxa_begin_catch')
    if [ -n "$syms" ]; then
        echo "WARNING: $obj references exception machinery:"
        echo "$syms"
    fi
done
```

---

## Notes

- `-fno-exceptions` causes `throw` expressions to call `std::abort()` in GCC/Clang (or `std::terminate()` depending on the implementation). Never rely on this as an error handling path.
- `std::function` in libstdc++ uses RTTI internally. Under `-fno-rtti`, use function pointers, `etl::delegate`, or a hand-rolled type-erased callable.
- `std::any` requires both RTTI and exceptions - it is completely unusable under these flags. Use `std::variant` instead.
- When using third-party libraries (FreeRTOS C++ wrappers, ETL, etc.), always verify they build cleanly with both flags. Run `nm -C library.a | grep __cxa_throw` to detect hidden exception usage.
- The `noexcept` specifier still compiles under `-fno-exceptions` and is useful for optimizer hints and `std::is_nothrow_*` trait checks.
- AUTOSAR C++14 Rule A15-0-1 mandates `-fno-exceptions` for all safety-critical code (ASIL-B and above).
- Binary size reduction from disabling exceptions varies dramatically by codebase: template-heavy code sees 20-30% reduction; C-like code sees 5-10%.
