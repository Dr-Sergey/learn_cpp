# Understand C++ exception table overhead and when to use -fno-exceptions

**Category:** Embedded and Constrained Systems  
**Standard:** C++17  
**Reference:** <https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html>  

---

## Topic Overview

### The Cost of C++ Exceptions

C++ exceptions use **zero-cost exception handling** (on Itanium ABI / GCC / Clang): there is no runtime overhead when exceptions are **not thrown**. The phrase "zero-cost" trips people up, though, because it only refers to the dynamic cost in the normal (non-throwing) path. Enabling exceptions still has significant **static costs** that matter enormously on constrained systems - they take up flash space whether or not you ever throw.

| Cost type | Impact | Typical size |
| --- | --- | --- |
| **Unwind tables** (`.ARM.exidx`, `.eh_frame`) | Flash | 10-30% of `.text` size |
| **Landing pads** (catch clauses) | Flash | Per catch site |
| **Personality routine** | Flash | ~2-5 KB |
| **C++ runtime** (`__cxa_throw`, `__cxa_begin_catch`) | Flash | ~5-15 KB |
| **Stack usage** | RAM | Minimal (tables are in flash) |
| **Runtime cost** (when not throwing) | CPU | Zero on modern compilers |
| **Runtime cost** (when throwing) | CPU | Very slow (~1000x vs return) |

### Unwind Tables - Where the Bloat Comes From

For every function, the compiler emits a table entry describing how to unwind the stack frame if an exception propagates through it - how to restore saved registers and run destructors. On ARM these tables live in dedicated sections:

```cpp
.ARM.exidx section - one 8-byte entry per function
.ARM.extab section - extended unwind info for complex frames
```

For a firmware with 1,000 functions, that is at minimum 8 KB just for the index. Functions with local variables that have destructors need extended entries. All of this flash space is occupied regardless of whether an exception is ever actually thrown.

### Measuring the Impact

The best way to understand what you are paying is to measure it directly. Build the same project twice - once with and once without exceptions - and compare:

```bash
# Build with exceptions
arm-none-eabi-g++ -O2 -std=c++17 -o with_exc.elf main.cpp

# Build without exceptions
arm-none-eabi-g++ -O2 -std=c++17 -fno-exceptions -o no_exc.elf main.cpp

# Compare sizes
arm-none-eabi-size with_exc.elf no_exc.elf

# Show exception-related sections
arm-none-eabi-objdump -h with_exc.elf | grep -E "exidx|extab|eh_frame"
```

Typical results on a medium-sized project (~50 KB code):

| Build | .text | .ARM.exidx | .ARM.extab | Total flash |
| --- | --- | --- | --- | --- |
| With exceptions | 48 KB | 8 KB | 3 KB | 62 KB |
| Without exceptions | 45 KB | 0 | 0 | 48 KB |

That is a **23% reduction** from disabling exceptions. On a device with 64 KB of flash, that difference can determine whether your firmware fits at all.

### What `-fno-exceptions` Does

When you pass this flag, four things change:

1. **Disables `throw` and `try`/`catch`** - using them is a compile error.
2. **Removes unwind tables** - no `.ARM.exidx` / `.eh_frame` sections.
3. **Changes `std::terminate` behavior** - throwing (e.g., from `std::optional::value()`) calls `std::abort()` directly.
4. **Enables `operator new` to return null** - unless you link `--specs=nosys.specs` or use your own `operator new`.

### What Breaks Without Exceptions

This is the part you need to audit carefully before switching a codebase to `-fno-exceptions`. These patterns compile fine but will silently call `std::terminate()` at runtime instead of throwing:

```cpp
// These are compile errors with -fno-exceptions:
throw std::runtime_error("oops");
try { f(); } catch (...) { }

// These call std::terminate() / abort():
std::optional<int> opt;
int x = opt.value();     // throws bad_optional_access -> terminates

std::vector<int> v;
int y = v.at(10);        // throws out_of_range -> terminates

auto p = std::make_unique<LargeObj>(); // new may return nullptr if not handled
```

The reason this trips people up is that the code looks perfectly normal - `.value()` and `.at()` are standard library functions and there is no visible `throw` keyword anywhere. The exception is hidden inside the library. You must replace every such call with the non-throwing alternative: `opt.value_or(default)`, `v[i]` with manual bounds checking, `get_if` instead of `get`, and so on.

### Coding Patterns Without Exceptions

The standard replacement for exception-based error handling is to make errors explicit in the return type. Here are three patterns you will see in embedded codebases:

```cpp
// Instead of exceptions - return error codes or std::expected
#include <cstdint>

enum class Error : uint8_t {
    None = 0,
    InvalidInput,
    Timeout,
    HardwareFault,
};

// Option 1: Error codes
struct Result {
    float value;
    Error error;
};

Result read_sensor() {
    if (!sensor_ready()) {
        return {0.0f, Error::Timeout};
    }
    float v = adc_read();
    if (v < 0.0f || v > 3.3f) {
        return {0.0f, Error::HardwareFault};
    }
    return {v, Error::None};
}

// Option 2: bool return with out parameter
bool parse_command(const char* input, Command& out) {
    if (!input) return false;
    // ... parse ...
    out = parsed;
    return true;
}

// Option 3: assert/abort for truly unrecoverable errors
void critical_init() {
    bool ok = configure_clocks();
    if (!ok) {
        // Log error and halt - no recovery possible
        error_log("Clock init failed");
        __disable_irq();
        while (true) {}
    }
}
```

Option 3 is appropriate for initialization failures where there is genuinely no recovery path. The system either starts correctly or halts - a controlled halt is better than undefined behavior.

### Providing `operator new` for `-fno-exceptions`

If any code path uses `new` (even indirectly through standard containers), you should provide your own `operator new` that makes the failure behavior explicit rather than relying on the default:

```cpp
#include <cstdlib>
#include <new>

// With -fno-exceptions, the default operator new calls std::terminate
// on failure. Provide your own that returns nullptr:
void* operator new(std::size_t size, const std::nothrow_t&) noexcept {
    return malloc(size);
}

void* operator new(std::size_t size) noexcept {
    void* p = malloc(size);
    if (!p) {
        // In embedded: log and halt rather than silently returning null
        error_handler("Out of memory");
    }
    return p;
}

void operator delete(void* p) noexcept { free(p); }
void operator delete(void* p, std::size_t) noexcept { free(p); }
```

### `-fno-rtti` - the Companion Flag

RTTI (Run-Time Type Information) adds `type_info` objects for every polymorphic class (~50-100 bytes each) and the machinery to support `typeid` and `dynamic_cast`. Most embedded projects do not need either of these, so disabling RTTI is almost always the right call alongside `-fno-exceptions`.

```bash
arm-none-eabi-g++ -fno-exceptions -fno-rtti -o minimal.elf main.cpp
```

| Feature | Disabled by | Saves |
| --- | --- | --- |
| `throw`/`catch` | `-fno-exceptions` | 15-30% flash |
| `dynamic_cast`, `typeid` | `-fno-rtti` | 2-5% flash |
| Both | Both flags | 20-35% flash |

### CMake Integration

Here is how to apply these flags cleanly in CMake so they apply consistently to the whole firmware target:

```cmake
target_compile_options(firmware PRIVATE
    -fno-exceptions
    -fno-rtti
    -fno-unwind-tables       # Explicitly remove unwind tables
    -fno-asynchronous-unwind-tables
)

# GCC link-time flag to remove exception handling library
target_link_options(firmware PRIVATE
    -fno-exceptions
    --specs=nano.specs       # Use newlib-nano (smaller libc)
)
```

---

## Self-Assessment

### Q1: Why does zero-cost exception handling still cost flash

"Zero-cost" means no runtime overhead **when exceptions are not thrown** - no extra instructions in the normal code path. But the compiler must still emit **unwind tables** that describe how to restore each stack frame during unwinding. These tables are stored in flash (`.ARM.exidx`, `.ARM.extab`, or `.eh_frame`) and can be 10-30% of the text section size. On a 512 KB MCU, that could be 50-100 KB of flash wasted on data that is never used if you never throw.

### Q2: What's the main risk of using `-fno-exceptions`

Standard library functions that normally throw exceptions will call `std::terminate()` instead, causing an **immediate abort**. This includes:

- `std::vector::at()`, `std::optional::value()`, `std::get<>()` on `std::variant`
- `operator new` on allocation failure
- Any third-party library that uses exceptions internally

You must audit all code paths and replace exception-relying APIs with non-throwing alternatives (e.g., `operator[]` instead of `at()`, `std::get_if` instead of `std::get`).

### Q3: Show the CMake configuration for a minimal-overhead C++ embedded build

```cmake
add_executable(firmware main.cpp startup.cpp)

target_compile_features(firmware PRIVATE cxx_std_17)

target_compile_options(firmware PRIVATE
    -Os                     # Optimize for size
    -fno-exceptions         # No exception support
    -fno-rtti               # No RTTI
    -fno-unwind-tables
    -fno-asynchronous-unwind-tables
    -ffunction-sections     # Put each function in its own section
    -fdata-sections         # Put each variable in its own section
    -Wall -Wextra
)

target_link_options(firmware PRIVATE
    -Wl,--gc-sections       # Remove unused sections
    -Wl,--print-memory-usage
    --specs=nano.specs       # Tiny C library
    --specs=nosys.specs      # No system calls
    -T${CMAKE_SOURCE_DIR}/linker.ld  # Custom linker script
)
```

The combination of `-ffunction-sections -fdata-sections` and `-Wl,--gc-sections` is particularly powerful - it lets the linker discard every function and variable that nothing else references, which can cut significant flash usage from a project that only uses a subset of the standard library.

---

## Notes

- Always measure: build with and without exceptions and compare `arm-none-eabi-size` output before committing to the change.
- `-ffunction-sections` + `-Wl,--gc-sections` is essential - it removes unused code and pairs naturally with `-fno-exceptions`.
- Some vendors (e.g., Arduino, Mbed) default to `-fno-exceptions` for their platforms, so you may already be in this world without realizing it.
- If you need exception-like error propagation, use `std::expected` (C++23) or a lightweight Result type - they give you the expressiveness without the flash overhead.
- Clang and GCC behave identically with these flags.
