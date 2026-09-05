# Apply sanitizers (ASan, UBSan, TSan, MSan) in your test pipeline

**Category:** Testing in Practice  
**Standard:** Not version-specific  

---

## Topic Overview

**Sanitizers** are compiler-instrumented runtime checkers that detect undefined behavior, memory errors, data races, and uninitialized reads that regular tests and static analysis miss. Think of them as a very smart invisible observer that watches your program run and screams the moment something goes wrong - before that wrongness has a chance to corrupt state further down the line. They are the single highest-value testing tool for C++ after basic unit tests.

The reason this matters so much is that C++ bugs like out-of-bounds writes and data races are *silent* by default. Your tests may all pass while the program is quietly overwriting adjacent memory or reading garbage. Sanitizers make those silent bugs loud.

### Sanitizer Overview

| Sanitizer | Flag | Detects | Overhead |
| --- | --- | --- | --- |
| **ASan** (AddressSanitizer) | `-fsanitize=address` | Heap/stack buffer overflow, use-after-free, double-free, memory leaks | 2x slowdown, 2-3x memory |
| **UBSan** (UndefinedBehaviorSanitizer) | `-fsanitize=undefined` | Signed overflow, null deref, misaligned access, shift errors | Minimal (<5%) |
| **TSan** (ThreadSanitizer) | `-fsanitize=thread` | Data races, lock order violations | 5-15x slowdown, 5-10x memory |
| **MSan** (MemorySanitizer) | `-fsanitize=memory` | Uninitialized reads | 3x slowdown |

### Compatibility

Some sanitizers instrument the same memory regions and cannot coexist in the same binary. Keep this table in mind when planning your CI matrix - you will need separate build configurations for TSan and MSan.

| Combination | Compatible? |
| --- | --- |
| ASan + UBSan | Yes (recommended default) |
| ASan + TSan | No (mutually exclusive) |
| ASan + MSan | No (mutually exclusive) |
| TSan + UBSan | Yes |
| MSan + UBSan | Yes |

---

## Self-Assessment

### Q1: Set up sanitizer builds with CMake

**Answer:**

The approach here is to add CMake options for each sanitizer and guard against invalid combinations at configure time. This way you get a clear error message instead of a mysterious runtime crash when you accidentally mix incompatible sanitizers.

```cmake
# === CMakeLists.txt ===

# Sanitizer options
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)
option(ENABLE_MSAN "Enable MemorySanitizer" OFF)

# Validate incompatible combinations
if(ENABLE_ASAN AND ENABLE_TSAN)
    message(FATAL_ERROR "ASan and TSan cannot be used together")
endif()
if(ENABLE_ASAN AND ENABLE_MSAN)
    message(FATAL_ERROR "ASan and MSan cannot be used together")
endif()

# Apply sanitizer flags
set(SANITIZER_FLAGS "")
if(ENABLE_ASAN)
    list(APPEND SANITIZER_FLAGS -fsanitize=address)
    list(APPEND SANITIZER_FLAGS -fno-omit-frame-pointer)  # Better stack traces
endif()
if(ENABLE_UBSAN)
    list(APPEND SANITIZER_FLAGS -fsanitize=undefined)
    list(APPEND SANITIZER_FLAGS -fno-sanitize-recover=all)  # Abort on UB
endif()
if(ENABLE_TSAN)
    list(APPEND SANITIZER_FLAGS -fsanitize=thread)
endif()
if(ENABLE_MSAN)
    list(APPEND SANITIZER_FLAGS -fsanitize=memory)
    list(APPEND SANITIZER_FLAGS -fno-omit-frame-pointer)
    list(APPEND SANITIZER_FLAGS -fsanitize-memory-track-origins)  # Track where uninitialized data came from
endif()

if(SANITIZER_FLAGS)
    add_compile_options(${SANITIZER_FLAGS})
    add_link_options(${SANITIZER_FLAGS})
endif()
```

Notice that the same flags go to both the compiler and the linker - the sanitizer runtime is a shared library that needs to be linked in. With those flags in place, here is how you drive the builds from the command line:

```bash
# Build and run with ASan + UBSan (recommended default)
cmake -B build-asan \
    -DCMAKE_BUILD_TYPE=Debug \
    -DENABLE_ASAN=ON \
    -DENABLE_UBSAN=ON
cmake --build build-asan
ctest --test-dir build-asan --output-on-failure

# Build and run with TSan (separate build for thread issues)
cmake -B build-tsan \
    -DCMAKE_BUILD_TYPE=Debug \
    -DENABLE_TSAN=ON
cmake --build build-tsan
ctest --test-dir build-tsan --output-on-failure
```

You really do need two separate build trees if you want both ASan and TSan coverage. That is not a workaround - it is the intended workflow.

### Q2: Show what each sanitizer catches with concrete examples

**Answer:**

Here is a tour of each sanitizer with the exact kind of bug it catches. Pay attention to the error messages in the comments - sanitizer output is surprisingly readable and usually pinpoints the exact line.

```cpp
#include <vector>
#include <thread>
#include <cstring>

// === ASan: catches heap buffer overflow ===
void asan_example() {
    int* arr = new int[10];
    arr[10] = 42;  // OUT OF BOUNDS! ASan reports:
    // ERROR: AddressSanitizer: heap-buffer-overflow
    // WRITE of size 4 at 0x... is 0 bytes to the right of 40-byte region
    delete[] arr;
}

// === ASan: catches use-after-free ===
void asan_uaf_example() {
    int* p = new int(42);
    delete p;
    *p = 99;  // USE AFTER FREE! ASan reports:
    // ERROR: AddressSanitizer: heap-use-after-free
}

// === ASan: catches stack buffer overflow ===
void asan_stack_example() {
    char buf[10];
    std::strcpy(buf, "This is way too long for the buffer");
    // ERROR: AddressSanitizer: stack-buffer-overflow
}

// === UBSan: catches signed integer overflow ===
void ubsan_example() {
    int x = std::numeric_limits<int>::max();
    x += 1;  // SIGNED OVERFLOW IS UB! UBSan reports:
    // runtime error: signed integer overflow: 2147483647 + 1
}

// === UBSan: catches null dereference ===
void ubsan_null_example() {
    int* p = nullptr;
    *p = 42;  // UNDEFINED BEHAVIOR! UBSan reports:
    // runtime error: null pointer passed as argument 1
}

// === TSan: catches data race ===
void tsan_example() {
    int counter = 0;
    std::thread t1([&] {
        for (int i = 0; i < 1000; ++i) ++counter;  // RACE!
    });
    std::thread t2([&] {
        for (int i = 0; i < 1000; ++i) ++counter;  // RACE!
    });
    t1.join(); t2.join();
    // WARNING: ThreadSanitizer: data race
    // Write of size 4 by thread T1:
    // Previous write of size 4 by thread T2:
}

// === MSan: catches uninitialized read ===
void msan_example() {
    int arr[5];
    // arr not initialized
    if (arr[2] > 0) {  // READING UNINITIALIZED! MSan reports:
        // WARNING: MemorySanitizer: use-of-uninitialized-value
    }
}
```

The key insight is that each of these bugs is *silent* without a sanitizer. Your program might run for hours before the corrupted memory causes an observable symptom - and by then the original cause is long gone. Sanitizers catch the problem at the exact site where it happens.

### Q3: Integrate sanitizers into CI with suppressions and environment tuning

**Answer:**

Running sanitizers in CI is where they really pay off. You want a matrix build that runs ASan+UBSan and TSan on every push, with the sanitizer options set to abort immediately on any error.

```yaml
# === GitHub Actions CI with sanitizer matrix ===
name: Sanitizers
on: [push, pull_request]
jobs:
  sanitizer:
    strategy:
      matrix:
        include:
          - name: ASan+UBSan
            cmake_flags: -DENABLE_ASAN=ON -DENABLE_UBSAN=ON
            env_vars: "ASAN_OPTIONS=detect_leaks=1:halt_on_error=1"
          - name: TSan
            cmake_flags: -DENABLE_TSAN=ON
            env_vars: "TSAN_OPTIONS=halt_on_error=1:second_deadlock_stack=1"

    runs-on: ubuntu-latest
    name: ${{ matrix.name }}
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: |
          cmake -B build -DCMAKE_BUILD_TYPE=Debug ${{ matrix.cmake_flags }}
          cmake --build build -j$(nproc)
      - name: Test
        env:
          ASAN_OPTIONS: ${{ matrix.env_vars }}
        run: ctest --test-dir build --output-on-failure -j$(nproc)
```

Sometimes sanitizers will report issues in third-party code you cannot fix, or benign patterns the sanitizer misidentifies. Suppression files let you silence specific known issues without disabling the sanitizer entirely. Use them sparingly - each suppression is a place where a real bug could hide.

```bash
# === Suppression files for known issues ===

# asan_suppressions.txt
# Suppress leak in third-party library (will fix upstream)
leak:third_party::LibraryInit
leak:libprotobuf

# tsan_suppressions.txt
# Known benign race in logging (write-only, no read dependency)
race:Logger::instance

# ubsan_suppressions.txt
unsigned-integer-overflow:fast_hash.cpp  # Intentional wrapping
```

Finally, the environment variables give you fine-grained control over sanitizer behavior. Setting `halt_on_error=1` is especially important - without it, UBSan defaults to printing a warning and continuing, which means a test might "pass" even with detected undefined behavior.

```bash
# === Environment variables for tuning ===

# ASan options
export ASAN_OPTIONS="\
    detect_leaks=1:\
    halt_on_error=1:\
    check_initialization_order=1:\
    detect_stack_use_after_return=1:\
    suppressions=asan_suppressions.txt"

# UBSan options
export UBSAN_OPTIONS="\
    print_stacktrace=1:\
    halt_on_error=1:\
    suppressions=ubsan_suppressions.txt"

# TSan options
export TSAN_OPTIONS="\
    halt_on_error=1:\
    second_deadlock_stack=1:\
    history_size=7:\
    suppressions=tsan_suppressions.txt"
```

With `halt_on_error=1` in place, any sanitizer violation causes the process to exit non-zero, which CTest treats as a test failure. That is exactly the behavior you want in CI.

---

## Notes

- Always run ASan+UBSan in CI - they catch the most common C++ bugs with acceptable overhead.
- TSan requires a separate build because it is incompatible with ASan.
- MSan is Clang-only and requires all dependencies built with MSan (including libc++) - it is the hardest sanitizer to set up correctly.
- Use `-fno-sanitize-recover=all` with UBSan to make UB errors fatal, not just warnings.
- `-fno-omit-frame-pointer` gives much better stack traces in sanitizer reports and is worth the tiny performance cost.
- ASan's `detect_leaks=1` enables LeakSanitizer (LSan) and reports memory leaks at exit.
- Sanitizers work best with Debug or RelWithDebInfo builds so stack traces are readable.
- Never use sanitizer builds for performance benchmarking - the overhead is too high to be meaningful.
- Hardware-assisted ASan (HWASan) is available on AArch64 for lower overhead in production monitoring.
