# Use hot/cold function splitting to improve instruction cache density

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html>  

---

## Topic Overview

Every function in your binary occupies instruction cache space. The problem with error handling and logging code is that it lives right next to the hot loop that runs millions of times per second - even though the error path executes maybe once every few minutes. Those rarely-executed bytes still take up cache lines that the hot loop could be using.

Hot/cold splitting fixes this by physically moving the cold code to a different section of the binary. The hot path becomes a compact, contiguous block of instructions that fits in fewer cache lines:

```cpp
Without splitting:                    With splitting:
.text:                                .text.hot:
  [process entry]                       [process entry]
  [hot loop body]                       [hot loop body]
  [error handler - 200 bytes]           [process exit]
  [hot loop body continued]           .text.unlikely:
  [logging - 500 bytes]                 [error handler]
  [process exit]                        [logging]
  Total hot path: fragmented            Total hot path: compact!
  icache lines: scattered               icache lines: contiguous
```

| Technique | Mechanism |
| --- | --- |
| `[[gnu::cold]]` function | Entire function placed in `.text.unlikely` |
| `noinline` + cold | Prevent cold code from bloating hot functions |
| PGO (`-fprofile-use`) | Compiler automatically splits based on profile |
| `-freorder-functions` | GCC groups hot functions together |

---

## Self-Assessment

### Q1: `[[gnu::cold]]` on error paths for icache improvement

When error-handling code is inlined into the hot function, the compiler inserts the full error handler body directly inside the loop. Even with `[[unlikely]]` hints, those bytes still occupy icache space. Extracting the error handler into a separate `[[gnu::cold]]` function removes it from the hot path entirely:

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
#include <chrono>

// WITHOUT splitting: error handling code is inlined into hot function
[[gnu::hot]] int process_no_split(const int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] < 0) {
            // This error handling code is INSIDE the hot function!
            // It takes ~200 bytes of icache even though it rarely executes.
            std::cerr << "Error at index " << i
                      << ": negative value " << data[i] << '\n';
            throw std::runtime_error("negative value");
        }
        sum += data[i];
    }
    return sum;
}

// WITH splitting: error handling extracted to cold function
[[gnu::cold]] [[noreturn]]
void report_error(int idx, int val) {
    std::cerr << "Error at index " << idx
              << ": negative value " << val << '\n';
    throw std::runtime_error("negative value");
}

[[gnu::hot]] int process_split(const int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] < 0) [[unlikely]] {
            report_error(i, data[i]);  // cold call, out of hot section
        }
        sum += data[i];
    }
    return sum;
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N, 42);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile int s = 0;
        for (int r = 0; r < 100; ++r) s = fn(data.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_no_split, "No split ");
    bench(process_split,    "Split    ");
    // Split version: hot loop is more compact -> fewer icache misses
    // Improvement: ~5-15% for tight loops with error handling
}
```

The `[[noreturn]]` annotation on `report_error` tells the compiler this function never returns normally (it always throws), which allows slightly better code generation in the caller - the compiler knows the code after the call is unreachable on the cold path.

### Q2: `noinline` on cold paths prevents hot-function bloating

`[[gnu::cold]]` alone is not always enough. If the cold function is small enough, the compiler might inline it into the hot caller anyway - which defeats the purpose. Adding `[[gnu::noinline]]` forces the cold code to remain a separate call:

```cpp
#include <iostream>
#include <string>

// Problem: compiler may INLINE cold helper into hot function,
// which bloats the hot function's code size.

// GOOD: [[gnu::noinline]] prevents inlining of cold code
[[gnu::cold]] [[gnu::noinline]]
void log_debug(const char* msg, int val) {
    // This code is 100+ bytes of instructions.
    // Without noinline, compiler might inline it into hot function!
    std::string s = "DEBUG: ";
    s += msg;
    s += " value=";
    s += std::to_string(val);
    std::cerr << s << '\n';
}

[[gnu::cold]] [[gnu::noinline]]
void handle_boundary(int idx, int n) {
    std::cerr << "Boundary: idx=" << idx << " n=" << n << '\n';
}

[[gnu::hot]] int hot_compute(const int* data, int n) {
    int result = 0;
    for (int i = 0; i < n; ++i) {
        result += data[i] * data[i];

        // Without noinline: log_debug's code would be copied HERE,
        // expanding the loop body by ~100 bytes.
        // With noinline: just a `call` instruction (5 bytes).
        if (result > 1'000'000) [[unlikely]] {
            log_debug("overflow", result);
        }
        if (i == n - 1) [[unlikely]] {
            handle_boundary(i, n);
        }
    }
    return result;
}

// To verify:
//   g++ -O2 -S test.cpp
//   Look at hot_compute in assembly:
//   - With noinline: loop body is ~20 instructions (compact)
//   - Without noinline: loop body is ~80 instructions (bloated)
//   The compact version uses fewer icache lines.

int main() {
    int data[] = {1, 2, 3, 4, 5};
    std::cout << hot_compute(data, 5) << '\n';  // 55
}
```

The assembly comment gives you the key metric: a 4x difference in loop body size directly translates to fewer icache lines needed to hold the hot path. When the hot loop fits in 2-3 cache lines instead of 8-10, the instruction cache can hold more of your hot working set and misses drop accordingly.

### Q3: PGO-based automatic hot/cold section grouping

Profile-Guided Optimization (PGO) automates the hot/cold analysis. Instead of guessing which paths are hot, the compiler instruments the binary, you run it on representative input, and then the compiler uses the measured execution counts to make splitting decisions - including splitting *within* a single function, not just at function boundaries.

```cpp
#include <iostream>

// Profile-Guided Optimization (PGO) automates hot/cold splitting:
//
// Step 1: Instrumented build
//   g++ -O2 -fprofile-generate -o app app.cpp
//
// Step 2: Run with representative workload
//   ./app <typical_input>
//   // Generates .gcda profile data files
//
// Step 3: Optimized build using profile
//   g++ -O2 -fprofile-use -o app app.cpp
//
// What the compiler does with profile data:
//
// 1. FUNCTION REORDERING (-freorder-functions):
//    Hot functions grouped in .text.hot
//    Cold functions in .text.unlikely
//    Result: hot functions are contiguous in memory -> icache friendly
//
// 2. BASIC BLOCK REORDERING (-freorder-blocks):
//    Hot basic blocks placed contiguously (fall-through path)
//    Cold basic blocks (error handling) moved to end of function
//    Result: hot path has no jumps -> better icache and prefetcher
//
// 3. FUNCTION SPLITTING (-freorder-blocks-and-partition):
//    A SINGLE function is split: hot part in .text.hot, cold part in .text.unlikely
//    Example: process() has hot loop + cold error handler
//    -> Hot loop in .text.hot, error handler in .text.unlikely
//
// Linker integration:
//   -ffunction-sections -fdata-sections: each function gets its own section
//   Linker script or --symbol-ordering-file orders sections by hotness
//   Gold/LLD: --section-ordering-file=hot_order.txt

int main() {
    std::cout << "PGO hot/cold splitting workflow:\n";
    std::cout << "  1. g++ -O2 -fprofile-generate -o app app.cpp\n";
    std::cout << "  2. ./app <representative_workload>\n";
    std::cout << "  3. g++ -O2 -fprofile-use -o app app.cpp\n";
    std::cout << "\nCompiler optimizations enabled by PGO:\n";
    std::cout << "  -freorder-functions: group hot functions\n";
    std::cout << "  -freorder-blocks: reorder basic blocks\n";
    std::cout << "  -freorder-blocks-and-partition: split functions\n";
    std::cout << "\nTypical improvement: 10-20% for large codebases\n";
}
```

PGO's advantage over manual annotation is granularity: it can split within a single function, reorder basic blocks within a function, and make inlining decisions based on actual call frequency. The three-step build process is the main friction. It is worth setting up if you have a reproducible representative workload and a large codebase where manual annotation would be impractical.

---

## Notes

- `[[gnu::cold]]` + `[[gnu::noinline]]` is the manual equivalent of PGO splitting.
- PGO is superior because it uses actual execution data, not programmer guesses.
- BOLT (Binary Optimization and Layout Tool) can reorder functions post-link for even better results.
- For large codebases (Chrome, Firefox), PGO + BOLT gives 5-15% overall speedup.
- `perf stat -e L1-icache-load-misses` measures instruction cache pressure.
