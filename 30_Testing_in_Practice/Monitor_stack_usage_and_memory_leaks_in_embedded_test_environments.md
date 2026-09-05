# Monitor stack usage and memory leaks in embedded test environments

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

Stack overflow and memory leaks are among the most critical bugs in embedded systems, and they're particularly nasty because the hardware doesn't give you much warning. When a desktop application leaks memory, the OS will eventually kill it. When an embedded system overflows its stack, it silently corrupts whatever happens to live at the memory addresses below the stack pointer - often other variables, other tasks' stacks, or the heap. The system continues running but behaves unpredictably, and the real cause can be very far from where you observe the symptoms.

The reason this trips people up is that stack overflows are non-deterministic during development. Your code might work fine during testing with short call chains, then overflow in the field when an interrupt fires during a deep function call. That's why you need to measure and enforce stack budgets before the firmware ships, not after.

RAM on typical microcontrollers is very limited - often 8 to 256 KB for the entire system including stack, heap, and static data. The four main techniques for staying within budget are: **compile-time stack analysis** (know per-function maximums before you run), **runtime stack painting** (measure actual high-water marks during testing), **leak detection in test builds** (catch heap leaks on the host), and **linker map file analysis** (verify total static allocation fits the MCU).

### Memory Monitoring Approaches

| Technique | Stack | Heap | When | Overhead |
| --- | --- | --- | --- | --- |
| **`-fstack-usage` (GCC)** | Per-function analysis | No | Compile time | Zero |
| **Stack painting** | High-water mark | No | Runtime | Low |
| **ASan (host)** | Overflow detect | Leaks | Test time | 2x memory |
| **Valgrind/Memcheck** | No | Full leak report | Test time | 10-30x slower |
| **Linker map** | Total allocation | Static/BSS | Link time | Zero |
| **Custom new/delete** | No | Track heap | Runtime | Low |

---

## Self-Assessment

### Q1: Analyze stack usage at compile time and link time

**Answer:**

GCC's `-fstack-usage` flag makes the compiler emit a `.su` file alongside each `.o` file. Each line in the `.su` file tells you how many bytes of stack a particular function uses and whether that's a static (known at compile time) or dynamic (bounded or unbounded, depending on whether it uses VLAs) value. This is zero-overhead analysis - it happens at compile time and has no effect on the compiled binary.

```bash
# === GCC -fstack-usage: per-function stack analysis ===
arm-none-eabi-g++ -fstack-usage -c src/main.cpp -o build/main.o
# Creates main.su alongside main.o:
# main.cpp:15:6:main  128  static
# main.cpp:30:6:process_data  256  dynamic,bounded
# main.cpp:45:6:handle_isr  32  static

# Parse .su files to find stack hogs
find build -name '*.su' -exec cat {} \; | sort -t: -k5 -n -r | head -20
```

To turn that raw data into an automated pass/fail check, a small Python script can parse all `.su` files and flag any function that exceeds your stack budget per function:

```python
# === stack_analyzer.py: Parse .su files and check against limits ===
import sys
from pathlib import Path

def analyze_stack(build_dir: str, max_stack: int = 512):
    violations = []
    for su_file in Path(build_dir).rglob('*.su'):
        for line in su_file.read_text().splitlines():
            parts = line.split('\t')
            if len(parts) >= 2:
                func = parts[0]
                stack_bytes = int(parts[1])
                if stack_bytes > max_stack:
                    violations.append((func, stack_bytes))

    if violations:
        print(f"Stack violations (limit: {max_stack} bytes):")
        for func, size in sorted(violations, key=lambda x: -x[1]):
            print(f"  {func}: {size} bytes")
        return False
    return True

if __name__ == '__main__':
    ok = analyze_stack('build', max_stack=512)
    sys.exit(0 if ok else 1)
```

You can wire both the compiler flag and the post-build analysis check directly into CMake so they run automatically on every build:

```cmake
# === CMake: Enable stack analysis and linker map ===
target_compile_options(firmware PRIVATE
    -fstack-usage
    -Wstack-usage=512     # Warn if any function uses >512 bytes
)

target_link_options(firmware PRIVATE
    -Wl,-Map=${CMAKE_BINARY_DIR}/firmware.map
    -Wl,--print-memory-usage    # Show RAM/Flash usage summary
)

# Post-build: check stack limits
add_custom_command(TARGET firmware POST_BUILD
    COMMAND python3 ${CMAKE_SOURCE_DIR}/tools/stack_analyzer.py
            ${CMAKE_BINARY_DIR} --max-stack 512
    COMMENT "Checking stack usage..."
)
```

The `-Wl,--print-memory-usage` linker flag gives you a quick sanity check like `FLASH: 45KB / 256KB, RAM: 12KB / 64KB` right at the end of the build output. If you're anywhere near the limits, you'll know immediately.

### Q2: Runtime stack monitoring with stack painting

**Answer:**

Compile-time analysis tells you the worst case per function, but it can't tell you which combination of nested calls actually occurs at runtime. Stack painting fills this gap by writing a known byte pattern across the unused portion of the stack at startup, then later measuring how far into that pattern a real call chain reached. This "high-water mark" approach is the standard technique on RTOS systems.

The reason this works is simple: if you fill the whole stack with `0xDEADBEEF` at boot and later inspect it from the bottom upward, the first word that's no longer `0xDEADBEEF` marks where the stack was actually used.

```cpp
// === Stack painting: fill stack with known pattern at boot ===
// Implemented in startup code or RTOS task creation

#include <cstdint>
#include <cstring>
#include <algorithm>

constexpr uint32_t STACK_PAINT_PATTERN = 0xDEADBEEF;

void paint_stack(uint32_t* stack_bottom, size_t size_words) {
    // Fill unused stack with pattern
    // Called at boot before main (or at task creation in RTOS)
    for (size_t i = 0; i < size_words; ++i)
        stack_bottom[i] = STACK_PAINT_PATTERN;
}

size_t measure_stack_used(const uint32_t* stack_bottom, size_t size_words) {
    // Count from bottom up to find first unpainted word
    size_t unused = 0;
    for (size_t i = 0; i < size_words; ++i) {
        if (stack_bottom[i] == STACK_PAINT_PATTERN)
            unused++;
        else
            break;
    }
    return (size_words - unused) * sizeof(uint32_t);
}

float stack_usage_percent(const uint32_t* stack_bottom, size_t size_words) {
    size_t used = measure_stack_used(stack_bottom, size_words);
    return (static_cast<float>(used) / (size_words * sizeof(uint32_t))) * 100.0f;
}

// === Host test for the stack analysis functions ===
#include <gtest/gtest.h>
#include <vector>

TEST(StackPaintTest, MeasuresUsedStack) {
    // Simulate a 1KB stack, fully painted
    std::vector<uint32_t> stack(256, STACK_PAINT_PATTERN);

    // Simulate usage: overwrite top 64 words
    for (size_t i = 192; i < 256; ++i)
        stack[i] = 0x12345678;

    size_t used = measure_stack_used(stack.data(), stack.size());
    EXPECT_EQ(used, 64 * sizeof(uint32_t));  // 256 bytes used
}

TEST(StackPaintTest, EmptyStackShowsZeroUsage) {
    std::vector<uint32_t> stack(256, STACK_PAINT_PATTERN);
    EXPECT_EQ(measure_stack_used(stack.data(), stack.size()), 0);
}

TEST(StackPaintTest, FullStackShowsFullUsage) {
    std::vector<uint32_t> stack(256, 0x12345678);  // All overwritten
    size_t used = measure_stack_used(stack.data(), stack.size());
    EXPECT_EQ(used, 256 * sizeof(uint32_t));
}
```

Notice that the painting and measuring functions have no hardware dependencies - they just work on arrays of `uint32_t`. That means you can unit-test them on the host (exactly as shown above) before trusting them on real hardware.

### Q3: Integrate memory leak detection into test infrastructure

**Answer:**

For host tests of embedded code you want to know if any test leaves heap allocations uncollected. The most portable approach is to override the global `operator new` and `operator delete` to maintain an allocation counter. Any test that ends with a net-positive allocation count has a leak.

The reason to use a global test listener rather than per-test assertions is that leaks are often in teardown paths that only run after the test body has already passed. The listener catches them reliably regardless of where in the test lifecycle they happen.

```cpp
#include <gtest/gtest.h>
#include <cstdlib>
#include <atomic>
#include <iostream>

// === Global new/delete override for leak tracking ===
namespace {
    std::atomic<int64_t> g_alloc_count{0};
    std::atomic<int64_t> g_alloc_bytes{0};
    std::atomic<int64_t> g_peak_bytes{0};
}

void* operator new(size_t size) {
    void* ptr = std::malloc(size);
    if (!ptr) throw std::bad_alloc();
    g_alloc_count.fetch_add(1, std::memory_order_relaxed);
    g_alloc_bytes.fetch_add(size, std::memory_order_relaxed);
    auto current = g_alloc_bytes.load(std::memory_order_relaxed);
    auto peak = g_peak_bytes.load(std::memory_order_relaxed);
    while (current > peak &&
           !g_peak_bytes.compare_exchange_weak(peak, current,
               std::memory_order_relaxed)) {}
    return ptr;
}

void operator delete(void* ptr) noexcept {
    if (ptr) {
        g_alloc_count.fetch_sub(1, std::memory_order_relaxed);
        std::free(ptr);
    }
}

void operator delete(void* ptr, size_t) noexcept {
    ::operator delete(ptr);
}

// === Test listener that reports memory stats ===
class MemoryCheckListener : public ::testing::EmptyTestEventListener {
public:
    void OnTestStart(const ::testing::TestInfo&) override {
        start_count_ = g_alloc_count.load();
    }

    void OnTestEnd(const ::testing::TestInfo& info) override {
        auto end_count = g_alloc_count.load();
        auto leaked = end_count - start_count_;
        if (leaked > 0) {
            std::cerr << "[LEAK] " << info.test_suite_name() << "."
                      << info.name() << ": "
                      << leaked << " allocations leaked\n";
        }
    }

    void OnTestProgramEnd(const ::testing::UnitTest&) override {
        std::cout << "Peak heap usage: "
                  << g_peak_bytes.load() << " bytes\n";
    }

private:
    int64_t start_count_ = 0;
};

// Register in main:
// auto& listeners = ::testing::UnitTest::GetInstance()->listeners();
// listeners.Append(new MemoryCheckListener);
```

The peak heap usage printed at program end is a bonus - it tells you the maximum heap demand across all tests, which you can compare against your MCU's available SRAM to verify the code would fit on the real device.

---

## Notes

- `-fstack-usage` is GCC-specific; Clang has limited support - use `-Wframe-larger-than=` for warnings instead.
- Stack painting works on any platform but only detects the high-water mark, not per-call depth.
- Global `operator new/delete` override is a simple but effective leak detector for test builds.
- Parse linker `.map` files to verify total static RAM usage fits in the MCU's SRAM.
- RTOS task stacks should have 20-30% margin above the measured high-water mark.
- Combine compile-time (`-fstack-usage`) and runtime (painting) for defense in depth.
- In CI: fail the build if any function exceeds the stack budget or if leaks are detected.
- `-Wl,--print-memory-usage` gives a quick summary: `FLASH: 45KB / 256KB, RAM: 12KB / 64KB`.
