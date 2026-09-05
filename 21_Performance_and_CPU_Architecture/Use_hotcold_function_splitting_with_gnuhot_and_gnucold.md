# Use hot/cold function splitting with [[gnu::hot]] and [[gnu::cold]]

**Category:** Performance and CPU Architecture  
**Standard:** C++11  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html>  

---

## Topic Overview

The `[[gnu::hot]]` and `[[gnu::cold]]` attributes are your way of communicating execution frequency to the compiler and linker. `[[gnu::hot]]` says "optimize this aggressively and put it somewhere the instruction cache can easily keep it warm." `[[gnu::cold]]` says "this function rarely runs - minimize its footprint and push it away from the hot code."

The payoff comes from how the linker groups sections. All `.text.hot` sections from every translation unit end up adjacent in the final binary. All `.text.unlikely` sections end up in a completely separate region. The hot working set is therefore physically contiguous in virtual memory, which means it occupies the fewest possible icache lines.

```cpp
Object file layout with hot/cold sections:

.text.hot:            (icache-dense, fits together)
  fast_path()
  process_packet()
  decode_frame()

.text.unlikely:       (pushed away, never pollutes icache)
  handle_error()
  log_failure()
  print_diagnostics()

Linker groups .text.hot sections from ALL .o files together.
Result: the entire hot working set is contiguous in virtual memory.
```

| Attribute | Section | Effect on optimizer |
| --- | --- | --- |
| `[[gnu::hot]]` | `.text.hot` | Aggressive optimization, inline more |
| `[[gnu::cold]]` | `.text.unlikely` | Minimal optimization, never inline into hot |
| (default) | `.text` | Normal optimization |

---

## Self-Assessment

### Q1: Mark fast path `[[gnu::hot]]` and error handler `[[gnu::cold]]`, inspect layout

The two attributes work together: `[[gnu::hot]]` on the main processing function tells the compiler to apply its most aggressive optimizations and place it in the hot section, while `[[gnu::cold]]` on the error handler keeps that bulky code out of the hot path entirely:

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
#include <chrono>

// The fast path: [[gnu::hot]] tells the compiler:
//  1. Place in .text.hot section
//  2. Apply aggressive optimization (more inlining, unrolling)
//  3. Prefer this function for icache residency
[[gnu::hot]]
int fast_path(const int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += data[i];
        if (data[i] < 0) [[unlikely]] {
            handle_negative(i, data[i]);  // call to cold function
        }
    }
    return sum;
}

// The error handler: [[gnu::cold]] tells the compiler:
//  1. Place in .text.unlikely section
//  2. Never inline this into hot callers
//  3. Optimize for size, not speed
[[gnu::cold]] [[gnu::noinline]]
void handle_negative(int idx, int val) {
    std::cerr << "Warning: negative value " << val
              << " at index " << idx << '\n';
    // This 100+ bytes of code stays OUT of the hot loop
}

// Inspect the layout:
//   g++ -O2 -ffunction-sections -c test.cpp
//   readelf -S test.o | grep text
//     .text.hot.fast_path            (hot section)
//     .text.unlikely.handle_negative (cold section)
//
//   objdump -d test.o:
//     fast_path: compact loop, no inlined error handling
//     handle_negative: separate section, full error output

int main() {
    std::vector<int> data(1'000'000, 42);
    auto t0 = std::chrono::high_resolution_clock::now();
    volatile int s = 0;
    for (int r = 0; r < 100; ++r)
        s = fast_path(data.data(), static_cast<int>(data.size()));
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    std::cout << "Time: " << ms.count() << " ms\n";
}
```

The `readelf` and `objdump` commands in the comments are the way to verify this actually worked. If you see `.text.hot.fast_path` and `.text.unlikely.handle_negative` in the section list, the placement is correct.

### Q2: How cold functions go to `.text.unlikely` for icache improvement

To understand why this matters, it helps to think about how the linker lays out the binary and how the instruction cache works. Functions that call each other frequently benefit from being physically adjacent - if `compute` and `process` both fit in the same few icache lines, they can stay resident together:

```cpp
#include <iostream>

// The linker groups sections by name prefix:
//   .text.hot.*      -> contiguous block at low addresses
//   .text.*          -> normal code
//   .text.unlikely.* -> pushed to high addresses
//
// This means:
//   - All hot functions across ALL translation units are adjacent
//   - Cold functions are far away, never loaded into icache
//   - icache hit rate improves dramatically for hot paths

[[gnu::hot]] int compute(int x) { return x * x + 1; }  // .text.hot
[[gnu::hot]] int process(int x) { return compute(x) * 2; }  // .text.hot (adjacent!)

[[gnu::cold]] void log_error(int code) {  // .text.unlikely
    std::cerr << "Error: " << code << '\n';
}

[[gnu::cold]] void print_trace() {  // .text.unlikely (adjacent to log_error)
    std::cerr << "Stack trace...\n";
}

// Memory layout after linking:
// Address 0x400000: compute()      } .text.hot
// Address 0x400040: process()      } icache lines are contiguous!
// ...                              } (adjacent in L1 icache)
// Address 0x4F0000: log_error()    } .text.unlikely
// Address 0x4F0080: print_trace()  } far away, never loaded unless needed

// Verify:
//   g++ -O2 -ffunction-sections -Wl,--sort-section=name -o test test.cpp
//   nm --numeric-sort test | grep -E 'compute|process|log_error|print_trace'

int main() {
    // Hot path: compute + process are in adjacent icache lines
    volatile int r = 0;
    for (int i = 0; i < 1000; ++i)
        r = process(i);
    std::cout << r << '\n';
}
```

The `nm --numeric-sort` command gives you the symbol addresses in order, so you can confirm that `compute` and `process` are adjacent while `log_error` and `print_trace` are at much higher addresses. That address gap is the physical separation between the hot and cold regions of your binary.

### Q3: Explicit hot/cold splitting vs. PGO-based automatic splitting

When should you use manual attributes versus setting up a full PGO build? The answer depends on how confident you are about which code is actually hot and whether you can afford the build complexity.

```cpp
#include <iostream>

int main() {
    std::cout << "Explicit vs PGO hot/cold splitting comparison:\n\n";

    std::cout << "+-------------------+-----------------------------+-----------------------------+\n";
    std::cout << "| Feature           | Explicit [[gnu::hot/cold]]  | PGO-based (-fprofile-use)   |\n";
    std::cout << "+-------------------+-----------------------------+-----------------------------+\n";
    std::cout << "| Accuracy          | Programmer's guess          | Measured execution counts   |\n";
    std::cout << "| Granularity       | Whole functions only        | Basic blocks within funcs   |\n";
    std::cout << "| Maintenance       | Manual, can become stale    | Automatic with profile data |\n";
    std::cout << "| Build complexity  | Normal build                | 3-step build (gen/run/use)  |\n";
    std::cout << "| Branch prediction | No help                     | Teaches branch predictor    |\n";
    std::cout << "| Inlining          | Hints only                  | Data-driven decisions       |\n";
    std::cout << "| Function splitting | No (whole function placed)  | Yes (splits within func)    |\n";
    std::cout << "+-------------------+-----------------------------+-----------------------------+\n";

    // When to use each:
    //
    // USE EXPLICIT ATTRIBUTES when:
    //  1. PGO infrastructure not available
    //  2. Error handlers are clearly cold (100% certain)
    //  3. Library code where profile data varies by user
    //  4. Quick wins without build system changes
    //
    // USE PGO when:
    //  1. Large codebase where manual annotation is impractical
    //  2. Need basic-block-level splitting (not just functions)
    //  3. Want branch prediction hints automatically
    //  4. Can set up representative benchmark workload
    //
    // BEST: Use BOTH together!
    //  - Manual [[cold]] for obvious error paths
    //  - PGO for data-driven splitting of everything else
    //  - PGO respects explicit attributes (won't contradict them)

    // PGO + BOLT pipeline (state of the art):
    //  g++ -O2 -fprofile-generate -o app.instr app.cpp
    //  ./app.instr <workload>
    //  g++ -O2 -fprofile-use -o app.pgo app.cpp
    //  perf record -e cycles:u -j any,u -- ./app.pgo <workload>
    //  llvm-bolt app.pgo -o app.bolt -data=perf.data -reorder-functions
    //  // app.bolt: 10-20% faster than app.pgo for large binaries
}
```

The "use both together" recommendation is the practical takeaway. Explicit `[[gnu::cold]]` on obvious error paths is essentially free effort - error handlers are always cold, and marking them takes one line. PGO then handles the less obvious cases where your intuition might be wrong. The two approaches are complementary: PGO will not override explicit attributes, so you get the best of both.

---

## Notes

- `[[gnu::hot]]` + `[[gnu::cold]]` are complementary to #628 (splitting technique) and #546 (attribute basics).
- Compile with `-ffunction-sections -Wl,--sort-section=name` for best section grouping.
- `readelf -S <binary> | grep text` shows section layout.
- GCC also supports `__attribute__((hot))` / `__attribute__((cold))` (older syntax).
- Clang supports both `[[gnu::hot/cold]]` and `__attribute__((hot/cold))`.
