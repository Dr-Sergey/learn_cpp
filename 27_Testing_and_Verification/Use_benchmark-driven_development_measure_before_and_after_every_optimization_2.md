# Avoid misleading micro-benchmarks when measuring optimizations

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/google/benchmark>  

---

## Topic Overview

This file focuses on **micro-benchmark pitfalls and validation with full-system profiling**. (See file #686 for Google Benchmark basics, RegisterBenchmark, and the core measure -> optimize -> measure workflow.)

### Why Micro-Benchmarks Mislead

A micro-benchmark runs the same function thousands of times in a tight loop. That sounds like a good approximation of production usage, but it actually creates an artificial environment that can make a function look dramatically faster than it ever will be in a real program. The reason is cache: after the first few iterations, your data fits entirely in L1 or L2 cache and stays there. In production, other code runs between calls to your function and evicts your data. You end up measuring the cache-warm path when the real bottleneck is the cache-cold path.

```cpp
Micro-benchmark world:                  Real-world:
┌──────────────────────┐                ┌──────────────────────┐
│ Data fits in L1 cache│                │ Data evicted by other│
│ No branch mispredicts│                │  code running between│
│ Perfect instruction  │                │  function calls      │
│  prefetching         │                │ TLB misses from large│
│ No OS interrupts     │                │  working sets        │
└──────────────────────┘                └──────────────────────┘
     "50ns per call"                         "800ns per call"
```

| Pitfall | What Happens | Mitigation |
| --- | --- | --- |
| Hot cache | Data stays in L1/L2 | Use `benchmark::ClobberMemory()`, shuffle data |
| Dead code elimination | Compiler removes computation | `DoNotOptimize()` |
| Too-small input | O(n²) looks O(n) | Test multiple input sizes |
| CPU frequency scaling | Results vary 2-3× | Pin governor to `performance` |
| Alignment lottery | Random speedup/slowdown | Run multiple iterations |
| Branch prediction warm-up | Sorted data is 5× faster | Test random data too |

---

## Self-Assessment

### Q1: Write a Google Benchmark before attempting an optimization to establish a baseline

The baseline benchmark here measures string building with `+=`. Notice the growth pattern in the expected output: going from 1000 to 10000 items is 10x the input, but the time grows by roughly 18x. That super-linear growth is the first signal that something interesting is happening - in this case, repeated reallocation as the string outgrows its buffer.

```cpp
#include <benchmark/benchmark.h>
#include <string>
#include <sstream>
#include <vector>

// Function to benchmark: string building

// V1: string concatenation with +=
std::string build_string_concat(int n) {
    std::string result;
    for (int i = 0; i < n; ++i) {
        result += "item_" + std::to_string(i) + ",";
    }
    return result;
}

// Baseline benchmark
static void BM_StringConcat(benchmark::State& state) {
    const int n = state.range(0);
    for (auto _ : state) {
        auto result = build_string_concat(n);
        benchmark::DoNotOptimize(result.data());
        benchmark::ClobberMemory();  // Force memory to be visible
    }
    state.SetItemsProcessed(int64_t(state.iterations()) * n);
}
BENCHMARK(BM_StringConcat)
    ->RangeMultiplier(10)
    ->Range(10, 100000)
    ->Unit(benchmark::kMicrosecond);

BENCHMARK_MAIN();

// Baseline results:
// BM_StringConcat/10       0.3  us
// BM_StringConcat/100      4.2  us
// BM_StringConcat/1000     65   us    <- note: ~15x for 10x input (reallocation cost)
// BM_StringConcat/10000    1200 us
// BM_StringConcat/100000   28000 us   <- quadratic-ish growth
```

### Q2: Apply the optimization, re-run the benchmark, and record the speedup

Three versions are compared here, each addressing a different bottleneck. V2 moves away from `+=` with temporaries by using `ostringstream`. V3 goes further: `reserve` eliminates reallocations entirely, and `std::to_chars` avoids creating `std::string` temporary objects for each integer. The speedup compounds across both improvements.

```cpp
#include <benchmark/benchmark.h>
#include <string>
#include <sstream>
#include <vector>
#include <charconv>

// V1: Original (+=)
std::string build_v1(int n) {
    std::string result;
    for (int i = 0; i < n; ++i)
        result += "item_" + std::to_string(i) + ",";
    return result;
}

// V2: Optimized (reserve + ostringstream)
std::string build_v2(int n) {
    std::ostringstream oss;
    for (int i = 0; i < n; ++i)
        oss << "item_" << i << ',';
    return oss.str();
}

// V3: Further optimized (reserve + to_chars)
std::string build_v3(int n) {
    // Estimate: "item_" (5) + digits (up to 6) + "," (1) = ~12 chars per item
    std::string result;
    result.reserve(n * 12);
    char buf[16];
    for (int i = 0; i < n; ++i) {
        result.append("item_", 5);
        auto [ptr, ec] = std::to_chars(buf, buf + 16, i);
        result.append(buf, ptr - buf);
        result.push_back(',');
    }
    return result;
}

// Benchmarks
static void BM_V1_Concat(benchmark::State& s) {
    int n = s.range(0);
    for (auto _ : s) benchmark::DoNotOptimize(build_v1(n));
    s.SetItemsProcessed(int64_t(s.iterations()) * n);
}
static void BM_V2_Stream(benchmark::State& s) {
    int n = s.range(0);
    for (auto _ : s) benchmark::DoNotOptimize(build_v2(n));
    s.SetItemsProcessed(int64_t(s.iterations()) * n);
}
static void BM_V3_Reserve(benchmark::State& s) {
    int n = s.range(0);
    for (auto _ : s) benchmark::DoNotOptimize(build_v3(n));
    s.SetItemsProcessed(int64_t(s.iterations()) * n);
}

BENCHMARK(BM_V1_Concat)->Range(100, 100000)->Unit(benchmark::kMicrosecond);
BENCHMARK(BM_V2_Stream)->Range(100, 100000)->Unit(benchmark::kMicrosecond);
BENCHMARK(BM_V3_Reserve)->Range(100, 100000)->Unit(benchmark::kMicrosecond);

BENCHMARK_MAIN();

// Results (typical):
// BM_V1_Concat/100000     28000 us  (baseline)
// BM_V2_Stream/100000     12000 us  (2.3× faster)
// BM_V3_Reserve/100000     3500 us  (8× faster)
//
// Delta: V3 is 8× faster than V1
// Key insight: reserve() eliminates reallocation, to_chars avoids std::string temporaries
```

### Q3: Explain why micro-benchmarks can mislead and how to validate with full-system profiling

This is probably the most important conceptual point in benchmark-driven development. The two benchmark functions below measure the same binary search on the same data, but one warms the cache first and the other evicts it. The 32x difference in results is purely a measurement artifact - neither number is wrong, but only one is representative of what happens in a real application.

```cpp
// Micro-benchmark that lies

#include <benchmark/benchmark.h>
#include <vector>
#include <algorithm>
#include <random>

// Searching sorted data - micro-benchmark says "fast!"
static void BM_BinarySearch_Hot(benchmark::State& state) {
    std::vector<int> data(1000000);
    std::iota(data.begin(), data.end(), 0);
    // Data is in L1/L2 cache (warm)

    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, 999999);

    for (auto _ : state) {
        int target = dist(rng);
        bool found = std::binary_search(data.begin(), data.end(), target);
        benchmark::DoNotOptimize(found);
    }
}
BENCHMARK(BM_BinarySearch_Hot);
// Result: ~25 ns/call - "binary search is blazing fast!"

// But in a real system with cold caches:
static void BM_BinarySearch_Cold(benchmark::State& state) {
    // Allocate a HUGE buffer to flush L1/L2/L3 cache
    std::vector<int> data(1000000);
    std::iota(data.begin(), data.end(), 0);
    std::vector<char> cache_buster(32 * 1024 * 1024, 0);  // 32MB

    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(0, 999999);

    for (auto _ : state) {
        // Evict data from cache
        for (size_t i = 0; i < cache_buster.size(); i += 64)
            benchmark::DoNotOptimize(cache_buster[i]);

        int target = dist(rng);
        bool found = std::binary_search(data.begin(), data.end(), target);
        benchmark::DoNotOptimize(found);
    }
}
BENCHMARK(BM_BinarySearch_Cold);
// Result: ~800 ns/call - 32× slower due to cache misses!

BENCHMARK_MAIN();
```

The lesson is that micro-benchmarks are useful for comparing two implementations under the same conditions, but you need system-level profiling to know whether those conditions resemble production. If your micro-benchmark shows a 30% speedup but `perf` shows the function is only 0.01% of total runtime, optimizing that function is wasted effort regardless of how clean the benchmark numbers look.

**Validation with full-system profiling:**

```bash
# Step 1: Run micro-benchmark to identify candidates
./my_benchmark --benchmark_out=micro.json

# Step 2: Profile the REAL application with perf
perf record -g ./my_application --real-workload
perf report  # See where time is ACTUALLY spent

# Step 3: Compare
# If micro-benchmark says function X is 10ns but perf shows
# X takes 5% of total runtime -> optimization is worthwhile
# If micro-benchmark says function Y is 800ns but perf shows
# Y is 0.01% of total runtime -> DON'T optimize this

# Step 4: Validate optimization in production
perf stat -d ./my_application_before  # L1/LLC miss rates
perf stat -d ./my_application_after   # Compare miss rates
```

---

## Notes

- `benchmark::ClobberMemory()` - compiler barrier (force stores to memory), prevents reordering
- `benchmark::DoNotOptimize(x)` - prevents dead code elimination of `x`
- Always test with **representative data sizes** - tiny inputs hide algorithmic complexity
- `--benchmark_repetitions=10 --benchmark_report_aggregates_only=true` for statistical stability
- Use `perf stat -d` to show L1/LLC cache miss rates alongside benchmark times
