# Use UBSan, ASan, MSan, and Static Analyzers for UB Detection

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17, C++20, C++23  
**Reference:** [Clang Sanitizer Documentation](https://clang.llvm.org/docs/index.html)  

---

## Topic Overview

No single tool catches all undefined behavior. Effective UB detection requires layering **runtime sanitizers** (which catch UB during execution) with **static analyzers** (which catch UB at compile time) and **compiler warnings** (which catch common patterns). Each tool covers a different subset of UB categories.

Here is a quick reference for what each tool does and what it costs:

| Tool | Category | Catches | Overhead | False Positives |
| --- | --- | --- | --- | --- |
| **UBSan** | Runtime | Signed overflow, shift UB, null deref, alignment, etc. | 10-30% | Very low |
| **ASan** | Runtime | Heap/stack buffer overflow, use-after-free, double-free, leaks | 2x slowdown, 2-3x memory | Low |
| **MSan** | Runtime | Uninitialized memory reads | 2-3x slowdown | Low (but setup-sensitive) |
| **TSan** | Runtime | Data races, lock-order inversions | 5-15x slowdown, 5-10x memory | Rare |
| **Clang-Tidy** | Static | Patterns, style, some UB | Compile time | Moderate |
| **Clang Static Analyzer** | Static | Path-sensitive analysis, memory bugs | Minutes | Moderate |
| **PVS-Studio** | Static | Deep pattern matching, UB patterns | Minutes | Low-moderate |
| **Compiler warnings** | Compile | Common mistakes, some UB | Zero | Low |

The coverage map below shows which sanitizer detects which class of bug. A Yes means reliable detection; Partial means sometimes caught; No means not detected:

```cpp
Detection Coverage Map:

                  UBSan  ASan  MSan  TSan  Static
Signed overflow   Yes    No    No    No    Partial
Null deref        Yes    Yes   No    No    Yes
OOB access        Partial Yes  No    No    Partial
Use-after-free    No     Yes   No    No    Partial
Uninit read       No     No    Yes   No    Partial
Data race         No     No    No    Yes   Partial
Strict aliasing   No     No    No    No    Partial
Shift UB          Yes    No    No    No    Yes
Division by zero  Yes    No    No    No    Yes
Alignment         Yes    No    No    No    Partial
```

The recommended strategy is: **always build with `-Wall -Wextra -Wpedantic`**, run CI with **ASan + UBSan** together, run **TSan** separately (incompatible with ASan), and run **MSan** for code that processes external input. Complement with **Clang-Tidy** in CI and periodic **static analyzer** sweeps.

---

## Self-Assessment

### Q1: Set up a complete sanitizer build system with CMake

Setting up sanitizers in CMake is straightforward once you have the option flags right. The key decisions are: make UBSan fatal (not just a warning), and keep sanitizer builds as separate CMake configurations since you cannot mix ASan/TSan/MSan in one binary:

```cpp
// CMakeLists.txt content (store as CMakeLists.txt):
/*
cmake_minimum_required(VERSION 3.20)
project(ub_detection LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# Base warning flags (always on)
add_compile_options(
    -Wall -Wextra -Wpedantic -Werror
    -Wconversion -Wsign-conversion
    -Wnull-dereference -Wdouble-promotion
    -Wformat=2 -Wformat-overflow=2
    -Wimplicit-fallthrough
)

# Sanitizer configurations
option(ENABLE_ASAN   "Enable AddressSanitizer"   OFF)
option(ENABLE_UBSAN  "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_MSAN   "Enable MemorySanitizer"    OFF)
option(ENABLE_TSAN   "Enable ThreadSanitizer"    OFF)

if(ENABLE_ASAN)
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address)
endif()

if(ENABLE_UBSAN)
    add_compile_options(
        -fsanitize=undefined
        -fsanitize=float-divide-by-zero
        -fsanitize=float-cast-overflow
        -fno-sanitize-recover=all   # Make UBSan errors fatal
    )
    add_link_options(-fsanitize=undefined)
endif()

if(ENABLE_MSAN)
    add_compile_options(-fsanitize=memory -fno-omit-frame-pointer)
    add_link_options(-fsanitize=memory)
endif()

if(ENABLE_TSAN)
    add_compile_options(-fsanitize=thread)
    add_link_options(-fsanitize=thread)
endif()

add_executable(myproject main.cpp)
*/

// Usage:
// cmake -B build -DENABLE_ASAN=ON -DENABLE_UBSAN=ON
// cmake --build build
// ./build/myproject   // Runs with ASan + UBSan active

// main.cpp: test code with intentional bugs
#include <cstdio>
#include <cstdlib>
#include <vector>

// Bug 1: UBSan catches signed overflow
int overflow_bug(int x) {
    return x + 1;  // UBSan catches when x == INT_MAX
}

// Bug 2: ASan catches heap buffer overflow
void heap_overflow_bug() {
    int* arr = new int[10];
    // arr[10] = 42;  // ASan: heap-buffer-overflow
    delete[] arr;
}

// Bug 3: ASan catches use-after-free
void use_after_free_bug() {
    int* p = new int(42);
    delete p;
    // int val = *p;  // ASan: heap-use-after-free
}

// Bug 4: ASan catches stack buffer overflow
void stack_overflow_bug() {
    int arr[5] = {1, 2, 3, 4, 5};
    // int val = arr[5];  // ASan: stack-buffer-overflow
    (void)arr;
}

// Bug 5: UBSan catches shift UB
int shift_bug(int x, int shift) {
    return x << shift;  // UBSan catches when shift >= 32 or < 0
}

int main() {
    std::printf("Testing sanitizers...\n");

    // Uncomment to trigger specific sanitizers:
    // overflow_bug(INT_MAX);       // UBSan
    // heap_overflow_bug();         // ASan
    // use_after_free_bug();        // ASan
    // stack_overflow_bug();        // ASan
    // shift_bug(1, 32);            // UBSan

    std::printf("All checks passed.\n");
    return 0;
}
```

The CMake configuration provides four sanitizer modes that can be independently enabled. ASan and UBSan can be combined; TSan and MSan must run separately. `-fno-sanitize-recover=all` makes UBSan abort on first error, which is important for CI - without it, UBSan just prints a warning and keeps going, hiding subsequent errors.

---

### Q2: Demonstrate each sanitizer catching a specific bug

Here is a single file that contains one representative bug for each sanitizer. The compile flags differ per sanitizer, so read the comments before building each section:

```cpp
#include <atomic>
#include <climits>
#include <cstdio>
#include <cstring>
#include <thread>
#include <vector>

// === UBSan Examples ===
// Compile: clang++ -std=c++20 -fsanitize=undefined -fno-sanitize-recover=all

void ubsan_signed_overflow() {
    int x = INT_MAX;
    int y = x + 1;  // UBSan: signed integer overflow
    std::printf("overflow: %d\n", y);
}

void ubsan_division_by_zero() {
    int x = 42;
    int zero = 0;
    int y = x / zero;  // UBSan: division by zero
    std::printf("div: %d\n", y);
}

void ubsan_null_dereference() {
    int* p = nullptr;
    int x = *p;  // UBSan: null pointer dereference (also caught by ASan)
    std::printf("null: %d\n", x);
}

void ubsan_alignment() {
    char buf[16] = {};
    int* p = reinterpret_cast<int*>(buf + 1);  // Misaligned
    int x = *p;  // UBSan: misaligned access
    std::printf("misaligned: %d\n", x);
}

// === ASan Examples ===
// Compile: clang++ -std=c++20 -fsanitize=address -fno-omit-frame-pointer

void asan_heap_overflow() {
    std::vector<int> v(10);
    // Force out-of-bounds: bypass vector's bounds checking
    int* raw = v.data();
    int x = raw[10];  // ASan: heap-buffer-overflow
    std::printf("oob: %d\n", x);
}

void asan_use_after_free() {
    int* p = new int(42);
    delete p;
    int x = *p;  // ASan: heap-use-after-free
    std::printf("uaf: %d\n", x);
}

void asan_stack_use_after_scope() {
    int* p;
    {
        int local = 42;
        p = &local;
    }
    int x = *p;  // ASan: stack-use-after-scope (with -fsanitize-address-use-after-scope)
    std::printf("uas: %d\n", x);
}

// === MSan Example ===
// Compile: clang++ -std=c++20 -fsanitize=memory -fno-omit-frame-pointer
// NOTE: MSan requires ALL libraries to be instrumented (including libc++).

void msan_uninitialized_read() {
    int arr[10];
    // arr is not initialized
    int sum = 0;
    for (int i = 0; i < 10; ++i) {
        sum += arr[i];  // MSan: use of uninitialized value
    }
    std::printf("uninit sum: %d\n", sum);
}

// === TSan Example ===
// Compile: clang++ -std=c++20 -fsanitize=thread

int shared_data = 0;

void tsan_data_race() {
    auto writer = []() {
        for (int i = 0; i < 1000; ++i) {
            shared_data = i;  // TSan: data race
        }
    };

    std::thread t1(writer);
    std::thread t2(writer);
    t1.join();
    t2.join();
    std::printf("race result: %d\n", shared_data);
}

int main() {
    // Uncomment ONE group at a time, matching the compile flags:

    // UBSan tests:
    // ubsan_signed_overflow();
    // ubsan_division_by_zero();
    // ubsan_null_dereference();
    // ubsan_alignment();

    // ASan tests:
    // asan_heap_overflow();
    // asan_use_after_free();
    // asan_stack_use_after_scope();

    // MSan test:
    // msan_uninitialized_read();

    // TSan test:
    // tsan_data_race();

    std::printf("Done.\n");
}
```

Each sanitizer catches a specific category. UBSan catches language-level UB (overflow, shift, alignment). ASan catches memory errors (OOB, use-after-free, leaks). MSan catches uninitialized reads. TSan catches data races. They cannot all run simultaneously - ASan+UBSan is one build, TSan is another, MSan is a third.

---

### Q3: Integrate sanitizers and static analysis into CI

A complete CI pipeline needs three separate sanitizer jobs because the sanitizers are mutually incompatible. You also need suppression files for known false positives - real-world projects always have a few:

```cpp
// .github/workflows/sanitizers.yml (GitHub Actions)
/*
name: Sanitizer CI
on: [push, pull_request]

jobs:
  asan-ubsan:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4
      - name: Build with ASan + UBSan

        run: |
          cmake -B build \
            -DCMAKE_CXX_COMPILER=clang++ \
            -DENABLE_ASAN=ON \
            -DENABLE_UBSAN=ON
          cmake --build build

      - name: Run tests

        env:
          ASAN_OPTIONS: "detect_leaks=1:halt_on_error=1:print_stats=1"
          UBSAN_OPTIONS: "halt_on_error=1:print_stacktrace=1"
        run: cd build && ctest --output-on-failure

  tsan:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4
      - name: Build with TSan

        run: |
          cmake -B build \
            -DCMAKE_CXX_COMPILER=clang++ \
            -DENABLE_TSAN=ON
          cmake --build build

      - name: Run tests

        env:
          TSAN_OPTIONS: "halt_on_error=1:second_deadlock_stack=1"
        run: cd build && ctest --output-on-failure

  static-analysis:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4
      - name: Clang-Tidy

        run: |
          cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
          clang-tidy -p build \
            --checks='-*,bugprone-*,cert-*,cppcoreguidelines-*,\
              clang-analyzer-*,performance-*,readability-*' \
            --warnings-as-errors='bugprone-*,cert-*' \
            src/*.cpp
*/

// Suppression files for known false positives:

// --- ASan suppression (asan_suppressions.txt) ---
/*
# Suppress known leak in third-party library
leak:libthirdparty.so
# Suppress known issue in test framework
leak:testing::internal
*/

// --- UBSan suppression (ubsan_suppressions.txt) ---
/*
# Suppress known signed overflow in legacy parser
signed-integer-overflow:legacy_parser.cpp
# Suppress alignment issue in packed network struct handler
alignment:network_handler.cpp
*/

// --- TSan suppression (tsan_suppressions.txt) ---
/*
# Suppress known benign race in logging library
race:Logger::instance
# Suppress race in third-party lock-free queue
race:mpmc_queue::push
*/

// C++ code for suppression management:
#include <cstdio>
#include <cstdlib>

// Runtime suppression check
void configure_sanitizers() {
    // These environment variables control sanitizer behavior:
    //
    // ASAN_OPTIONS:
    //   detect_leaks=1           Enable leak detection
    //   halt_on_error=1          Abort on first error
    //   suppressions=file.txt    Load suppression file
    //   allocator_may_return_null=1  Return null instead of crashing on OOM
    //   check_initialization_order=1 Detect static init order fiasco
    //
    // UBSAN_OPTIONS:
    //   halt_on_error=1          Abort on first error
    //   print_stacktrace=1       Print stack trace on each error
    //   suppressions=file.txt    Load suppression file
    //
    // TSAN_OPTIONS:
    //   halt_on_error=1          Abort on first race
    //   second_deadlock_stack=1  Show both stacks in deadlock
    //   suppressions=file.txt    Load suppression file
    //   history_size=7           Increase history for better reports

    // Programmatic check: are sanitizers active?
    #if defined(__SANITIZE_ADDRESS__)
        std::printf("ASan is active\n");
    #endif
    #if defined(__has_feature)
        #if __has_feature(thread_sanitizer)
            std::printf("TSan is active\n");
        #endif
        #if __has_feature(memory_sanitizer)
            std::printf("MSan is active\n");
        #endif
    #endif
}

// Annotating intentional behavior to suppress false positives:
#if defined(__clang__)
    #define NO_SANITIZE_ADDRESS __attribute__((no_sanitize("address")))
    #define NO_SANITIZE_THREAD  __attribute__((no_sanitize("thread")))
    #define NO_SANITIZE_UB      __attribute__((no_sanitize("undefined")))
#elif defined(__GNUC__)
    #define NO_SANITIZE_ADDRESS __attribute__((no_sanitize_address))
    #define NO_SANITIZE_THREAD  __attribute__((no_sanitize_thread))
    #define NO_SANITIZE_UB      __attribute__((no_sanitize("undefined")))
#else
    #define NO_SANITIZE_ADDRESS
    #define NO_SANITIZE_THREAD
    #define NO_SANITIZE_UB
#endif

// Example: benign race for statistics counter (acceptable loss)
NO_SANITIZE_THREAD
void increment_stats_counter(int& counter) {
    ++counter;  // Benign race: we accept approximate stats
}

int main() {
    configure_sanitizers();
    std::printf("Sanitizer CI integration ready.\n");
}
```

A complete CI pipeline runs three separate sanitizer builds (ASan+UBSan, TSan, static analysis) because sanitizers are mutually incompatible. Suppression files manage known false positives. Function-level attributes (`no_sanitize`) suppress specific checks for documented acceptable behavior. Environment variables control runtime behavior.

---

## Notes

- **ASan + UBSan can run together** in one build. TSan and MSan must be separate builds (they conflict with ASan and each other).
- **MSan requires instrumented libraries:** all code including libc++ must be built with MSan. Use `-stdlib=libc++ -fsanitize=memory` with Clang's instrumented libc++ build.
- `-fno-sanitize-recover=all` is critical for CI: without it, UBSan prints a warning and continues, hiding subsequent errors.
- **`ASAN_OPTIONS=detect_stack_use_after_return=1`** (was opt-in before Clang 16, now default) catches stack-use-after-return bugs at the cost of significant memory overhead.
- For **Windows/MSVC**, ASan is supported (`/fsanitize=address`), but UBSan, MSan, and TSan are not. Use Clang-cl or WSL for full sanitizer coverage.
- Run sanitizer builds with **your full test suite** - sanitizers only catch UB that is actually executed. Increase code coverage to maximize sanitizer effectiveness.
- **Clang-Tidy checks** relevant to UB: `bugprone-signed-char-misuse`, `bugprone-integer-division`, `bugprone-undefined-memory-manipulation`, `cert-err34-c`, `cppcoreguidelines-pro-type-reinterpret-cast`.
