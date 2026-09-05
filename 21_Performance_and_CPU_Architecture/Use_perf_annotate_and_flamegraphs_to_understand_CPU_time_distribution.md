# Use perf annotate and flamegraphs to understand CPU time distribution

**Category:** Performance and CPU Architecture  
**Standard:** Not version-specific  
**Reference:** <https://www.brendangregg.com/flamegraphs.html>  

---

## Topic Overview

This topic focuses on the practical workflow of using `perf` for CPU time analysis and diagnosing cache miss hotspots - complementing #635 which covers the fundamentals.

While #635 covers the basic record/report/flamegraph pipeline, here you'll use `perf mem` and `perf c2c` to go beyond cycle sampling and pinpoint exactly *which memory accesses* are causing your stalls, and from which cache level the misses are occurring.

```cpp
Perf workflow for cache analysis:
  perf record -e cache-misses -g ./app      (sample on cache miss events)
  perf mem record ./app                     (detailed memory access profiling)
  perf mem report --sort=symbol,mem         (which symbols cause misses + NUMA info)
  perf c2c record ./app                     (false sharing detection)
```

| perf subcommand | What it profiles | Best for |
| --- | --- | --- |
| `perf stat` | Summary counters | Quick overview |
| `perf record` | Sampled stacks | Flamegraphs, hotspot ID |
| `perf annotate` | Assembly per-instruction | Identifying exact stall |
| `perf mem` | Memory access patterns | Cache miss sources |
| `perf c2c` | Cache-to-cache transfers | False sharing detection |

---

## Self-Assessment

### Q1: Generate flamegraph from `perf record` output

This example creates a program with three distinct hotspots at roughly 50/40/10 percent. After generating the flamegraphs, you'll also see how to create a *differential* flamegraph that visually highlights regressions (red) and improvements (blue) between two runs.

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// Complex program with multiple hotspots:
void compute_heavy(std::vector<double>& v) {
    for (auto& x : v) x = x * x + 1.0 / (x + 1.0);
}

void sort_heavy(std::vector<double>& v) {
    std::sort(v.begin(), v.end());
}

void alloc_heavy() {
    for (int i = 0; i < 1000; ++i) {
        auto p = new double[10000];
        delete[] p;
    }
}

int main() {
    std::vector<double> data(5'000'000, 3.14);
    for (int i = 0; i < 5; ++i) {
        compute_heavy(data);  // expect: 50% of samples
        sort_heavy(data);     // expect: 40% of samples
        alloc_heavy();        // expect: 10% of samples
    }
}

// Complete flamegraph workflow:
//
// # Build:
// g++ -O2 -g -fno-omit-frame-pointer -o app test.cpp
//
// # Record (multiple event types for different views):
// perf record -g -F 999 -o perf_cpu.data -- ./app          # CPU time
// perf record -e cache-misses -g -o perf_cache.data -- ./app # cache misses
//
// # Generate CPU flamegraph:
// perf script -i perf_cpu.data > cpu.perf
// stackcollapse-perf.pl cpu.perf | flamegraph.pl > cpu.svg
//
// # Generate cache-miss flamegraph:
// perf script -i perf_cache.data > cache.perf
// stackcollapse-perf.pl cache.perf | flamegraph.pl \
//   --title="Cache Miss Flamegraph" --colors=mem > cache.svg
//
// # Differential flamegraph (before/after optimization):
// perf script -i before.data > before.perf
// perf script -i after.data > after.perf
// stackcollapse-perf.pl before.perf > before.folded
// stackcollapse-perf.pl after.perf > after.folded
// difffolded.pl before.folded after.folded | flamegraph.pl > diff.svg
// # Red = regression, blue = improvement
```

The cache-miss flamegraph is particularly useful when you have a function that appears small on the CPU flamegraph but is responsible for a disproportionate share of cache misses. That's the tell-tale sign of a memory-latency bottleneck rather than a compute bottleneck, and the two require different fixes.

### Q2: `perf annotate` to find cycle-consuming instructions

When a function shows up as a hotspot, `perf annotate` breaks it down to individual instructions and shows how many samples landed on each one. A concentration of 90%+ on a single `mov` or `add` instruction is almost always a cache miss - the instruction itself is fast, but the CPU is stalled waiting for the data to arrive from memory.

```cpp
#include <iostream>
#include <vector>
#include <random>

// This function has a hidden performance problem:
int lookup_table(const std::vector<int>& table,
                 const std::vector<int>& indices) {
    int sum = 0;
    for (size_t i = 0; i < indices.size(); ++i) {
        sum += table[indices[i]];  // random access -> cache misses
    }
    return sum;
}

int main() {
    std::vector<int> table(10'000'000, 42);
    std::vector<int> indices(5'000'000);
    std::mt19937 rng(42);
    for (auto& idx : indices) idx = rng() % table.size();

    volatile int s = 0;
    for (int i = 0; i < 10; ++i)
        s = lookup_table(table, indices);
    std::cout << s << '\n';
}

// perf annotate workflow:
//
// g++ -O2 -g -fno-omit-frame-pointer -o app test.cpp
// perf record -- ./app
// perf annotate --symbol=lookup_table --stdio
//
// Output:
//  Percent | Disassembly
//  --------+------------------------------------------
//    2.30% |   mov   eax, DWORD PTR [rsi+rcx*4]     ; load indices[i]
//   91.40% |   add   edx, DWORD PTR [rdi+rax*4]     ; load table[idx] <-- 91% HERE
//    1.20% |   add   rcx, 1                          ; i++
//    0.80% |   cmp   rcx, r8                         ; i < size
//    0.30% |   jl    .loop                           ; branch
//
// Diagnosis: 91.4% of cycles spent on ONE instruction
// This means the CPU is STALLED waiting for table[idx] from DRAM.
// Random access pattern -> no hardware prefetch -> constant L3 misses.
//
// Fix: software prefetch
//   for (size_t i = 0; i < indices.size(); ++i) {
//       if (i + 8 < indices.size())
//           __builtin_prefetch(&table[indices[i+8]], 0, 0);
//       sum += table[indices[i]];
//   }
// After fix: 91% drops to ~30% (prefetch hides 60% of latency)
```

The 91% concentration is diagnostic in itself. If the stall were due to a computation bottleneck (too many operations chained together), you'd see the samples spread across multiple instructions. A single instruction consuming almost all samples means the pipeline is stalled on memory bandwidth or latency.

### Q3: Identify cache miss hotspot with `perf mem`

`perf mem` goes further than cycle sampling by recording *where in the memory hierarchy* each load or store was served from. This lets you distinguish between L1 hits (fast), L3 hits (slower), and DRAM misses (much slower). It also reports whether any loads hit a "modified" line in another core's cache, which is the fingerprint of false sharing.

```cpp
#include <iostream>
#include <vector>
#include <random>

// Program with two access patterns: sequential vs random
void sequential_access(const std::vector<int>& data) {
    volatile long sum = 0;
    for (size_t i = 0; i < data.size(); ++i)
        sum += data[i];  // sequential: always L1 hit
}

void random_access(const std::vector<int>& data,
                   const std::vector<size_t>& idx) {
    volatile long sum = 0;
    for (size_t i = 0; i < idx.size(); ++i)
        sum += data[idx[i]];  // random: constant cache misses
}

int main() {
    std::vector<int> data(10'000'000, 1);
    std::vector<size_t> idx(5'000'000);
    std::mt19937 rng(42);
    for (auto& i : idx) i = rng() % data.size();

    for (int r = 0; r < 10; ++r) {
        sequential_access(data);
        random_access(data, idx);
    }
}

// perf mem workflow:
//
// g++ -O2 -g -fno-omit-frame-pointer -o app test.cpp
//
// # Record memory access samples:
// perf mem record -o mem.data -- ./app
//
// # Report by symbol:
// perf mem report -i mem.data --sort=symbol,mem,snoop
//
// Output:
//   Overhead  Symbol             Data Src    Snoop
//   95.20%    random_access      L3 miss     HitM    <-- 95% of misses!
//    3.10%    sequential_access  L1 hit      None
//    1.70%    random_access      L1 hit      None
//
// The report shows:
//  - random_access causes 95% of L3 misses (Data Src = L3 miss)
//  - sequential_access has almost all L1 hits
//  - "HitM" = Hit Modified (another core had the line) -> false sharing
//
// Correlate with source using perf annotate:
//   perf annotate --symbol=random_access -i mem.data
//   Points to: sum += data[idx[i]]; (the random load)
//
// Fix options:
//   1. Sort indices for cache-friendly access order
//   2. Use software prefetch (__builtin_prefetch)
//   3. Restructure algorithm to avoid random access
```

The "HitM" snoop result is particularly important in multi-threaded programs. If you see `HitM` on a line that shouldn't be shared between threads, you likely have false sharing - two threads that write to different variables which happen to share the same 64-byte cache line. That's best detected with `perf c2c`.

---

## Notes

- `perf mem` requires hardware support (Intel PEBS or AMD IBS).
- `perf c2c` detects false sharing (cache-to-cache transfers between cores).
- Differential flamegraphs (`difffolded.pl`) compare before/after optimization.
- `perf stat -d` provides a quick summary with L1/LLC miss rates.
- Alternative tools: Intel VTune (more detailed), AMD uProf, Apple Instruments.
