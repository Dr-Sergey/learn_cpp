# Use Quick Bench or Google Benchmark for micro-benchmarking

**Category:** Tooling and Debugging  
**Standard:** Not version-specific  
**Reference:** <https://github.com/google/benchmark>  

---

## Topic Overview

If you have ever wondered "is `emplace_back` actually faster than `push_back` for my type?", Google Benchmark is the right tool for getting a real answer. It provides a framework for writing accurate micro-benchmarks - small, focused measurements of individual operations. Quick Bench at quick-bench.com is a browser-based front-end that runs Google Benchmark online, which is great when you just want a fast sanity check without a local setup.

The key thing micro-benchmarking gets right that a simple `std::chrono` loop gets wrong is statistical stability. Google Benchmark automatically runs each test for enough iterations to get a meaningful average, reports both wall-clock time and CPU time, and gives you the raw iteration count so you can judge confidence.

Here is what a typical benchmark output looks like:

```text
Benchmark output:
-----------------------------------------------------------
Benchmark               Time             CPU   Iterations
-----------------------------------------------------------
BM_push_back          12.3 ns         12.2 ns    56000000
BM_emplace_back       10.1 ns         10.0 ns    68000000
                      ^^^^^^          ^^^^^^
                    wall clock       CPU time
```

The framework ran each variant millions of times to arrive at those stable numbers. Notice that wall-clock time and CPU time are reported separately - if they diverge, it often means the OS was competing for resources during the run.

---

## Self-Assessment

### Q1: Benchmark `push_back` vs `emplace_back`

The key thing to understand about this benchmark is the structure: each function receives a `benchmark::State&` loop that the framework drives. Everything inside `for (auto _ : state)` is timed, and anything outside that loop is setup that runs only once. The `benchmark::DoNotOptimize` call at the end is essential - without it, the compiler may decide the vector is never read and silently delete the whole loop.

```cpp
// bench.cpp — compile:
// g++ -O2 -std=c++20 bench.cpp -lbenchmark -lpthread -o bench
#include <benchmark/benchmark.h>
#include <vector>
#include <string>

static void BM_push_back(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<std::string> v;
        v.reserve(1000);
        for (int i = 0; i < 1000; ++i)
            v.push_back(std::string("hello world"));  // creates temp, then moves
        benchmark::DoNotOptimize(v.data());
    }
}
BENCHMARK(BM_push_back);

static void BM_emplace_back(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<std::string> v;
        v.reserve(1000);
        for (int i = 0; i < 1000; ++i)
            v.emplace_back("hello world");  // constructs in-place, no temp
        benchmark::DoNotOptimize(v.data());
    }
}
BENCHMARK(BM_emplace_back);

// Parameterized benchmark:
static void BM_push_back_n(benchmark::State& state) {
    const int n = state.range(0);
    for (auto _ : state) {
        std::vector<std::string> v;
        v.reserve(n);
        for (int i = 0; i < n; ++i)
            v.push_back(std::string("hello"));
        benchmark::DoNotOptimize(v.data());
    }
    state.SetItemsProcessed(state.iterations() * n);
}
BENCHMARK(BM_push_back_n)->Range(8, 8 << 10);

BENCHMARK_MAIN();
```

Build and run it, and the output will look something like this:

```bash
# Build and run:
$ g++ -O2 -std=c++20 bench.cpp -lbenchmark -lpthread -o bench
$ ./bench
# ----------------------------------------------------------
# Benchmark              Time           CPU      Iterations
# ----------------------------------------------------------
# BM_push_back        15234 ns       15200 ns         46000
# BM_emplace_back     12456 ns       12400 ns         56000
#                     ~~~~~~ emplace_back ~18% faster
# BM_push_back_n/8       45 ns          45 ns      15000000
# BM_push_back_n/64     320 ns         319 ns       2100000
# BM_push_back_n/512   2890 ns        2880 ns        240000
# BM_push_back_n/4096 24500 ns       24400 ns         28000
```

The parameterized variant is useful for spotting when a performance difference is size-dependent. Here you can see the cost per element stays roughly linear, which is what you would expect.

### Q2: `DoNotOptimize` prevents dead code elimination

This is the part that trips people up the most when they first write benchmarks. An optimizing compiler is allowed to remove any computation whose result is never observed - and a `for` loop that adds numbers to a local variable that nothing reads afterward is a perfect candidate for deletion. If you omit `DoNotOptimize`, your benchmark may report 0 ns because there is literally nothing left to measure.

```cpp
#include <benchmark/benchmark.h>

// WITHOUT DoNotOptimize — compiler eliminates the work:
static void BM_bad(benchmark::State& state) {
    for (auto _ : state) {
        int sum = 0;
        for (int i = 0; i < 1000; ++i)
            sum += i;          // Compiler sees sum is unused -> eliminates loop!
        // No DoNotOptimize -> measures NOTHING
    }
}
BENCHMARK(BM_bad);  // reports ~0 ns (useless!)

// WITH DoNotOptimize — compiler must keep the computation:
static void BM_good(benchmark::State& state) {
    for (auto _ : state) {
        int sum = 0;
        for (int i = 0; i < 1000; ++i)
            sum += i;
        benchmark::DoNotOptimize(sum);  // treats 'sum' as observable
    }
}
BENCHMARK(BM_good);  // reports actual loop time

// ClobberMemory forces memory writes to be visible:
static void BM_container(benchmark::State& state) {
    std::vector<int> v(1000);
    for (auto _ : state) {
        for (auto& x : v) x += 1;
        benchmark::ClobberMemory();  // writes to v are "observed"
    }
}
BENCHMARK(BM_container);

BENCHMARK_MAIN();
```

The reason `DoNotOptimize` works is that it uses an inline assembly constraint to tell the compiler "this value has been seen by something outside your control." Here is the simplified implementation:

```cpp
// Simplified implementation:
template <class T>
void DoNotOptimize(T& value) {
    asm volatile("" : "+r"(value));  // value in register, compiler can't eliminate
}
// The asm volatile is a barrier: compiler must compute 'value' but the asm does nothing.
```

The assembly instruction itself does nothing at runtime. The `volatile` keyword is what prevents the compiler from reasoning past it. Use `DoNotOptimize` for scalar results and `ClobberMemory` when you are writing to a container and want to ensure those writes are not skipped.

### Q3: Filter and interpret benchmark output

Once you have multiple benchmarks, you do not always want to run all of them. Google Benchmark provides filtering and several output formats that make it easy to compare runs over time.

```bash
# Run only benchmarks matching a regex:
$ ./bench --benchmark_filter="emplace"
# Runs only BM_emplace_back

$ ./bench --benchmark_filter="BM_push_back_n/.*"
# Runs all parameterized variants

# Output columns explained:
# Benchmark       Time        CPU       Iterations
# BM_X           15.2 ns     15.1 ns    46000000
# |              |           |          |
# name           wall-clock  CPU-only   auto-determined
#                per iter    per iter   for statistical
#                                       stability

# Output formats:
$ ./bench --benchmark_format=json > results.json
$ ./bench --benchmark_format=csv  > results.csv

# Compare two runs:
$ ./bench --benchmark_out=baseline.json --benchmark_out_format=json
# ... make changes
$ ./bench --benchmark_out=improved.json --benchmark_out_format=json
$ python3 -m google_benchmark.compare baseline.json improved.json
# Benchmark              Time       CPU
# BM_push_back        -0.18%    -0.19%  (18% faster!)

# Statistical analysis (multiple runs):
$ ./bench --benchmark_repetitions=10 --benchmark_report_aggregates_only=true
# Shows mean, median, stddev
```

The comparison script is particularly valuable in CI workflows - you can catch performance regressions the same way you catch test failures.

---

## Notes

- Quick Bench at quick-bench.com runs Google Benchmark online - great for quick tests.
- Always benchmark with `-O2` or `-O3`; `-O0` results are meaningless for real-world comparisons.
- Use `benchmark::ClobberMemory()` when your benchmark mutates a container and you need those writes to be treated as observable.
- `SetItemsProcessed()` and `SetBytesProcessed()` add throughput metrics to the output.
- Install via: `vcpkg install benchmark` or `apt install libbenchmark-dev`.
