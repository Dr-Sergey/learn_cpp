# Understand Freestanding vs Hosted C++ Implementations

**Category:** Embedded and Constrained Systems  
**Standard:** C++23 (P1642R11, P2338R4)  
**Reference:** <https://en.cppreference.com/w/cpp/freestanding>  

---

## Topic Overview

The C++ standard defines two kinds of implementations: **hosted** and **freestanding**. A hosted implementation provides the full standard library with OS-level services - file I/O, threading, networking, dynamic memory. A freestanding implementation provides only a minimal subset of headers and makes no assumptions about an underlying operating system. Every embedded firmware, kernel module, bootloader, or hypervisor targets a freestanding environment at some level.

Understanding the boundary matters because code that accidentally depends on hosted facilities will fail to link or behave unpredictably on bare-metal targets. Senior engineers must know exactly which headers and utilities are available freestanding, and how to structure libraries that work across both worlds.

| Aspect | Hosted | Freestanding |
| --- | --- | --- |
| OS required | Yes | No |
| `main()` signature | `int main()` / `int main(int, char**)` | Implementation-defined entry point |
| Heap (`new`/`delete`) | Available | Not guaranteed |
| Thread support | `<thread>`, `<mutex>`, etc. | Not available |
| I/O streams | `<iostream>`, `<fstream>` | Not available |
| Program startup | Runtime calls `main()` after full init | Platform-specific (reset vector, `_start`) |
| Exceptions/RTTI | Available by default | Often disabled (`-fno-exceptions`) |

C++23 significantly expanded the freestanding header set via **P1642R11** and **P2338R4**. Previously freestanding was limited to `<cstddef>`, `<cstdlib>` (partial), `<cstdint>`, `<type_traits>`, `<limits>`, `<new>`, `<initializer_list>`, and a few more. C++23 adds freestanding-compatible portions of `<optional>`, `<variant>`, `<expected>`, `<string_view>`, `<span>`, `<array>`, `<ranges>`, `<functional>`, `<tuple>`, `<utility>`, and `<bit>`. This is a big deal - it means you can write expressive, modern C++ on bare metal without pulling in any OS-dependent headers.

Here is how to visualize the spectrum from fully freestanding to fully hosted:

```cpp
┌──────────────────────────────────────────────────────┐
│              C++ Implementation Spectrum              │
├──────────────┬───────────────────┬───────────────────┤
│  Freestanding│   Partial Hosted  │   Fully Hosted    │
│  (bare-metal)│ (RTOS, minimal OS)│ (Linux, Windows)  │
├──────────────┼───────────────────┼───────────────────┤
│ <cstdint>    │ + <vector>        │ + <iostream>      │
│ <type_traits>│ + <string>        │ + <filesystem>    │
│ <atomic>     │ + <algorithm>     │ + <thread>        │
│ <optional>   │ + custom allocator│ + <fstream>       │
│ <span>       │                   │ + <regex>         │
│ <expected>   │                   │ + <networking>    │
└──────────────┴───────────────────┴───────────────────┘
```

---

## Self-Assessment

### Q1: Write a compile-time check that detects whether you are in a freestanding environment, and conditionally provide a logging facility that works in both modes

The key macro here is `__STDC_HOSTED__` - it is 1 for hosted, 0 for freestanding, and it is guaranteed by both the C and C++ standards. You can use it to gate any hosted-only code at the preprocessor level.

```cpp
// platform_log.h - Works in both freestanding and hosted
#pragma once

#include <cstdint>
#include <type_traits>

// __STDC_HOSTED__ is 1 for hosted, 0 for freestanding
#if __STDC_HOSTED__
  #include <cstdio>
#endif

namespace platform {

// Hardware UART register (example: ARM UART0 at 0x4000'C000)
inline volatile std::uint32_t* const UART0_DR =
    reinterpret_cast<volatile std::uint32_t*>(0x4000'C000);

inline void put_char_hw(char c) {
    *UART0_DR = static_cast<std::uint32_t>(c);
}

inline void log_string(const char* msg) {
#if __STDC_HOSTED__
    // Hosted: use stdio
    std::fputs(msg, stderr);
    std::fputc('\n', stderr);
#else
    // Freestanding: bitbang to UART
    while (*msg) {
        put_char_hw(*msg++);
    }
    put_char_hw('\r');
    put_char_hw('\n');
#endif
}

// Type-safe severity using enum class (works freestanding)
enum class Severity : std::uint8_t { Debug, Info, Warn, Error, Fatal };

template <Severity S>
inline void log(const char* msg) {
    // Compile-time filter: strip debug in release
    if constexpr (S >= Severity::Warn) {
        log_string(msg);
    }
}

} // namespace platform

// Usage in any translation unit:
// platform::log<platform::Severity::Error>("Sensor timeout");
```

The `if constexpr` on the severity filter means debug-level logging compiles away entirely in release builds - no runtime check, no code generated at all.

### Q2: Demonstrate which C++23 freestanding utilities you can use to build a safe register-access wrapper without any hosted headers

One of the practical benefits of C++23's expanded freestanding support is that you can now use `std::expected` for error handling and `std::span` for buffer views - both without any hosted dependency. This example builds a full register-access abstraction using only freestanding headers:

```cpp
// register_access.h - Pure freestanding C++23
#pragma once

// All of these are freestanding in C++23
#include <cstdint>       // freestanding since C++11
#include <optional>      // freestanding since C++23
#include <expected>      // freestanding since C++23
#include <bit>           // freestanding since C++23
#include <span>          // freestanding since C++23
#include <type_traits>   // freestanding since C++11
#include <limits>        // freestanding since C++11

namespace hw {

enum class RegError : std::uint8_t {
    Timeout,
    InvalidOffset,
    ReadOnly
};

// Peripheral register block base
template <std::uintptr_t Base, std::size_t Size>
class RegisterBlock {
    static_assert(Size > 0 && Size % 4 == 0,
                  "Register block size must be positive multiple of 4");
public:
    static constexpr std::uintptr_t base_addr = Base;
    static constexpr std::size_t    num_regs  = Size / 4;

    [[nodiscard]]
    static auto read32(std::size_t offset)
        -> std::expected<std::uint32_t, RegError>
    {
        if (offset >= num_regs) {
            return std::unexpected(RegError::InvalidOffset);
        }
        auto* reg = reinterpret_cast<volatile std::uint32_t*>(
            Base + offset * 4);
        return *reg;
    }

    static auto write32(std::size_t offset, std::uint32_t val)
        -> std::expected<void, RegError>
    {
        if (offset >= num_regs) {
            return std::unexpected(RegError::InvalidOffset);
        }
        auto* reg = reinterpret_cast<volatile std::uint32_t*>(
            Base + offset * 4);
        *reg = val;
        return {};
    }

    // Read a bitfield: bit position P, width W
    template <unsigned P, unsigned W>
    [[nodiscard]]
    static auto read_field(std::size_t offset)
        -> std::expected<std::uint32_t, RegError>
    {
        static_assert(P + W <= 32, "Bitfield exceeds register width");
        constexpr std::uint32_t mask = ((1u << W) - 1u) << P;

        auto val = read32(offset);
        if (!val) return std::unexpected(val.error());
        return (*val & mask) >> P;
    }

    // Count leading zeros in a register value (C++23 <bit>)
    [[nodiscard]]
    static auto count_leading_zeros(std::size_t offset)
        -> std::expected<int, RegError>
    {
        auto val = read32(offset);
        if (!val) return std::unexpected(val.error());
        return std::countl_zero(*val);
    }
};

// Example: SPI peripheral on Cortex-M at 0x4001'3000, 64 bytes
using SPI1 = RegisterBlock<0x4001'3000, 64>;

// CR1 = offset 0, SR = offset 2, DR = offset 3
inline bool spi_is_busy() {
    // BSY flag = bit 7, width 1 in status register (offset 2)
    auto busy = SPI1::read_field<7, 1>(2);
    return busy.value_or(1) != 0;   // assume busy on error
}

} // namespace hw
```

Notice that `std::expected` gives you the same expressiveness as exceptions - callers must explicitly handle the error case - without any of the exception table overhead. This is a pattern that will become increasingly common as C++23 compilers mature.

### Q3: Write a CMake toolchain file and C++ source that compiles for both freestanding (ARM Cortex-M) and hosted (x86-64 Linux) from the same codebase

The key to dual-target code is keeping all hosted-only code behind `#if __STDC_HOSTED__` guards. The core application logic - data structures, algorithms, state machines - should live in files that include only freestanding headers.

```cmake
# toolchain-arm-none-eabi.cmake
set(CMAKE_SYSTEM_NAME Generic)       # No OS -> freestanding
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER   arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Freestanding flags
set(COMMON_FLAGS "-ffreestanding -nostdlib -mthumb -mcpu=cortex-m4 \
    -mfloat-abi=hard -mfpu=fpv4-sp-d16 -fno-exceptions -fno-rtti")

set(CMAKE_C_FLAGS_INIT   "${COMMON_FLAGS}")
set(CMAKE_CXX_FLAGS_INIT "${COMMON_FLAGS}")

set(CMAKE_EXE_LINKER_FLAGS_INIT "-T ${CMAKE_SOURCE_DIR}/linker.ld \
    -nostartfiles -Wl,--gc-sections")
```

```cpp
// main.cpp - Dual-target source
#include <cstdint>
#include <type_traits>

#if __STDC_HOSTED__
  #include <cstdio>
  #include <cstdlib>
#endif

namespace app {

struct SensorReading {
    std::int16_t temperature_centi;  // degrees C x 100
    std::uint16_t humidity_pct_x10;  // %RH x 10
    std::uint32_t timestamp_ms;
};
static_assert(std::is_trivially_copyable_v<SensorReading>,
              "Must be trivially copyable for DMA / memcpy");

constexpr std::size_t MAX_READINGS = 64;
static SensorReading ring_buffer[MAX_READINGS];
static std::size_t   write_idx = 0;

void store_reading(SensorReading r) {
    ring_buffer[write_idx % MAX_READINGS] = r;
    ++write_idx;
}

const SensorReading& latest() {
    return ring_buffer[(write_idx - 1) % MAX_READINGS];
}

} // namespace app

// ---------- Entry points ----------
#if __STDC_HOSTED__
int main() {
    app::store_reading({2350, 655, 1000});
    app::store_reading({2410, 670, 2000});
    auto& r = app::latest();
    std::printf("T=%d.%02d°C  RH=%u.%u%%  @%ums\n",
                r.temperature_centi / 100,
                r.temperature_centi % 100,
                r.humidity_pct_x10 / 10,
                r.humidity_pct_x10 % 10,
                r.timestamp_ms);
    return 0;
}
#else
// Bare-metal entry point (called from startup.s after .data/.bss init)
extern "C" [[noreturn]] void app_main() {
    app::store_reading({2350, 655, 0});
    while (true) {
        // Main loop - read sensors, store, sleep
        asm volatile("wfi");  // Wait For Interrupt
    }
}
#endif
```

```cmake
# CMakeLists.txt - Build for either target
cmake_minimum_required(VERSION 3.25)
project(sensor_fw LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)

add_executable(sensor_app main.cpp)

if(CMAKE_SYSTEM_NAME STREQUAL "Generic")
    message(STATUS "Building freestanding for ${CMAKE_SYSTEM_PROCESSOR}")
    target_compile_definitions(sensor_app PRIVATE NDEBUG)
    # Size-optimized for flash
    target_compile_options(sensor_app PRIVATE -Os -ffunction-sections -fdata-sections)
else()
    message(STATUS "Building hosted for ${CMAKE_HOST_SYSTEM_PROCESSOR}")
    target_compile_options(sensor_app PRIVATE -O2 -Wall -Wextra)
endif()
```

---

## Notes

- `__STDC_HOSTED__` is the **only** portable macro to detect freestanding vs hosted at the preprocessor level; it is mandated by both the C and C++ standards.
- C++23 freestanding additions (`<optional>`, `<expected>`, `<span>`, `<string_view>`, `<variant>`, `<tuple>`, `<ranges>`, `<functional>`, `<bit>`, `<array>`) do **not** include their throwing members - `.value()` on `optional`/`expected` is **not** freestanding because it would throw.
- GCC uses `-ffreestanding` to set `__STDC_HOSTED__` to 0; Clang does the same. MSVC does not natively support freestanding.
- When writing code for both modes, put all hosted-only code behind `#if __STDC_HOSTED__` guards and keep the core logic header-only with freestanding headers.
- Freestanding does **not** guarantee `operator new`/`operator delete` - any dynamic allocation must be user-supplied.
- ARM CMSIS headers and vendor HALs are C headers; wrap with `extern "C"` and validate they compile under `-ffreestanding -std=c++23`.
- Always `static_assert` layout-critical types with `is_trivially_copyable_v` and `is_standard_layout_v` for DMA and hardware buffer compatibility.
