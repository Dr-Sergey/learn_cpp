# Structure a multi-target embedded project (bootloader, app, tests)

**Category:** Project Architecture  
**Standard:** Not version-specific  

---

## Topic Overview

Embedded projects often produce multiple firmware images from a single codebase: a **bootloader** that handles firmware updates and jumps to the main application, the main **application** itself, and **host-based tests** that run on your development PC without any hardware. Each of these targets has different linker scripts, different startup code, and different memory layouts - but they all share the same HAL interfaces, device drivers, and business logic.

Managing all of this cleanly with CMake requires understanding how to conditionally include targets based on whether you are cross-compiling for the target hardware or building natively for the host.

### Multi-Target Project Structure

| Target | Memory Region | Size | Purpose |
| --- | --- | --- | --- |
| **Bootloader** | 0x0800_0000 - 0x0800_7FFF | 32KB | Firmware update, jump to app |
| **Application** | 0x0800_8000 - 0x080F_FFFF | 992KB | Main application firmware |
| **Tests** | Host PC | N/A | Unit tests (no hardware) |
| **Factory test** | Full flash | 1MB | Production line testing |

The address split between bootloader and application is intentional. The bootloader owns the low part of flash at 0x08000000, which is where the processor looks for the reset vector on power-up. The application starts at 0x08008000, which is 32KB up - exactly enough room for the bootloader. The bootloader validates the application image and then jumps to that address.

---

## Self-Assessment

### Q1: Structure the project directory and CMake

The key CMake insight for embedded multi-target builds is the `CMAKE_CROSSCOMPILING` variable. When you configure with an ARM toolchain file, CMake sets this to true and you build bootloader and app. When you configure without a toolchain file (using your host GCC or Clang), it is false and you build only the host tests with a mock BSP. The same `CMakeLists.txt` serves both scenarios.

**Answer:**

```cpp
project/
├── CMakeLists.txt              # Top-level: defines targets
├── cmake/
│   ├── toolchain-arm.cmake     # ARM cross-compiler toolchain
│   ├── toolchain-host.cmake    # Host compiler for tests
│   ├── linker/
│   │   ├── bootloader.ld       # 0x08000000, 32KB
│   │   └── application.ld      # 0x08008000, 992KB
│   └── targets.cmake           # Target-specific config
├── hal/                        # Shared HAL interfaces
│   ├── include/hal/
│   │   ├── gpio.h
│   │   ├── flash.h
│   │   └── uart.h
│   └── CMakeLists.txt
├── bsp/                        # Board support
│   ├── stm32f4/
│   │   ├── startup.s           # Vector table + reset handler
│   │   ├── system_init.cpp
│   │   ├── gpio_impl.cpp
│   │   └── CMakeLists.txt
│   └── host/                   # Mock BSP for tests
│       └── CMakeLists.txt
├── drivers/                    # Shared device drivers
│   ├── include/drivers/
│   │   ├── sensor.h
│   │   └── motor.h
│   ├── sensor.cpp
│   └── CMakeLists.txt
├── bootloader/                 # Bootloader-specific code
│   ├── main.cpp
│   ├── firmware_update.cpp
│   ├── crc32.cpp
│   └── CMakeLists.txt
├── app/                        # Application-specific code
│   ├── main.cpp
│   ├── control_loop.cpp
│   ├── state_machine.cpp
│   └── CMakeLists.txt
└── tests/                      # Host-based tests
    ├── test_control_loop.cpp
    ├── test_state_machine.cpp
    ├── test_crc32.cpp
    └── CMakeLists.txt
```

```cmake
# === Top-level CMakeLists.txt ===
cmake_minimum_required(VERSION 3.20)
project(embedded_project C CXX ASM)

set(CMAKE_CXX_STANDARD 20)

# Shared libraries (no target-specific code)
add_subdirectory(hal)
add_subdirectory(drivers)

if(CMAKE_CROSSCOMPILING)
    # Firmware targets (ARM)
    add_subdirectory(bsp/stm32f4)
    add_subdirectory(bootloader)
    add_subdirectory(app)
else()
    # Host targets (tests)
    add_subdirectory(bsp/host)
    add_subdirectory(tests)
endif()

# === bootloader/CMakeLists.txt ===
add_executable(bootloader
    main.cpp
    firmware_update.cpp
    crc32.cpp
)
target_link_libraries(bootloader PRIVATE hal bsp_stm32f4)
target_link_options(bootloader PRIVATE
    -T${CMAKE_SOURCE_DIR}/cmake/linker/bootloader.ld
    -Wl,--gc-sections
)
# Generate .bin and .hex
add_custom_command(TARGET bootloader POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:bootloader>
            bootloader.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:bootloader>
            bootloader.hex
)

# === app/CMakeLists.txt ===
add_executable(application
    main.cpp
    control_loop.cpp
    state_machine.cpp
)
target_link_libraries(application PRIVATE hal bsp_stm32f4 drivers)
target_link_options(application PRIVATE
    -T${CMAKE_SOURCE_DIR}/cmake/linker/application.ld
    -Wl,--gc-sections
)
```

The `--gc-sections` linker flag is important for embedded targets. It removes any code sections that are not reachable from the entry point, which keeps the firmware image as small as possible. You only get this benefit if you also compile with `-ffunction-sections -fdata-sections`, which puts each function and variable in its own section so the linker can individually discard unused ones.

### Q2: Linker scripts and memory layout

Linker scripts tell the linker where each section of your program lives in physical memory. Without them, an ARM Cortex-M processor will not boot correctly because it will not find the interrupt vector table at the right address. The bootloader and application need separate linker scripts because they live at different addresses.

**Answer:**

```cmake
/* === bootloader.ld === */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 32K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

ENTRY(Reset_Handler)

SECTIONS
{
    .isr_vector : { KEEP(*(.isr_vector)) } > FLASH
    .text       : { *(.text*) *(.rodata*) } > FLASH
    .data       : { *(.data*) } > RAM AT > FLASH
    .bss        : { *(.bss*) *(COMMON) } > RAM
}

/* === application.ld === */
MEMORY
{
    /* Application starts after bootloader */
    FLASH (rx)  : ORIGIN = 0x08008000, LENGTH = 992K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

ENTRY(Reset_Handler)

SECTIONS
{
    .isr_vector : { KEEP(*(.isr_vector)) } > FLASH
    .text       : { *(.text*) *(.rodata*) } > FLASH
    .data       : { *(.data*) } > RAM AT > FLASH
    .bss        : { *(.bss*) *(COMMON) } > RAM

    /* Shared memory for bootloader<->app communication */
    .shared_data (NOLOAD) : {
        KEEP(*(.shared_data))
    } > RAM
}
```

```cpp
// === Bootloader: jump to application ===
void jump_to_application() {
    constexpr uint32_t APP_ADDRESS = 0x08008000;

    // Read application's stack pointer and reset handler
    uint32_t* app_vector = reinterpret_cast<uint32_t*>(APP_ADDRESS);
    uint32_t app_sp = app_vector[0];
    uint32_t app_reset = app_vector[1];

    // Validate: stack pointer should be in RAM
    if (app_sp < 0x20000000 || app_sp > 0x20020000) {
        // Invalid application image
        return;
    }

    // Disable interrupts, set stack pointer, jump
    __disable_irq();
    __set_MSP(app_sp);
    SCB->VTOR = APP_ADDRESS;  // Relocate vector table

    auto app_entry = reinterpret_cast<void(*)()>(app_reset);
    app_entry();  // Never returns
}

// === Shared memory between bootloader and app ===
struct SharedData {
    uint32_t magic;           // 0xDEADBEEF = valid
    uint32_t boot_reason;     // Why did we boot?
    uint32_t firmware_version;
    uint32_t crc32;
} __attribute__((section(".shared_data")));
```

The stack pointer validation before jumping is not optional - it is the safety check that prevents the bootloader from jumping to garbage if the application flash region is erased or corrupt. An erased STM32 flash reads as 0xFFFFFFFF, so `app_sp` would be 0xFFFFFFFF, which is well outside valid RAM and correctly rejected. The `SCB->VTOR` assignment relocates the vector table so the application's interrupt handlers are found at the right address after the jump.

### Q3: Build presets for all targets

With three distinct configurations (ARM debug, ARM release, and host tests), `CMakePresets.json` is particularly valuable here. Without presets, you would need separate shell scripts to invoke CMake with the right toolchain file and options for each case.

**Answer:**

```json
// === CMakePresets.json ===
{
  "version": 6,
  "configurePresets": [
    {
      "name": "arm-debug",
      "displayName": "ARM Debug (bootloader + app)",
      "toolchainFile": "cmake/toolchain-arm.cmake",
      "binaryDir": "build/arm-debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "MCU": "STM32F407VG"
      }
    },
    {
      "name": "arm-release",
      "displayName": "ARM Release (optimized)",
      "toolchainFile": "cmake/toolchain-arm.cmake",
      "binaryDir": "build/arm-release",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release",
        "MCU": "STM32F407VG"
      }
    },
    {
      "name": "host-test",
      "displayName": "Host Tests (no hardware)",
      "binaryDir": "build/host-test",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "BUILD_TESTING": "ON"
      }
    }
  ],
  "buildPresets": [
    { "name": "bootloader", "configurePreset": "arm-release",
      "targets": ["bootloader"] },
    { "name": "application", "configurePreset": "arm-release",
      "targets": ["application"] },
    { "name": "all-firmware", "configurePreset": "arm-release" },
    { "name": "tests", "configurePreset": "host-test" }
  ]
}
```

```cmake
# === cmake/toolchain-arm.cmake ===
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)

set(CMAKE_C_FLAGS_INIT "-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16")
set(CMAKE_CXX_FLAGS_INIT "${CMAKE_C_FLAGS_INIT} -fno-exceptions -fno-rtti")

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

The `CMAKE_FIND_ROOT_PATH_MODE_*` settings at the bottom of the toolchain file tell CMake not to look for programs (like `cmake`, `python`) in the sysroot, but to look for libraries and headers only there. Without these, CMake might accidentally find host system headers or tools and try to use them when building for ARM.

---

## Notes

- Bootloader and app are separate ELF files with separate linker scripts and separate flash regions.
- Shared code (HAL, drivers) is compiled for each target but linked separately.
- Host tests use mock BSP (no vendor HAL headers); test business logic and algorithms on PC.
- Use `CMakePresets.json` for reproducible builds: `cmake --preset arm-release`.
- The bootloader validates the app image (CRC, magic number) before jumping.
- Shared memory (`.shared_data` section) allows bootloader/app communication (boot reason, update status).
- Always validate the stack pointer before jumping - an erased flash (0xFFFFFFFF) would crash.
