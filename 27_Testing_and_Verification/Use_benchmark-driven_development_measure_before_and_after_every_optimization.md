# Use benchmark-driven development: measure before and after every optimization

**Category:** Testing and Verification  
**Standard:** Not version-specific  
**Reference:** <https://github.com/google/benchmark>  

---

## Topic Overview

**Benchmark-driven development** applies the scientific method to optimization: measure first, change one thing, measure again to isolate the variable. Without a baseline, you are guessing. The intuition is simple - if you do not have a number from before the change, you have no idea whether your optimization actually helped, and you definitely cannot tell by how much.

### Workflow

The process is deliberately mechanical because that is what keeps you honest. Changing more than one thing at a time makes it impossible to know which change produced the improvement (or regression).

```cpp
1. Write Google Benchmark for the hot function
2. Run -> establish BASELINE (e.g., 450 ns/op)
3. Apply ONE optimization
4. Re-run -> measure DELTA (e.g., 320 ns/op -> 29% faster)
5. Commit if improved, revert if not
6. Repeat from step 3
```

### Google Benchmark Quick Reference

| Feature | Syntax |
| --- | --- |
| Basic benchmark | `BENCHMARK(BM_Name)` |
| Range input | `->Range(8, 1<<20)` |
| Arguments | `->Args({rows, cols})` |
| Time unit | `->Unit(benchmark::kMicrosecond)` |
| Iterations | `->Iterations(1000)` |
| Prevent dead code | `benchmark::DoNotOptimize(result)` |
| Memory fence | `benchmark::ClobberMemory()` |
| Dynamic register | `benchmark::RegisterBenchmark(...)` |

---

## Self-Assessment

### Q1: Write a Google Benchmark before optimizing a function to establish a performance baseline

Before you touch the implementation, you need a number you can point to. This benchmark measures `sum_of_squares_v1` at multiple input sizes. The `DoNotOptimize` call is essential - without it, the compiler is allowed to see that `result` is never used and eliminate the entire loop body, giving you a measurement of literally nothing.

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <algorithm>
#include <numeric>
#include <random>

// Original function - establish baseline BEFORE optimizing

// Naive: sum of squares using accumulate with lambda
double sum_of_squares_v1(const std::vector<double>& data) {
    return std::accumulate(data.begin(), data.end(), 0.0,
        [](double acc, double x) { return acc + x * x; });
}

// Benchmark: measure the BASELINE
static void BM_SumOfSquares_Baseline(benchmark::State& state) {
    // Setup: create random data (NOT timed)
    const int n = state.range(0);
    std::mt19937 rng(42);
    std::uniform_real_distribution<double> dist(-100.0, 100.0);
    std::vector<double> data(n);
    std::generate(data.begin(), data.end(), [&]{ return dist(rng); });

    // Timed loop
    for (auto _ : state) {
        double result = sum_of_squares_v1(data);
        benchmark::DoNotOptimize(result);  // Prevent dead code elimination
    }

    // Report throughput
    state.SetBytesProcessed(
        static_cast<int64_t>(state.iterations()) * n * sizeof(double));
    state.SetItemsProcessed(
        static_cast<int64_t>(state.iterations()) * n);
}

BENCHMARK(BM_SumOfSquares_Baseline)
    ->Range(64, 1 << 20)    // 64 to 1M elements
    ->Unit(benchmark::kMicrosecond);

BENCHMARK_MAIN();

// Run:  ./benchmark --benchmark_format=console
// Output:
// BM_SumOfSquares_Baseline/64        0.012 us    <- BASELINE
// BM_SumOfSquares_Baseline/1024      0.190 us
// BM_SumOfSquares_Baseline/1048576   195   us
```

Notice that setup code goes outside the timed loop. The random data generation happens once, and the benchmark framework measures only the iterations inside `for (auto _ : state)`. Any initialization or teardown that is not part of what you are measuring should stay outside.

### Q2: Apply one optimization, re-run the benchmark, and record the delta

Once you have the baseline, make exactly one change. Here the change is manual loop unrolling with four accumulators. Placing both versions in the same benchmark file and running them together makes the comparison trivial.

```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <algorithm>
#include <numeric>
#include <random>

// V1: Original (baseline)
double sum_of_squares_v1(const std::vector<double>& data) {
    return std::accumulate(data.begin(), data.end(), 0.0,
        [](double acc, double x) { return acc + x * x; });
}

// V2: Manual loop with 4-way unrolling
double sum_of_squares_v2(const std::vector<double>& data) {
    const size_t n = data.size();
    const double* p = data.data();
    double s0 = 0, s1 = 0, s2 = 0, s3 = 0;

    size_t i = 0;
    for (; i + 4 <= n; i += 4) {
        s0 += p[i]   * p[i];
        s1 += p[i+1] * p[i+1];
        s2 += p[i+2] * p[i+2];
        s3 += p[i+3] * p[i+3];
    }
    double sum = s0 + s1 + s2 + s3;
    for (; i < n; ++i)
        sum += p[i] * p[i];
    return sum;
}

// Benchmarks: side by side
static void BM_V1_Accumulate(benchmark::State& state) {
    const int n = state.range(0);
    std::vector<double> data(n, 1.5);  // Deterministic
    for (auto _ : state) {
        benchmark::DoNotOptimize(sum_of_squares_v1(data));
    }
    state.SetBytesProcessed(int64_t(state.iterations()) * n * 8);
}

static void BM_V2_Unrolled(benchmark::State& state) {
    const int n = state.range(0);
    std::vector<double> data(n, 1.5);
    for (auto _ : state) {
        benchmark::DoNotOptimize(sum_of_squares_v2(data));
    }
    state.SetBytesProcessed(int64_t(state.iterations()) * n * 8);
}

BENCHMARK(BM_V1_Accumulate)->Range(1024, 1 << 20)->Unit(benchmark::kMicrosecond);
BENCHMARK(BM_V2_Unrolled)->Range(1024, 1 << 20)->Unit(benchmark::kMicrosecond);

BENCHMARK_MAIN();

// Compare output:
// BM_V1_Accumulate/1048576   195 us       <- BEFORE
// BM_V2_Unrolled/1048576     148 us       <- AFTER (24% faster)
//
// Use --benchmark_out=results.json --benchmark_out_format=json
// to store results for CI comparison
```

The JSON output flag is not cosmetic - it is how you build a paper trail. Save `results.json` as a CI artifact alongside your baseline, and you have a permanent record of what each optimization actually produced.

### Q3: Use benchmark::RegisterBenchmark dynamically to compare multiple implementations side by side

When you have four competing implementations and three input sizes, that is twelve benchmark runs. Writing twelve static functions by hand is tedious and repetitive. `RegisterBenchmark` lets you loop over the implementations and sizes programmatically instead.

<!-- compile: needs third-party library header `benchmark/benchmark.h` -->
```cpp
#include <benchmark/benchmark.h>
#include <vector>
#include <string>
#include <algorithm>
#include <cstring>

// Multiple implementations to compare

void copy_loop(const std::vector<int>& src, std::vector<int>& dst) {
    for (size_t i = 0; i < src.size(); ++i) dst[i] = src[i];
}

void copy_stl(const std::vector<int>& src, std::vector<int>& dst) {
    std::copy(src.begin(), src.end(), dst.begin());
}

void copy_memcpy(const std::vector<int>& src, std::vector<int>& dst) {
    std::memcpy(dst.data(), src.data(), src.size() * sizeof(int));
}

void copy_assign(const std::vector<int>& src, std::vector<int>& dst) {
    dst = src;
}

// Dynamic registration
using CopyFunc = void(*)(const std::vector<int>&, std::vector<int>&);

struct CopyImpl {
    const char* name;
    CopyFunc func;
};

CopyImpl implementations[] = {
    {"Manual_Loop",  copy_loop},
    {"std::copy",    copy_stl},
    {"memcpy",       copy_memcpy},
    {"Assignment",   copy_assign},
};

int main(int argc, char** argv) {
    for (auto& impl : implementations) {
        for (int size : {1024, 65536, 1 << 20}) {
            std::string name = std::string(impl.name) + "/" + std::to_string(size);

            benchmark::RegisterBenchmark(name, [&impl, size](benchmark::State& state) {
                std::vector<int> src(size, 42);
                std::vector<int> dst(size);

                for (auto _ : state) {
                    impl.func(src, dst);
                    benchmark::ClobberMemory();
                }
                state.SetBytesProcessed(
                    int64_t(state.iterations()) * size * sizeof(int));
            })->Unit(benchmark::kMicrosecond);
        }
    }

    benchmark::Initialize(&argc, argv);
    benchmark::RunSpecifiedBenchmarks();
    benchmark::Shutdown();

    return 0;
}

// Output (all side by side):
// Manual_Loop/1024        0.15 us
// std::copy/1024          0.12 us
// memcpy/1024             0.10 us
// Assignment/1024         0.13 us
// Manual_Loop/1048576     980  us
// std::copy/1048576       520  us  <- compiler vectorizes
// memcpy/1048576          450  us  <- fastest
// Assignment/1048576      540  us
```

---

## Notes

- Install: `vcpkg install benchmark` or FetchContent in CMake
- Always use `benchmark::DoNotOptimize` on the result to prevent the compiler from removing the work
- Use `--benchmark_out=file.json` to save results for CI regression tracking
- `benchmark_compare.py` (shipped with Google Benchmark) diffs two JSON files with statistical significance
- Pin CPU frequency (`cpupower frequency-set -g performance`) for stable results on Linux
