# Use perf annotate and flamegraphs for source-level hotspot analysis

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://www.brendangregg.com/flamegraphs.html>  

---

## Topic Overview

`perf` is Linux's primary profiling tool, and combined with flamegraphs it gives you a visual map of where your CPU time is actually going. The important thing to understand is that `perf` works by *sampling* the call stack at high frequency - typically hundreds of times per second - and building a statistical picture of which functions were executing most often. The wider a bar in a flamegraph, the more samples landed in that function, which means the more time the CPU spent there.

The workflow and reading guide below are worth committing to memory:

```cpp
Workflow:

  1. perf record -g ./app         (sample call stacks)
  2. perf script > out.perf        (dump samples)
  3. stackcollapse-perf.pl < out.perf | flamegraph.pl > flame.svg
  4. Open flame.svg in browser     (interactive flamegraph)

Flamegraph reading:
  Width = % of total samples (wider = hotter)
  Y-axis = call stack depth (bottom = main, top = leaf)
  Color = random (no meaning)

  |  malloc  |   free   |
  |    process_data()   |  log()  |
  |         main()                 |
  +------------ 100% CPU ----------+
  process_data() = 70% of CPU time (hottest stack)
```

Here is a quick reference for the `perf` subcommands you'll use most often:

| Tool | Purpose | Key flags |
| --- | --- | --- |
| `perf record` | Sample CPU events | `-g` (call graph), `-F 99` (99Hz) |
| `perf report` | Interactive TUI browser | `--sort=dso,symbol` |
| `perf annotate` | Assembly-level hotspots | `--symbol=func_name` |
| `perf stat` | Hardware counter summary | `-e cycles,instructions,cache-misses` |
| flamegraph.pl | Visualize as SVG | Brendan Gregg's FlameGraph repo |

---

## Self-Assessment

### Q1: Record perf profile and generate flamegraph

Here is a sample program designed so that most time lands in `hot_function`. After you've seen what a clear flamegraph looks like with obvious hotspots, you'll have a reference point for when the picture is messier in a real application.

```cpp
#include <iostream>
#include <vector>
#include <cmath>
#include <algorithm>
#include <numeric>

// Sample program to profile:
void hot_function(std::vector<double>& data) {
    for (auto& x : data)
        x = std::sin(x) * std::cos(x) + std::sqrt(std::abs(x));
}

void cold_function(std::vector<double>& data) {
    std::sort(data.begin(), data.end());
}

int main() {
    std::vector<double> data(10'000'000);
    std::iota(data.begin(), data.end(), 0.1);

    for (int i = 0; i < 10; ++i) {
        hot_function(data);   // ~90% of time
        cold_function(data);  // ~10% of time
    }
    std::cout << data[0] << '\n';
}

// Step-by-step flamegraph generation:
//
// 1. Compile with debug info (keep optimizations!):
//    g++ -O2 -g -fno-omit-frame-pointer -o app test.cpp -lm
//    // -g: debug symbols for source mapping
//    // -fno-omit-frame-pointer: reliable stack unwinding
//
// 2. Record samples with call stacks:
//    perf record -g -F 997 -- ./app
//    // -g: capture call graphs (stacks)
//    // -F 997: sample at 997 Hz (prime to avoid aliasing)
//    // Generates perf.data (~10-100 MB)
//
// 3. Quick check with perf report:
//    perf report --sort=symbol --no-children
//    // Shows: 90.2%  hot_function
//    //         9.1%  cold_function (std::sort internals)
//
// 4. Generate flamegraph:
//    perf script > out.perf
//    stackcollapse-perf.pl out.perf > out.folded
//    flamegraph.pl out.folded > flame.svg
//    firefox flame.svg
//
// 5. Read the flamegraph:
//    - main() is at the bottom (widest bar = 100%)
//    - hot_function() is a wide tower (90% of width)
//    - Inside hot_function: sin(), cos(), sqrt() are visible
//    - cold_function() is a narrow tower (10%)
//    - Click any box to zoom in
```

Notice that you compile with `-O2` and also with `-g`. The debug symbols are for source mapping - they don't prevent optimization. If you compile without `-O2`, you're profiling code that doesn't resemble what users run, and the results are misleading.

### Q2: `perf annotate` for assembly-level hotspots

Flamegraphs tell you *which function* is hot. `perf annotate` goes one level deeper and tells you *which instruction* is consuming cycles. This is how you distinguish between a function that's slow because of heavy computation versus one that's slow because of cache misses.

```cpp
#include <iostream>
#include <vector>

// This function has a cache miss hotspot:
void sum_columns(const int* matrix, int* result, int rows, int cols) {
    for (int j = 0; j < cols; ++j)
        for (int i = 0; i < rows; ++i)
            result[j] += matrix[i * cols + j];  // stride = cols (cache unfriendly)
}

int main() {
    constexpr int R = 4096, C = 4096;
    std::vector<int> mat(R * C, 1), res(C, 0);
    for (int iter = 0; iter < 10; ++iter)
        sum_columns(mat.data(), res.data(), R, C);
    std::cout << res[0] << '\n';  // R * 10
}

// Using perf annotate:
//
// 1. g++ -O2 -g -fno-omit-frame-pointer -o app test.cpp
// 2. perf record -g -- ./app
// 3. perf annotate --symbol=sum_columns
//
// Output (annotated assembly):
//    Percent | Source code / Assembly
//    --------+-------------------------------------------
//     0.50%  |   mov    eax, DWORD PTR [rsi+rcx*4]    ; load result[j]
//    87.30%  |   add    eax, DWORD PTR [rdi+rdx*4]    ; load matrix[i*cols+j]  <-- HOTSPOT!
//     2.10%  |   mov    DWORD PTR [rsi+rcx*4], eax    ; store result[j]
//     0.80%  |   add    rdx, 1                        ; i++
//
// The 87.3% on the `add` instruction means:
//   - The CPU stalls HERE waiting for data from memory
//   - Column-major access pattern causes cache misses
//   - Each access jumps 4096*4 = 16KB -> misses L1 (32KB) after 2 rows!
//
// Fix: transpose the loop order (row-major access):
//   for (int i = 0; i < rows; ++i)
//     for (int j = 0; j < cols; ++j)
//       result[j] += matrix[i * cols + j];  // stride = 1 (sequential)
//
// After fix: the 87% hotspot drops to ~5% (hits L1 cache).
```

The 87.3% concentration on a single `add` instruction is a classic cache-miss fingerprint. The instruction itself is trivial - an integer add - but the CPU is stalled for hundreds of cycles waiting for `matrix[i*cols+j]` to arrive from DRAM. Swapping the loop order makes that access sequential and eliminates most of the stalls.

### Q3: CPU cycles, instructions, and IPC

IPC (Instructions Per Cycle) is one of the most useful quick diagnostics for understanding what kind of problem you have. A high IPC means the CPU is executing instructions efficiently. A low IPC means it's spending most of its time waiting - for memory, for branches to resolve, or for dependent instructions to complete.

```cpp
#include <iostream>

// perf stat measures hardware performance counters:
//   perf stat -e cycles,instructions,cache-misses,branch-misses ./app
//
// Key metrics:
//
// CYCLES: number of CPU clock ticks consumed
//   - 1 GHz CPU: 1 billion cycles/second
//   - More cycles = slower (stalls, waits)
//
// INSTRUCTIONS: number of CPU instructions retired
//   - Each add, mov, cmp, jmp counts as 1 instruction
//   - More instructions = more work done
//
// IPC (Instructions Per Cycle): instructions / cycles
//   - IPC = 1.0: average (one instruction per clock)
//   - IPC > 3.0: excellent (superscalar execution, many in-flight)
//   - IPC < 0.5: bad (stalled on memory, branches, or dependencies)
//
// Modern CPUs can execute 4-6 instructions/cycle (superscalar).
// Low IPC means the CPU is waiting, not computing.

int main() {
    std::cout << "IPC interpretation guide:\n";
    std::cout << "+--------+-----------------------------+----------------------------+\n";
    std::cout << "| IPC    | Diagnosis                   | Action                     |\n";
    std::cout << "+--------+-----------------------------+----------------------------+\n";
    std::cout << "| < 0.5  | Memory-bound (cache misses) | Improve data locality      |\n";
    std::cout << "| 0.5-1  | Branch-heavy or dependency  | Reduce branches, unroll    |\n";
    std::cout << "| 1-2    | Reasonable                  | Moderate optimization room |\n";
    std::cout << "| 2-4    | Good utilization            | Near optimal               |\n";
    std::cout << "| > 4    | Excellent (SIMD-heavy)      | ~Theoretically optimal     |\n";
    std::cout << "+--------+-----------------------------+----------------------------+\n";

    // Example perf stat output:
    //   perf stat ./matrix_multiply
    //
    //   Performance counter stats for './matrix_multiply':
    //      12,345,678,901  cycles         # 3.5 GHz
    //       5,432,109,876  instructions   # 0.44 IPC  <-- BAD! memory-bound
    //       1,234,567,890  cache-misses   # High!
    //
    //   After optimization (tiled):
    //      3,456,789,012   cycles         # 3.5 GHz
    //      5,432,109,876   instructions   # 1.57 IPC  <-- GOOD! 3.5x improvement
    //         12,345,678   cache-misses   # 100x fewer
}
```

The before/after comparison in the comments is instructive. The instruction count stays the same - you're doing the same computation - but the cycle count drops by 3.5x because you stopped waiting on cache misses. This is the typical pattern for memory-bound code: optimization reduces cycles without changing instructions.

---

## Notes

- Always compile with `-g -fno-omit-frame-pointer` for accurate profiling.
- `perf record -F 997` uses prime frequency to avoid sampling bias.
- Flamegraphs show WHERE time is spent; `perf annotate` shows WHY (which instructions stall).
- Low IPC + high cache misses = memory-bound. Low IPC + high branch misses = branch-bound.
- FlameGraph scripts: `git clone https://github.com/brendangregg/FlameGraph`.
